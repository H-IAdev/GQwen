# REGISTRO DE PROGRESO Y DESARROLLO TÉCNICO: SISTEMA DE BOSSES, MODELOS 3D E INTELIGENCIA ARTIFICIAL
# Lógica de Oleadas, Entidades Infectadas, Coordinador Multi-Sectorial y Red P2P Host/Client
# Auditado directamente desde coop.html — Versión 4.8 FINAL BALANCED

Manual de referencia técnico completo para el sistema de inteligencia artificial de enemigos, tabla de parámetros `GAME_CONFIG.zombies` / `ZDEF`, algoritmo de oleadas balanceadas, coordinador multi-sectorial de incursión, canalización multi-modelo 3D y mecánicas especiales en **GQwen Co-Op 3D** (`coop.html`).

---

## 1. FICHA TÉCNICA DEL COMPONENTE

| Parámetro | Especificación Técnica |
|:---|:---|
| **Módulo** | IA de Infectados, Spawner Multi-Sectorial, Sistema de Bosses y Hitboxes |
| **Archivos Impactados** | `coop.html` (`GAME_CONFIG.zombies`, `applyZombieType`, `resetZombieSlotTransforms`, `pickSpawnParams`, `updateZombies`) |
| **Arquitectura Red** | **Host-Authoritative**: Solo el Host calcula IA, navegación, colisiones y spawns de zombies |
| **Tasa de Broadcast** | `zstate` a 10Hz (0.10s) desde Host a todos los Clientes P2P |
| **Modelos 3D GLTF** | Multi-Asset Paralelo (`Zombie_Basic`, `Zombie_Chubby`, `skeleton_-_lowpoly_character`, `Zombie_Arm`) + Fallback Procedural Zero-GC |
| **Total de Entidades** | 9 tipos de infectados (3 comunes + 3 mini-jefes + 3 jefes de élite) |

---

## 2. TABLA COMPLETA DE ENTIDADES Y BALANCEO (`GAME_CONFIG.zombies` / `ZDEF`)

Todos los parámetros de combate y movimiento han sido calibrados a promedios óptimos para permitir maniobra, recarga y retroceso táctico:

| Entidad ID | HP Base | Rango Velocidad | Daño Ataque | Score | Escala 3D (`sc`) | Armadura (`armor`) | Modelo 3D Activo | Comportamiento Táctico |
|:---|:---|:---|:---|:---|:---|:---|:---|:---|
| `walker` | 65 HP | 0.95 – 1.10 m/s | 10 HP | 10 pts | 1.00× | 0% | `Zombie_Basic.gltf` | Infectado común. Avance moderado y constante. Windup 1.60s / Cool 4.0s. |
| `runner` | 48 HP | 1.35 – 1.50 m/s | 8 HP | 15 pts | 0.88× | 0% | `skeleton_-_lowpoly_character.glb` | Trotador ágil con ojos carmesí brillantes. Windup 1.40s / Cool 3.5s. |
| `brute` | 120 HP | 0.80 – 0.95 m/s | 15 HP | 30 pts | 1.35× | 0% | `Zombie_Chubby` / `Basic` | Tanque pesado. Armadura reforzada oscura. Windup 1.85s / Cool 4.5s. |
| `mini_stalker` | 200 HP | 1.15 – 1.30 m/s | 12 HP | 75 pts | 1.25× | 15% | `skeleton_-_lowpoly_character.glb` | Espectro óseo. Cuchillas y aura púrpura. Windup 1.50s / Cool 4.2s. |
| `mini_rockhurler` | 280 HP | 0.70 – 0.85 m/s | 14 HP | 90 pts | 1.50× | 20% | `Zombie_Arm` / Hombreras Roca | Artillería de rocas media. Charge 0.9s / Cool 4.5s. |
| `mini_juggernaut` | 380 HP | 0.75 – 0.90 m/s | 18 HP | 140 pts | 1.70× | 25% | `Zombie_Chubby.gltf` | Sub-jefe colosal. Núcleo verde esmeralda. Windup 2.00s / Cool 5.0s. |
| `boss_stalker` | 460 HP | 1.20 – 1.35 m/s | 18 HP | 180 pts | 1.60× | **30%** | `skeleton_-_lowpoly_character.glb` | **Asesino de Neón**: Aureola magenta, corte lateral doble. |
| `boss_rockhurler` | 580 HP | 0.75 – 0.90 m/s | 22 HP | 220 pts | 2.10× | **35%** | `Zombie_Arm` / Puño Magma | **Jefe Volcánico**: Proyectiles AoE y agarre de infectados. |
| `boss_juggernaut` | 880 HP | 0.80 – 0.95 m/s | 28 HP | 350 pts | 2.40× | **45%** | `Zombie_Chubby.gltf` | **Jefe Final Alpha**: Núcleo neón, cuernos de titan, pisadas sísmicas. |

