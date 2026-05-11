# Unidad 8

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
### Bitácora Técnica — Unidad 8: Performance Audiovisual en Vivo

#### Adapters utilizados

Para la performance final se utilizó el modo `strudel-osc-microbit` del `bridgeServer.js`,
que levanta tres adapters simultáneamente:

- **StrudelOscMicrobitAdapter** — adapter compuesto que integra las tres fuentes en un
  solo proceso. Se arranca con:

  - **MicrobitBinaryAdapter** — maneja la comunicación serial con el micro:bit físico.
  Lee frames binarios de 8 bytes a 115200 baudios.
- **StrudelAdapter** — escucha eventos OSC de Strudel en el puerto 8080.
- **OpenStageControlAdapter** — recibe mensajes OSC del controlador en el puerto 9000.

---

### Contrato de mensajes por fuente

Todos los mensajes llegan al frontend normalizados por el `bridgeClient.js` con un campo
`type` que identifica la fuente.

### micro:bit
El micro:bit envía frames binarios de 8 bytes por serial:

| Byte | Contenido |
|------|-----------|
| 0 | Header `0xAA` |
| 1-2 | X acelerómetro (Int16 big-endian) |
| 3-4 | Y acelerómetro (Int16 big-endian) |
| 5 | Botón A (0 o 1) |
| 6 | Botón B (0 o 1) |
| 7 | Checksum (suma bytes 1-6 % 256) |

El bridge normaliza esto y lo reenvía al frontend como:
```json
{ "type": "microbit", "x": -340, "y": 120, "btnA": false, "btnB": false }
```

El código MicroPython flasheado en el micro:bit:
```python
from microbit import *
import struct

uart.init(baudrate=115200)

while True:
    x = accelerometer.get_x()
    y = accelerometer.get_y()
    a = 1 if button_a.is_pressed() else 0
    b = 1 if button_b.is_pressed() else 0
    xy = struct.pack('>hh', x, y)
    chk = (xy[0] + xy[1] + xy[2] + xy[3] + a + b) & 0xFF
    frame = bytes([0xAA]) + xy + bytes([a, b, chk])
    uart.write(frame)
    sleep(33)
```

### Strudel
Strudel envía eventos OSC al bridge con cada beat. El bridge los reenvía como:
```json
{
  "type": "strudel",
  "timestamp": 1715000000000,
  "payload": {
    "args": ["s", "bd", "delta", 0.25]
  }
}
```

Para que Strudel envíe datos al bridge sin dejar de sonar, se usó la siguiente
estructura en el REPL:
```javascript
const pat = stack(
  // patrón completo
).cpm(124.5/4).room(0.3)

$: pat
$: pat.osc()
```

### Open Stage Control (OSC)
El controlador envía mensajes OSC con las siguientes direcciones:

| Dirección | Efecto |
|-----------|--------|
| `/rgb_1` | Color de las figuras (3 args: R, G, B) |
| `/size` | Escala del tamaño (0 a 1) |
| `/bg_alpha` | Transparencia del fondo (0 a 1) |
| `/enabled` | Activa o desactiva el dibujo (0 o 1) |

El bridge los reenvía como:
```json
{ "type": "osc", "payload": { "address": "/rgb_1", "args": [255, 0, 128] } }
```

---

### Arquitectura del sistema
<img width="482" height="381" alt="image" src="https://github.com/user-attachments/assets/3dc3da74-e520-4b0f-acf5-9dbc9053b06b" />

---

### Decisión de diseño central

La decisión más importante del sistema es que **las tres fuentes se cruzan en un único
evento visual**. Cuando Strudel dispara un beat, la posición donde ocurre la explosión
la determina el micro:bit en ese instante, y el color lo está modulando OSC en tiempo
real. No son tres sistemas paralelos sino uno solo donde cada fuente tiene un rol:

- **Strudel** → dispara el evento (cuándo)
- **micro:bit** → define la posición (dónde)
- **OSC** → moldea el color y tamaño (cómo se ve)

---

### Pruebas técnicas de integración


#### Prueba 1 — Adapter de MicrobitV2
No funcionó. Tuvimos que usar el binary.

**Solución:** se reemplazó el código MicroPython por la versión que genera frames
binarios con header `0xAA` y checksum.

---

