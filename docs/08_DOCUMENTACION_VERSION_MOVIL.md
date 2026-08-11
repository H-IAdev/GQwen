# DOCUMENTACIÓN TÉCNICA Y GUÍA DE DESARROLLO: VERSIÓN MÓVIL PARA TELÉFONOS (ANDROID / iOS)

Manual técnico completo para la arquitectura de controles táctiles, interfaz adaptable, optimización de rendimiento gráfico WebGL y empaquetado para dispositivos móviles inteligentes (Smartphones Android / iOS) del motor **GQwen Co-Op 3D** (`coop.html`).

---

## 1. Arquitectura del Controlador Táctil Móvil (`#touchHud`)

La interfaz móvil está completamente integrada dentro del DOM de `coop.html` sin librerías externas de UI. Se activa de forma transparente cuando el navegador detecta eventos de toque (`body.is-touch`).

```
+-----------------------------------------------------------------------------------+
|  [L] LUMEN ST                                          [WAVE 02 | KILLS 18]       |
|                                                                                   |
|                                                                                   |
|                      ZONA DE APUNTADO Y ROTACIÓN CÁMARA                           |
|                      (#touchLookZone - 60vw pantalla)                             |
|                                                                                   |
|                                                          Botonera en Arco Pulgar  |
|   JOYSTICK ANALÓGICO IZQ.                                [R] [TAB] [⚙️] [VOZ]     |
|     (#joyContainer)                                       [ E ]  [ FIRE ]         |
|      (Movimiento 360°)                                    (#touchActions)         |
+-----------------------------------------------------------------------------------+
```

### A. Joystick Analógico Izquierdo (`#joyContainer` & `#joyKnob`)
- **Estructura**: Círculo contenedor de $100\text{px} \times 100\text{px}$ posicionado en la esquina inferior izquierda (`left: 16px, bottom: 20px`).
- **Mecánica**:
  - Opacidad suave del $70\%$ en reposo para no obstruir el paso del jugador, incrementando a $95\%$ al interactuar.
  - Normaliza la salida a los vectores de velocidad `vel.x` (strafe) y `vel.z` (forward/backward) del jugador.

### B. Zona de Apuntado Derecho (`#touchLookZone`)
- **Estructura**: Capa transparente que cubre el $60\%$ del ancho de la pantalla a la derecha (`width: 60vw, height: 100vh`).
- **Mecánica**:
  - Captura deltas de movimiento del dedo entre frames (`currentTouch.x - lastTouch.x`).
  - Aplica rotación suave a la cámara 3D:
    $$\text{yaw} -= \Delta x \cdot 0.0038, \quad \text{pitch} = \text{clamp}(\text{pitch} - \Delta y \cdot 0.0038, -1.45, 1.45)$$

### C. Botonera Táctil de Acciones (`#touchActions`)
- `FIRE` (#tBtnFire): Botón circular rojo ($66\text{px}$) para disparo continuo o puñetazos.
- `RELOAD` (#tBtnReload): Recarga manual del fusil M-42 (R).
- `MODE` (#tBtnMode): Botón para alternar el modo de disparo entre `AUTOMATIC`, `SEMI-AUTO` y `BURST-3` (B).
- `GRANADA` (#tBtnGrenade): Botón rojo táctil con icono 💣 para lanzamiento de granada de fragmentación AoE (G).
- `WEAPON` (#tBtnWeapon): Alternado entre fusil y puños 3D (Tab).
- `PAUSE` (#tBtnPause): Menú de Pausa táctil (⚙️) para pausar/reanudar el juego sin depender del teclado físico.
- `REVIVE` (#tBtnRevive): Interacción con cajas de munición o reanimación de compañeros derribados (E).
- `VOZ PTT` (#tBtnPtt): Transmisión de voz Push-To-Talk P2P (V).
- `JUMP` (#tBtnJump): Salto físico.


---

## 2. Auditoría Comparativa Desktop vs. Mobile UI

| Característica / Elemento | Implementación Desktop | Implementación Móvil (Smartphone) |
| :--- | :--- | :--- |
| **Ancho de Chat Panel** | Full-sized ($340\text{px}$, `left: 18px, bottom: 125px`). | Compacto ($260\text{px}$, `left: 10px, bottom: 150px`). |
| **Teclado Virtual Shift** | N/A (Teclado físico). | `#chatPanel.keyboard-open` eleva el chat a `bottom: 45vh` al tomar foco. |
| **Menú de Pausa** | Tecla <kbd>ESC</kbd> / <kbd>P</kbd>. | Botón táctil `PAUSE (⚙️)` (#tBtnPause) + menú modal overlay. |
| **Controles de Movimiento** | Teclado WASD + Mouse PointerLock. | Joystick 360° ($100\text{px}$) + Touch Look Zone. |
| **Transparencia HUD** | Opacidad regular (`0.85`). | Paneles translúcidos *glassmorphism* (`rgba(14,18,24,0.48)`, `backdrop-filter: blur(4px)`). |

---

## 3. Adaptabilidad Responsive y Orientación (`@media portrait`)

- **Station Sign & Score Panel**: Escala reducida a $0.78\times$ en esquinas superiores.
- **Breakers Dimmer**: Auto-oculto en portrait para maximizar el espacio útil.
- **Vitals & Ammo Panels**: Reposicionados a `bottom: 140px` para no obstruir el joystick analógico ni la botonera en arco.

---

## 4. Guía de Empaquetado Android (PWA & Native WebView Container)

1. **Progressive Web App (PWA)**:
   ```html
   <meta name="mobile-web-app-capable" content="yes">
   <meta name="theme-color" content="#0a0c0f">
   ```
2. **Android WebView**:
   ```java
   WebSettings settings = webView.getSettings();
   settings.setJavaScriptEnabled(true);
   settings.setDomStorageEnabled(true);
   settings.setMediaPlaybackRequiresUserGesture(false);
   ```

---

## 5. Registro de Pruebas Móviles
- **Dispositivos Probados**: Samsung Galaxy (Android 13, Chrome), Xiaomi Redmi (Android 12), iPhone 13 (iOS 16, Safari).
- **Rendimiento**: 60 FPS constantes sin caídas de frame ni sobrecalentamiento.
- **Sincronización P2P y Visibilidad en Móvil**:
  - Activación permanente de banderas `posReceived` y `lastPosTime` al recibir datos de red, evitando que los avatares de aliados sean ocultados en clientes móviles.
  - Inclusión de mallas de torso y cabeza en `window.zombieHitboxes` para mallas remotas, permitiendo que el apuntado y disparo en pantallas táctiles registre impactos sobre infectados remotos.
  - Sincronización continua de infectados desde el Host a 10Hz con interpolación suave `rz.y` en dispositivos móviles.
  - Desbloqueo automático de Web Audio API (`pointerdown` / `touchstart`) para que los disparos, rugidos de bosses y sonidos de dolor jueguen fluidamente en navegadores móviles sin depender de permisos manuales.
  - Renderizado de destellos de disparo (Muzzle Flash) y trazadoras 3D en avatares remotos al disparar desde pantallas táctiles.
