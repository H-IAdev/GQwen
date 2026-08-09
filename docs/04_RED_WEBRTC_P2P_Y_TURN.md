# REGISTRO DE PROGRESO Y DESARROLLO TÉCNICO: RED WEBRTC P2P Y METERED TURN

## 1. Ficha Técnica del Componente
- **Módulo**: Motor Multijugador Co-op P2P Full Mesh, Chat de Texto y Voz PTT
- **Archivos Impactados**: `coop.html` (Integración PeerJS, Metered TURN API, handlers `netSend`, `netSync`, `initLocalAudio`, `setPTTState`, `setupMediaCall`)
- **Estado**: Producción (Milestone 40 — Commit `0b6dbfa`)
- **Protocolos**: WebRTC DataChannels, MediaConnection, Encrypted TURNS TLS (Puerto 443), Metered API

---

## 2. Arquitectura de Red y CGNAT Traversal

```
                                +-----------------------------------+
                                |     Host Client (PeerJS Peer)     |
                                +-----------------+-----------------+
                                                  |
              +-----------------------------------+-----------------------------------+
              |                                   |                                   |
    +---------v----------+              +---------v----------+              +---------v----------+
    | Encrypted TURNS    |              | Direct P2P STUN    |              | Encrypted TURNS    |
    | Relay (TLS 443)    |              | Connection         |              | Relay (TLS 443)    |
    +---------+----------+              +---------+----------+              +---------+----------+
              |                                   |                                   |
              +-----------------------------------+-----------------------------------+
                                                  |
                                +-----------------v-----------------+
                                |     Guest Client (PeerJS Peer)    |
                                +-----------------------------------+
```

### Integración Metered TURN TLS
Para garantizar conexiones sin fallos a través de routers 4G/5G estrictos y redes con CGNAT doble:
- El motor realiza un fetch automático a `https://shoot.metered.live/api/v1/turn/credentials` al iniciar.
- Utiliza servidores TURN sobre TLS (puerto 443 TCP), lo que permite atravesar firewalls corporativos que bloquean tráfico UDP.

---

## 3. Protocolo de Paquetes JSON (`netSend` / `netSync`)

| Tipo de Paquete | Origen → Destino | Parámetros del Payload | Propósito |
| :--- | :--- | :--- | :--- |
| **`pos`** | **Todos → Todos (20hz)** | **`{ from, x, y, z, p, ry, w, rel, t, state, hp }`** | **Sincronización 3D (20hz): Coordenadas $X,Y,Z$, inclinación `p` (pitch), yaw `ry`, arma activa `w`, recarga `rel` y estado.** |
| **`zsound`** | **Host → Guests** | **`{ sound }`** | **RPC de Audio**: Difusión de rugidos de Bosses (`juggernaut`, `stalker`, `rockhurler`) y gemidos de ambiente a todos los clientes. |
| **`pfx`** | **Todos → Todos** | **`{ sound, pId }`** | **RPC de Audio/FX**: Difusión de sonidos de recarga, quejidos de daño, caída e inicio de reanimación entre jugadores. |
| `pstate` | Ambos | `{ state }` | Cambio de estado vital (alive/downed/dead). |
| `wave` | Host → Guest | `{ n }` | Transición e inicio de nueva oleada de infectados. |
| `zhit` | Ambos | `{ id, dmg }` | Notificación de impacto y daño a un zombie. |
| `zhit_fx` | Ambos | `{ pos, hs }` | **RPC Visual/Audio**: Instancia partículas de sangre `ichor` y sonido de impacto `sfx.flesh()`. |
| `zkill` | Ambos | `{ id, hs }` | Notificación de eliminación de zombie (con o sin cranial headshot) y sonido `sfx.die()`. |
| `shot` | Ambos | `{ pId, isFists, fx, fy, fz, tx, ty, tz }` | **RPC Visual/Audio de Disparo**: Instancia trazadora 3D exacta desde el cañón `(fx,fy,fz)` al punto de impacto `(tx,ty,tz)` con Muzzle Flash sprite + light. |
| `rock_fx` | Ambos | `{ pos }` | **RPC Visual/Audio**: Partículas de explosión de roca `stone` y gemido de choque. |
| `pdmg` | Ambos | `{ dmg, target }` | Propagación de daño recibido por un jugador. |
| `chat` | Ambos | `{ sender, msg, color }` | **Chat de Texto Confiable**: Transmisión de mensajes de texto en vivo (Desktop/Mobile). |
| `start` | Host -> Guest | `{ lateJoin: true }` | Sincronización de estado para jugadores que entran a mitad de partida. |

### Optimizaciones de Sincronización (Sync Fixes)
1. **Desacoplamiento de Micrófono y Datos (Non-Blocking DataChannel)**: La inicialización del canal de datos `peer.connect()` es síncrona e independiente de la captura del micrófono (`initLocalAudio()`). La denegación o espera del micrófono en navegadores móviles ya no bloquea la conexión del juego.
2. **Desbloqueo de `netSend` en Lobby**: Transmisión abierta de paquetes de presencia `pos` y `player` durante el Lobby para descubrimiento de avatares antes de iniciar oleada.
3. **Paquete de Presencia Instantáneo**: Envío inmediato de coordenadas e ID al abrir la conexión (`hostConn.on('open')`).
4. **Posicionamiento 3D Directo**: Asignación instantánea `mesh.position.set(x, feetY, z)` en la recepción del primer paquete para evitar deslizamientos desde el origen $(0,0,0)$.
5. **Log de Diagnóstico en Pantalla (`#diagLog`)**: Consola en vivo en la interfaz del Lobby/HUD que registra cada micro-evento de red en teléfonos móviles y PC.

---

## 4. Chat de Voz P2P Push-To-Talk (`MediaConnection`)
- **Captura Única del Micrófono**: Se ejecuta `initLocalAudio()` al conectarse, solicitando `getUserMedia({ audio: true })`. Los tracks de audio inician deshabilitados por defecto (`track.enabled = false`).
- **Llamada de Voz Única**: Se establece una llamada `peer.call(remotePeerId, localAudioStream)` persistente al iniciar la sesión P2P.
- **Elementos `<audio>` Dinámicos**: Los streams recibidos se adjuntan a elementos `<audio id="remoteAudio_..." autoplay style="display:none">` inyectados al `document.body`, garantizando la reproducción fluida sin bloqueos de Autoplay.
- **Push-To-Talk (PTT)**:
  - Al presionar <kbd>V</kbd> (o el botón táctil **VOZ PTT** en móvil), se habilita temporalmente el track (`track.enabled = true`) y aparece la insignia `#pttIndicator` en el HUD.
  - Al soltar la tecla <kbd>V</kbd> / botón táctil, el track regresa a estado silencioso.

---

## 5. Registro de Pruebas y Verificación
- **Prueba 4G/5G**: Conexión P2P exitosa entre dispositivo móvil con datos 4G y escritorio con fibra.
- **Prueba PTT y Chat**: Transmisión limpia de audio sin interrupción de FPS y libre de conflicto entre la escritura en chat y el movimiento del personaje WASD.
