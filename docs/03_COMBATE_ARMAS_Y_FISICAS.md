# REGISTRO DE PROGRESO Y DESARROLLO TÉCNICO: COMBATE, ARMAS Y FÍSICAS DE JUEGO
# Especificaciones de Armamento, Balística, Raycasting, Puños 3D y HUD Adaptativo
# Auditado directamente desde coop.html — Versión 4.7 UX POLISH

Manual de referencia técnico detallado para el sistema de combate, especificaciones de armamento, balística por raycasting, retroceso, alternado automático de armas, cajas de suministros y sistema de controles adaptativos Desktop/Mobile en **GQwen Co-Op 3D** (`coop.html`).

---

## 1. FICHA TÉCNICA DEL COMPONENTE

| Parámetro | Especificación Técnica |
|:---|:---|
| **Módulo** | Controlador del Jugador, Motor Balístico, Sistema de Armas y Touch HUD |
| **Archivos Impactados** | `coop.html` (Líneas 940–1000 armas, 1738–1835 `shoot`/`punch`, 3100–3145 touch controls) |
| **Detección de Impactos** | 3D Raycasting exacto sobre `window.zombieHitboxes` (con fallback de proximidad 2D) |
| **Modos de Control** | Teclado/Mouse (Desktop) + Dual Touch Joysticks (Móvil) |
| **Frecuencia de Disparo** | $0.092\text{s}$ (652 disparos/min en Fusil M-42) |

---

## 2. CATÁLOGO Y ESPECIFICACIONES DE ARMAMENTO

### 2.1 Tabla Comparativa de Armas

| Característica / Propiedad | Fusil M-42 "Token" | Puños 3D de Combate (`fistsGroup`) |
|:---|:---|:---|
| **Tipo de Ataque** | Fuego automático por raycasting balístico | Combate cuerpo a cuerpo (Melee strike) |
| **Capacidad Cargador (`MAG`)** | 30 rondas (9mm) | Infinito (cuerpo a cuerpo) |
| **Reserva por Defecto / Máx** | 120 rondas / 240 rondas máx. | N/A |
| **Daño al Torso (Body Impact)** | **25 HP** | **25 HP** |
| **Daño a Cabeza (Headshot)** | **50 HP** (2.0× multiplicador) | **40 HP** (1.6× multiplicador) |
| **Cadencia de Disparo / Cadencia** | $0.092\text{s}$ (652 RPM) | $0.200\text{s}$ (300 golpes/min) |
| **Tiempo de Recarga (`relT`)** | $1.0\text{s}$ | Instantáneo |
| **Alcance Efectivo** | $70.0\text{m}$ (Raycast far limit) | $3.5\text{m}$ (Raycast far limit, fallback $3.2\text{m}$) |
| **Dispersión (Spread Base)** | $0.006\text{ rad}$ (base en reposo) | N/A |

### 2.2 Balística, Dispersión y Retroceso del Fusil M-42

Al llamar a `shoot()` (Líneas 1787–1835):

1. **Fórmula de Dispersión Dinámica (`spread`)**:
   $$\text{spread} = 0.006 + (\text{si en movimiento } > 1.2\text{m/s} ? 0.010 : 0) + (\text{si en salto } > 0.02\text{m} ? 0.018 : 0)$$
   - En reposo: Dispersión mínima de $0.006\text{ rad}$ ($0.34^\circ$).
   - Caminando / corriendo: Dispersión incrementada a $0.016\text{ rad}$.
   - Disparando en el aire (salto): Dispersión máxima de $0.034\text{ rad}$ ($1.95^\circ$).

2. **Físicas de Retroceso Impulsivo (`kickP` y `kickY`)**:
   $$\text{kickP} += 0.011 + \text{Random}(0, \, 0.004) \quad \text{(Elevación vertical de cámara)}$$
   $$\text{kickY} += \text{Random}(-0.003, \, +0.003) \quad \text{(Desviación horizontal aleatoria)}$$
   - El retroceso se amortigua exponencialmente en cada frame: `kickP *= exp(-9 * dt)`.

3. **Muzzle Flash y Luz 3D de Disparo**:
   - `flashLight.intensity = 50` (PointLight de $6\text{m}$ parentada a la cámara).
   - `flashSpr`: Sprite 2D de canvas radial (`getFlashTex()`) de escala $0.12 \dots 0.22$, visible por $0.045\text{s}$ con rotación aleatoria.

4. **Trazadoras de Bala (`fireTracer`)**:
   - Se instancian cilindros luminosos `BoxGeometry` con `AdditiveBlending` que conectan la boca del fusil `tip` con el punto de impacto exacto `end`.

---

## 3. SISTEMA DE PUÑOS 3D Y AUTO-SWITCHING POR AGOTAMIENTO

### 3.1 Animación Dinámica de Puñetazos (`punch()`, Líneas 1740–1785)

Cuando el jugador equipa los **Puños 3D de Combate** (`activeWeapon === 'fists'`):

```js
// Extensión senoidal del puño activo (38cm de extensión)
const pOff = Math.sin(Math.min(1, punchT) * Math.PI) * 0.38;
fistsGroup.children[0].position.set(-.22, -.22 + Math.sin(bobT)*.005, -.32 - (punchSide === 0 ? pOff : 0));
fistsGroup.children[1].position.set( .22, -.22 + Math.sin(bobT)*.005, -.32 - (punchSide === 1 ? pOff : 0));
```

