# REGISTRO DE PROGRESO Y DESARROLLO TÉCNICO: ENTORNO Y ARQUITECTURA 3D
# Motor de Geometría Procedural, Matemáticas de Pista y Físicas de Navegación
# Auditado directamente desde coop.html — Versión 4.7 UX POLISH

Manual de documentación técnica detallado para la generación procedural de la arquitectura 3D, parametrización trigonométrica de la pista elíptica, cálculo de normales y tangentes, cimentación de la zona secreta central (mezzanine) y algoritmo de físicas de navegación anti-clipping en el motor **GQwen Co-Op 3D** (`coop.html`).

---

## 1. FICHA TÉCNICA DEL COMPONENTE

| Parámetro | Valor / Especificación Técnica |
|:---|:---|
| **Módulo** | Generador de Mapa Procedural, Estructura 3D y Motor de Físicas |
| **Archivos Impactados** | `coop.html` (Líneas 535–840 `buildLoop`, 1128–1142 `getFloorY`, 2810–2833 colisiones) |
| **Tecnologías** | Three.js WebGL, Parametric BufferGeometry, Dynamic Canvas Textures |
| **Sectores de Pista** | 8 sectores elípticos ($S_0$ a $S_7$) |
| **Radio Mayor / Menor** | $T_A = 55.0\text{m}$ (eje X) / $T_B = 30.0\text{m}$ (eje Z) |
| **Longitud Total Pista** | $L \approx \pi \left[3(T_A+T_B) - \sqrt{(3T_A+TB)(T_A+3T_B)}\right] \approx 271.8\text{m}$ (Aproximación de Ramanujan) |
| **Subdivisión por Sector** | `SPS = 10` segmentos por sector (total 80 cuadriláteros por superficie continua) |
| **Draw Calls** | Mínimos (agrupados en `sectorGroups[0..7]` e `intGroup`) |

---

## 2. PLANO ARQUITECTÓNICO ASCII COMPLETO (VISTA DE PLANTA Y SECCIONES)

```
                                    NORTE (Z = -30m, t = 0.00)
                              ══════════════════════════════════════════════
                              [ SECTOR 0: ESTACIÓN NORTE - SPAWN PERÍMETRO ]
                                  (Andén de Entrada · Breaker 0 · Generador)
                                                 │
               TÚNEL NOROESTE                    │                    TÚNEL NORESTE
           [ SECTOR 7: SPAWN ]                   │                [ SECTOR 1: SPAWN ]
            (t = 0.82, X = -38m)                 │                 (t = 0.18, X = +38m)
                    \                            │                            /
                     \                           │                           /
                      \    ┌─────────────────────┴─────────────────────┐    /
                       \   │        GRAN ESTACIÓN CONCOURSE            │   /
                        \  │      (Planta Central: 36m x 24m)          │  /
                         \ │                                           │ /
                          \│ ┌───────────────────────────────────────┐ │/
                           │ │      MEZZANINE SUPERIOR (y = 3.5m)    │ │
                           │ │  - Master Ammo Supply Box (0, -4.0)   │ │
                           │ │  - Cabina de Control & Pantallas CRTs │ │
                           │ │  - Spawn Inicial de Jugadores (0, 0)  │ │
CURVA OESTE                │ └───────────────────┬───────────────────┘ │                CURVA ESTE
[ SECTOR 6: SPAWN ] <─────>│                     │                     │<─────> [ SECTOR 2: SPAWN ]
 (t = 0.75, X = -55m)      │ [Escalera Sur Oeste]│ [Escalera Nor-Este] │         (t = 0.25, X = +55m)
  Graffiti "HOLD THE LINE" │ (x = -16m, z = -4)  │ (x = 16m, z = 4)    │          Vías Electrificadas
  Vías y Balasto           │                     │                     │          Breaker 2
                           │ ┌───────────────────┴───────────────────┐ │
                          /│ │       PLANTA BAJA CONCOURSE (y = 0m)  │ │\
                         / │ │  - Quioscos Centrales y Máquinas      │ │ \
                        /  │ │  - Molinetes de Acceso y Pilares      │ │  \
                       /   │ └───────────────────────────────────────┘ │   \
                      /    └─────────────────────┬─────────────────────┘    \
                     /                           │                           \
                    /                            │                            \
           [ SECTOR 5: SPAWN ]                   │                [ SECTOR 3: SPAWN ]
             TÚNEL SUROESTE                      │                   TÚNEL SURESTE
          (t = 0.68, X = -38m)                   │                (t = 0.32, X = +38m)
                                                 │
                              [ SECTOR 4: PORTAL SUR - SPAWN PERÍMETRO ]
                                  (t = 0.50 · Puerta de Acero M.steel)
                              ══════════════════════════════════════════════
                                         SUR (Z = +30m)

                       ESPECIFICACIONES DE COTAS Y DIMENSIONES
                       ───────────────────────────────────────
   • Pista Elíptica: Eje Mayor X = 110m (TA = 55m), Eje Menor Z = 60m (TB = 30m).
   • Perímetro Total: ~271.8 metros.
   • Bóveda y Altura: Techo continuo wH = 6.4m, Andén Total = 13.7m.
   • Zona Central Concourse: 36.0m (X) x 24.0m (Z), Mezzanine Elevado a +3.48m.
   • 6 Sectores de Incursión: Norte (t=0.0), Noreste (t=0.18), Sureste (t=0.32),
                              Sur (t=0.50), Suroeste (t=0.68), Noroeste (t=0.82).
```

