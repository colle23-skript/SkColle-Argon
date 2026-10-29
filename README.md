# SkColle-Argon
<div align="center">

# ArgonAC & SkColle

**Packet-Level Minecraft Anticheat & Netty Physics Engine for Paper 1.20.4 - 1.21.11+**

[![Release](https://img.shields.io/badge/Release-BETA--1.0.0-6f42c1?style=flat-square)](https://github.com/colle23-skript)
[![Platform](https://img.shields.io/badge/Platform-Paper%201.20.4--1.21.11+-2b6cb0?style=flat-square)](https://papermc.io/)
[![Java](https://img.shields.io/badge/Java-21-b07219?style=flat-square)](https://adoptium.net/)
[![Skript](https://img.shields.io/badge/Skript-2.9.3+-2ea44f?style=flat-square)](https://github.com/SkriptLang/Skript)
[![Discord](https://img.shields.io/badge/Discord-Support%20Server-5865F2?style=flat-square&logo=discord&logoColor=white)](https://discord.gg/3tr9aYWZDR)

</div>

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. What is SkColle.jar and How It Leverages Java Packets](#2-what-is-skcollejar-and-how-it-leverages-java-packets)
  - [2.1 Netty Pipeline Interception](#21-netty-pipeline-interception)
  - [2.2 Core Physics and Mathematical Engines](#22-core-physics-and-mathematical-engines)
- [3. SkColle Skript API Reference](#3-skcolle-skript-api-reference)
  - [3.1 Events, Conditions, and Effects](#31-events-conditions-and-effects)
  - [3.2 Player State & Physics Expressions](#32-player-state--physics-expressions)
  - [3.3 Packet Field Dictionary](#33-packet-field-dictionary)
- [4. ArgonAC Check Directory (22 Modules)](#4-argonac-check-directory-22-modules)
  - [4.1 Combat Checks](#41-combat-checks)
  - [4.2 Movement Checks](#42-movement-checks)
  - [4.3 World & Protocol Checks](#43-world--protocol-checks)
- [5. Commands, Permissions & GUI Administration](#5-commands-permissions--gui-administration)
  - [5.1 Command Reference](#51-command-reference)
  - [5.2 GUI Control Panel & Mitigation Modes](#52-gui-control-panel--mitigation-modes)
- [6. Compilation & Installation](#6-compilation--installation)
- [7. Bug Reports & Support](#7-bug-reports--support)

---

## 1. Overview

**ArgonAC** is a packet-driven anticheat ecosystem built on a dual-layer architecture:

1. **`SkColle.jar` (Java 21 Addon):** A low-level Netty packet interceptor and 3D mathematical physics engine. It hooks directly into each player's network channel, analyzes raw inbound and outbound Minecraft packets before standard Bukkit processing, compensates for latency using historical bounding-box rewind buffers, and exposes custom syntax to Skript.
2. **`Argon.sk` (Heuristic & Mitigation Engine):** A modular detection script that consumes pre-computed telemetry from `SkColle`, manages Violation Levels (VL), executes packet-isolated setbacks, and provides an in-game control panel for server staff.

---

## 2. What is SkColle.jar and How It Leverages Java Packets

Standard Bukkit events (`PlayerMoveEvent`, `EntityDamageByEntityEvent`) are triggered late in the server tick loop after entity positions and collisions have already been processed. This introduces tick-alignment jitter, latency desynchronization, and bypass vectors for packet-based cheats.

`SkColle.jar` solves this by operating directly at the network transport layer.

### 2.1 Netty Pipeline Interception

When a player connects, `SkColle` uses cached reflection (`ReflectCacheEngine` with `MethodHandle` and `VarHandle` lookup tables) to locate the player's underlying Netty `io.netty.channel.Channel` (`ServerPlayer -> ServerGamePacketListenerImpl -> Connection -> Channel`). It injects a custom `ChannelDuplexHandler` named `skcolle_interceptor` directly before the vanilla `packet_handler`.

```text
[Client Socket]
       |
       v
[Netty ChannelPipeline]
       |
       +---> [skcolle_interceptor (Inbound)]
       |        |-- 1. Enforces 5000ms Join Grace Period (Passive Sync)
       |        |-- 2. Drops stale in-flight packets during active Setback Lock
       |        |-- 3. Extracts raw fields (x, y, z, yaw, pitch, onGround, entityId)
       |        |-- 4. Tracks 20-Tick Client CPS, FakeLag stalls & NoSwing protocol
       |        +-- 5. Synchronizes 3D Physics & dispatches PacketSkriptEvent
       |
       v
[Minecraft Server packet_handler]
       |
       v
[skcolle_interceptor (Outbound)]
       |-- Intercepts SetEntityMotion (Knockback vectors vx, vy, vz)
       |-- Intercepts KeepAlive / Ping packets for true RTT measurement
       v
[Client Socket]
```

### 2.2 Core Physics and Mathematical Engines

#### 5-Second Join Grace Period (`5000ms` Minimum)
Upon login, clients experience chunk loading, resource pack initialization, and teleportation handshakes. `SkColle` enforces a strict `5000ms` grace period (`isInJoinGracePeriod()`) from the exact millisecond of connection:
* All packet checks, watchdog timers, and event dispatches are suppressed for the first 5 seconds.
* Incoming `POSITION` and `LOOK` packets are passively synchronized (`syncPassiveJoinState`) so the anticheat tracks the player's exact coordinates without false-flagging when the 5-second window expires.

#### Packet-Isolated Setback Engine (`<= 0.50m` Ground Clamping)
Standard Bukkit teleports cause rubberbanding loops when in-flight client packets arrive after the setback is issued. `SkColle` isolates setbacks at the packet level:
* **Surface Clamping (`computeSetbackWithinHalfBlockOfGround`):** Scans downward from the last verified safe coordinate to locate the solid ground collision box and clamps the setback Y coordinate within `<= 0.50` blocks of the surface.
* **In-Flight Packet Dropping (`shouldDropPacketDuringSetback`):** Locks the player's Netty pipeline for `135ms` upon setback, dropping any incoming `POSITION` packet that deviates by more than `0.40m` (`distSq > 0.16`) from the setback target until the client acknowledges the teleport.
* **High-Frequency Ground Pull (`tickHighFrequencySetbackWatchdog`):** Prevents players from disabling Fly mid-air right before a flag to hover or fall freely; pulls the player down to the clamped ground coordinate on the next tick.

#### Historical Rewind & Diagonal-Normalized 3D Reach
* **Bounding-Box Rewind History:** Every tick, `SkColle` snapshots the 3D `BoundingBox` of all living entities into a 45-tick concurrent ring buffer (`~1600ms`).
* **Dual-Ping Rewind & Interpolation:** When `USE_ENTITY` arrives, `SkColle` rewinds the target's bounding box by the attacker's effective RTT plus half the victim's RTT, generating interpolated sub-tick bounding boxes (`lerpBox`).
* **3D Slab Method & Diagonal Normalization:** Computes ray-box intersection across candidate eye heights and look vectors. Because hitting a square hitbox at a 45-degree corner yields a longer geometric segment, `SkColle` calculates the horizontal attack angle `theta = atan2(min(|dx|, |dz|), max(|dx|, |dz|))` in `[0, 45 deg]` and applies an oblique correction factor `0.045 * sin(2 * theta)`.
* **I-Frame Filtering (`< 3.60m`):** Hits below `3.60m` that occur while the victim is in damage invulnerability frames (`noDamageTicks > 10`) are ignored to eliminate false flags caused by modern Minecraft knockback desynchronization.

#### Pairwise Statistical Mode GCD Rotation Analysis
* `SkColle` computes the pairwise Greatest Common Divisor (`pairwiseRotationGcd`) between consecutive `dYaw` and `dPitch` deltas scaled by `2^24` (`16777216.0`).
* It stores the last 25 rotation GCDs in an evicting ring buffer (`yawGcdWindow` and `pitchGcdWindow`) and extracts the **Statistical Mode** (`computeMode`) to learn each player's mouse sensitivity grid.
* Rotations during combat that break the learned grid (`delta % learnedGcd`) trigger `KillAura (Type B)` or `AimAssist (Type C)`.

#### Multi-Tick Velocity Curve & Inter-Hit Analyzer
* Intercepts outbound `SetEntityMotion` packets to capture exact server knockback vectors (`expectedVelXZ`, `expectedVelY`), classifies the hit type (`NORMAL`, `SPRINT`, `CRIT`, `SWEEP`, `KB1`, `KB2`), and measures `interHitMs` (`dt` between consecutive hits).
* Tracks the client's response across ticks 1 to 6, flagging 1st-tick vertical/horizontal reduction (`Velocity A/B`), mid-air horizontal braking on ticks 2-6 (`Velocity C - BrakeTick`), and delayed knockback release (`Velocity D`).

#### NoFall (A & B) and Dynamic Speed Envelope (#4)
* **NoFall A (Ground-State Spoofing):** Flags clients sending `onGround = true` while both current and previous server AABB probes confirm the player is airborne (`!serverGround && !lastTickOnGround`).
* **NoFall B (500ms Fall Damage Suppression):** Accumulates real server-side downward displacement (`realServerFallDistance`). If a player lands after falling `> 3.50m` (`+1.0m` per Jump Boost level) on a non-cancelling block and `EntityDamageEvent(FALL)` does not fire within `500ms`, `SkColle` dispatches a `NOFALL_CHECK` packet to flag and apply the suppressed fall damage.
* **Speed A (#4 Dynamic Envelope & Friction):** Enforces horizontal speed limits combining base ground/air envelopes, air friction decay (`prevDXZ * 0.91 + 0.026`), Speed potion compensation, and timed environmental grace memory for Ice (`1200ms`), Slime/Beds (`900ms`), Stairs/Slabs (`500ms`), and Head-Hitters (`650ms`).

---

## 3. SkColle Skript API Reference

### 3.1 Events, Conditions, and Effects

```skript
# Event triggered on the Main Thread for every analyzed packet
on skcolle packet receive:
    # Alias: on skcolle inbound packet:

# Condition to match packet category or unified NMS name
if skcolle packet type is "POSITION", "POSITION_LOOK", "LOOK", or "FLYING":

# Packet-isolated setback (clamps within <= 0.50m of solid ground)
skcolle setback {_player}

# Netty-level packet stream freeze (blocks movement, attacks, and block actions)
skcolle set freeze state of {_player} to true

# Real-time 3D physics prediction stream (P = Predicted, A = Actual, O = Offset)
skcolle toggle argon debug for {_staff} target {_target}
```

### 3.2 Player State & Physics Expressions

| Expression | Return Type | Description |
| :--- | :--- | :--- |
| `skcolle netty cps of %player%` | `Number` | Lag-proof CPS calculated over a 20-client-tick ring buffer. |
| `skcolle netty click stddev of %player%` | `Number` | Standard deviation of click intervals across the last 20 swings. |
| `skcolle effective ping of %player%` | `Number` | Maximum of Bukkit ping and real-time Netty KeepAlive RTT (ms). |
| `skcolle packet jitter of %player%` | `Number` | Standard deviation of `Flying` packet arrival intervals (ms). |
| `skcolle safe location of %player%` | `Location` | Last verified ground-clamped coordinate (`<= 0.50m` from ground). |
| `skcolle ground state of %player%` | `Boolean` | Server-side 3D AABB ground collision probe (`-0.08Y` foot box). |
| `skcolle liquid state of %player%` | `Boolean` | `true` if player bounding box intersects water, lava, or bubble columns. |
| `skcolle climbable state of %player%` | `Boolean` | `true` if player is on ladders, vines, scaffolding, or cobwebs. |
| `skcolle phase state of %player%` | `Boolean` | `true` if player bounding box is inside a full solid occluding block. |
| `skcolle surface bonus of %player%` | `Number` | Combined speed bonus from ice, slime, stairs, slabs, and ceiling blocks. |

### 3.3 Packet Field Dictionary

Fields are read inside `on skcolle packet receive:` using `skcolle packet number field "<name>"` or `skcolle packet text field "<name>"`.

<details>
<summary><strong>Click to expand full Packet Field Dictionary</strong></summary>

| Packet Type(s) | Field Name | Type | Description |
| :--- | :--- | :--- | :--- |
| `USE_ENTITY` | `"entityId"` | `Number` | Target entity numeric ID. |
| `USE_ENTITY` | `"packetReachDist"` | `Number` | 3D hitbox distance corrected for diagonal obliquity and RTT rewind. |
| `USE_ENTITY` | `"packetRayDist"` | `Number` | Slab-method ray intersection distance (`-1.0` if ray missed hitbox). |
| `USE_ENTITY` | `"dynamicMaxReach"` | `Number` | Maximum allowed reach based on dual ping and closing speed. |
| `USE_ENTITY` | `"inHurtTime"` | `Number` | `1` if target is in invulnerability frames (`noDamageTicks > 10`), else `0`. |
| `USE_ENTITY` | `"attackerSpeed"` | `Number` | Attacker horizontal speed (`dXZ`) at the moment of attack. |
| `USE_ENTITY` | `"attackAngleDeg"` | `Number` | Horizontal obliquity angle (`0.0` frontal to `45.0` diagonal). |
| `USE_ENTITY` | `"reachAngleType"` | `Text` | `"DIAG"` if angle `>= 15.0 deg`, otherwise `"FRONT"`. |
| `USE_ENTITY` | `"headAngleErr"` | `Number` | Angular deviation (degrees) between look vector and target center. |
| `LOOK`, `POSITION_LOOK` | `"aimAssistFlag"` | `Number` | `1`=Aim A, `2`=Aim B, `3`=Scaffold C, `4`=KillAura B (GCD), `5`=Aim C (GCD). |
| `LOOK`, `POSITION_LOOK` | `"learnedYawGcd"` | `Number` | Learned statistical mode GCD for Yaw over 25 rotations. |
| `LOOK`, `POSITION_LOOK` | `"learnedPitchGcd"` | `Number` | Learned statistical mode GCD for Pitch over 25 rotations. |
| `LOOK`, `POSITION_LOOK` | `"gcdRemainder"` | `Number` | Modulo remainder error against the learned sensitivity grid. |
| `POSITION`, `FLYING` | `"packetFlyFlag"` | `Number` | `1`=Fly A (Hover), `2`=Fly B (Ascend), `3`=Fly C (Glide), `4`=GroundSpoof A. |
| `POSITION`, `FLYING` | `"packetSpeedFlag"` | `Number` | `1`=Speed A (#4 Dynamic Envelope/Friction), `2`=Speed B (High Burst). |
| `POSITION`, `FLYING` | `"speedContextTag"` | `Text` | Active environment tag (`"GROUND"`, `"AIR"`, `"JUMP"`, `"ICE"`, `"SLIME"`, `"STAIR"`, `"HEADHIT"`). |
| `POSITION`, `FLYING` | `"packetNoFallFlag"` | `Number` | `1` if client spoofed `onGround=true` while airborne (NoFall A). |
| `POSITION`, `FLYING` | `"serverFallDist"` | `Number` | Accumulated real server-side fall distance in meters. |
| `POSITION`, `FLYING` | `"moveDeltaXZ"` | `Number` | Horizontal displacement in the current packet tick. |
| `POSITION`, `FLYING` | `"allowedMaxXZ"` | `Number` | Maximum allowed horizontal displacement for the current tick. |
| `POSITION`, `FLYING` | `"packetDY"` | `Number` | Vertical displacement in the current packet tick. |
| `POSITION`, `FLYING` | `"expectedDY"` | `Number` | Predicted vertical displacement from gravity kinematics. |
| `POSITION`, `FLYING` | `"packetAirTicks"` | `Number` | Consecutive ticks spent airborne according to server AABB probe. |
| `POSITION`, `FLYING` | `"noSwingViolation"` | `Number` | `1` if client attacked without sending `ARM_ANIMATION`. |
| `POSITION`, `FLYING` | `"fakeLagStallMs"` | `Number` | Duration of packet stall (ms) if a combat FakeLag burst is detected. |
| `NOFALL_CHECK` | `"noFallSubType"` | `Number` | `2` when 500ms landing window expires without `EntityDamageEvent(FALL)`. |
| `NOFALL_CHECK` | `"fallDist"` | `Number` | Total server-side distance fallen before landing (meters). |
| `NOFALL_CHECK` | `"expectedDmg"` | `Number` | Suppressed fall damage in health points (`half-hearts`). |
| `VELOCITY_CHECK` | `"velSubType"` | `Number` | `1`=Zero/Low V, `2`=Reduced H/V, `3`=Horizontal Cancel / Mid-Air Brake. |
| `VELOCITY_CHECK` | `"pctH"` / `"pctV"` | `Number` | Percentage of expected horizontal and vertical knockback taken. |
| `VELOCITY_CHECK` | `"interHitMs"` | `Number` | Time elapsed (ms) since the previous combat hit (`dt`). |
| `VELOCITY_CHECK` | `"brakeTick"` | `Number` | Airborne tick (`2..6`) where sudden horizontal braking occurred. |
| `VELOCITY_CHECK` | `"hitType"` | `Text` | Knockback context (`"NORMAL"`, `"SPRINT"`, `"CRIT"`, `"SWEEP"`, `"KB1"`, `"SPRINT_KB2"`). |
| `SCAFFOLD_CHECK` | `"scaffoldSubType"` | `Number` | `1`=Pitch A, `2`=Sprint B, `3`=Rot/Clutch C, `4`=Multi-Air Clutch D. |

</details>

---

## 4. ArgonAC Check Directory (22 Modules)

> [!IMPORTANT]
> All 22 modules respect the **5-Second Join Grace Period** (`5000ms`) and support independent **Detection** (`ENABLED`/`DISABLED`) and **Mitigation** (`ACTIVE`/`SILENT`) states.

### 4.1 Combat Checks

| Check | Subtypes | Detection Method | Mitigation (`ACTIVE`) |
| :--- | :--- | :--- | :--- |
| **Reach** | `Type A` | Historical 3D bounding-box rewind, diagonal corner normalization (`0.045 * sin(2*theta)`), dual-ping allowance, and `< 3.60m` I-Frame filter. | Cancels the next attack damage event (`cancelNextHit`). |
| **HitBox** | `Type A` | 3D Slab-Method ray-box intersection across historical target positions and candidate eye vectors. | Cancels the next attack damage event (`cancelNextHit`). |
| **Velocity** | `Type A, B, C, D` | Outbound `SetEntityMotion` tracking, inter-hit timing (`dt`), 1st-tick V/H ratio, ticks 2-6 mid-air brake detection, and delayed KB release. | Alert & VL tracking only (never setbacks to prevent KB loops). |
| **KillAura** | `Type A, B, C, D` | `A`: NoSwing protocol; `B`: Pairwise Mode Pitch/Yaw GCD; `C`: Same-tick multi-target; `D`: Invalid pitch (`> 90.01 deg`). | Cancels attack hit (`Type B, C`) or setbacks (`Type D`). |
| **AimAssist** | `Type A, B, C` | `A`: Adaptive combat yaw tracking; `B`: High-angle target snap; `C`: Statistical Mode Yaw GCD modulo remainder. | Alert & VL tracking. |
| **AutoClicker** | `Type A, B` | 20-client-tick lag-proof CPS ring buffer (`Type A >= 19 CPS`) and tick-interval standard deviation (`Type B < 5.0ms`). | Alert & VL tracking. |
| **KeepSprint** | `Type A` | Detects failure to apply horizontal deceleration (`0.6x`) on the tick following a sprint-attack against a player. | Alert & VL tracking. |

### 4.2 Movement Checks

| Check | Subtypes | Detection Method | Mitigation (`ACTIVE`) |
| :--- | :--- | :--- | :--- |
| **Fly** | `Type A, B, C` | `A`: 2-tick horizontal air hover lock; `B`: Unauthorized upward air acceleration; `C`: Gravity parabola deviation (`(v - 0.08) * 0.98`). | Instant packet-isolated setback (`<= 0.50m` from ground). |
| **NoFall** | `Type A, B` | `A`: Client `onGround=true` spoofing in mid-air; `B`: Server fall distance accumulator with `500ms` `EntityDamageEvent(FALL)` verification window. | `A`: Instant setback; `B`: Applies suppressed fall damage to player. |
| **Speed** | `Type A, B, C` | `A (#4)`: Dynamic envelope (ice, slime, stairs, head-hitters, potions, air friction `0.91`); `B`: High burst; `C`: Omni-sprint packet spam. | Instant packet-isolated setback. |
| **GroundSpoof** | `Type A` | Persistent `onGround` spoofing after `>= 5` consecutive server air ticks. | Instant packet-isolated setback. |
| **Kinetic** | `Type A` | Mid-air horizontal vector acceleration failure (`AirStrafe`). | Instant packet-isolated setback. |
| **Step** | `Type A` | Instantaneous vertical step height `> 0.62m` without Jump Boost or Levitation. | Instant packet-isolated setback. |
| **Spider** | `Type A` | Sustained vertical wall climbing (`0.11m .. 0.43m` per tick) without climbable blocks. | Instant packet-isolated setback. |
| **Jesus** | `Type A` | Liquid surface walking (`|dY| < 0.004` with `dXZ > 0.18` on water/lava while not grounded). | Instant packet-isolated setback. |
| **Phase** | `Type A` | Horizontal movement (`dXZ > 0.20`) while bounding box intersects full solid occluding blocks. | Instant packet-isolated setback. |
| **BackTrack** | `Type A` | Combat packet stalling (`150ms..1600ms`) followed by `< 10ms` burst release while moving (`FakeLag`). | Alert & VL tracking. |

### 4.3 World & Protocol Checks

| Check | Subtypes | Detection Method | Mitigation (`ACTIVE`) |
| :--- | :--- | :--- | :--- |
| **Scaffold** | `Type A, B, C, D` | `A`: Invalid pitch bridging; `B`: Un-sneaked sprint bridging; `C`: Post-place rotation snapback; `D`: Mid-air downward clutch streak. | **Position-Lock Setback** (locks player in position; placed blocks are **not** removed). |
| **Inventory** | `Type A, B` | `A`: Clicking inventory slots while moving/sprinting; `B`: Sustained walking (`dXZ > 0.105`) while inventory GUI is open. | Closes open inventory + Instant setback. |
| **FastBreak** | `Type A` | Block digging completion delta (`START_DESTROY` to `STOP_DESTROY`) faster than minimum physical break time. | Alert & VL tracking. |
| **FastPlace** | `Type A` | Sustained block placement intervals `< 22ms` while holding a valid block. | Alert & VL tracking. |
| **Timer** | `Type A` | Packet balance clock regulator (`balance += 50 - deltaMs`) detecting game speed acceleration. | Instant packet-isolated setback. |

---

## 5. Commands, Permissions & GUI Administration

### 5.1 Command Reference

**Base Command:** `/argon` (Aliases: `/ac`, `/argonac`)

| Command | Permission | Description |
| :--- | :--- | :--- |
| `/argon gui` | `argon.admin` | Opens the 6-row ArgonAC Control Panel GUI. |
| `/argon alerts` | `argon.admin` / `argon.alerts` | Toggles real-time violation alerts in chat for the executing staff member. |
| `/argon broadcast` | `argon.admin` | Toggles kick/punishment announcements between `PUBLIC (All Players)` and `SILENT (Staff Only)`. |
| `/argon inspect <player>` | `argon.admin` | Opens the live diagnostic GUI for a target player (Ping, Jitter, VL, Freeze, Kick, Teleport). |
| `/argon debug [player]` | `argon.admin` | Toggles the real-time 3D physics vector stream (`P`, `A`, `O`) for the target player. |
| `/argon bypass` | `argon.admin` | Toggles anticheat exemption for the executing administrator. |
| `/argon reset` | `argon.admin` | Clears all active Violation Levels (VL) and buffers for all online players. |
| `/argon discord` | `argon.admin` | Displays clickable links to the official Discord and GitHub repositories. |

### 5.2 GUI Control Panel & Mitigation Modes

Running `/argon gui` opens the main control panel:
* **Category Submenus (Slots 11, 13, 15):** Filter checks by `COMBAT`, `MOVEMENT`, or `WORLD`.
  * **Left-Click on a Check:** Toggles **Detection** (`ENABLED` / `DISABLED`).
  * **Right-Click on a Check:** Toggles **Mitigation** (`ACTIVE` / `SILENT`).
    * **`ACTIVE` Mitigation:** Physically blocks cheats in real time (cancels illegal Reach/HitBox/KillAura hits, executes position-lock setbacks on Fly/Speed/NoFall/Scaffold, applies suppressed fall damage on NoFall B, and closes cheated inventories).
    * **`SILENT` Mitigation:** Disables physical intervention (no setbacks or hit cancellations) while continuing to log Violation Levels and send staff alerts—ideal for recording video evidence of suspected cheaters.
* **Kick/Ban Broadcasts (Slot 30):** Switches between public server-wide punishment broadcasts and silent staff-only notifications.
* **Player Inspector (Slot 32):** Lists all online players with live RTT ping, packet jitter, total VL, and 1-click **Freeze Packet Stream** (`skcolle set freeze state`).

---

## 6. Compilation & Installation

### Requirements
* **Server Software:** Paper `1.20.4` through `1.21.11+` (or Folia/Purpur equivalents).
* **Java Runtime:** Java 21.
* **Dependencies:** Skript `2.9.3+`.

### Building `SkColle.jar` from Source
1. Clone the repository and navigate to the `SkColle` directory:
   ```bash
   git clone [https://github.com/colle23-skript/ArgonAC.git](https://github.com/colle23-skript/ArgonAC.git)
   cd ArgonAC/SkColle
   ```
2. Compile with Maven (requires JDK 21):
   ```bash
   mvn clean package
   ```
3. The compiled jar will be generated in `target/SkColle.jar`.

### Server Installation
1. Copy `SkColle.jar` and `Skript.jar` into your server's `plugins/` folder.
2. Restart the server completely (do not use `/reload` when loading Netty pipeline hooks).
3. Copy `Argon.sk` into `plugins/Skript/scripts/`.
4. Load the script in-game or via console:
   ```text
   /sk reload Argon.sk
   ```

---

## 7. Bug Reports & Support

If you encounter a false flag or bug, please include a **video recording** showing the violation alert details (`VL`, `ping`, and check info bracket) and submit it via:

* **Discord Server:** [https://discord.gg/3tr9aYWZDR](https://discord.gg/3tr9aYWZDR)
* **GitHub Issues:** [https://github.com/colle23-skript](https://github.com/colle23-skript)