---

## 3. LÓGICA DE 10 OLEADAS CALIBRADAS (`WAVE_PLAN`) Y RITMO DE COMBATE

### 3.1 Tabla Determinista de Oleadas (`WAVE_PLAN`)

| Oleada | Comunes | Brutos (`brute`) | Mini-Bosses (`minis`) | Jefe Principal (`boss`) | Total Enemigos | Hostiles Activos Máx (`maxActive`) |
|:---|:---|:---|:---|:---|:---|:---|
| **1** | 5 | 0 | Ninguno | Ninguno | **5** | 4 en vivo |
| **2** | 6 | 0 | `mini_stalker` (1) | Ninguno | **7** | 4 en vivo |
| **3** | 7 | 1 | `mini_rockhurler` (1) | Ninguno | **9** | 5 en vivo |
| **4** | 8 | 1 | `mini_juggernaut` (1) | Ninguno | **10** | 5 en vivo |
| **5** | 8 | 1 | Ninguno | `boss_stalker` (1) | **10** | 6 en vivo |
| **6** | 9 | 2 | `mini_stalker` (1) | Ninguno | **12** | 6 en vivo |
| **7** | 10 | 1 | Ninguno | `boss_rockhurler` (1) | **12** | 6 en vivo |
| **8** | 10 | 2 | `mini_juggernaut` (1) | Ninguno | **13** | 7 en vivo |
| **9** | 11 | 2 | `mini_rockhurler` (1) | `boss_stalker` (1) | **14** | 7 en vivo |
| **10 (Final)** | 10 | 2 | `mini_stalker` (1) | **`boss_juggernaut` (1)** | **14** | **8 en vivo** |

### 3.2 Fórmulas de Escalado y Pacing
- **Salud Lineal Suave**: `hpMul = (1.0 + (n - 1) * 0.075) * (1 + 0.10 * (N - 1))` (+7.5% por ola, eliminados los picos exponenciales inmanejables).
- **Tope de Zombis Simultáneos en Escena**:
  $$\text{maxActive} = \min\left(8, \, 4 + \lfloor\text{wave} \cdot 0.4\rfloor + (P - 1) \cdot 2\right)$$
- **Intervalo de Aparición Telegrafiado**:
  $$\text{spawnInt} = \max\left(1.65\text{s}, \, 2.4\text{s} - (n - 1) \cdot 0.08\text{s}\right)$$

---

## 4. COORDINADOR MULTI-SECTORIAL DE INCURSIÓN (`SPAWN_SECTORS`)

Para evitar acumulaciones en un único punto cuando los jugadores limpian el centro de la estación, el motor utiliza un **Coordinador Estratégico de 6 Sectores Perimetrales**:

```
                              [0: Norte Central (t=0.00)]
                                         |
               [5: Túnel Noroeste]                 [1: Túnel Noreste]
                      (t=0.82)                         (t=0.18)
                         \                                 /
                          \        +-------------+        /
                           \       |  ESTACIÓN   |       /
                            \      |  CONCOURSE  |      /
                           /       +-------------+      \
                          /                              \
                         /                                \
               [4: Túnel Suroeste]                 [2: Túnel Sureste]
                      (t=0.68)                         (t=0.32)
                                         |
                               [3: Sur Central (t=0.50)]
```

