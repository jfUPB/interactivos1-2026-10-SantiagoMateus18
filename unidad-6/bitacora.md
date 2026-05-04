# Unidad 6

## Bitácora de proceso de aprendizaje

### ¿Cuál es la diferencia entre recibir un mensaje y ejecutarlo?
Recibir es guardar el mensaje en la cola. Ejecutar es activarlo cuando el reloj llega a su timestamp. Puedes recibir ahora y ejecutar en 500ms.

### ¿Por qué un sistema audiovisual puede necesitar timestamp además de los datos del evento?
Porque el audio tiene sincronización precisa — si ejecutas la animación apenas llega el mensaje, la latencia de red y del navegador la desincroniza del sonido. El timestamp dice exactamente cuándo debe ocurrir.

### ¿Qué aspectos de la arquitectura de las unidades 4 y 5 permanecen intactos aunque ahora la fuente de datos ya no sea hardware?
Todo lo estructural: el Adapter normaliza, el BridgeServer transporta, el BridgeClient recibe, la FSM organiza el estado, y el draw solo dibuja. Solo cambia la fuente de datos — antes era hardware serial, ahora es una app web.

## Bitácora de aplicación 
### sketch.js
con mejoras visuales y cambios en tiempo real. 
```.js
const EVENTS = {
      CONNECT: "CONNECT",
      DISCONNECT: "DISCONNECT",
      DATA: "DATA",
      STRUDEL_DATA: "STRUDEL_DATA",
      FRAME_TICK: "FRAME_TICK",
      KEY_PRESSED: "KEY_PRESSED",
      KEY_RELEASED: "KEY_RELEASED",
};


class PainterTask extends FSMTask {
      constructor() {
            super();

            this.rxData = {
                  x: 0,
                  y: 0,
                  btnA: false,
                  btnB: false,
                  ready: false
            };

            this.circleResolution = 5;
            this.radius = 0;
            this.fillEnabled = false;

            this.eventQueue = [];
            this.activeVisuals = [];
            this.queueLookaheadMs = 16;

            this.transitionTo(this.estado_esperando);
      }

      estado_esperando = (ev) => {
            if (ev.type === "ENTRY") {
                  cursor();
                  console.log("Waiting for connection...");
            } else if (ev.type === EVENTS.CONNECT) {
                  this.transitionTo(this.estado_corriendo);
            }
      };

      estado_corriendo = (ev) => {
            if (ev.type === "ENTRY") {
                  noCursor();
                  strokeWeight(2);
                  stroke(0, 25);
                  background(0);
                  console.log("Microbit ready to draw");
                  this.rxData = {
                        x: 0,
                        y: 0,
                        btnA: false,
                        btnB: false,
                        ready: false
                  };
                  this.circleResolution = 5;
                  this.radius = 0;
                  this.fillEnabled = false;
                  this.eventQueue = [];
                  this.activeVisuals = [];
            }

            else if (ev.type === EVENTS.DISCONNECT) {
                  this.transitionTo(this.estado_esperando);
            }

            else if (ev.type === EVENTS.DATA) {
                  this.updateLogic({ kind: "microbit", ...ev.payload });
            }

            else if (ev.type === EVENTS.STRUDEL_DATA) {
                  this.updateLogic({ kind: "strudel", ...ev.payload });
            }

            else if (ev.type === EVENTS.FRAME_TICK) {
                  this.advanceTimeline(ev.now);
            }

            else if (ev.type === EVENTS.KEY_PRESSED) {
                  this.handleKeys(ev.keyCode, ev.key);
            }

            else if (ev.type === EVENTS.KEY_RELEASED) {
                  this.handleKeyRelease(ev.keyCode, ev.key);
            }

            else if (ev.type === "EXIT") {
                  cursor();
            }
      };

      updateLogic(data) {
            if (data.kind === "strudel") {
                  const normalized = {
                        timestamp: Number(data.timestamp),
                        payload: data.payload || {},
                  };

                  if (!Number.isFinite(normalized.timestamp)) return;
                  this.eventQueue.push(normalized);
                  this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
                  return;
            }

            this.rxData.ready = true;
            this.rxData.x = data.x;
            this.rxData.y = data.y;
            this.rxData.btnA = data.btnA;
            this.rxData.btnB = data.btnB;

            this.circleResolution = int(map(this.rxData.y, -2048, 2047, 2, 10));
            this.radius = map(this.rxData.x, -2048, 2047, -width / 2, width / 2);
            this.fillEnabled = this.rxData.btnB;
      }
      advanceTimeline(nowMs) {
            const now = Number(nowMs);
            if (!Number.isFinite(now)) return;

            while (this.eventQueue.length > 0 && this.eventQueue[0].timestamp <= now + this.queueLookaheadMs) {
                  const ev = this.eventQueue.shift();
                  this.activateVisual(ev, now);
            }

            this.activeVisuals = this.activeVisuals.filter((v) => now <= v.endAt);
      }

      activateVisual(ev, now) {
            const payload = ev.payload || {};
            const soundType = String(payload.soundType || "other");
            const durationMs = Math.max(90, Math.min(1400, Number(payload.delta || 0.25) * 1000));

            const profileBySound = {
                  bd: { color: [128, 0, 32], baseSize: 160 },
                  sdcp: { color: [243, 130, 15], baseSize: 120 },
                  hh: { color: [27, 140, 151], baseSize: 84 },
                  other: { color: [45, 45, 45], baseSize: 100 },
            };

            const profile = profileBySound[soundType] || profileBySound.other;
            this.activeVisuals.push({
                  soundType,
                  startAt: now,
                  endAt: now + durationMs,
                  durationMs,
                  color: profile.color,
                  baseSize: profile.baseSize,
            });
      }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();
const simButtons = { btnA: false, btnB: false };
const simAxes = { x: 0, y: 0 };
const AXIS_STEP = 256;
const AXIS_MIN = -2048;
const AXIS_MAX = 2047;

function pushSimButtons() {
      bridge?.send({ cmd: "setSimButtons", btnA: simButtons.btnA, btnB: simButtons.btnB });
}

function clampAxis(value) {
      return Math.max(AXIS_MIN, Math.min(AXIS_MAX, value));
}

function pushSimAxes() {
      bridge?.send({ cmd: "setSimAxes", x: simAxes.x, y: simAxes.y });
}

function setup() {
      createCanvas(windowWidth, windowHeight);
      background(255);
      painter = new PainterTask();
      bridge = new BridgeClient();

      bridge.onConnect(() => {
            connectBtn.html("Disconnect");
            painter.postEvent({ type: EVENTS.CONNECT });
      });

      bridge.onDisconnect(() => {
            connectBtn.html("Connect");
            painter.postEvent({ type: EVENTS.DISCONNECT });
      });

      bridge.onStatus((s) => {
            console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
      });

      bridge.onData((data) => {
            painter.postEvent({
                  type: EVENTS.DATA, payload: {
                        x: data.x,
                        y: data.y,
                        btnA: data.btnA,
                        btnB: data.btnB
                  }
            });
            console.log(data);
      });
      bridge.onStrudel((evt) => {
            painter.postEvent({
                  type: EVENTS.STRUDEL_DATA,
                  payload: {
                        timestamp: evt.timestamp,
                        payload: evt.payload,
                  }
            });
      });

      connectBtn = createButton("Connect");
      connectBtn.position(10, 10);
      connectBtn.mousePressed(() => {
            if (bridge.isOpen) bridge.close();
            else bridge.open();
      });

      renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
      painter.postEvent({ type: EVENTS.FRAME_TICK, now: Date.now() });
      painter.update();
      renderer.get(painter.state)?.();
}

function drawRunning() {
      let mb = painter.rxData;

      // Renderiza visuales musicales ya programados por timestamp.
      for (const v of painter.activeVisuals) {
            const elapsed = Date.now() - v.startAt;
            const progress = constrain(elapsed / v.durationMs, 0, 1);
            const alpha = 1 - progress;
            const size = v.baseSize + progress * 180;

            push();
            noStroke();
            fill(v.color[0], v.color[1], v.color[2], 180 * alpha);
            translate(width / 2, height / 2);

            if (v.soundType === "bd") {
                  rectMode(CENTER);
                  rect(0, 0, size, size * 0.6, 12);
            } else if (v.soundType === "sdcp") {
                  let offsetY = lerp(-height / 4, height / 4, progress);
                  rectMode(CENTER);
                  rect(0, offsetY, size, size * 0.3, 8);
            } else if (v.soundType === "hh") {
                  noFill();
                  stroke(v.color[0], v.color[1], v.color[2], 210 * alpha);
                  strokeWeight(2 + (1 - progress) * 5);
                  circle(0, 0, size * 0.9);
            } else {
                  rectMode(CENTER);
                  square(0, 0, size * 0.4);
            }
            pop();
      }

      if (!mb.ready) return;

      if (mb.btnA) {
            push();
            translate(width / 2, height / 2);

            const angle = TAU / painter.circleResolution;

            if (painter.fillEnabled) {
                  fill(34, 45, 122, 50);
            } else {
                  noFill();
            }

            beginShape();
            for (let i = 0; i <= painter.circleResolution; i++) {
                  const x = cos(angle * i) * painter.radius;
                  const y = sin(angle * i) * painter.radius;
                  vertex(x, y);
            }
            endShape();

            pop();
      }
}

function windowResized() {
      resizeCanvas(windowWidth, windowHeight);
}

function keyPressed() {
      const k = String(key || "").toLowerCase();
      let changed = false;
      let axisChanged = false;

      if (k === "a" && !simButtons.btnA) {
            simButtons.btnA = true;
            changed = true;
      }

      if (k === "b" && !simButtons.btnB) {
            simButtons.btnB = true;
            changed = true;
      }

      if (keyCode === LEFT_ARROW) {
            simAxes.x = clampAxis(simAxes.x - AXIS_STEP);
            axisChanged = true;
      }

      if (keyCode === RIGHT_ARROW) {
            simAxes.x = clampAxis(simAxes.x + AXIS_STEP);
            axisChanged = true;
      }

      if (keyCode === UP_ARROW) {
            simAxes.y = clampAxis(simAxes.y - AXIS_STEP);
            axisChanged = true;
      }

      if (keyCode === DOWN_ARROW) {
            simAxes.y = clampAxis(simAxes.y + AXIS_STEP);
            axisChanged = true;
      }

      if (k === "r") {
            simAxes.x = 0;
            simAxes.y = 0;
            axisChanged = true;
      }

      if (changed) pushSimButtons();
      if (axisChanged) {
            pushSimAxes();
            return false;
      }
}

function keyReleased() {
      const k = String(key || "").toLowerCase();
      let changed = false;

      if (k === "a" && simButtons.btnA) {
            simButtons.btnA = false;
            changed = true;
      }

      if (k === "b" && simButtons.btnB) {
            simButtons.btnB = false;
            changed = true;
      }

      if (changed) pushSimButtons();
}
```

