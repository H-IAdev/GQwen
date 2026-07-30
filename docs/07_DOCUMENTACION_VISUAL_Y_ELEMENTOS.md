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

## 3. EFECTOS DE POSTPROCESO Y COMBATE

1. **ACES Filmic Tone Mapping**:
   - `renderer.toneMapping = THREE.ACESFilmicToneMapping`
   - `renderer.toneMappingExposure = 0.98`
2. **UnrealBloomPass Selectivo (Capa 2)**:
   - Umbral de Bloom fijado en `0.85`, permitiendo que únicamente los emisivos intensos (ojos de bosses `6.0`, `CORE_NEON` `4.5`, destellos de arma `45`) generen resplandor luminoso sin saturar el entorno.
3. **Chispas Aditivas de Impacto**:
   - Material `pMats.spark` configurado con `THREE.AdditiveBlending` y `depthWrite = false`.
4. **Telégrafo Térmico del Rockhurler**:
   - La roca volcánica proyectada incrementa dinámicamente su intensidad emisiva de `0.4` a `2.2` durante la carga de 0.9s.