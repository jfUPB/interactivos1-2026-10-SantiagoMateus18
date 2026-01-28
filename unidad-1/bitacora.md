# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 01
#### ¿Qué es un sistema físico interactivo?
Es un (como la palabra lo dice) sistema diseñado por una o varias personas de forma digital, con el objetivo de fusionar la realidad física, la experiencia de usuario y la tecnología, ya sea con arte generativo, para entretenimiento, o para solucionar problemas específicos. Existen distintas formas de hacer "merge" entre los dos mundos y estáen las manos del diseñador elegir cuál usar.

#### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Personalmente, opino que la aparición de sistemas interactivos que mezclen física, realidad, mercado y usuario abre puertas en muchos campos de acción laboral, especialmente en un entretenimiento orientado al mundo mas ejectuvio (muy por la linea del branding), por ejemplo: Campañas publicitarias con experiencia de usuario, sistemas de interacción usuario-máquina para parques, escenarios y museos, entre otros. 

### Actividad 02
#### ¿Qué es el diseño/arte generativo?
Es una forma de diseñar y plasmar ideas creativas usualmente orientadas al arte visual (digital y a veces llevado a medios físicos) utilizando principalmente códigos y metodlogías Input-Output con información y datos trasnformados finalmente en distintos elementos publicitarios, artísticos, interactivos, musicales, animados, entre otros. 

#### ¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Sería muy interesante generar un brand propio por medio de la ayuda de la IA generativa para mis proyectos personales de tal manera que no se use como un "artista" o un reemplazo, sino como el píncel que ayuda a diseñar mis productos y obras, controlado totalmente por mí mismo, logrando llevar mi visión del arte a un medio digital. 

### Actividad 03
#### (Realizada en clase)

### Actividad 04
#### En tu bitácora: ¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.
Debido a la diferencia o delay entre las señales. Con la función 'is.Pressed' la función envía señales inmediatas bajo un 'true' cuando oprimimos el botón en cuestión. La señal es enviada inmediatamente al presionarlo, por lo que el programa P5.js alcanza leerlo en tiempo real. Sin embargo, con 'was.Pressed', la señal devuelve un true solo una vez cuando el botón fue presionado anteriormente a la pregunta del sistema, si es un true, el Micro:bit realizará la acción requerida, pero lo hará **después** de la interacción del usuario, por ende, para cuando el código p5.js quiere leerlo al mismo tiempo, ya no hay señal que recibir e interpreta que nunca fue presionado ningún botón.

### Bitácora de aplicación (Actividad 05)

#### El programa de p5.js:
```.h
let port;
let connectBtn;
let X;

function setup() {
    createCanvas(400, 400);
    background(220);
    X = width;
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
    let sendBtn = createButton('Send Love');
    sendBtn.position(220, 300);
    sendBtn.mousePressed(sendBtnClick);
    fill('white');
    ellipse(width / 2, height / 2, 100, 100);
}


function draw() {
  

    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
          fill('red')
          X-=100
        }
        else if(dataRx == 'B'){
          fill('yellow')
          X+=100
        }
        else{
            fill('green');
          X-=0
        }
        

        background(220);
        ellipse(X, height / 2, 100, 100);
    }


    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    }
    else {
        connectBtn.html('Disconnect');
    }
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}

function sendBtnClick() {
    port.write('h');
}
```

### El programa de Micro:bit
```.h
from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
    if accelerometer.was_gesture('shake'):
        uart.write('C')
        sleep(500)
    if uart.any():
        data = uart.read(1)
        if data:
            if data[0] == ord('h'):
                display.show(Image.HEART)
                sleep(500)
                display.show(Image.HAPPY)

```

### ¿Cómo funciona el programa?
El programa utiliza la misma estructura de código que la actividad anterior, también realizada en clase. El código de Micro:bit no recibió edición alguna. En el programa de P5.js realicé una adición, la variable X=width que servía como la variable a editar para que pudiéramos cambiar la posición en X de la elipse. Tenía inicialmente los valores por defecto del canvas realizado por el profesor (400, 400) y a ese número (en X) se le añaden o restan 100 unidades por medio del botón **A (hacia la izquierda)** o el botón **B (hacia la derecha)**. Decidí dejar las funciones 'Fill' por razones estéticas para que cuando la elipse cambie de lugar, también se cambie su color. Al mover el Micro:bit físico, podemos cambiar el color de la elipse a verde, sin afectar la coordenada modificada. 

> Nota: Debido a que la función para cambiar a color verde no recibe propiamente un dato emitido por el Micro:bit hacia la consola **(dataRX)**, si hacemos 'Shake' al Micro:bit antes de mover en la coordenada X la elipse, corremos el riesgo de que la coordenada X se mueva hacia la coordenada emitida por el dato recibido más reciente (es decir, se moverá hacia la coordenada X del botón 'B').

## Bitácora de reflexión
### ¿Cómo funciona el sistema físico interactivo de la Actividad 04?
El sistema físico de la actividad 04 es a simple vista una convergencia entre un emisor físico que genera estímulos basados en contacto (botones y su presión) y sensores de movimiento (la información recibida al hacerle 'shake') y un receptor completamente digital que recibe los datos de los estímulos físicos transformados en variables o datos (como 'A', 'B' y 'C'). De esta forma se genera una comunicación máquina-usuario y mundo físico-digital de forma correcta utilizando métodos pre-estipulados para llevar a cabo tareas sencillas. De la misma forma, podemos intuir, planificar y posteriormente probar que la comunicación es **bilateral** por lo que la máquina puede enviarle información (con base a los mismos datos) al medio físico para generar una respuesta por medio de estímulos visibles por el usuario (luces, letras, movimientos, etc.). 