### BridgeServer.js 
con la implementación de los nuevos códigos: 
```.js

//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --baud 115200
//     node bridgeServer.js --device microbit-bin --wsPort 8081 --baud 115200
//     node bridgeServer.js --device strudel --wsPort 8081

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitV2Adapter = require("./adapters/MicrobitV2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");

const log = {
   info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
   warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
   error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};


function getArg(name, def = null) {
   const i = process.argv.indexOf(`--${name}`);
   if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
   return def;
}

function hasFlag(name) {
   return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
   try {
      return JSON.parse(s);

   } catch (e) {
      log.warn("Failed to parse JSON: ", s, e);
      return null;
   }
}

function broadcast(wss, obj) {
   const text = JSON.stringify(obj);
   for (const client of wss.clients) {
      if (client.readyState === 1) client.send(text);
   }
}

function status(wss, state, detail = "") {
   broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
   const ports = await SerialPort.list();
   const microbit = ports.find(p =>
      p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
   );
   return microbit?.path ?? null;
}

async function createAdapter() {
   if (DEVICE === "microbit") {
      const path = SERIAL_PATH ?? await findMicrobitPort();
      if (!path) {
         log.error("micro:bit not found. Use --serialPort to specify manually.");
         process.exit(1);
      }
      log.info(`micro:bit found at ${path}`);
      return new MicrobitV2Adapter({ path, baud: BAUD, verbose: VERBOSE });
   }

   if (DEVICE === "microbit-bin") {
      const path = SERIAL_PATH ?? await findMicrobitPort();
      if (!path) {
         log.error("micro:bit not found. Use --serialPort to specify manually.");
         process.exit(1);
      }
      return new MicrobitBinaryAdapter({ path, baud: BAUD });
   }

   if (DEVICE === "strudel") {
      return new StrudelAdapter({
         host: getArg("strudelHost", "127.0.0.1"),
         port: parseInt(getArg("strudelPort", "8080"), 10),
         verbose: VERBOSE,
      });
   }

   return new SimAdapter({ hz: SIM_HZ });
}

async function main() {
   const wss = new WebSocketServer({ port: WS_PORT });
   log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

   const adapter = await createAdapter();

   adapter.onConnected = (detail) => {
      log.info(`[ADAPTER] Device Connected: ${detail}`);
      status(wss, "connected", detail);
   };

   adapter.onDisconnected = (detail) => {
      log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
      status(wss, "disconnected", detail);
   };

   adapter.onError = (detail) => {
      log.error(`[ADAPTER] Device Error: ${detail}`);
      status(wss, "error", detail);
   };

  adapter.onData = (d) => {
  if (d?.type === "strudel") {
    broadcast(wss, d);
    return;
  }

  broadcast(wss, {
    type: "microbit",
    x: d.x,
    y: d.y,
    btnA: !!d.btnA,
    btnB: !!d.btnB,
    t: nowMs(),
  });
};

   status(wss, "ready", `bridge up (${DEVICE})`);

   wss.on("connection", (ws, req) => {
      log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

      const state = adapter.connected ? "connected" : "ready";

      const detail = adapter.connected
         ? adapter.getConnectionDetail()
         : `bridge (${DEVICE})`;

      ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

      ws.on("message", async (raw) => {
         const msg = safeJsonParse(raw.toString("utf8"));
         if (!msg) return;

         if (msg.cmd === "connect") {
            log.info(`[NETWORK] Client requested adapter connect`);

            if (adapter.connected) {
               log.info(`[HW-POLICY] Adapter already open. Sending current status to incoming client.`);
               ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
               return;
            }

            try {
               await adapter.connect();
            } catch (e) {
               const detail = `connect failed: ${e.message || e}`;
               log.error(`[ADAPTER] ` + detail);
               status(wss, "error", detail);
            }
            return;
         }

         if (msg.cmd === "disconnect") {
            log.info(`[NETWORK] Client requested adapter disconnect`);
            if (wss.clients.size > 1) {
               log.info(`[HW-POLICY] Adapater kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
               ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
               return;
            }

            try {
               await adapter.disconnect();
            } catch (e) {
               const detail = `disconnect failed: ${e.message || e}`;
               log.error(`[ADAPTER] ` + detail);
               status(wss, "error", detail);
            }
            return;
         }

         if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
            log.info(`Setting Sim Hz to ${msg.hz}`);
            await adapter.handleCommand(msg);
            status(wss, "connected", `sim hz=${adapter.hz}`);
            return;
         }

         if (msg.cmd === "setSimButtons" && adapter instanceof SimAdapter) {
            await adapter.handleCommand(msg);
            return;
         }

         if ((msg.cmd === "setSimAxes" || msg.cmd === "clearSimAxes") && adapter instanceof SimAdapter) {
            await adapter.handleCommand(msg);
            return;
         }

         if (msg.cmd === "setLed") {
            try {
               await adapter.handleCommand?.(msg);
            } catch (e) {
               const detail = `command failed: ${e.message || e}`;
               log.error(`[ADAPTER] ` + detail);
               status(wss, "error", detail);
            }
            return;
         }
      });

      ws.on("close", () => {
         log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
         if (wss.clients.size === 0) {
            log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
            adapter.disconnect();
         }
      });
   });

   if (DEVICE === "sim") {
      await adapter.connect();
   }
}

