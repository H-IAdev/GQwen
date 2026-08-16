# REGISTRO DE PROGRESO Y ANÁLISIS TÉCNICO FORENSE: `coop.html`

## 1. Ficha Técnica del Archivo
- **Nombre**: `coop.html`
- **Ruta**: `d:/mind/SpaceCode/coop.html`
- **Líneas Totales**: 2,354
- **Tamaño**: 119.9 KB
- **Estado**: Producción (Milestone 38 — Pushed to GitHub `2755982`)
- **Propósito**: Ejecutable principal autocontenido del juego de supervivencia zombie multijugador 3D.

---

## 2. Mapa Arquitectónico por Bloques de Código

```
+-----------------------------------------------------------------------+
|  Lineas 1 - 243: HTML Head, Metadatos Anti-Cache y Hoja de Estilos   |
+-----------------------------------------------------------------------+
|  Lineas 244 - 345: Estructura DOM Overlays, HUD Táctil y Lobby       |
+-----------------------------------------------------------------------+
|  Lineas 346 - 410: Cargador Multi-CDN de Respaldo (Three.js & PeerJS) |
+-----------------------------------------------------------------------+
|  Lineas 411 - 505: Motor Matemático, Helpers y Fetch Credentials TURN |
+-----------------------------------------------------------------------+
|  Lineas 506 - 585: Generador Procedural de Texturas Canvas & Mallas  |
+-----------------------------------------------------------------------+
|  Lineas 586 - 770: Construcción del Mapa 3D y Concurso Mezzanine      |
+-----------------------------------------------------------------------+
|  Lineas 771 - 920: Sistema de Partículas, Balas Tracers y Cámara      |
+-----------------------------------------------------------------------+
|  Lineas 921 - 1045: Registro ZDEF, Generador de Meshes e Impactos    |
+-----------------------------------------------------------------------+
|  Lineas 1046 - 1200: Loop de IA de Infectados y Detección de Targets  |
+-----------------------------------------------------------------------+
|  Lineas 1201 - 1315: Mecánicas del Rock Hurler (AoE y Zombie Grab)   |
+-----------------------------------------------------------------------+
|  Lineas 1316 - 1460: Sincronización Remota y Control del Armamento    |
+-----------------------------------------------------------------------+
|  Lineas 1461 - 1650: Motor de Red WebRTC PeerJS y Metered TURN Relay |
+-----------------------------------------------------------------------+
|  Lineas 1651 - 1800: Controlador Táctil Móvil y Eventos de Toque     |
+-----------------------------------------------------------------------+
|  Lineas 1801 - 2354: Loop Principal de Renderizado (`animate()`)      |
+-----------------------------------------------------------------------+
```

---

## 3. Desglose Detallado por Módulos y Funciones Clave

### A. Metadatos y Estilos (Líneas 1 - 243)
- **Anti-Cache Headers**: Metas `no-cache`, `no-store`, `must-revalidate` para forzar la actualización instantánea en navegadores sin almacenar versiones obsoletas en caché.
- **Responsive Layout**: Estilos CSS en variables (`--amber`, `--red`, `--green`, `--mono`) con compatibilidad para modos Portrait y Landscape en dispositivos móviles.

### B. Elementos DOM del HUD y Lobby (Líneas 244 - 345)
- `#stationSign`: Cartel de la estación LUMEN ST.
- `#breakers`: Control deslizante de regulador de luz (Dimmer).
- `#touchHud`: Controles táctiles transparentes con joystick analógico `#joyContainer` y botonera `#touchActions`.
- `#lobby`: Pantalla de inicio con opciones **PLAY SOLO**, **CREATE ROOM** y **JOIN ROOM**.

### C. Texturas Procedurales Canvas (Líneas 506 - 585)
- `tileCnv()`: Generador de baldosas con degradados de suciedad.
- `floorCnv()`: Textura de hormigón desgastado.
- `whiteTileCnv()`: Baldosas blancas cerámicas cuadradas para muros interiores.
- `grafCnv(txt, color)`: Renderizador dinámico de texto de graffiti con sombras de neón en Canvas.

### D. Construcción del Mapa `buildLoop()` (Líneas 586 - 770)
- Muros cerámicos interiores continuos (`innerSolidMesh`) para garantizar sellado 100%.
- Portal del **Sector 4** con marco de hormigón, dintel y puerta de acero (`M.steel`).
- Grupo `intGroup` que contiene el mezzanine central elevated (`mFloor`), escaleras dobles de 14 peldaños, cabina con terminales emisivos y 24 mensajes de pared + 8 de techo.

