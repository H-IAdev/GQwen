# GAME ARCHITECTURE & SYSTEM SPECIFICATION MANUAL — GQWEN CO-OP 3D

Comprehensive technical manual, system architecture blueprint, WebRTC P2P networking protocols, combat mechanics formulas, entity registries, performance optimization strategies, visual direction specifications, and historical changelog for the **GQwen Co-Op 3D Zombie Survival Engine** (`coop.html`).

---

## 1. Executive Summary & Core Concept

**GQwen Co-Op 3D** (`coop.html`) is a zero-dependency, single-file browser-based survival horror FPS engine built using modern WebGL (Three.js r128) and PeerJS WebRTC peer-to-peer networking with enterprise TURN fallback. 

Set within a quarantined subterranean transit system (**LUMEN ST · LOOP LINE**), up to 4 players must defend against dynamic waves of infected hosts, high-speed elite stalkers, heavily armored juggernauts, and boulder-hurling monstrosities while managing limited ammunition, breaker lighting, and interior access pathways.

---

## 2. System Architecture & Tech Stack

```
                                +-----------------------------------+
                                |    GQwen Browser Game Client      |
                                |            (coop.html)            |
                                +-----------------+-----------------+
                                                  |
              +-----------------------------------+-----------------------------------+
              |                                   |                                   |
    +---------v----------+              +---------v----------+              +---------v----------+
    |   Rendering Engine  |              |  Networking Engine |              |    Audio & FX     |
    |  Three.js WebGL    |              |  PeerJS / WebRTC   |              | Web Audio API Synth|
    | Dual Camera Layers |              | Metered TURN Relay |              | Procedural Canvas  |
    +--------------------+              +--------------------+              +--------------------+
```

- **Core Runtime**: HTML5 / ES6 JavaScript (No bundlers, 100% self-contained).
- **Graphics Pipeline**: Three.js WebGL Renderer with dynamic shadow maps (`PCFSoftShadowMap`), 74° FOV camera, 38m far-clip frustum, pitch-black linear fog shroud (`0x040609`), ACES Filmic Tone Mapping (exposure `0.98`), and layer-partitioned bloom glowing elements (Layer 2 with `UnrealBloomPass` threshold `0.85`).
- **Networking Architecture**: Full Mesh WebRTC P2P connection via PeerJS with Metered TURN/STUN credential fetch (`https://shoot.metered.live`) for guaranteed connection across strict 4G/5G mobile routers and CGNAT.
- **UI & Controls**: Responsive HTML5 DOM overlays + Dual Virtual Touch Joysticks for mobile browsers + WASD/Mouse Pointer Lock controls for desktop browsers.

---

## 3. Procedural Map & Environmental Geometry

The game world consists of a continuous curved subterranean loop track divided into **8 Procedural Sectors (`S0` to `S7`)** connected to a central hidden concourse mezzanine.

```
                                    NORTH (Z = -30m, t = 0.00)
                              ══════════════════════════════════════════════
                              [ SECTOR 0: NORTH STATION - PERIMETER SPAWN ]
                                  (Platform Entry · Breaker 0 · Generator)
                                                 │
               NORTHWEST TUNNEL                  │                   NORTHEAST TUNNEL
           [ SECTOR 7: SPAWN ]                   │                [ SECTOR 1: SPAWN ]
            (t = 0.82, X = -38m)                 │                 (t = 0.18, X = +38m)
                    \                            │                            /
                     \                           │                           /
                      \    ┌─────────────────────┴─────────────────────┐    /
                       \   │      GRAND CENTRAL METRO CONCOURSE        │   /
                        \  │        (Floor Area: 36m x 24m)            │  /
                         \ │                                           │ /
                          \│ ┌───────────────────────────────────────┐ │/
                           │ │      UPPER MEZZANINE DECK (y = 3.5m)  │ │
                           │ │  - Master Ammo Supply Box (0, -4.0)   │ │
                           │ │  - Security Control Booth & CRTs      │ │
                           │ │  - Local Player Initial Spawn (0, 0)  │ │
WEST BEND                  │ └───────────────────┬───────────────────┘ │                EAST BEND
[ SECTOR 6: SPAWN ] <─────>│                     │                     │<─────> [ SECTOR 2: SPAWN ]
 (t = 0.75, X = -55m)      │ [South-West Stairs] │ [North-East Stairs] │         (t = 0.25, X = +55m)
  Graffiti "HOLD THE LINE" │ (x = -16m, z = -4)  │ (x = 16m, z = 4)    │          Electrified Rails
  Ballast & Rails          │                     │                     │          Breaker 2
                           │ ┌───────────────────┴───────────────────┐ │
                          /│ │      LOWER CONCOURSE FLOOR (y = 0m)   │ │\
                         / │ │  - Center Kiosks & Vending Machines   │ │ \
                        /  │ │  - Turnstiles & Structural Pillars    │ │  \
                       /   │ └───────────────────────────────────────┘ │   \
                      /    └─────────────────────┬─────────────────────┘    \
                     /                           │                           \
                    /                            │                            \
           [ SECTOR 5: SPAWN ]                   │                [ SECTOR 3: SPAWN ]
             SOUTHWEST TUNNEL                    │                   SOUTHEAST TUNNEL
          (t = 0.68, X = -38m)                   │                (t = 0.32, X = +38m)
                                                 │
                              [ SECTOR 4: SOUTH PORTAL - PERIMETER SPAWN ]
                                  (t = 0.50 · Reinforced Steel Door)
                              ══════════════════════════════════════════════
                                         SOUTH (Z = +30m)
```