main().catch((e) => {
   log.error("Fatal:", e);
   process.exit(1);
});
```

### Strudel 
```.js
const { WebSocketServer } = require("ws");
const BaseAdapter = require("./BaseAdapter");

function toArgsMap(args) {
  const map = {};
  if (!Array.isArray(args)) return map;

  for (let i = 0; i + 1 < args.length; i += 2) {
    const k = args[i];
    const v = args[i + 1];
    if (typeof k === "string") map[k] = v;
  }

  return map;
}

function classifySound(sample) {
  const s = String(sample || "").toLowerCase();
  if (/(^|[^a-z])bd([^a-z]|$)/.test(s)) return "bd";
  if (/(^|[^a-z])(sd|cp)([^a-z]|$)/.test(s)) return "sdcp";
  if (/(^|[^a-z])hh([^a-z]|$)/.test(s)) return "hh";
  return "other";
}

class StrudelAdapter extends BaseAdapter {
  constructor({ port = 8080, host = "127.0.0.1", verbose = false } = {}) {
    super();
    this.port = port;
    this.host = host;
    this.verbose = verbose;
    this.server = null;
    this.clients = new Set();
  }

  async connect() {
    if (this.connected) return;

    await new Promise((resolve, reject) => {
      this.server = new WebSocketServer({ port: this.port, host: this.host }, () => resolve());
      this.server.once("error", reject);
    });

    this.connected = true;
    this.onConnected?.(`strudel ws://${this.host}:${this.port}`);

    this.server.on("connection", (ws) => {
      this.clients.add(ws);

      ws.on("message", (raw) => {
        const evt = this._normalize(raw);
        if (!evt) return;
        this.onData?.(evt);
      });

      ws.on("close", () => this.clients.delete(ws));
      ws.on("error", (err) => this._fail(err));
    });

    this.server.on("error", (err) => this._fail(err));
    this.server.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    for (const c of this.clients) {
      try {
        c.close();
      } catch (_e) {
        // Ignore close races.
      }
    }
    this.clients.clear();

    if (this.server) {
      await new Promise((resolve) => this.server.close(() => resolve()));
    }

    this.server = null;
    this.onDisconnected?.("strudel socket closed");
  }

