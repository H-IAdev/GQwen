# REGISTRO DE PROGRESO Y DESARROLLO TÉCNICO: SISTEMA DE BOSSES E INTELIGENCIA ARTIFICIAL
# Lógica de Oleadas, Entidades Infectadas, IA de Combate y Red P2P Host/Client
# Auditado directamente desde coop.html — Versión 4.7 UX POLISH

Manual de referencia técnico completo para el sistema de inteligencia artificial de enemigos, tabla de parámetros `ZDEF`, algoritmo de oleadas, mecánicas especiales de bosses y sincronización de red P2P Host-Cliente en **GQwen Co-Op 3D** (`coop.html`).

---

## 1. FICHA TÉCNICA DEL COMPONENTE

| Parámetro | Especificación Técnica |
|:---|:---|
| **Módulo** | IA de Infectados, Spawner de Oleadas, Sistema de Bosses y Hitboxes |
| **Archivos Impactados** | `coop.html` (`ZDEF` Línea 1030, `buildZombieMesh`, `spawnZombie`, `updateZombies` 1150–1260) |
| **Arquitectura Red** | **Host-Authoritative**: Solo el Host calcula IA, navegación y colisiones de zombies |
| **Tasa de Broadcast** | `zstate` a 10Hz (0.10s) desde Host a todos los Clientes P2P |
| **Total de Entidades** | 6 tipos de infectados (3 comunes + 3 bosses especiales) |

---

## 2. TABLA COMPLETA DE ENTIDADES Y BALANCEO (`ZDEF` — Líneas 1030–1037)

Todos los parámetros de combate están registrados formalmente en el objeto constante `ZDEF`:

| Entidad ID | HP Base | Rango Velocidad | Daño Ataque | Score | Escala 3D (`sc`) | Armadura (`armor`) | Comportamiento Táctico |
|:---|:---|:---|:---|:---|:---|:---|:---|
| `walker` | 25 HP | 3.2 – 4.0 m/s | 12 HP | 10 pts | 1.00× | 0% | Infectado estándar. Ataque frontal con windup de 0.45s. |
| `runner` | 18 HP | 5.5 – 6.8 m/s | 10 HP | 15 pts | 0.88× | 0% | Sprint veloz. Emerge de la niebla a velocidad 2×. |
| `brute` | 60 HP | 2.5 – 3.2 m/s | 20 HP | 30 pts | 1.35× | 0% | Tanque pesado. Torso ensanchado (40% más ancho). |
| `boss_juggernaut` | 550 HP | 2.6 – 3.2 m/s | 45 HP | 350 pts | 2.40× | **45%** | **Jefe Final**: Núcleo neón verde, suprime otros bosses. |
| `boss_stalker` | 280 HP | 6.5 – 8.0 m/s | 30 HP | 180 pts | 1.60× | **30%** | **Asesino de Neón**: Aureola magenta, aproximación silenciosa. |
| `boss_rockhurler` | 380 HP | 2.2 – 2.8 m/s | 35 HP | 220 pts | 2.10× | **35%** | **Jefe de Rango**: Ataque AoE de rocas y agarre/lanzamiento de zombies. |

### Cálculo de Daño Efectivo con Armadura
Cuando una bala o puñetazo impacta a un boss con armadura:
$$\text{Daño Recibido} = \text{Daño Base} \cdot (1 - \text{armor})$$
- Un disparo al torso del Juggernaut ($25\text{ HP}$) inflige: $25 \cdot (1 - 0.45) = 13.75\text{ HP}$.
- Un headshot de fusil ($50\text{ HP}$) al Juggernaut inflige: $50 \cdot (1 - 0.45) = 27.5\text{ HP}$.

---

## 3. LÓGICA DE OLEADAS Y MULTIPLICADORES ESCALARES

### 3.1 Fórmulas de Escalado por Oleada ($N$) y Jugadores ($P$)

