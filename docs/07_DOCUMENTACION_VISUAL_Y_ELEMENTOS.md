# DOCUMENTACIÓN VISUAL COMPLETA — GQWEN CO-OP 3D
# Composición de Mundo, Materiales PBR, Iluminación y Efectos
# Auditado directamente desde coop.html (GQW-VIS-6.0 REV 03 MASTER)

Manual de referencia definitivo para la composición visual, materiales PBR, iluminación, efectos de postproceso y arquitectura del mundo 3D del motor **GQwen Co-Op 3D** (`coop.html`). Todos los valores fueron verificados directamente desde el código fuente consolidado.

---

## 1. PALETA DE MATERIALES PBR — Registro `M`

El sistema de materiales reutilizables se define en el objeto global `M` utilizando materiales estándar PBR `THREE.MeshStandardMaterial` / `THREE.MeshPhysicalMaterial` con `sRGBEncoding` únicamente en los mapas Albedo y decodificación lineal en los mapas Normales/Roughness/AO:

```js
const pbrSet = (prefix, rep, opts = {}) => {
  function map(suf, srgb) {
    const t = texLoader.load(prefix + '_' + suf + '.jpg');
    t.wrapS = t.wrapT = THREE.RepeatWrapping;
    t.repeat.set(rep[0], rep[1]);
    t.anisotropy = MAX_ANISO;
    if (srgb) t.encoding = THREE.sRGBEncoding;
    return t;
  }
  return new THREE.MeshStandardMaterial({
    map: map('albedo', true),
    normalMap: map('normal'),
    roughnessMap: map('rough'),
    aoMap: map('ao'),
    normalScale: new THREE.Vector2(0.8, 0.8),
    envMapIntensity: opts.env != null ? opts.env : 0.45
  });
};
```

| Clave `M.*` | Set de Texturas PBR | Roughness | Metalness | envMap | Uso en Escena |
|:---|:---|:---|:---|:---|:---|
| `M.floor` | ambientCG "Ground037" (2K) + Máscara Puddle | `clearcoat: 0.65` | 0.08 | 0.50 | Suelo de andén cerámico húmedo con reflejos |
| `M.tile` | ambientCG "Tiles071" (1K) Albedo/Normal/AO | 0.35 | — | 0.35 | Muros cerámicos principales de estación |
| `M.tileDk` | ambientCG "Tiles071" Tinte Oscuro (`#8f8d86`) | 0.50 | — | 0.20 | Bóveda y muros oscuros de túneles (Clonado) |
| `M.concrete` | ambientCG "Concrete032" (1K) | 0.85 | — | 0.15 | Jambas, pilares y estructuras de hormigón |
| `M.steel` | ambientCG "Metal032" (1K) | 0.35 | 0.85 | 0.90 | Puertas acorazadas, torniquetes y soportes |
| `M.rail` | ambientCG "Metal032" Cromo Pulido | 0.25 | 1.00 | 0.90 | Barandillas y rieles de la estación |

---

## 2. JERARQUÍA DE ILUMINACIÓN Y PALETA DE COLORIMETRÍA

### 2.1 Reequilibrio de Luces (Unidades Legacy r128)

```
JERARQUÍA DE ILUMINACIÓN — GQwen Co-Op 3D (GQW-VIS-6.0)
════════════════════════════════════════════════════════════════════

[1] AmbientLight
    Color: 0x3a4552 (Azul acero desaturado)
    Intensidad: 0.20 (Antes 0.60 - Incrementa el contraste de sombras)

[2] HemisphereLight
    Color Cielo: 0x5a6a7a | Color Suelo: 0x141210
    Intensidad: 0.24 (Antes 0.50)

[3] Key SpotLight (Sombra Dinámica Principal)
    Color: 0xc8d4e8 (Frío fluorescente)
    Intensidad: 1.90 | Distancia: 26m | Ángulo: PI * 0.42 | Penumbra: 0.65
    Sombra: 1024 x 1024, bias -0.0004
```