- **Alternancia de Puño (`punchSide`)**: Alterna entre el brazo izquierdo ($0$) y derecho ($1$) en cada golpe.
- **Raycasting Melee de Impacto**:
  - `ray.far = 3.5m`: Detecta colisiones exactas en 3D sobre la cabeza o torso de mallas registradas en `window.zombieHitboxes`.
  - **Fallback de Proximidad 2D ($3.2\text{m}$)**: Si el raycast 3D falla por imprecisión de inclinación de cámara en pantallas táctiles móviles, escanea zombies en un radio 2D de $3.2\text{m}$ y aplica $25\text{ HP}$ de daño automático, garantizando golpe constante en móvil.

### 3.2 Lógica de Auto-Switching Automático

```
                    ┌────────────────────────────────────────┐
                    │      INTENTO DE DISPARO (shoot)        │
                    └───────────────────┬────────────────────┘
                                        │
                    ┌───────────────────┴────────────────────┐
                    │   ¿Queda munición en el cargador?      │
                    └─────────┬──────────────────────┬───────┘
                           SÍ │                      │ NO
                              v                      v
                       [DISPARAR M-42]    ┌──────────────────────────┐
                       (ammo--)           │ ¿Queda munición reserva? │
                                          └────┬─────────────────┬───┘
                                            SÍ │                 │ NO
                                               v                 v
                                         [RECARGAR]      [AUTO-SWITCH PUÑOS]
                                         (startReload)   switchWeapon('fists')
                                                         feed('OUT OF AMMO')
                                                         punch()
```

- Si el jugador intenta disparar sin balas en el cargador pero con reserva ($ammo = 0, reserve > 0$), se activa la recarga automática (`startReload()`).
- Si la munición total cae a cero absoluto ($ammo = 0, reserve = 0$), el motor cambia instantáneamente a los **Puños 3D de Combate**, emite la alerta `OUT OF AMMO — BARE FISTS ENGAGED` y lanza un puñetazo inmediato sin trabar el loop de disparo.

---

## 4. CAJAS DE SUMINISTROS 3D Y PAQUETES DINÁMICOS

Existen 2 fuentes de reabastecimiento de munición en el mapa:

### 4.1 Cajas Estacionales 3D de Estación (`ammoCrateMat`)
- **Ubicación**: Cajas estacionales fijas situadas en las plataformas del túnel y en el Mezzanine Central (`mBox`).
- **Recompensa**: **$+60$ rondas de reserva** al interactuar con **[E]** (o botón táctil **REVIVE**).
- **VFX**: Trigger de **24 partículas de chispas doradas** (`spark`, velocidad $4.0\text{ m/s}$) + sonido de recarga metálica `sfx.pickup()`.

### 4.2 Paquetes Dinámicos de Munición (`window.ammoCrates`)
- **Spawning**: Cajas flotantes con resplandor dorado (`emissiveIntensity: 0.7`) que aparecen en el suelo.
- **Recompensa**: **$+30$ rondas de reserva** al pasar sobre ellas.
- **VFX**: Burst de **12 partículas de chispas doradas** (`spark`, velocidad $2.5\text{ m/s}$).

---

## 5. CONTROLES Y HUD TÁCTIL MÓVIL ADAPTATIVO

El juego soporta detección transparente de dispositivos táctiles (`body.is-touch`):

### 5.1 Esquema de Mapeo Desktop vs. Mobile

| Acción de Gameplay | Control Teclado / Mouse (Desktop) | Control Táctil Móvil (`#touchHud`) |
|:---|:---|:---|
| **Movimiento 360°** | Teclas <kbd>W</kbd> <kbd>A</kbd> <kbd>S</kbd> <kbd>D</kbd> | Joystick Analógico Izquierdo (`#joyContainer`, $100\text{px}$) |
| **Rotación de Cámara** | Movimiento de Mouse (PointerLock) | Zona de Apuntado Derecho (`#touchLookZone`, $60\text{vw}$) |
| **Disparo / Golpe** | Botón Izquierdo del Mouse | Botón Táctil `FIRE` (#tBtnFire, $66\text{px}$ rojo) |
| **Recarga Manual** | Tecla <kbd>R</kbd> | Botón Táctil `RELOAD` (#tBtnReload) |
| **Cambio de Arma** | Tecla <kbd>TAB</kbd> / Teclas <kbd>1</kbd> <kbd>2</kbd> | Botón Táctil `WEAPON` (#tBtnWeapon) |
| **Reanimación / Munición** | Tecla <kbd>E</kbd> (mantener) | Botón Táctil `REVIVE (E)` (resplandor automático en aliado caído) |
| **Voz Push-To-Talk** | Tecla <kbd>V</kbd> (mantener) | Botón Táctil `VOZ PTT` (#tBtnPtt, activación directa de micrófono) |
| **Menú de Pausa** | Tecla <kbd>ESC</kbd> / <kbd>P</kbd> | Botón Táctil `PAUSE (⚙️)` (#tBtnPause) |
| **Abrir / Cerrar Chat** | Tecla <kbd>T</kbd> (toggle bidireccional) | Teclado virtual con elevación `#chatPanel.keyboard-open` |

---

## 6. REGISTRO DE VERIFICACIÓN Y PRUEBAS DE COMBATE

- **Prueba de Balística**: Headshots verificados con multiplicador $2.0\times$ ($50\text{ HP}$ vs $25\text{ HP}$ cuerpo).
- **Prueba de Agotamiento de Munición**: Verificado que al llegar a 0 balas el cambio a puños 3D es instantáneo sin perder el flujo de combate.
- **Prueba de Melee Fallback**: Los puños 3D detectan correctamente zombies a $<3.2\text{m}$ en dispositivos móviles aunque la cámara no apunte directamente al centro del hitbox.
- **Prueba de Controles Táctiles**: Verificado funcionamiento de los botones `VOZ PTT` y `PAUSE` tras la actualización VER 4.7.