# Unidad 2

## Bitácora de proceso de aprendizaje

### Actividad 01

#### Cuáles son los estados en el programa?
1. estado_waitTimeout: único estado donde el pixel espera a que ocurra el evento de tiempo (Timeout) para alternar entre encendido y apagado.
2. estado_waitInON: estado en el que el pixel está encendido (valor 9 en el display).
3. estado_waitInOFF: estado en el que el pixel está apagado (valor 0 en el display).

#### ¿Cuáles son los eventos en el programa?
1. ENTRY: Evento especial que ocurre al entrar en un estado (inicialización del estado).
2. EXIT: Evento especial que ocurre al salir de un estado (aunque en este código no se usa mucho).
3. Timeout: Evento generado por el temporizador (Timer) cuando expira el intervalo definido.

#### ¿Cuáles son las acciones en el programa?
1. Encender pixel: display.set_pixel(x, y, 9) → brillo máximo.
2. Apagar pixel: display.set_pixel(x, y, 0) → pixel apagado.
3. Iniciar temporizador: self.myTimer.start() → arranca el conteo para el próximo Timeout.
4. Alternar estado del pixel: cambiar pixelState entre 0 y 9.
5. Procesar eventos en la cola: self.event_queue.pop(0) dentro de update().

### Actividad 02

#### Código del semáforo funcional: 

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
    def  __init__(self,_x,_y):
        self.redX= _x
        self.redY= _y
        self.event_queue = []
        self.timers = []

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

    def estado_WaitRed(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY,9)
            self.myTimer.start(2000)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY,0)
           self.transicion_a(self.estado_WaitGreen) 
            
    def estado_WaitGreen(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+2,9)
            self.myTimer.start(1000)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY+2,0)
           self.transicion_a(self.estado_WaitYellow) 
       

    def estado_WaitYellow(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+1,9)
            self.myTimer.start(700)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY+1,0)
           self.transicion_a(self.estado_WaitRed) 
        

semaforo1= Semaforo (0,0)

while True:
    semaforo1.update()
    utime.sleep_ms(20)
```
#### Código del semáforo con la edición de (button "A) para cambiar a amarillo:
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
    def  __init__(self,_x,_y):
        self.redX= _x
        self.redY= _y
        self.event_queue = []
        self.timers = []

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

    def estado_WaitRed(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY,9)
            self.myTimer.start(2000)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY,0)
           self.transicion_a(self.estado_WaitGreen) 
            
    def estado_WaitGreen(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+2,9)
            self.myTimer.start(1000)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY+2,0)
           self.transicion_a(self.estado_WaitYellow) 
        if ev == "A":
            self.myTimer.stop()
            self.transicion_a(self.estado_WaitYellow)
            display.set_pixel(self.redX,self.redY+2,0) 

    def estado_WaitYellow(self, ev):
        if ev == "ENTRY":
            display.set_pixel(self.redX,self.redY+1,9)
            self.myTimer.start(700)
        if ev == "Timeout":
           display.set_pixel(self.redX,self.redY+1,0)
           self.transicion_a(self.estado_WaitRed) 
        

semaforo1= Semaforo (0,0)

while True:
    semaforo1.update()
    if button_a.is_pressed():
        semaforo1.post_event("A")
    utime.sleep_ms(20)
```

### Actividad 04

#### Diagrama UML:
<img width="389" height="617" alt="image" src="https://github.com/user-attachments/assets/ce4a4b25-12f1-4c01-a4d3-e0ac1642a054" />

#### Código del Micro:bit

```.h
from microbit import *
import utime
import music

def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()
# Para mostrar usas display.show(FILL[n]) donde n será
# un valor de 0 a 25

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

class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        # Personalizas el nombre del evento y la duración
        self.myTimer = self.createTimer("Timeout",1000)
        self.count_pixels=20
        self.remaining=0
        self.estado_actual = None
        self.transicion_a(self.estado_estado1)

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


    def estado_estado1(self, ev):
        if ev == "ENTRY":
            self.count_pixels=20
            display.show(FILL[self.count_pixels])
        if ev == "A":
            if self.count_pixels < 25:
                self.count_pixels = self.count_pixels +1
                display.show(FILL[self.count_pixels])
        if ev == "B":
            if self.count_pixels > 15:
                self.count_pixels = self.count_pixels -1
                display.show(FILL[self.count_pixels])
        if ev == "S":
            self.transicion_a(self.estado_estado2)
            
    def estado_estado2(self, ev):
        if ev == "ENTRY":
            self.remaining = self.count_pixels
            display.show(FILL[self.remaining])
            self.myTimer.start(1000)
        if ev == "Timeout":
            self.remaining= self.remaining-1
            if self.remaining >0:
                display.show(FILL[self.remaining])
                self.myTimer.start(1000)
            else:
                display.show(Image.SKULL)
                music.play(music.BA_DING)
        if ev == "A":
            self.transicion_a(self.estado_estado1)
                
task = Task()

while True:
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")
    task.update()
    utime.sleep_ms(20)
```

## Bitácora de aplicación 



## Bitácora de reflexión