En la función `startWave(n)` (Líneas 1640–1650), el Host calcula los multiplicadores de vida y velocidad para adaptar la dificultad al número de jugadores conectados:

$$\text{hpMul} = \left(1 + (N - 1) \cdot 0.7\right) \cdot \left(1 + 0.3 \cdot (P - 1)\right)$$

$$\text{spMul} = 1 + \min\left(0.3, \, (N - 1) \cdot 0.05\right)$$

- **`hpMul`**: Aumenta un $+70\%$ por cada oleada transcurrida y un $+30\%$ por cada jugador adicional en la sala.
- **`spMul`**: Aumenta $+5\%$ de velocidad por oleada hasta un límite máximo de $+30\%$.

### 3.2 Límite de Enemigos Activos Simultáneos (`maxActive`)

Para mantener 60 FPS estables en dispositivos móviles y reducir la sobrecarga de red:

$$\text{maxActive} = \min\left(4, \, 2 + \lfloor N / 2 \rfloor\right)$$

- Oleada 1: Máximo 2 zombies simultáneos en pista.
- Oleada 2-3: Máximo 3 zombies simultáneos.
- Oleada 4+: Límite máximo de **4 zombies simultáneos activos**.

### 3.3 Regla de Exclusividad del Boss Final (`Alpha Juggernaut`)

Cuando el generador de colas selecciona un `boss_juggernaut`:
1. Probabilidad de aparición: $P_{\text{boss}} = \min(0.90, \, 0.20 + (N - 1) \cdot 0.08)$.
2. **Regla de Supresión**: Mientras el Juggernaut esté vivo en la cola o en el mapa, el motor bloquea la generación de cualquier otro infectado especial (`boss_stalker`, `boss_rockhurler` o `brute`).
3. Únicamente se permite el acompañamiento de infectados comunes (`walker` o `runner`), focalizando el fuego del equipo en el Boss Final.

---

## 4. MECÁNICAS DETALLADAS DEL ROCK HURLER (`boss_rockhurler`)

El `boss_rockhurler` posee dos patrones de ataque independientes en `updateZombies()` (Líneas 1214–1260):

```
                      +-----------------------------------------+
                      |   IA DEL ROCK HURLER (DECISIÓN DE ATAQUE)|
                      +--------------------+--------------------+
                                           |
                  +------------------------+------------------------+
                  |                                                 |
         [DISTANCIA JUGADOR > 4.0m]                       [ZOMBIE CERCANO < 10m]
      - Carga de Roca (Telegraph 0.9s)                - Intercepción veloz (+40% spd)
      - Proyectil octaédrico 19 m/s                   - Elevación a y = 2.2m (0.55s)
      - Explosión AoE (Radio 3.5m)                    - Lanzamiento a 20 m/s (30 dmg)
```

### 4.1 Ataque Ranged: Lanzamiento de Roca en Área (`doRockThrow` & `rockExplode`)

1. **Telegraphed Charge ($0.9\text{s}$)**:
   - Se detiene, eleva el brazo derecho $-150^\circ$ (`aR.p.rotation.x = -2.6`).
   - Sintetiza la roca octaédrica de piedra lava (`OctahedronGeometry(0.55)`, `0x8b5c2a`, emisivo `0xff4000`) sobre su mano derecha.
   - Emite aviso en el feed de combate: `ROCK HURLER cargando ataque!`.
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

## 6. REGISTRO DE VERIFICACIÓN Y PRUEBAS

- **Prueba de Armadura**: Confirmado en consola que el Juggernaut absorbe el $45\%$ del daño base por bala.
- **Prueba de Exclusividad**: Verificado en 15 oleadas que ningún `boss_stalker` ni `boss_rockhurler` aparece mientras el `boss_juggernaut` está activo.
- **Prueba de Red P2P**: Clientes móviles y PC reciben el paquete `zstate` a 10Hz manteniendo sincronía exacta de infectados.