### Errores encontrados y soluciones

#### Error 1 — Strudel sonaba pero no enviaba datos al bridge
**Causa:** el patrón de Strudel no tenía `.osc()` al final.

**Solución:** se agregó `.osc()` pero esto cortó el audio porque redirigía la salida.
Se resolvió separando el patrón en dos streams:
```javascript
const pat = stack(...).cpm(124.5/4).room(0.3)
$: pat         // reproduce audio
$: pat.osc()   // envía datos al bridge
```

#### Error 2 — Adapter incorrecto para el micro:bit
**Causa:** el modo `strudel-osc-microbit` seleccionaba `MicrobitBinaryAdapter`
automáticamente, pero el código MicroPython flasheado enviaba datos en formato
ASCII (texto), incompatible con el adapter binario.

**Solución:** se reescribió el código MicroPython para generar frames binarios
de 8 bytes con header `0xAA`, campos X e Y como Int16 big-endian y checksum.

#### Error 3 — Lag visible en el cursor del micro:bit
**Causa:** el micro:bit envía datos a ~30Hz. Se había implementado un cursor visual
(mira tipo videojuego) con dos anillos concéntricos pulsantes y una cruz de mira
que seguía la posición del micro:bit en tiempo real. Al ser 30Hz, el movimiento
del cursor se veía frame a frame, con un lag visual evidente que no correspondía
a la estética de la obra.

**Solución:** se eliminó el cursor visual completamente. El micro:bit sigue
controlando la posición de las explosiones, pero sin representación visual del
puntero. Esto hace que el lag sea imperceptible porque el espectador no tiene
un elemento de referencia que lo evidencie, y la posición solo se revela en el
momento en que ocurre la explosión.

#### Error 4 — Tres sketches separados sin integración
**Causa:** el proyecto originalmente tenía tres sketches independientes
(`sketch_strudel.js`, `sketch_microbit.js`, `sketch_strudel+osc.js`) donde cada
uno escuchaba solo su propia fuente. Esto no cumplía el requisito de la rúbrica
de tener al menos una decisión que dependiera de las tres fuentes combinadas.

**Solución:** se fusionaron los tres sketches en un único `sketch_unificado.js`
donde el `updateLogic` maneja los tres tipos de mensaje y las tres fuentes
participan en cada evento visual.

### Concepto de la obra

La obra es una performance audiovisual en vivo donde el espacio visual se comporta
como un universo en expansión. Cada beat de la música detona una explosión cósmica
en el punto del espacio que el performer señala con su cuerpo. El universo reacciona
al ritmo, obedece al movimiento, y su paleta de color y densidad es moldeada en
tiempo real por un segundo performer desde un panel de control.

La metáfora central es la de un director de orquesta cósmica: uno dirige el ritmo
y la textura sonora desde Strudel, otro moldea el ambiente visual desde Open Stage
Control, y el cuerpo del performer con el micro:bit decide dónde ocurre la acción.

---

### Rol de cada fuente

**Strudel** es el disparador. Cada evento musical — kick, snare, hi-hat, otros — genera
una explosión visual distinta. El ritmo es el pulso del universo. Sin Strudel no hay
eventos, la pantalla solo muestra estrellas titilando en silencio.

**micro:bit** es el cuerpo. La inclinación del dispositivo en el espacio físico determina
el punto exacto donde nace cada explosión. El performer sostiene el micro:bit con la mano
y al inclinarlo apunta hacia una zona del universo. Cuando llega el próximo beat, la
explosión nace ahí. El cuerpo del performer es literalmente parte del sistema visual.

**Open Stage Control** es el ambiente. Desde un panel táctil, el segundo performer
controla en tiempo real el color de todas las figuras, su tamaño, la transparencia
del fondo (que determina qué tanto persiste el rastro de las explosiones anteriores),
y puede silenciar visualmente el sistema completo con un toggle.

---

### Decisiones visuales

Se eligió una estética cósmica sobre fondo negro por varias razones. Primero, la
música trabajada tiene una energía expansiva y repetitiva que evoca el pulso del
universo. Segundo, el negro permite que cada explosión sea completamente protagonista
sin competir con un fondo complejo. Tercero, las estrellas titilando en el fondo dan
sensación de profundidad y de que el espacio existe incluso cuando no hay beats.

