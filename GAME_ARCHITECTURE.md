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
       [SECTOR 0: ENTRY]  <--->  [SECTOR 1: NORTH TUNNEL]  <--->  [SECTOR 2: BEND]
              ^                                                         |
              |                                                         v
  [SECTOR 7: WEST BEND]       +---------------------------+       [SECTOR 3: TUNNEL]
              ^               | CENTRAL SECRET MEZZANINE  |             |
              |               |  - Dual 14-Step Stairs    |             v
  [SECTOR 6: SOUTH TUNNEL]    |  - Master Ammo Supply     |     [SECTOR 4: ACCESS PORTAL]
              ^               |  - Control Booth & Screens|             |
              |               +---------------------------+             v
       [SECTOR 5: BEND]   <--------------------------------------- [SECTOR 5: EAST BEND]
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

## 4. Entity & Combat Registries

### Enemy Entity Matrix (`ZDEF`)

| Entity ID | Base HP | Speed Range | Damage | Score | Armor Res. | Special Ability / Mechanics |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `walker` | 25 | 3.2 - 4.0 m/s | 12 HP | 10 pts | 0% | Basic infected infantry. |
| `runner` | 18 | 5.5 - 6.8 m/s | 10 HP | 15 pts | 0% | Fast flanker, lower HP. |
| `brute` | 60 | 2.5 - 3.2 m/s | 20 HP | 30 pts | 0% | Heavy tank zombie. |
| `boss_juggernaut` | 550 | 2.6 - 3.2 m/s | 45 HP | 350 pts | 45% | **Final Boss**: Emerald core glowing green (`0x00ffaa`), high health & armor. |
| `boss_stalker` | 280 | 6.5 - 8.0 m/s | 30 HP | 180 pts | 30% | High-speed neon assassin, magenta eyes (`0xff00cc`), ghost aura. |
| `boss_rockhurler` | 380 | 2.2 - 2.8 m/s | 35 HP | 220 pts | 35% | **Ranged Heavy Boss**: Magma rock glowing charge (0.4 to 2.2 emissive intensity), 19m/s boulders. |

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