  getConnectionDetail() {
    return `strudel ws://${this.host}:${this.port}`;
  }

  _normalize(raw) {
    let msg;
    try {
      msg = JSON.parse(raw.toString("utf8"));
    } catch (_e) {
      if (this.verbose) {
        console.warn("[StrudelAdapter] Ignored non-JSON message");
      }
      return null;
    }

    if (!msg || msg.address !== "/dirt/play") return null;

    const args = toArgsMap(msg.args);
    const s = String(args.s || "");
    const delta = Number(args.delta);
    const timestamp = Number(msg.timestamp);

    if (!Number.isFinite(timestamp)) return null;

    return {
      type: "strudel",
      timestamp,
      payload: {
        eventType: "noteEvent",
        address: msg.address,
        s,
        soundType: classifySound(s),
        delta: Number.isFinite(delta) && delta > 0 ? delta : 0.25,
        cycle: Number.isFinite(Number(args.cycle)) ? Number(args.cycle) : null,
        cps: Number.isFinite(Number(args.cps)) ? Number(args.cps) : null,
        bank: args.bank ?? null,
      },
    };
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.server = null;
    this.clients.clear();
    this.onDisconnected?.("strudel socket closed (event)");
  }
}

module.exports = StrudelAdapter;
```

### Paso a paso:
Creé un nuevo adapter llamado StrudelAdapter.js que en vez de conectarse por serial como el microbit, abre un servidor WebSocket en el puerto 8080 para recibir los eventos de Strudel. Este adapter normaliza los mensajes al formato del sistema y los entrega al bridge.
Luego registré ese adapter en BridgeServer.js agregando el caso strudel en createAdapter(), y modifiqué el adapter.onData para que reenvíe los mensajes de Strudel tal cual sin transformarlos.
También tuve que tocar BridgeClient.js para agregar un callback separado onStrudel que maneja msg.type === "strudel", porque sin eso el sketch no podía distinguir entre eventos del microbit y de Strudel.
Finalmente adapté el sketch.js del profe para que use la arquitectura FSM — la cola de eventos y las animaciones viven dentro de PainterTask, updateLogic recibe los datos, advanceTimeline activa los eventos cuando llega su timestamp, y drawRunning solo dibuja.
Lo que no toqué fue FSM.js, BaseAdapter.js, y la lógica general del bridge — la arquitectura de las unidades anteriores quedó intacta.

### Preguntas: 

Recibir vs ejecutar: recibir es guardar el mensaje en la cola, ejecutar es activarlo en el momento correcto según su timestamp. 

¿Por qué timestamp? Porque si ejecutas apenas llega el mensaje, la latencia de red desincroniza el visual del audio. El timestamp dice exactamente cuándo debe ocurrir. 

¿Qué permanece intacto? La arquitectura completa: adapter normaliza, bridge transporta, FSM organiza, draw solo dibuja. Solo cambió la fuente de datos.

## Bitácora de reflexión
reflexion final. 5 lineas de texto. no más9:03Claude ha respondido: Esta actividad me mostró que una buena arquitectura de software permite agregar cosas nuevas sin romper lo que ya existe.Esta actividad me mostró que una buena arquitectura de software permite agregar cosas nuevas sin romper lo que ya existe. Cambié la fuente de datos de hardware físico a una app web y el sistema siguió funcionando igual. Entendí la diferencia entre recibir un mensaje y ejecutarlo — la cola con timestamps es lo que hace que el audio y el visual estén sincronizados. También aprendí que el adapter es la pieza clave que aísla al resto del sistema de los detalles del protocolo. En general, separar bien las capas hace que el código sea más fácil de mantener y extender.