Cada tipo de drum tiene su propia forma visual con lógica interna:

- **Kick (bd):** nova con cuatro anillos concéntricos que se expanden desde el punto
  de impacto y un núcleo blanco brillante que se desvanece. Es la explosión más grande
  y dramática, correspondiente al elemento rítmico más grave y contundente.

- **Snare (sd):** dieciocho partículas que se dispersan radialmente como debris estelar,
  acompañadas de una onda de choque circular. Evoca la dispersión de materia tras
  una colisión.

- **Hi-hat (hh):** destello estelar en forma de cruz de ocho puntas que aparece y
  desaparece rápidamente. Pequeño, preciso, como el parpadeo de una estrella lejana.

- **Otros sonidos:** nebulosa giratoria con tres capas que rotan en direcciones opuestas,
  evocando la formación de estructuras estelares complejas.

Los colores base son naranja para el kick, azul celeste para el snare y amarillo para
el hi-hat. OSC puede sobreescribir estos colores mezclándolos con el color del panel,
lo que permite transiciones de paleta durante la performance.

---

### Decisiones musicales

La pieza musical es una reinterpretación de Pump Up The Jam de Technotronic, trabajada
en Strudel con síntesis propia usando osciladores `z_sawtooth` y `z_square`. Se trabaja
en C menor a 124.5 BPM. La estructura tiene capas que entran progresivamente usando
`pickRestart`, lo que genera una sensación de construcción gradual que se refleja
también en lo visual: al principio hay pocas explosiones, luego el canvas se llena.

Para que Strudel enviara datos al bridge sin perder el audio, fue necesario separar
el patrón en dos streams paralelos, uno para el audio y otro para el OSC. Esto fue
una decisión técnica que también tuvo impacto performático: permite que el audio y
los visuales puedan desincronizarse intencionalmente si se modifica uno de los dos
streams durante la performance.

---

### Decisiones performáticas

La performance tiene dos roles claramente diferenciados. El primer performer maneja
el micro:bit con una mano y con la otra controla Strudel desde el teclado, pudiendo
modificar el patrón en vivo. El segundo performer opera el panel de Open Stage Control
desde una tablet o segundo monitor.

La duración objetivo es de 3 a 4 minutos, estructurada en tres momentos: apertura
con pocas capas y explosiones dispersas, desarrollo con todas las capas activas y
el canvas acumulando rastros, y cierre donde OSC sube el bgAlpha para borrar el
rastro progresivamente hasta dejar solo las estrellas.

---

### Cambios entre iteración ingenieril e iteración estética

En la iteración ingenieril el sistema tenía tres sketches separados que no se
comunicaban entre sí. Cada sketch escuchaba solo su propia fuente y dibujaba
figuras simples: círculos sólidos para el kick, barras rectangulares para el snare,
cuadrados pequeños para el hi-hat. El micro:bit dibujaba líneas en una pantalla
blanca, completamente desconectado de los otros dos sistemas.

En la iteración estética se tomaron las siguientes decisiones de cambio:

Se fusionaron los tres sketches en uno solo donde las tres fuentes participan en
cada evento visual. Esto no fue solo una decisión técnica sino conceptual: la obra
tiene sentido solo cuando las tres fuentes están presentes al mismo tiempo.

Se reemplazaron las figuras geométricas simples por animaciones cósmicas con
múltiples capas, transparencias y movimiento. El objetivo era que cada explosión
se sintiera como un evento físico real, no como un elemento de interfaz.

Se agregó un campo de 200 estrellas titilando en el fondo que existe
independientemente de los beats, dando vida al espacio incluso en los silencios.

Se implementó y luego se eliminó un cursor visual tipo mira de videojuego para el
micro:bit. Inicialmente se dibujaron dos anillos concéntricos pulsantes con una
cruz de mira que seguían la posición del micro:bit en tiempo real. El problema fue
que al operar a 30Hz el movimiento del cursor era visiblemente laggeado, frame a
frame, lo que rompía la estética fluida del resto de la obra. Se decidió eliminar
el cursor completamente: el micro:bit sigue controlando la posición pero sin
representación visual del puntero. El lag desaparece porque el espectador no tiene
un elemento de referencia que lo evidencie, y la posición del performer solo se
revela en el momento exacto en que ocurre la explosión.