### Algoritmo Anti-Clustering:
1. **Evaluación de Distancia 3D Real**: Se mide la distancia tridimensional exacta $\text{hypot}(spX - px, spZ - pz)$ a todos los jugadores y la cámara.
2. **Memoria de Rotación (`recentSpawnSectors`)**: Se guardan los últimos 3 sectores utilizados. Un sector recién usado queda bloqueado, forzando a la horda a emerger en pinza desde flancos opuestos.
3. **Filtro de Seguridad 3D**: Distancia mínima de aparición $\ge 20.0\text{m}$, garantizando que ningún enemigo aparezca en el cono de visión directo del jugador.

---

## 5. ARQUITECTURA MULTI-MODELO 3D Y RESPAWN LIMPIO

### 5.1 Canalización de Carga y Fallback Zero-GC
- **CDN R2 / Servidor Local**:
  - `Zombie_Basic.gltf` (1.7 MB, 16 animaciones): Infectados comunes y caminantes.
  - `Zombie_Chubby.gltf` (1.6 MB, 16 animaciones): Juggernauts, Titans y Brutos.
  - `skeleton_-_lowpoly_character.glb` (341 KB): Shadow Stalkers y Runners.
  - `Zombie_Arm.gltf` (1.3 MB): Rock Hurlers.
- **Protección de Protocolo `file:///`**: Si el juego se abre por archivo local directo, se suprimen peticiones AJAX locales que producen errores de CORS con origen `null`, activando el motor procedural **Zero-GC** con apéndices completos (cuernos de titan, hombreras de piedra, núcleo neón y cuchillas de hueso).

### 5.2 Reinicio Integral de Transformaciones (`resetZombieSlotTransforms`)
Al eliminar un zombi (`releaseZombieSlot`) y al reutilizar el slot (`acquireZombieSlot` / `spawnZombie`):
- Se ejecutan `slot.mixer.stopAllAction()` y se limpian los clamps de la animación `Death`.
- Se restauran a cero todas las posturas procedurales (`torso`, `head`, `jaw`, `aL.p`, `aR.p`, `lL.p`, `lR.p` y `g.rotation.set(0, 0, 0)`).
- Se resetean flags: `isCrawling = false; isBiting = false; isThrown = false; dead = false; deathT = 0; loosened = false;`.
- **Resultado**: 100% de los zombis reaparecen erguidos, caminando con naturalidad y sin deformidades heredadas.

---

## 6. COLISIONES EN CONCOURSE Y NAVEGACIÓN DE OBSTÁCULOS

- **Resolución Autoritativa**: Se ejecuta `resolveWorldCollisions(z.g.position, 0.45)` de forma continua en todas las zonas (pista, túneles y concourse central).
- **Repulsión Anticipatoria**: Detección de proximidad ante quioscos y máquinas (`OBSTACLES`) con vector de desvío ortogonal reforzado (`side * 3.5`), garantizando que los enemigos rodeen limpiamente esquinas y obstáculos sin atravesarlos.
2. **Proyección Físico-Parabólica**:
   - Velocidad inicial: $19\text{ m/s}$ dirigida hacia la posición del jugador más cercano.
   - Gravedad aplicada: $9.8\text{ m/s}^2$ en el eje Y.
   - Alcance efectivo máximo: $\sim 60\text{m}$.
3. **Explosión AoE (`rockExplode`, Líneas 1322–1340)**:
   - Al impactar el suelo o a un jugador, la roca explota generando un burst de **18 partículas de roca + 8 chispas**.
   - Daño en área (Radio $3.5\text{m}$):
     $$\text{Daño Recibido} = \text{Daño Base} \cdot \left(1 - \frac{d}{3.5}\right)$$
   - Impacta tanto a jugadores locales como a clientes remotos y causa **empujón físico lateral (stagger)** a infectados comunes en el radio.

### 4.2 Ataque Melee/Combo: Agarre y Lanzamiento de Zombie

1. **Escaneo de Víctima (Radio 10m)**:
   - Escanea infectados comunes a su alrededor. Ignora a otros bosses o zombies ya muertos.