### Architectural Specifications
- **Track Layout**: Elliptical track parameterized by `TA = 55m`, `TB = 30m`.
- **Unified Geometry**: Constant ceiling height `wH = 6.4m` and constant concourse width `outerW = 8.5m` (total floor width `13.7m`) across all 8 sectors.
- **Partition Walls**: Double-sided 100% solid white ceramic tile inner partition walls (`M.tile` / `M.tileDk`) along all sectors (`s = 0..7`) preventing void exposure.
- **Sector 4 Portal Doorway**: Curvature-aligned framed portal (`frameL`, `frameR`, `frameTop`) with a heavy steel security door (`M.steel`) and warning status LED (`doorLed`, `0xff0044`).
- **Central Secret Room (`intGroup`)**:
  - **Floor Base**: `12.0m x 0.3m x 24.0m` reinforced concrete slab (`secretMapBase`).
  - **Balcony Mezzanine**: `6.0m x 0.25m x 10.0m` elevated concourse at `y = 3.48m`.
  - **Dual Staircases**: 14-step southern (`z = -12` to `-4`) and northern (`z = 4` to `12`) concrete stairways with step physics height lookup (`getFloorY()`).
  - **Security Control Booth**: Elevated booth featuring blue emissive monitoring terminals (`0x3f8cff`) and turnstiles.
  - **Environmental Graffiti**: 24 survival wall messages + flush curved `"HOLD THE LINE"` decal in Sector S6 at `-5.45m`.

---

## 4. Entity, Model & Combat Registries

### Enemy Entity Matrix (`GAME_CONFIG.zombies` / `ZDEF`)

| Entity ID | Base HP | Speed Range | Damage | Score | Armor Res. | Active 3D Asset | Special Ability / Mechanics |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `walker` | 65 HP | 0.95 - 1.10 m/s | 10 HP | 10 pts | 0% | `Zombie_Basic.gltf` | Standard infected infantry. Windup 1.60s / Cool 4.0s. |
| `runner` | 48 HP | 1.35 - 1.50 m/s | 8 HP | 15 pts | 0% | `skeleton_-_lowpoly_character.glb` | Agile skeletal sprinter, glowing crimson optics. Windup 1.40s / Cool 3.5s. |
| `brute` | 120 HP | 0.80 - 0.95 m/s | 15 HP | 30 pts | 0% | `Zombie_Chubby` / `Basic` | Heavy tank zombie. Windup 1.85s / Cool 4.5s. |
| `mini_stalker` | 200 HP | 1.15 - 1.30 m/s | 12 HP | 75 pts | 15% | `skeleton_-_lowpoly_character.glb` | Skeletal shade. Bone blades & violet aura. |
| `mini_rockhurler` | 280 HP | 0.70 - 0.85 m/s | 14 HP | 90 pts | 20% | `Zombie_Arm` / Stone Pauldrons | Medium volcanic artillery. |
| `mini_juggernaut` | 380 HP | 0.75 - 0.90 m/s | 18 HP | 140 pts | 25% | `Zombie_Chubby.gltf` | Colossal sub-boss. Glowing emerald core. |
| `boss_stalker` | 460 HP | 1.20 - 1.35 m/s | 18 HP | 180 pts | 30% | `skeleton_-_lowpoly_character.glb` | **Neon Assassin**: Magenta aura, telegraphed twin-slash. |
| `boss_rockhurler` | 580 HP | 0.75 - 0.90 m/s | 22 HP | 220 pts | 35% | `Zombie_Arm` / Magma Fist | **Volcanic Behemoth**: Area-of-effect boulder bombardment & minion toss. |
| `boss_juggernaut` | 880 HP | 0.80 - 0.95 m/s | 28 HP | 350 pts | 45% | `Zombie_Chubby.gltf` | **Final Alpha Boss**: Emerald power core, seismic ground-pound. |

### Strategic Multi-Sector Incursion Coordinator (`SPAWN_SECTORS`)
- **6 Peripheral Incursion Sectors**: North Central (`t=0.00`), Northeast Tunnel (`t=0.18`), Southeast Tunnel (`t=0.32`), South Central (`t=0.50`), Southwest Tunnel (`t=0.68`), and Northwest Tunnel (`t=0.82`).
- **Anti-Clustering Memory**: Prevents consecutive spawns from the same sector by tracking the last 3 active sectors in `recentSpawnSectors`.
- **True 3D Euclidean Spacing**: Guarantees a $\ge 20\text{m}$ buffer distance from all living players and the active camera.