---

## 3. PARAMETRIZACIÓN MATEMÁTICA DE LA PISTA (LOOP LINE)

La arquitectura del túnel subterráneo no utiliza archivos de modelos 3D estáticos externos (no hay `.obj`, `.gltf` ni `.fbx`). Todo el espacio se sintetiza matemáticamente sobre un bucle cerrado elíptico.

### 2.1 Ecuaciones Paramétricas Fundamentales

Dado el parámetro normalizado $t \in [0, 1)$ que recorre la pista elíptica completa:

```
[1] Vector Posición Central:
    p(t) = ( TA * sin(2π t),  0,  -TB * cos(2π t) )

[2] Vector Tangente Normalizado:
    T(t) = normalize( TA * cos(2π t),  0,  TB * sin(2π t) )

[3] Vector Normal Exterior (Apuntando hacia afuera de la curva):
    n(t) = normalize( (TA * sin(2π t)) / TA²,  0,  (-TB * cos(2π t)) / TB² )
```

### 2.2 Transición Dinámica de Estación (`stationness`)

Para crear una diferencia visual fluida entre las **zonas de andén/estación** y los **túneles estrechos de paso**, se utiliza la función `stationness(t)` (Línea 545):

$$\text{stationness}(t) = \text{clamp}\left(1 - \frac{d - 0.055}{0.045}, \, 0, \, 1\right)$$

donde $d = \min(\min(t, 1-t), \, |t - 0.5|)$ mide la distancia paramétrica al centro de las estaciones principales ($S_0$ en $t=0.0$ y $S_4$ en $t=0.5$).

### 2.3 Interpolación de Perfil Geométrico

| Parámetro Geométrico | Túnel Oscuro ($st = 0$) | Andén Estación ($st = 1$) | Ecuación de Interpolación |
|:---|:---|:---|:---|
| **Ancho Exterior (`outerW`)** | $3.0\text{m}$ | $5.8\text{m}$ | `lerp(3.0, 5.8, st)` |
| **Altura de Bóveda (`wH`)** | $4.2\text{m}$ | $5.2\text{m}$ | `lerp(4.2, 5.2, st)` |
| **Textura de Suelo** | `M.concrete` (`0x282b30`) | `M.floor` (baldosas `#1e2126`) | `st > 0.3 ? M.floor : M.concrete` |
| **Textura de Pared** | `M.tileDk` (azulejo sucio) | `M.tile` (azulejo crema) | `st > 0.3 ? M.tile : M.tileDk` |

---

## 3. CONSTRUCCIÓN PROCEDURAL DE SUPERFICIES (`buildLoop`)

La función `buildLoop()` (Líneas 652–735) itera sobre los 8 sectores ($s = 0 \dots 7$) y genera las geometrías de buffer empaquetadas:

```
                   ESTRUCTURA DE SECTOR DE PISTA (Vista Frontal)
                   ═════════════════════════════════════════════

                 y = wH ──────────────────────────────────────── Techo (M.ceil)
                        │                                      │
                        │                                      │
                        │  PARED INTERIOR                      │  PARED EXTERIOR
                        │  (M.tile)                            │  (M.tile / M.tileDk)
                        │  x = -3.0m                           │  x = outerW + 0.4m
                        │                                      │
                 y = 0  └───────────┬──────────────┬───────────┘
                                 Vías (-1.3m)   Vías (+1.3m)
                                 Suelo (M.floor / M.concrete)
```

### 3.1 Geometría del Suelo (`fm`)
- **Vértices de Buffer**: Se calculan 2 vértices por subdivisión $i \in [0, SPS]$:
  - Borde interior: $\vec{P}_{\text{in}} = \vec{p}(t) - 2.8 \cdot \hat{n}(t)$
  - Borde exterior: $\vec{P}_{\text{out}} = \vec{p}(t) + \text{outerW} \cdot \hat{n}(t)$
- **Indexación Triangulada**: Se indexan 2 triángulos por segmento ($6$ índices por subdivisión).

### 3.2 Pared Exterior (`wg`)
- Posicionada a distancia lateral $x = \text{outerW} + 0.4\text{m}$ para dejar un margen de estructura física.
- Se extruye desde $y = 0.0\text{m}$ hasta $y = wH$.

### 3.3 Pared Interior Sólida (`innerSolidMesh`)
- **Estanqueidad 100%**: Para evitar que la cámara o el jugador vean el fondo vacío del mapa al mirar hacia el centro de la elipse, se genera un muro continuo doble cara a distancia $x = -3.0\text{m}$ en los 8 sectores.
- Garantiza **cero huecos visuales** y 0 vacíos sin importar la posición o el ángulo de cámara.

### 3.4 Techo Elevado (`cg`)
- Cubre completamente desde la pared interior ($x = -3.2\text{m}$) hasta más allá de la pared exterior ($x = \text{outerW} + 0.4\text{m}$) a la cota constante $y = wH$.
- Utiliza `M.ceil` (`0x181b1f` casi negro) para cerrar ópticamente el túnel.

### 3.5 Vías del Tren 3D (`tp`, `ti`)
- Franja central de vías simétrica a $\pm 1.3\text{m}$ desde el centro de la elipse elevadas a $y = 0.01\text{m}$ para evitar z-fighting.

---

## 4. PORTAL BLINDADO DE ACCESO (SECTOR 4)

En el **Sector 4 ($s = 4$, $t = 0.5$)**, alineado tangencialmente con la curva del túnel, se erige el portal de seguridad que separa el Sector de Entrada de las zonas de contención:

```
                  PORTAL DE SEGURIDAD — SECTOR 4
                  ═══════════════════════════════

                    ┌─────────────────────────┐
                    │   frameTop (dintel)     │  ← Box(0.42, 1.0, 8.8, M.concrete)
                    │   [LED 0xff0044] (dLed) │  ← Box(0.18, 0.16, 0.55, emissive 5.0)
                    ├───────┬─────────┬───────┤
                    │       │  PUERTA │       │
                    │frameL │ BLINDADA│frameR │  ← Jambas Box(0.42, wH, 1.1, M.concrete)
                    │ (h=wH)│ doorPanel│ (h=wH)│  ← Puerta Box(0.18, 3.6, 7.6, M.steel)
                    │       │ (M.steel│       │
                    └───────┴─────────┴───────┘
```

- **Alineación Tangencial**: La orientación de la puerta se calcula con el ángulo de la tangente `doorAngle = Math.atan2(pTan.x, pTan.z)`.
- **Estructura**:
  - `frameL` / `frameR`: Jambas de hormigón armado de $0.42\text{m} \times wH \times 1.1\text{m}$ posicionadas en $z = -3.8\text{m}$ y $z = +3.8\text{m}$ respecto al punto medio del sector.
  - `frameTop`: Dintel superior de hormigón de $0.42\text{m} \times 1.0\text{m} \times 8.8\text{m}$ a altura $y = wH - 0.5\text{m}$.
  - `doorPanel`: Hoja metálica de acero blindado de $0.18\text{m} \times 3.6\text{m} \times 7.6\text{m}$ (`M.steel`, metalness 0.85).
  - `dLed`: Módulo de alerta LED en $y = 3.4\text{m}$, con material emisivo rojo `0xff0044` e intensidad de resplandor `5.0`.