### 2.2 Paleta Dominante "Noir Industrial Cálido"

- **Ámbar (#FFB400)**: Identidad de estación, señalética, objetivos y HUD.
- **Sodio (#FFAA22)**: Luces de refugio cálido en el Mezzanine y cabina de control.
- **Frío Fluorescente (#C8D4E8)**: Dirección de la amenaza en los túneles.
- **Verde Infección (#00FFAA)**: Núcleo y ojos del Juggernaut (Capa 2 Bloom).
- **Magenta Stalker (#FF00CC)**: Ojos y aureola fantasma del Stalker (Capa 2 Bloom).

---

## 4. MODELO DE JUGADOR HUMANO (`Soldier.glb`), ARMAS EN MANO Y CÁMARA 3D

1. **Skin Unificada CDN para Jugadores Humanos**:
   - Modelo: `https://cdn.jsdelivr.net/gh/mrdoob/three.js@r128/examples/models/gltf/Soldier.glb`.
   - Asignado exclusivamente a **jugadores humanos** (local y remotos P2P).
   - **Armas 3D en Mano (`weaponPropGroup`)**: Props de fusil de asalto (`wGun`) y escopeta táctica (`wShotgun`) adjuntadas al hueso de la mano derecha (`rightHandBone`).
   - **Animación Dinámica de Ataque y Golpe**: Retroceso de brazo al disparar (`recoilT`) y animación de puñetazo frontal (`punchT`).

2. **Sistema de Vista de Cámara (Tecla `C` / Botón Táctil `C / CAM 3D`)**:
   - Alterna entre 4 modos de vista:
     - `1ª Persona (FPS Clásico)`: Vista limpia de HUD tradicional.
     - `1ª Persona Inmersiva (Full Body 3D)`: La cámara se sitúa en los ojos del soldado y se oculta la cabeza (`scale = 0.0001`), permitiendo mirar hacia abajo y ver tu torso, chaleco táctico, piernas caminando y el arma en tu mano derecha en 3D real.
     - `3ª Persona (Posterior)`: Cámara jalada $2.5\text{m}$ hacia atrás para ver a tu soldado en tercera persona.
     - `3ª Persona (Frontal)`: Cámara ubicada $2.5\text{m}$ enfrente para inspeccionar tu equipamiento de cara.

---

## 4. DISTRIBUCIÓN ERGONÓMICA DE CONTROLES TÁCTILES MÓVILES (3 ZONAS)

La interfaz táctil en dispositivos móviles sigue el estándar industrial de juegos de disparos en 3D (PUBG / CoD Mobile):

1. **Zona Izquierda (Desplazamiento y Utilitarios)**:
   - **Joystick Analógico Virtual (`#joyContainer`)**: Diámetro de $100\text{px}$ ubicado en la esquina inferior izquierda.
   - **Linterna Táctica (`#tBtnFlashlight`) y PTT Voz (`#tBtnPtt`)**: Distribuidos verticalmente sobre el joystick.

2. **Zona Superior Derecha (Sistema)**:
   - **Pausa (`#tBtnPause`) y Cambio de Cámara 3D (`#tBtnCam`)**: Situados en la esquina superior para evitar interrupciones durante el combate.

3. **Zona Inferior Derecha (Arco Ergonómico de Combate)**:
   - **Botón de Disparo Principal (`#tBtnFire`)**: Botón rojo prominente de $72\text{px}$ situado en la esquina inferior derecha.
   - **Arco Táctico Envolvente**: Botones de Salto (`#tBtnJump`), Recarga (`#tBtnReload`), Cambio de Arma (`#tBtnWeapon`), Granada (`#tBtnGrenade`), Modo de Fuego (`#tBtnMode`) y Reanimación (`#tBtnRevive`) organizados en abanico ergonómico natural.