2. **Aproximación e Intercepción (+40% Velocidad)**:
   - Corre hacia la víctima a velocidad incrementada. Al estar a $<2.2\text{m}$, congela la IA del zombie común y lo eleva a $y = 2.2\text{m}$ sobre sus hombros.
3. **Lanzamiento Misil (`doThrowVictim`)**:
   - Lanza al zombie como misil teledirigido a $20\text{ m/s}$ hacia el jugador.
   - Si el zombie volador impacta al jugador ($<1.2\text{m}$), inflige $30\text{ HP}$ de daño directo con efecto visual de 16 partículas de sangre (`ichor`).

---

## 5. SINCRONIZACIÓN DE RED P2P HOST-CLIENTE (`zstate`)

Para evitar problemas de desincronización en partidas cooperativas:

1. **Autoridad del Host**: El Host calcula todas las trayectorias de los zombies, ataques y estados de salud.
2. **Broadcast `zstate` a 10Hz**: Cada $0.10\text{s}$, el Host envía un paquete `zstate` comprimido con el arreglo de todos los zombies activos:
   ```json
   {
     "type": "zstate",
     "zs": [
       { "id": 1, "t": 0.125, "lat": 2.1, "y": 0, "rotY": 1.57, "type": "walker", "state": "alive", "hp": 25 }
     ]
   }
   ```
3. **Interpolación en Cliente**: Los clientes reciben el paquete y aplican interpolación suave a las coordenadas $X, Y, Z$ y ángulo `rotY`, garantizando un movimiento fluido a 60 FPS sin saltos de red.
4. **Hitbox Registration**: Mallas de infectados se registran en `window.zombieHitboxes` en todos los clientes para permitir raycasting local de disparos y puñetazos.

---

## 7. MOTOR DE INTELIGENCIA DE UTILIDAD PARA BOTS DE ESCUADRÓN (`evaluateBotUtility`)

El escuadrón de acompañantes IA (`bots`: VEGA, RUIZ, KIDO) opera mediante un motor de **Utility AI** determinista de 4 estados con umbral de histéresis ($0.12$) y acoplamiento de escuadrón (*Squad Tethering*):

1. **Percepción del Entorno y Distancia al Jugador**:
   - FOV cónico de $110^\circ$ (`dot > -0.34`) + detección omnidireccional cercana ($< 5.0\text{m}$).
   - Threat Score dinámico: $(100 / \text{dist}) \cdot \text{multiplicadores (Bosses 2.2×, Agarre 3.0×)}$.
   - Squad Tethering: Si el bot se distancias más de $16\text{m}$ del jugador, la prioridad de formación se eleva a $u = 0.90$ para reunificar al escuadrón.

2. **Estados Evaluados en Tiempo Real**:
   - **`rescue`** ($u = 0.98$): Rescate incondicional del jugador caído sin importar la distancia inicial, avanzando a $7.5\text{m/s}$.
   - **`retreat`** ($u = 0.85$ / $0.78$): Retirada táctica a sprint ($+30\%$ velocidad) cuando $\text{HP} < 28$ o $\text{HP} < 60$ en regeneración pasiva, efectuando *backpedal* y fuego de supresión.
   - **`combat`** ($u = \min(0.80, \text{threat}/100)$): Mantener distancia idónea de fuego ($9\text{m}$ avance, $3.8\text{m}$ retroceso).
   - **`formation`** ($u = 0.20$ o $0.90$ tether): Cobertura de ángulos muertos + evasión suave de la línea de mira del jugador ($\text{dot} > 0.82$).

---

## 8. ARQUITECTURA DE RESILIENCIA DE RESPAWN Y SEGURIDAD DE COLA (FASE 14)

Para garantizar la estabilidad ininterrumpida de las oleadas y prevenir excepciones en tiempo de ejecución:

1. **Definición Dual y Retrocompatibilidad de Detección**:
   - `getActivePlayerEntities()`: Extrae entidades tridimensionales activas con coordenadas $\{x, z\}$ y vector frontal de visión $\{fwdX, fwdZ\}$ para oclusión de campo visual.
   - `getActivePlayerPositions()`: Alias nativo expuesto que provee una lista mapeada de posiciones $\{x, z\}$ para cálculo de distancias y rejillas de emboscada.
