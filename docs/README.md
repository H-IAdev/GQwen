# ÍNDICE GENERAL DE DOCUMENTACIÓN TÉCNICA Y REGISTROS DE PROGRESO — GQWEN CO-OP 3D

Este directorio contiene los registros de progreso, guías visuales y especificaciones técnicas detalladas para cada uno de los sistemas principales del motor de juego **GQwen Co-Op 3D Zombie Survival** (`coop.html`).

---

## 📂 Registros Módulo por Módulo

1. 🏛️ **[01_ENTORNO_Y_ARQUITECTURA_3D.md](file:///d:/mind/SpaceCode/docs/01_ENTORNO_Y_ARQUITECTURA_3D.md)**
   - Motor trigonométrico de pista elíptica (`TA = 55m`, `TB = 30m`).
   - Muros cerámicos divisorios continuos sellados al 100% en los 8 sectores (`s = 0..7`).
   - Portal curvado de acceso en Sector 4 con puerta acorazada de acero (`M.steel`).
   - Geometría unificada: techo continuo `wH = 6.4m`, andén de `13.7m` de ancho total.

2. 🧟 **[02_SISTEMA_DE_BOSSES_Y_IA.md](file:///d:/mind/SpaceCode/docs/02_SISTEMA_DE_BOSSES_Y_IA.md)**
   - Matriz de entidades (`ZDEF`): Walkers, Runners, Brutes, Alpha Juggernaut, Neon Stalker y Rock Hurler.
   - Sistema de Oleadas Exclusivas para el Boss Final (Juggernaut esmeralda `0x00ffaa`).
   - Mecánicas del Rock Hurler: Telégrafo de carga de roca con rampa emisiva (0.4 a 2.2) y explosión AoE (3.5m).

3. 🔫 **[03_COMBATE_ARMAS_Y_FISICAS.md](file:///d:/mind/SpaceCode/docs/03_COMBATE_ARMAS_Y_FISICAS.md)**
   - Controlador del Jugador (WASD + Dual Joysticks Táctiles Móviles).
   - Sistema de Armamento: Fusil M-42 vs Puños 3D de Combate.
   - Chispas de impacto aditivas (`pMats.spark`, `THREE.AdditiveBlending`).

4. 🌐 **[04_RED_WEBRTC_P2P_Y_TURN.md](file:///d:/mind/SpaceCode/docs/04_RED_WEBRTC_P2P_Y_TURN.md)**
   - Red P2P Full Mesh con PeerJS e integración Metered TURN TLS (Puerto 443 TCP).
   - Interpolación angular `lerpAngle(a, b, t)` con resolución de ruta más corta $\pm\pi$.
   - Presupuesto de sombras P2P: `castShadow = !isRemote`.

5. ⚡ **[05_OPTIMIZACION_Y_RENDIMIENTO.md](file:///d:/mind/SpaceCode/docs/05_OPTIMIZACION_Y_RENDIMIENTO.md)**
   - Expediente **IT-42-C**: Generador `normalFromCanvas()` con TypedArrays ($10\times$ más rápido).
   - Culling de animación en CPU (`z.anim.mixer.timeScale = 0` a $>28\text{m}$).
   - Sanitización XSS en Chat (`textContent`) y recubrimiento dinámico de 800 partículas de polvo.

6. 📄 **[06_REPORTE_TECNICO_COOP_HTML.md](file:///d:/mind/SpaceCode/docs/06_REPORTE_TECNICO_COOP_HTML.md)**
   - Registro forense y desglose detallado bloque por bloque de `coop.html` (VER 4.6 P2P RESILIENT).

7. 🎨 **[07_DOCUMENTACION_VISUAL_Y_ELEMENTOS.md](file:///d:/mind/SpaceCode/docs/07_DOCUMENTACION_VISUAL_Y_ELEMENTOS.md)**
   - Guía visual **GQW-VIS-6.0 REV 03**: Paleta ambientCG PBR, IBL reflections (`scene.environment`), iluminación legacy reequilibrada y UnrealBloomPass selectivo en Capa 2.

8. 📱 **[08_DOCUMENTACION_VERSION_MOVIL.md](file:///d:/mind/SpaceCode/docs/08_DOCUMENTACION_VERSION_MOVIL.md)**
   - Manual técnico para versión móvil (Android / iOS): Joystick analógico 360°, Touch Look Zone, adaptabilidad `@media (orientation: portrait)` y tier `REDUCED`.

---

## 🛠️ Comandos de Verificación del Desarrollador
- **Sintaxis de Scripts**: `node --check scratch/apply_master_optimization_fixes.js`
- **Sintaxis de Game Engine**: `node -e "new Function(scriptText)"`
- **Push Aislado a Repositorio**: `git push origin main` (`coop.html`)