---

## 5. Player Controllers, Combat & Physics

### Player Movement & Collision Physics
- **Base Velocity**: Forward/Backward 4.8 m/s, Strafe 3.8 m/s, Sprint multiplier $1.45\times$.
- **Anti-Clipping Pushback**: Clamped track lateral coordinate `playerLat` in range `[-2.65m, outerW - 0.4m]`. Dynamic normal pushback prevents wall geometry clipping.
- **Vertical Physics & Stairs**: `getFloorY(x, z)` handles floor elevation on staircases ($0.44\text{m}$ step incline per meter) and mezzanine concourse ($y = 3.5\text{m}$).

### Weapon Systems
- **M-42 "Token" Rifle**: 30 clip size, 120 reserve (Max 240), 25 Base Dmg / 50 Headshot Dmg, 0.092s fire rate, 1.0s fast reload.
- **3D Combat Fists**: Infinite ammo, 25 Base Dmg / 40 Headshot Dmg, 0.200s attack rate.
- **Additive Spark FX**: Bullet impacts on surfaces trigger additive glowing sparks (`pMats.spark`, `THREE.AdditiveBlending`).

---

## 6. Networking Protocol & WebRTC Peer Sync

Full mesh peer-to-peer data synchronization over WebRTC Data Channels using compact payload formats:

| Packet Type | Direction | Payload Parameters | Description |
| :--- | :--- | :--- | :--- |
| `pstate` | Both | `{ state, trackT, trackLat }` | Real-time position and vitality sync. |
| `wave` | Host -> Guest | `{ n }` | Wave start broadcast. |
| `zhit` | Both | `{ id, dmg }` | Zombie damage event notification. |
| `zkill` | Both | `{ id, hs }` | Zombie kill synchronization. |
| `pdmg` | Both | `{ dmg, target }` | Player damage propagation. |
| `hurler_rock` | Host -> Guest | `{ pos, dst, dmg }` | Rock Hurler projectile RPC event. |

- **Shortest-Angle Interpolation**: Y-axis rotation lerped via `lerpAngle(a, b, t)` to prevent 350° spinning when crossing $\pm\pi$.
- **Shadow Budgeting**: Remote peer meshes use `castShadow = false`, while local players and AI bots (NOAH, ELENA, MARCUS) retain full shadow casting (`castShadow = true`).

---

## 7. Performance Optimizations & Memory Management

1. **Pre-Allocated Scratch Vectors**: Zero GC latency spikes by recycling static global vector instances (`_v1`, `_v2`, `_v3`, `_vRock`, `_vCam`, `_vHand`, `_vDir`).
2. **Fast TypedArray Normal Generator**: `normalFromCanvas()` optimized with direct array index arithmetic in `Uint8ClampedArray` ($10\times$ speedup, $<15\text{ms}$).
3. **Animation Mixer TimeScale Culling**: `z.anim.mixer.timeScale = 0.0` for zombies at distance $> 28\text{m}$ from the player camera.
4. **Dynamic Particle Recycling**: 800 dust motes continuously recycled within a 20m sphere around the active player camera.
5. **Throttled DOM Operations**: Chat messages sanitised with `textContent` (XSS C1 fix); UI overlays updated on dirty flags at 10 Hz (`_uiAccumulator`).

---

## 8. Historical Changelog & Development Timeline

### Milestones 1 to 38
- Core WebGL renderer, WebRTC PeerJS networking, zombie AI, boss mechanics, map geometry, and scratch vector pooling.

### Milestone 39 — Master Visual Overhaul (GQW-VIS-5.0 / VIS-6.0 REV 03)
- Unified all 8 sectors to constant `wH = 6.4m` ceiling and `outerW = 8.5m` concourse width (13.7m total floor width).
- Integrated ambientCG PBR materials (`Tiles071`, `Concrete032`, `Metal032`, `Ground037`) with `MeshPhysicalMaterial` wet floor puddle mask (`clearcoat: 0.65`).
- Added additive spark particle blending (`pMats.spark`) and Rockhurler emissive charge glow ramping (0.4 to 2.2).
- Built 800 dynamic camera-culled dust particles recycled within 20m of player position.

### Milestone 40 — Master Performance & Security Audit (IT-42-C)
- Closed XSS vulnerability in `addChatMessage()` using safe `textContent` nodes.
- Implemented `lerpAngle(a, b, t)` shortest-path angular lerping for remote player yaws.
- Optimized `normalFromCanvas()` with direct typed array indexing ($10\times$ faster).
- Added `AnimationMixer` distance timeScale culling for distant zombies ($>28\text{m}$).
- Created interactive ops console `LUMEN-ST_plan-maestro.html` and master integration documentation.