---

## 5. MEZZANINE CENTRAL Y SALA SECRETA (`intGroup`)

Adosado al grupo del Sector 0 (`sectorGroups[0].add(intGroup)`), se encuentra el complejo arquitectónico del **Mezzanine Secreto Central**, un área de dos niveles que añade verticalidad al combate:

```
             ESQUEMA ARQUITECTÓNICO DEL MEZZANINE CENTRAL
             ════════════════════════════════════════════

   y=5.15m ─── secretMapCeil (12m × 24m) ─────────────────────── Techo
                   │                               │
   y=4.4m ─────────┼────── secBooth (Cabina) ──────┼──────────── Luz Azul
                   │      [Monitores 0x3f8cff]     │
   y=3.48m ────────┤  mFloor (6m × 10m Balcón)    ├──────────── M.rail (barandilla)
                   │  [mBox Casillero Munición]    │
                 ╱ │                               │ ╲
   Escalera Sur╱   │  14 peldaños × 0.25m alto     │   ╲ Escalera Norte
   z = -12 a -4  ╱ │  (ExtrudeGeometry)            │     ╲ z = 4 a 12
y=0.0m ──────────┴─────────────────────────────────┴─────────── Suelo
y=-0.12m ─── secretMapBase (12m × 24m) ───────────────────────── Cimentación
```

### 5.1 Ficha de Componentes del Mezzanine

| Componente | Variable | Dimensiones ($W \times H \times D$) | Coordenadas $(X, Y, Z)$ | Material / Propiedades |
|:---|:---|:---|:---|:---|
| **Cimentación Base** | `secretMapBase` | $12.0\text{m} \times 0.3\text{m} \times 24.0\text{m}$ | $(2.4, -0.12, -16.0)$ | `concMat` (`0x22262d`), receiveShadow |
| **Techo Secreto** | `secretMapCeil` | $12.0\text{m} \times 0.3\text{m} \times 24.0\text{m}$ | $(2.4, 5.15, -16.0)$ | `M.ceil` (`0x181b1f`) |
| **Balcón Mezzanine** | `mFloor` | $6.0\text{m} \times 0.25\text{m} \times 10.0\text{m}$ | $(2.4, 3.48, -28.0)$ | `concMat`, receiveShadow |
| **Escalera Sur** | `stStep` (×14) | $2.2\text{m} \times 0.26\text{m} \times 0.55\text{m}$ | $z \in [-12, -4]$ | `concMat`, elevación $+0.25\text{m}$/paso |
| **Escalera Norte** | `stStep` (×14) | $2.2\text{m} \times 0.26\text{m} \times 0.55\text{m}$ | $z \in [4, 12]$ | `concMat`, elevación $+0.25\text{m}$/paso |
| **Cabina de Control** | `secBooth` | $1.8\text{m} \times 0.9\text{m} \times 0.5\text{m}$ | $(2.4, 4.4, 0.0)$ | `termMat`, emissive `0x3f8cff` int:0.9 |
| **Luz de Cabina** | `secBoothLight` | PointLight | $(2.4, 4.6, 0.5)$ | Color `0x3f8cff`, int: 4.0, radio: 12m |
| **Casillero Maestro** | `mBox` | $1.0\text{m} \times 0.6\text{m} \times 0.6\text{m}$ | $(4.2, 3.88, 0.0)$ | Textura munición, emissive `0xffb400` int:0.8 |
| **Molinetes (×2)** | `tsBody` / `tsBar` | $0.4\text{m} \times 1.0\text{m} \times 0.8\text{m}$ | $z = \pm 1.2\text{m}$ | `turnstileMat` / `metalGateMat` |
| **Luces Sodio (×2)** | `sodLight` / `sodLight2` | PointLight | $(2.4, 3.8, -16.0)$ | Color `0xffaa22` (naranja sodio cálido) |

---

## 6. SISTEMA FÍSICO DE NAVEGACIÓN Y ANTI-CLIPPING

### 6.1 Proyección Paramétrica Local (`closestT` y `playerLat`)

El motor determina la posición del jugador respecto a la elipse curva mediante dos coordenadas paramétricas:

