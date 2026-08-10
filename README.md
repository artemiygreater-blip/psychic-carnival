#!/usr/bin/env bash
set -e

echo "Создаю структуру файлов для speedswap-scaffold..."

# Создаём каталоги
mkdir -p src/main/java/com/artemiygreater/speedswap
mkdir -p src/main/resources/META-INF
mkdir -p .github/workflows

# build.gradle
cat > build.gradle <<'EOF'
plugins {
    id 'net.minecraftforge.gradle.forge' version '5.1.+'
}

group = 'com.artemiygreater'
version = '1.0'
archivesBaseName = 'speedswap'

sourceCompatibility = JavaVersion.VERSION_17
targetCompatibility = JavaVersion.VERSION_17

repositories {
    mavenCentral()
    maven { url 'https://maven.minecraftforge.net' }
}

dependencies {
    // Forge dependencies are resolved by the ForgeGradle plugin
}

minecraft {
    // Forge/MC version. If you need a different patch, change here.
    version = "1.21.11-61.2.0"
    mappings channel: 'official', version: '1.21.11'
    runs {
        client {
            workingDirectory project.file('run')
        }
        server {
            workingDirectory project.file('run_server')
        }
    }
}
EOF

# gradle.properties
cat > gradle.properties <<'EOF'
org.gradle.jvmargs=-Xmx4G
EOF

# settings.gradle
cat > settings.gradle <<'EOF'
rootProject.name = 'psychic-carnival'
EOF

# README.md
cat > README.md <<'EOF'
# Speedswap mod

This branch adds a scaffold of the SpeedSwap Forge mod and a GitHub Actions workflow that builds the mod on CI and uploads the jar artifact.

How this works
- I created a branch `speedswap-scaffold` with the mod source and a workflow `.github/workflows/build.yml`.
- The workflow runs Gradle on GitHub Actions to download Forge/Minecraft/ForgeGradle artifacts and build the jar.

To get the jar
1. Push or open a PR from this branch on GitHub.
2. On the GitHub Actions page for the run, download the build artifact named `speedswap-jar`.

Notes
- I aimed for a conservative, compile-friendly Java implementation; if the build fails on CI due to mappings or minor API differences, open the Actions log and paste the first compile errors here and I'll fix them.
- Typical fixes will be around compass API or tick/event signatures (Forge mappings change frequently). The workflow will show exact compile errors so we can iterate.
EOF

# mods.toml
cat > src/main/resources/META-INF/mods.toml <<'EOF'
modLoader="javafml"
loaderVersion="61.2.0"
license="MIT"
[[mods]]
modId="speedswap"
version="1.0.0"
displayName="SpeedSwap"
description='''
A Forge mod that swaps the "speedrunner" among players on a timer while hunters track them.
'''

[[dependencies.speedswap]]
modId="forge"
mandatory=true
versionRange="[61.2.0,61.2.0]"
ordering="NONE"
side="BOTH"
EOF

# GitHub Actions workflow
cat > .github/workflows/build.yml <<'EOF'
name: Build and publish Speedswap JAR