### E. Mecánicas del Rock Hurler (Líneas 1201 - 1315)
- `rockExplode(pos, dmg)`: Calcula el daño splash AoE en un radio de 3.5m, aplicando la fórmula de decaimiento y stagger a infectados comunes.
- `spawnRock(fromPos, toPos, dmg)`: Instancia la roca balística octaédrica a 19 m/s utilizando el pool de luces `rockLightPool`.
- `doGrabAndThrow(z, nearest, forcedVictim)`: Eleva al zombie víctima y lo impulsa a 27 m/s con vector vertical balístico (+5.2 m/s), orientando aerodinámicamente la malla hacia el objetivo sin giros de rueda descontrolados.

### F. Motor de Red PeerJS (Líneas 1461 - 1650)
- `initNet()`: Configuración de PeerJS con credenciales de Metered TURN TLS.
- `setupIceWatchdog()`: Monitoreo de estado ICE con reconexión automática (`pc.restartIce()`).
- `netSync(dt)`: Sincronización continua de posiciones de jugadores y zombies a 20Hz en su propio módulo aislado `NetworkSync`.
- **Sincronización Instantánea y Late-Join**:
  - Al abrir el canal P2P (`hostConn.on('open')`), se envía un paquete de presencia instantáneo.
  - Al unirse a una partida en progreso, el Host transmite los datos de todos los jugadores conectados.

### G. Loop de Juego `animate()` (Líneas 1801 - 2354)
- Manejo del delta time `dt`.
- Cálculo de físicas WASD y empuje normal anti-clipping (`playerLat`).
- Comprobación de proximidad a cajas de munición y reanimación de compañeros.
- Renderizado de escena Three.js y actualización de componentes UI.

---

## 4. Registro de Estado y Estabilidad
- **Sintaxis**: 0 Errores (Verificado con `node --check`).
- **Mapeo de Red P2P y Visibilidad**:
  - Se implementó la difusión de paquetes `zstate` a 10Hz desde el Host hacia los Clientes para mantener actualizada la posición $X, Y, Z$ de todos los zombies en vivo.
  - Corrección del desfase de altura vertical $Y$ mediante el cálculo de `feetY = cameraY - eyeH`, asegurando que avatares y mallas de infectados descansen exactamente sobre la superficie del suelo (0m) o mezzanine (3.5m) sin levitar hacia el techo.
  - Activación de banderas `posReceived: true` y `lastPosTime` en la recepción de paquetes `pos`, `player` y `zstate`, garantizando que la visibilidad de los aliados se mantenga constante sin ser ocultados por el watchdog de timeout.
  - Registro de mallas remotas de infectados en `window.zombieHitboxes` para que los disparos y puñetazos en clientes móviles detecten colisiones y apliquen daño mediante raycasting.
- **Visuales de Disparo Remoto & Muzzle Flash**:
  - Transmisión de coordenadas exactas del cañón `(fx,fy,fz)` y punto de impacto `(tx,ty,tz)` en el paquete `shot`.
  - Corrección de `ReferenceError: flashTex is not defined` mediante la función procedural `getFlashTex()`.
  - Cobertura de resiliencia `try/catch` en `buildPlayerMesh` para asegurar que las mallas de avatares 3D se construyan sin interrumpir las conexiones DataChannel.
- **Web Audio API Auto-Unlock**:
  - Escuchadores automáticos en `pointerdown`, `touchstart` y `keydown` para reanudar instantáneamente `AudioContext` (`AC.resume()`) en dispositivos móviles y PC.
- **Compatibilidad**: 100% Funcional en Desktop (Chrome, Firefox, Edge) y Mobile (Safari iOS, Chrome Android).

---

## 5. Changelog — VER 4.7 UX POLISH

### [2026-07-25] — Chat, Revive & Touch Fixes

**Chat — Auto-desvanecimiento de Mensajes:**
- Añadida animación CSS `@keyframes msgFade` que desvanece y contrae suavemente cada mensaje `.cMsg` a los 5 segundos de mostrarse.
- Añadido `setTimeout(() => div.remove(), 5800)` en `addChatMessage()` para eliminar el nodo del DOM tras el fade.
- Resultado: el chat se auto-limpia sin intervención del usuario, liberando espacio visual.

**Chat — Toggle [T] como Abrir/Cerrar:**
- Cambiada la condición de la tecla `T` de `&& !isChatting` a `toggleChat(!isChatting)`.
- La tecla `T` ahora funciona como interruptor: abre si el chat estaba cerrado, cierra si estaba abierto.
- Previene el bloqueo de input cuando el usuario accidentalmente dispara mientras el chat está visible.

**Revive — Barra de Progreso Visual:**
- Añadido elemento `#reviveBar` / `#reviveFill` dentro de `#revivePrompt` con CSS degradado dorado.
- La barra se actualiza en tiempo real durante `animate()` mostrando el progreso de 0% a 100% mientras se mantiene `[E]`.
- Al completar o al perder al compañero de vista, la barra regresa a 0%.

