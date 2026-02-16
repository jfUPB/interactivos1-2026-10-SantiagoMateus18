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

## Bitácora de aplicación 



## Bitácora de reflexión