---

### Evidencias de ensayo

Durante los ensayos se probaron las siguientes configuraciones:

Se probó el sistema completo con SimAdapter en lugar del micro:bit físico para
verificar la integración Strudel + OSC antes de tener el hardware disponible.
Las figuras aparecían en posiciones aleatorias, lo que permitió validar que el
pipeline de animaciones funcionaba correctamente.

Se probó el micro:bit físico con el código MicroPython incorrecto (formato ASCII
en lugar de binario) y se observó que el cursor no aparecía y no había respuesta
al movimiento. Esto llevó a identificar que el adapter esperaba frames binarios
de 8 bytes con header 0xAA, lo que derivó en reescribir el código MicroPython.

Se probó el cursor visual de la mira durante un ensayo completo y se observó el
lag frame a frame al mover el micro:bit con naturalidad. La decisión de eliminarlo
se tomó en ese ensayo al comparar la experiencia con y sin el cursor: sin él la
obra se sentía más limpia y el lag era imperceptible.

Se probó la estructura del patrón de Strudel con `.osc()` directo al final del
stack, lo que cortó el audio. Se resolvió durante el ensayo separando el patrón
en dos streams y se verificó que audio y visuales corrían simultáneamente sin
interferirse.

### código de micropython usado: 

```.py
from microbit import *
import struct

uart.init(baudrate=115200)

while True:
    x = accelerometer.get_x()
    y = accelerometer.get_y()
    a = 1 if button_a.is_pressed() else 0
    b = 1 if button_b.is_pressed() else 0
    
    # Empacar X e Y como Int16 big-endian
    xy = struct.pack('>hh', x, y)
    
    # Checksum: suma de bytes 1 a 6 % 256
    chk = (xy[0] + xy[1] + xy[2] + xy[3] + a + b) & 0xFF
    
    # Frame completo: header + X + Y + A + B + CHK
    frame = bytes([0xAA]) + xy + bytes([a, b, chk])
    
    uart.write(frame)
    sleep(33)
```

### SKetch completo:

```.py
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
};

class UnifiedTask extends FSMTask {
    constructor() {
        super();

        this.eventQueue = [];
        this.activeAnimations = [];
        this.latencyCorrection = 0;
        this.stars = [];

        this.microbit = {
            x: 0,
            y: 0,
            ready: false
        };

        this.controls = {
            bgAlpha: 18,
            minSize: 40,
            maxSize: 500,
            sizeScale: 1,
            enabled: true,
            color: [180, 120, 255],
            useColorOverride: false,
        };

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Esperando conexión...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            background(0);
            console.log("Sistema listo");
            this.eventQueue = [];
            this.activeAnimations = [];
            // Generar campo de estrellas de fondo
            this.stars = Array.from({ length: 200 }, () => ({
                x: random(width),
                y: random(height),
                r: random(0.5, 2.5),
                bright: random(80, 200),
                twinkleSpeed: random(0.01, 0.04),
                twinkleOffset: random(TWO_PI),
            }));
        } else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        } else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        } else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(msg) {
        if (!msg) return;
        if (msg.type === "strudel") { this._handleStrudel(msg); return; }
        if (msg.type === "osc")     { this._handleOsc(msg);     return; }
        if (msg.type === "microbit"){ this._handleMicrobit(msg); return; }
    }

    _handleMicrobit(msg) {
        this.microbit.x = map(msg.x, -2048, 2047, width * 0.1, width * 0.9);
        this.microbit.y = map(msg.y, -2048, 2047, height * 0.1, height * 0.9);
        this.microbit.ready = true;
    }

    _handleStrudel(msg) {
        const payload = msg.payload || {};
        const args = payload.args || msg.args;
        if (!Array.isArray(args)) return;

        const params = {};
        for (let i = 0; i < args.length; i += 2) params[args[i]] = args[i + 1];

        const timestamp = Number(msg.timestamp ?? payload.timestamp ?? Date.now());
        const sound = params.s || "unknown";
        const delta = params.delta ?? 0.25;

        const x = this.microbit.ready ? this.microbit.x : random(width * 0.2, width * 0.8);
        const y = this.microbit.ready ? this.microbit.y : random(height * 0.2, height * 0.8);

        this.eventQueue.push({ timestamp, sound, delta, params, x, y });
        this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
    }

    _handleOsc(msg) {
        const payload = msg.payload || {};
        const address = payload.address || "";
        const args = Array.isArray(payload.args) ? payload.args : [];

        if (address === "/rgb_1" && args.length >= 3) {
            this.controls.color = [args[0], args[1], args[2]];
            this.controls.useColorOverride = true;
            return;
        }
        if (address === "/size") {
            const v = Number(args[0]);
            if (Number.isFinite(v)) this.controls.sizeScale = v <= 1 ? lerp(0.2, 3, v) : Math.max(0.2, v);
            return;
        }
        if (address === "/bg_alpha") {
            const v = Number(args[0]);
            if (Number.isFinite(v)) this.controls.bgAlpha = v <= 1 ? Math.round(lerp(0, 255, v)) : Math.max(0, Math.min(255, v));
            return;
        }
        if (address === "/enabled") {
            this.controls.enabled = !!Number(args[0]);
        }
    }

    drawRunning() {
        background(0, this.controls.bgAlpha);

        // Estrellas de fondo titilando
        noStroke();
        for (let s of this.stars) {
            const b = s.bright + sin(frameCount * s.twinkleSpeed + s.twinkleOffset) * 40;
            fill(255, 255, 255, b);
            ellipse(s.x, s.y, s.r * 2);
        }

        // Cursor estelar — posición actual del micro:bit
        if (this.microbit.ready) {
            const cx = this.microbit.x;
            const cy = this.microbit.y;
            const t = frameCount * 0.04;

            push();
            translate(cx, cy);

            // Anillo exterior pulsante
            noFill();
            stroke(255, 255, 255, 60 + sin(t) * 40);
            strokeWeight(1);
            ellipse(0, 0, 80 + sin(t) * 10, 80 + sin(t) * 10);

            // Anillo interior
            stroke(255, 255, 255, 120 + sin(t * 1.5) * 50);
            strokeWeight(1.5);
            ellipse(0, 0, 40 + sin(t * 1.5) * 5, 40 + sin(t * 1.5) * 5);

            // Cruz de mira
            stroke(255, 255, 255, 180);
            strokeWeight(1);
            line(-25, 0, -10, 0);
            line(10, 0, 25, 0);
            line(0, -25, 0, -10);
            line(0, 10, 0, 25);

            // Punto central brillante
            noStroke();
            fill(255, 255, 255, 200 + sin(t * 2) * 55);
            ellipse(0, 0, 5, 5);

            pop();
        }

        const now = Date.now() + this.latencyCorrection;

        while (this.eventQueue.length > 0 && now >= this.eventQueue[0].timestamp) {
            const ev = this.eventQueue.shift();
            this.activeAnimations.push({
                startTime: ev.timestamp,
                duration: Math.max(1, ev.delta * 1000),
                type: ev.sound,
                x: ev.x,
                y: ev.y,
                color: this.getColorForSound(ev.sound),
                seed: random(1000),
            });
        }

        if (!this.controls.enabled) return;

        for (let i = this.activeAnimations.length - 1; i >= 0; i--) {
            const anim = this.activeAnimations[i];
            const elapsed = now - anim.startTime;
            const progress = elapsed / anim.duration;

            if (progress <= 1) {
                this.drawElement(anim, progress);
            } else {
                this.activeAnimations.splice(i, 1);
            }
        }
    }

    drawElement(anim, p) {
        const kind = this.getSoundKind(anim.type);
        const base = anim.color;
        const c = this.controls.useColorOverride
            ? this._blend(base, this.controls.color, 0.6)
            : base;
        const ss = this.controls.sizeScale;

        push();
        translate(anim.x, anim.y);

        if (kind === "bd") {
            // Kick: explosión de nova — anillos que se expanden y desvanecen
            const maxR = lerp(80, 340, p) * ss;
            for (let ring = 0; ring < 4; ring++) {
                const rp = constrain(p - ring * 0.08, 0, 1);
                const r = lerp(10, maxR, rp);
                const alpha = lerp(220, 0, rp);
                noFill();
                stroke(c[0], c[1], c[2], alpha);
                strokeWeight(lerp(4, 0.5, rp));
                ellipse(0, 0, r * 2, r * 2);
            }
            // Núcleo brillante
            const coreSize = lerp(60, 0, p) * ss;
            noStroke();
            fill(255, 255, 255, lerp(255, 0, p));
            ellipse(0, 0, coreSize, coreSize);
            fill(c[0], c[1], c[2], lerp(180, 0, p));
            ellipse(0, 0, coreSize * 1.6, coreSize * 1.6);

        } else if (kind === "sd") {
            // Snare: dispersión de partículas como debris estelar
            randomSeed(anim.seed * 100);
            const count = 18;
            for (let i = 0; i < count; i++) {
                const angle = (TWO_PI / count) * i + random(-0.3, 0.3);
                const dist = lerp(0, random(80, 200), p) * ss;
                const px = cos(angle) * dist;
                const py = sin(angle) * dist;
                const sz = lerp(random(4, 10), 0, p) * ss;
                const alpha = lerp(255, 0, p * p);
                noStroke();
                fill(c[0], c[1], c[2], alpha);
                ellipse(px, py, sz, sz);
            }
            // Línea de choque
            noFill();
            stroke(c[0], c[1], c[2], lerp(180, 0, p));
            strokeWeight(1.5);
            const shockR = lerp(20, 160, p) * ss;
            arc(0, 0, shockR * 2, shockR * 2, 0, TWO_PI);

        } else if (kind === "hh") {
            // Hi-hat: destello estelar — cruz con rayos cortos
            const len = lerp(40, 5, p) * ss;
            const alpha = lerp(255, 0, p);
            stroke(c[0], c[1], c[2], alpha);
            strokeWeight(lerp(2.5, 0.5, p));
            // Cruz principal
            line(-len, 0, len, 0);
            line(0, -len, 0, len);
            // Diagonal
            const diagLen = len * 0.6;
            line(-diagLen, -diagLen, diagLen, diagLen);
            line(diagLen, -diagLen, -diagLen, diagLen);
            // Punto central
            noStroke();
            fill(255, 255, 255, alpha);
            ellipse(0, 0, lerp(8, 1, p), lerp(8, 1, p));

        } else {
            // Otros: nebulosa giratoria con capas
            randomSeed(anim.seed * 77);
            const layers = 3;
            for (let l = 0; l < layers; l++) {
                const angle = p * TWO_PI * (l % 2 === 0 ? 1 : -1) + (l * TWO_PI / layers);
                const r = lerp(20, 120, p) * ss * (0.6 + l * 0.3);
                const alpha = lerp(160, 0, p) / layers;
                noFill();
                stroke(c[0], c[1], c[2], alpha * 2);
                strokeWeight(lerp(3, 0.5, p));
                push();
                rotate(angle);
                arc(0, 0, r * 2, r * 1.2, 0, PI + HALF_PI);
                pop();
            }
        }

        pop();
    }

    _blend(a, b, t) {
        return [
            Math.round(lerp(a[0], b[0], t)),
            Math.round(lerp(a[1], b[1], t)),
            Math.round(lerp(a[2], b[2], t)),
        ];
    }

    getSoundKind(s) {
        const name = String(s || "").toLowerCase();
        if (name.includes("bd")) return "bd";
        if (name.includes("sd") || name.includes("cp")) return "sd";
        if (name.includes("hh") || name.includes("oh")) return "hh";
        return "other";
    }

    getColorForSound(s) {
        const kind = this.getSoundKind(s);
        if (kind === "bd") return [255, 140, 50];   // naranja supernova
        if (kind === "sd") return [100, 200, 255];  // azul celeste
        if (kind === "hh") return [255, 255, 180];  // amarillo estrella
        const cc = String(s || "?").charCodeAt(0) || 0;
        return [(cc * 123) % 255, (cc * 456) % 255, (cc * 789) % 255];
    }
}

let task;
let bridge;
let connectBtn;

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(0);

    task = new UnifiedTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        task.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        task.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((msg) => {
        task.postEvent({ type: EVENTS.DATA, payload: msg });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });
}

function draw() {
    task.update();
    if (task.state === task.estado_corriendo) {
        task.drawRunning();
    }
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```

### Diagrama final de las interacciones: 
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/e61a9b0c-eb02-45ae-b5ed-49851ce95211" />


## Bitácora de reflexión

(esperando la exposición final).