2. **Consumo Transaccional de Cola de Oleadas (*Queue Rollback Safety*)**:
   - `queue.pop()` solo descuenta el enemigo si `spawnZombie()` concluye satisfactoriamente y devuelve un objeto de ranura válido.
   - Si `spawnZombie` no puede instanciar la entidad en ese instante (por ejemplo, pool temporalmente saturado), el tipo de zombi permanece en la cola y reintenta en el siguiente ciclo ($0.5\text{s}$).
3. **Manejo Defensivo en `feed()`**:
   - `feed(msg, type)` encapsulado con salvaguardas contra colecciones DOM nulas o ausentes, asegurando que los anuncios de jefes y avisos de HUD nunca interrumpan la instanciación de infectados.

---

## 9. SISTEMA DE DETECCIÓN BALÍSTICA DUAL-PASS Y ATAQUES ESPECIALES DE JEFES (FASE 15)

### 9.1 Detección Balística Dual-Pass (Hitbox Proxy + Swept-Ray Cylinder Test)
Para eliminar el problema clásico de Three.js donde las mallas deformadas por huesos (`SkinnedMesh`) no actualizan sus bounding boxes en CPU durante animaciones complejas:
1. **Proxy Hitbox Cilíndrico Dedicado**: Cada slot del pool de 60 zombis incluye una malla cilíndrica invisible centrada en su torso (`userData.isHitbox = true`, $r=0.52\text{m}, h=1.85\text{m}$) que escala con `slot.scale` y rota con la entidad.
2. **Swept-Ray Cylinder Intersection**:
   - Pasada 1: Raycasting Three.js directo contra `window.zombieHitboxes`.
   - Pasada 2 (Fallback Proximidad): Si el raycast no colisiona directamente por desfasaje de vértices, se evalúa la distancia horizontal $d_{\text{horiz}} \le 0.68\text{m} \cdot \text{scale}$ sobre el segmento de rayo proyectado y altura $y \in [y_z - 0.15, y_z + 2.15 \cdot \text{scale}]$.
   - **Cálculo de Headshot**: Si el punto de impacto proyectado $y_{\text{pt}} > y_z + 1.30\text{m} \cdot \text{scale}$, se aplica el multiplicador de impacto crítico a la cabeza ($50\text{ DMG}$).

### 9.2 Purga de Ranuras y Ciclo de Vida en Arreglo `zombies`
- Al expirar la animación de muerte ($z.\text{deathT} > 2.4\text{s}$), se invoca `releaseZombieSlot(z)` y de forma simultánea `zombies.splice(i, 1)`. Esto previene duplicaciones de slots muertos que generaban anomalías de spawn en el centro.
- La entidad global `_vCam` fue eliminada de las listas de jugadores activos, garantizando que el punto $(0, 0, 0)$ no sea considerado un jugador estacionario.

### 9.3 Catálogo y Dinámicas de Ataques Especiales
- **Rock Hurler / Mini Rock Hurler**: Ciclo de carga (`z.charging = true`, roca incandescente en mano derecha `z.chargeRockMesh`), lanzamiento parabólico balístico y agarre de infectados débiles como proyectiles vivos. Preservación estricta de la malla de carga a través de reciclajes del pool.
- **Juggernaut / Mini Juggernaut**: Embestida sísmica en línea recta a $15.0\text{m/s}$ ($12.0\text{m/s}$ para mini) que aplasta infectados normales en su trayectoria y arrastra al jugador contra los muros perimetrales infligiendo daño masivo y sacudida de pantalla.
- **Shadow Stalker / Mini Stalker**: Ataque de desvanecimiento y teletransporte táctico (*Shadow Warp / Flank Dash*), disolviéndose en partículas de icor para reaparecer súbitamente en la retaguardia o flanco del jugador a una distancia de $3.8\text{m} - 5.5\text{m}$.