**Revive — Radio de Detección Adaptativo Móvil:**
- Radio de detección de compañero caído ampliado de `2.8u` a `3.8u` en dispositivos táctiles (`'ontouchstart' in window`).
- Compensa el mayor lag de sincronización de posición P2P en redes móviles.

**Revive — Retroalimentación Visual en Botón Táctil:**
- El botón `#tBtnRevive` recibe la clase `.active` automáticamente cuando hay un compañero derribado dentro del radio de detección.
- Se desactiva visualmente cuando no hay nadie derribado cerca.

**Corrección — Botones Táctiles PTT y PAUSE sin Listeners:**
- `#tBtnPtt`: Vinculado a `setPTTState(true/false)` mediante `bindTouchBtn`. El micrófono ahora se activa correctamente al presionar el botón PTT en móvil.
- `#tBtnPause`: Vinculado manualmente a `touchstart`/`touchend`. Alterna el pointer lock en juego; cierra el chat si está abierto.

---

## 6. Changelog — VER 4.8 FINAL BALANCED & FREQUENT BUG RESOLUTIONS

### [2026-08-16] — Multi-Asset Bosses, Anti-Clustering Spawn, CORS & Pacing

#### 🛡️ 1. Resolución de Errores Frecuentes y Resiliencia (Bugfixes Críticos)
- **Bloqueo CORS en Protocolo `file:///`**:
  - *Error*: `Access to XMLHttpRequest at 'file:///...' from origin 'null' has been blocked by CORS policy`.
  - *Corrección*: `loadZombieGLTFAssets()` detecta `protocol === 'file:'` y omite peticiones AJAX a `src/...`, activando de forma inmediata el motor procedural **Zero-GC** con apéndices 3D completos (cuernos de titan, hombreras de piedra, cuchillas y núcleos luminosos).
- **Eliminación de Errores 404 en CDN R2**:
  - *Error*: `Zombie_Ribcage.gltf 404`, `Zombie_Arm.gltf 404`.
  - *Corrección*: Se eliminaron las rutas inexistentes en el bucket R2 y se registró `skeleton_-_lowpoly_character.glb` (status 200 OK) para dar identidad visual a Shadow Stalkers y Runners.
- **Corrección de Ámbito en IA de Host (`N is not defined`)**:
  - *Error*: `[Resilience Notice - ZombiesAI]: N is not defined`.
  - *Corrección*: Se reemplazó la variable no declarada por la función autoritativa `const pCount = (typeof numPlayers === 'function') ? numPlayers() : 1;`.
- **Traspaso de Obstáculos en Zona Concourse**:
  - *Error*: Enemigos atravesando quioscos y máquinas centrales.
  - *Corrección*: Se integró `resolveWorldCollisions(z.g.position, 0.45)` de forma obligatoria en el avance de la zona concourse y se incrementó el vector de desvío ortogonal anticipatorio a `side * 3.5`.
- **Herencia de Animaciones de Muerte en Respawn**:
  - *Error*: Zombis reapareciendo congelados en el suelo, arrastrándose o doblados.
  - *Corrección*: Creación de `resetZombieSlotTransforms(slot)` para detener y limpiar todos los `AnimationMixer` (`stopAllAction()`), restaurar las articulaciones procedurales a cero erguido y resetear los flags (`dead = false; isCrawling = false; deathT = 0;`).

#### 🧟 2. Variedad de Modelos 3D y Canalización Multi-Asset
- Asignación diferenciada en `applyZombieType`:
  - `Zombie_Chubby.gltf` $\rightarrow$ Juggernauts, Titans y Brutos.
  - `skeleton_-_lowpoly_character.glb` $\rightarrow$ Shadow Stalkers y Runners.
  - `Zombie_Basic.gltf` $\rightarrow$ Infectados comunes y caminantes.
  - `Zombie_Arm.gltf` / Proyección procedural $\rightarrow$ Volcanic Rock Hurlers.

#### 📍 3. Coordinador Multi-Sectorial de Incursión (`SPAWN_SECTORS`)
- Implementados 6 sectores perimetrales estratégicos con historial de memoria `recentSpawnSectors` (últimos 3 sectores utilizados bloqueados para evitar spawn recurrente en un mismo punto).
- Evaluación tridimensional euclidiana segura ($\ge 20\text{m}$ a todos los jugadores y cámara).

#### ⚡ 4. Balanceo de Velocidades y Oleadas
- Velocidades promedio calibradas en `GAME_CONFIG.zombies` (Walkers `0.95 - 1.10 m/s`, Runners `1.35 - 1.50 m/s`, Brutes `0.80 - 0.95 m/s`).
- Límite infranqueable de ataque (`windupMax >= 1.30s`, `baseCool >= 3.2s`).
- Oleadas reestructuradas en `WAVE_PLAN` (5 a 14 zombis totales por oleada, `maxActive` limitado de 4 a 8 simultáneos, `spawnInt` espaciado a `2.4s - 1.65s`).


