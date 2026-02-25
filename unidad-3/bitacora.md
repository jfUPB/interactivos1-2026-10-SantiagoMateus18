# Unidad 3

## Bitácora de proceso de aprendizaje

### Actividad 01:
 1) Si el botón A se presiona el semáforo debe cambiar a modo “peatonal”. En este modo el semáforo se pone en rojo, pero antes debe pasar por amarillo para darle tiempo a los autos a detenerse.
 2) Si el botón B se presiona el semáforo pasa a modo nocturno. En este modo el semáforo parpadea en amarillo. Si el botón A se presiona en este modo, el semáforo vuelve a modo normal.

#### Código:

```.h
from microbit import *
import utime

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration

        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

class Semaforo:
    def  __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow,):
        self.redX= _x
        self.redY= _y
        self.event_queue = []
        self.timers = []
        self.timeInYellow =_timeInYellow
        self.timeInRed =_timeInRed
        self.timeInGreen =_timeInGreen
        
        self.myTimer = self.createTimer("Timeout", 2000)
        self.estado_actual = None
        self.transicion_a(self.estado_WaitRed)
        
    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def clear(self):
        display.set_pixel(self.redX,self.redY,0)
        display.set_pixel(self.redX,self.redY+1,0)
        display.set_pixel(self.redX,self.redY+2,0)

    def estado_WaitRed(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY,9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY,0)
           self.transicion_a(self.estado_WaitGreen) 
        if ev == "B":
            self.myTimer.stop()
            self.transicion_a(self.estado_WaitNocturn)

            
    def estado_WaitGreen(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+2,9)
            self.myTimer.start(self.timeInGreen)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY+2,0)
           self.transicion_a(self.estado_WaitYellow) 
        if ev == "A":
            self.myTimer.stop()
            self.transicion_a(self.estado_WaitYellow)
            display.set_pixel(self.redX,self.redY+2,0) 
        if ev == "B":
            self.myTimer.stop()
            self.transicion_a(self.estado_WaitNocturn)

    def estado_WaitYellow(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY+1,0)
           self.transicion_a(self.estado_WaitRed) 

    def estado_WaitNocturn(self,ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.redX,self.redY+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            pixelState = display.get_pixel(self.redX,self.redY+1)
            if pixelState== 9: display.set_pixel(self.redX, self.redY+1,0)
            else: display.set_pixel(self.redX,self.redY+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "A":
            self.transicion_a(self.estado_WaitRed)
            display.set_pixel(self.redX, self.redY+1,0)
        

semaforo1= Semaforo (0,0, 2000, 1000,500)

while True:
    semaforo1.update()
    if button_a.is_pressed():
        semaforo1.post_event("A")
    elif button_b.is_pressed():
        semaforo1.post_event("B")
    utime.sleep_ms(20)
```
### Actividad 02:

#### Código: 

```.h
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music

class Temporizador(FSMTask):
    def __init__(self):
        super().__init__()
        self.event_queue = []
        self.timers = []
        self.counter = 20
        self.myTimer = self.add_timer("Timeout",1000)
        self.estado_actual = None
        self.transition_to(self.estado_config)
        self.passWord = ["A","B","A"]
        self.stopSequence = []


    def estado_config(self, ev):
        if ev == ENTRY:
            self.counter = 20
            display.show(FILL[self.counter])
            self.myTimer.start()
        if ev == "A":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])
        if ev == "B":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])
        if ev == "S":
            self.transition_to(self.estado_armed)

    def estado_armed(self, ev):
        if ev == ENTRY:
            self.stopSequence.clear()
            self.myTimer.start()
        if ev == "Timeout":
            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])
                if self.counter == 0:
                    self.transition_to(self.estado_timeout)
                else:
                    self.myTimer.start()
        if ev == "S":
            self.transition_to(self.estado_paused)

        if ev == "A" or ev == "B":
            self.stopSequence.append(ev)
            if len(self.stopSequence)== 3:
                if self.stopSequence ==self.passWord:
                    self.transition_to(self.estado_config)
                else:
                    self.stopSequence.clear()
                

    def estado_paused(self,ev):
        if ev == "ENTRY":
            self.myTimer.stop()
        if ev == "S":
            self.transition_to(self.estado_armed)

        
    def estado_timeout(self, ev):
        if ev == ENTRY:
            display.show(Image.SKULL)
            music.play(music.FUNERAL)
        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)

temporizador = Temporizador()

while True:

    if button_a.was_pressed():
        temporizador.post_event("A")
    if button_b.was_pressed():
        temporizador.post_event("B")
    if accelerometer.was_gesture("shake"):
        temporizador.post_event("S")

    temporizador.update()
    utime.sleep_ms(20)
```
### Actividad 03:

Preparación inicial:  
Verifiqué que Git Bash estuviera instalado y que Visual Studio Code estuviera listo para abrir proyectos.

Clonación del repositorio:  
Ejecuté el comando git clone https://github.com/juanferfranco/sf1-2026-10-u3-a3.git en Git Bash para traer el proyecto desde GitHub.

Apertura en Visual Studio Code:  
Una vez clonada la carpeta, la abrí en Visual Studio Code.

Configuración del entorno:  
Instalé la extensión p5.vscode para poder ejecutar el proyecto.

Visualización del proyecto:  
Usé la opción Go Live para levantar un servidor local y visualizar la aplicación en el navegador.

### Actividad 04:

#### Código del sketch.js
```.h
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};

const UI = {
  dialSize: 250,
  ringWeight: 20,
  bigText: 100,
  configText: 120,
  helpText: 18,
};


class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);

  }

  get currentState() {
    return this.state;
  }

  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
    }
    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };


  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start();
    } else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      }
    } else if (ev === EXIT) {
      this.myTimer.stop();
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_pause);
    }

  };

  estado_pause = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.stop();
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_armed);
    }
    else if (ev === EVENTS.INC) {
      this.transitionTo(this.estado_pause2);
    }
  };

  estado_pause2 = (ev) => {
    if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    } else if (ev === EVENTS.INC) {
      this.transitionTo(this.estado_pause3);
    }
  };

  estado_pause3 = (ev) => {
    if (ev === EVENTS.INC) {
      this.transitionTo(this.estado_armed);
    }
  };



  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  }
}

let temporizador;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
  textAlign(CENTER, CENTER);

  renderer.set(temporizador.estado_config, () => drawConfig(temporizador.configValue));
  renderer.set(temporizador.estado_armed, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds));
  renderer.set(temporizador.estado_timeout, () => drawTimeout());
}

function draw() {
  temporizador.update();
  renderer.get(temporizador.currentState)?.();
}

function drawConfig(val) {
  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);
  textSize(18);
  fill(200);
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total) {
  background(20, 20, 20);
  let pulse = sin(frameCount * 0.1) * 10;

  noFill();
  strokeWeight(20);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, 250);

  stroke(255, 150, 0);
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100 + pulse);
  text(val, width / 2, height / 2);
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent("A");
  if (key === "b" || key === "B") temporizador.postEvent("B");
  if (key === "s" || key === "S") temporizador.postEvent("S");
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```

### Actividad 05:



## Bitácora de aplicación 



## Bitácora de reflexión