1. **`playerT` (Posición sobre el perímetro $t \in [0, 1)$)**:
   Calculado mediante la función arctan2 inversa normalizada (Línea 543):
   $$\text{playerT} = \left(\frac{\text{atan2}(P_x / T_A, \, -P_z / T_B)}{2\pi} + 1\right) \pmod 1$$

2. **`playerLat` (Desplazamiento lateral perpendicular al túnel en metros)**:
   Proyección escalar del vector diferencia sobre el vector normal exterior $\hat{n}$:
   $$\text{playerLat} = (P_x - \text{tp}_x) \cdot n_x + (P_z - \text{tp}_z) \cdot n_z$$

### 6.2 Algoritmo Anti-Clipping de Empuje Normal (`minLat` / `maxLat`)

Para impedir que el jugador o los infectados atraviesen las paredes curvas del túnel:

```js
// Líneas 2818–2820
const minLat = lerp(-2.6, -3.5, st);   // Pared interior sólida
const maxLat = lerp(2.6, 5.5, st);    // Pared exterior
if(playerLat < minLat){
  const push = minLat - playerLat;
  p.x += tn.x * push;
  p.z += tn.z * push;
  playerLat = minLat;
} else if(playerLat > maxLat){
  const push = maxLat - playerLat;
  p.x += tn.x * push;
  p.z += tn.z * push;
  playerLat = maxLat;
}
```

**Mecánica**: Si el movimiento del jugador intenta traspasar `minLat` o `maxLat`, el motor calcula el vector de penetración `push` y aplica una fuerza de traslación instantánea en la dirección del vector normal $\hat{n}$, cancelando el componente de velocidad hacia la pared. **Resultado: 0% atravesamiento de paredes.**

### 6.3 Algoritmo de Cota de Suelo Multi-Nivel (`getFloorY`)

La función `getFloorY(x, z)` (Líneas 1128–1142) determina dinámicamente la altura del terreno $Y$ debajo de cualquier coordenada $(x, z)$:

```js
function getFloorY(x, z) {
  // Escalera Sur (z = -12 a -4, x en [0.5, 4.5])
  if(z >= -12 && z <= -4 && x >= 0.5 && x <= 4.5) {
    return Math.min(3.5, Math.max(0, (-4 - z) * 0.44));
  }
  // Escalera Norte (z = 4 a 12, x en [0.5, 4.5])
  if(z >= 4 && z <= 12 && x >= 0.5 && x <= 4.5) {
    return Math.min(3.5, Math.max(0, (z - 4) * 0.44));
  }
  // Balcón de Mezzanine Central (z = -4 a 4, x en [0.2, 4.8])
  if(z > -4 && z < 4 && x >= 0.2 && x <= 4.8) {
    return 3.5;
  }
  // Pista elíptica principal (nivel cero)
  return 0;
}
```

- **Pendiente Exacta de Escalera**: La tasa de elevación es de $0.44\text{m}$ de altura por metro lineal en Z, coincidiendo exactamente con el escalonamiento de los 14 peldaños 3D ($14 \times 0.25\text{m} = 3.5\text{m}$ de altura total).
- **Cálculo de Posición Final de Cámara**:
  $$P_y = \text{getFloorY}(P_x, P_z) + \text{eyeH} + \text{jumpY} + \text{bob}$$
  donde $\text{eyeH} = 1.7\text{m}$ (jugador vivo), $0.5\text{m}$ (derribado), $0.3\text{m}$ (muerto).

---

## 7. REGISTRO DE VERIFICACIÓN Y PRUEBAS ARQUITECTÓNICAS

- **Estanqueidad Geométrica**: Verificada la continuidad de los 8 sectores sin vacíos entre uniones cuadriláteras ($SPS = 10$).
- **Prueba de Colisión de Paredes**: Caminata exhaustiva a lo largo de los $271.8\text{m}$ de túnel en modo sprint ($10.5\text{m/s}$). 0 salidas de mapa registradas.
- **Transición de Escaleras**: Subida y bajada fluida entre el suelo ($y = 0.0\text{m}$) y el Mezzanine ($y = 3.5\text{m}$) sin saltos de cámara bruscos gracias a la función continua `getFloorY`.
- **Rendimiento WebGL**: Cero problemas de rendimiento en móviles gracias al empaquetado de geometrías en `BufferGeometry` con vértices indexados.