on:
  push:
    branches:
      - 'speedswap-scaffold'
  workflow_dispatch: {}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - name: Run Gradle build
        uses: gradle/gradle-build-action@v2
        with:
          arguments: build
      - name: Upload built jar
        uses: actions/upload-artifact@v4
        with:
          name: speedswap-jar
          path: build/libs/*.jar
EOF

# Java source files
cat > src/main/java/com/artemiygreater/speedswap/SpeedSwapMod.java <<'EOF'
package com.artemiygreater.speedswap;

import com.mojang.logging.LogUtils;
import net.minecraftforge.event.RegisterCommandsEvent;
import net.minecraftforge.eventbus.api.SubscribeEvent;
import net.minecraftforge.fml.common.Mod;
import net.minecraftforge.fml.event.lifecycle.FMLCommonSetupEvent;
import net.minecraftforge.fml.javafmlmod.FMLJavaModLoadingContext;
import net.minecraftforge.common.MinecraftForge;
import org.slf4j.Logger;

@Mod(SpeedSwapMod.MODID)
public class SpeedSwapMod {
    public static final String MODID = "speedswap";
    public static final Logger LOGGER = LogUtils.getLogger();

    public SpeedSwapMod() {
        // Register ourselves for server & mod lifecycle events
        FMLJavaModLoadingContext.get().getModEventBus().addListener(this::setup);
        MinecraftForge.EVENT_BUS.register(this);
        // Initialize game manager
        GameManager.init();
    }

    private void setup(final FMLCommonSetupEvent event) {
        LOGGER.info("SpeedSwap mod setup completed");
    }

    @SubscribeEvent
    public void onRegisterCommands(RegisterCommandsEvent event) {
        SpeedSwapCommand.register(event.getDispatcher());
    }
}
EOF

cat > src/main/java/com/artemiygreater/speedswap/SpeedSwapCommand.java <<'EOF'
package com.artemiygreater.speedswap;

import com.mojang.brigadier.CommandDispatcher;
import com.mojang.brigadier.arguments.StringArgumentType;
import net.minecraft.commands.CommandSourceStack;
import net.minecraft.commands.Commands;
import net.minecraft.network.chat.Component;

import java.util.UUID;

public class SpeedSwapCommand {
    public static void register(CommandDispatcher<CommandSourceStack> dispatcher) {
        dispatcher.register(Commands.literal("speedswap")
                .then(Commands.literal("join")
                        .then(Commands.argument("role", StringArgumentType.word())
                                .executes(ctx -> {
                                    String role = StringArgumentType.getString(ctx, "role");
                                    UUID player = ctx.getSource().getPlayerOrException().getUUID();
                                    if ("speedrunner".equalsIgnoreCase(role)) {
                                        GameManager.get().joinAsSpeedrunner(player);
                                        ctx.getSource().sendSuccess(Component.literal("Joined as speedrunner"), false);
                                    } else {
                                        GameManager.get().joinAsHunter(player);
                                        ctx.getSource().sendSuccess(Component.literal("Joined as hunter"), false);
                                    }
                                    return 1;
                                })))
                .then(Commands.literal("status")
                        .executes(ctx -> {
                            ctx.getSource().sendSuccess(Component.literal(GameManager.get().status()), false);
                            return 1;
                        }))
        );
    }
}
EOF

cat > src/main/java/com/artemiygreater/speedswap/GameManager.java <<'EOF'
package com.artemiygreater.speedswap;

import java.util.*;
import java.util.concurrent.atomic.AtomicReference;

public class GameManager {
    private static final AtomicReference<GameManager> INSTANCE = new AtomicReference<>();

    public enum State { WAITING, RUNNING }

    private State state = State.WAITING;
    private final Set<UUID> hunters = new HashSet<>();
    private UUID speedrunner;
    private int intervalSeconds = 300; // default 5 minutes

    private GameManager() {
    }

    public static void init() {
        INSTANCE.compareAndSet(null, new GameManager());
    }

    public static GameManager get() {
        return INSTANCE.get();
    }

    public synchronized void joinAsSpeedrunner(UUID player) {
        this.speedrunner = player;
        this.state = State.RUNNING;
    }

    public synchronized void joinAsHunter(UUID player) {
        hunters.add(player);
    }

    public synchronized String status() {
        return "state=" + state + " speedrunner=" + (speedrunner == null ? "none" : speedrunner.toString()) + " hunters=" + hunters.size();
    }

    // TODO: swap logic, timers, mirroring, compass updates
}
EOF

cat > src/main/java/com/artemiygreater/speedswap/PlayerSyncHandler.java <<'EOF'
package com.artemiygreater.speedswap;

public class PlayerSyncHandler {
    // This class will contain logic to mirror player state every tick from speedrunner to other players.
    // Left intentionally minimal here; implementation will be added or adjusted based on compile-time Forge mappings.
}
EOF

cat > src/main/java/com/artemiygreater/speedswap/CompassHandler.java <<'EOF'
package com.artemiygreater.speedswap;

public class CompassHandler {
    // Logic to retarget hunters' compasses to the active speedrunner.
}
EOF

echo "Файлы созданы. Проверьте git status. Теперь создайте ветку, закоммитьте и запушьте:"
echo "  git checkout -b speedswap-scaffold"
echo "  git add ."
echo "  git commit -m \"Add SpeedSwap scaffold: build files, mod source, and CI build workflow\""
echo "  git push origin speedswap-scaffold"
echo ""
echo "После пуша откройте PR на GitHub и/или проверьте Actions — workflow соберёт jar и загрузит артефакт 'speedswap-jar'."
