# Unidad 5
## Bitácora de proceso de aprendizaje
### Actividad 01: 
#### ¿Qué ventajas y desventajas ves en usar un formato binario en lugar de texto ASCII?
Los formatos binarios conllevan a una mejor comunicación entre hardware, la facilidad de enviar datos sin ningún tipo de falla lógica (con los mecanismos de framing adecuados) y la optimización de almacenamiento ya que archivos grandes almacenan más bits en ASCII que los paquetes en binario.

Sin embargo, entender y manipular el binario conlleva más horas de estudio, es más complejo a la hroa de depurar o leer archivos y editarlos por personas. Hay, además, más ports y plugins adaptados o soportados para y por ASCII, a diferencia de binario. 

#### Si xValue=500, yValue=524, aState=True, bState=False, ¿cómo se vería el paquete en hexadecimal? 
Xvalue= 500 = 0x01F4. (si lo leemos en Big Endian) Por lo que el valor es= [01 F4]
Xvalue= 524 = 0x020C. (si lo leemos en Big Endian) Por lo que el valor es= [02 0C]
aState= 01. (true)
bState= 00. (false)

[Respuesta final] 01 F4 02 0C 01 00

#### ¿Por qué el protocolo ASCII de la unidad anterior no tenía este problema de sincronización?
Porque usábamos el salto de línea (/n) como delimitador de los datos para que el receptor supiera cuando empezar y terminar de leer los paquetes. 

#### ¿Por qué en binario no podemos usar \n como delimitador?
Por que podríamos delimitar los datos por medio del salto de línea (/n) lo que facilitaba que el receptor supiera cuando empezar y terminar de leer. En binario el factor (/n) puede significar otro tipo de byte o información por lo que el receptor no lo va a tomar como un salto de línea sino como otro dato más por leer. 

#### ¿Cuántos bytes tiene el paquete completo con framing? ¿Cuántos más que sin framing?
El paquete tiene 8 bytes con framing (contando el chk y el header) y son 6 datos puros sin framing.

#### ¿Qué pasa si un byte de datos tiene el valor 0xAA (170 en decimal)? ¿Podría el receptor confundirlo con un header? ¿Cómo ayuda el checksum en este caso?
No puede confundirlo, por la función del checksum que compara los datos recibidos con la suma de los datos añadidos. La suma no se realiza si existe un valor duplicado o existe alguna confusión. En ese caso el checksum indicará que hay un error de interpretación y tomará el valor como no válido y obligaría a volver a realizar la entrega del paquete o descartarlo completamente. 

## Bitácora de aplicación 
### Actividad 02: 
#### 1. Objetivo de la Actividad
Migrar el sistema de comunicación de un protocolo basado en texto (ASCII) a un protocolo binario compacto. El reto principal fue implementar el "Framing" (delimitación de paquetes) y la verificación de integridad mediante Checksum.

#### 2. Proceso de Construcción y Pruebas Intermedias
Fase 1: Creación del MicrobitBinaryAdapter
Decisión Técnica: Utilicé la clase Buffer de Node.js en lugar de strings, ya que los datos binarios pueden contener bytes que un string interpretaría erróneamente.

Lógica de Sincronización: Implementé un bucle while que busca el byte de cabecera 0xAA. Si el primer byte no es el header, el sistema lo descarta y sigue buscando hasta alinearse con un paquete válido de 8 bytes.

#### 3: Prueba de Error Provocado (Validación de Checksum)
Para verificar que mi lógica de seguridad funcionaba, realicé un "Stress Test" modificando el firmware del Micro:bit:

Error forzado: Cambié la línea checksum = sum(data) % 256 por un valor fijo checksum = 180.

Resultado esperado: El adaptador debería detectar que la suma de los datos no coincide con el 180 recibido y descartar el paquete.

Solución al problema de visibilidad: Inicialmente no veía nada en la terminal. Lo solucioné agregando console.log detallados para comparar el Checksum Calculado vs. el Recibido.

#### 5: Conclusiónn
La arquitectura desacoplada mediante el Patrón Adapter permitió que el frontend siguiera funcionando perfectamente sin saber que cambiamos el idioma del hardware. El uso de protocolos binarios redujo el peso de los mensajes de ~19 bytes a solo 8 bytes, haciendo el sistema más eficiente.

## Bitácora de reflexión

### Actividad 3: 

### 1: tabla comparativa: 
<img width="896" height="551" alt="image" src="https://github.com/user-attachments/assets/011c8efd-6097-4c20-a735-6d37738e8a65" />

### 2: ventajas de la arquitectura desacoplada:
La arquitectura basada en los patrones Adapter + Bridge + FSM permite integrar un protocolo binario sin modificar el frontend o el transporte por las siguientes razones:
     1. Contrato de Interfaz: El sketch.js (frontend) no sabe cómo se obtienen los datos; solo espera recibir un objeto JSON con las propiedades {x, y,           btnA, btnB}. Mientras el Adapter entregue ese formato, el resto del sistema permanece intacto.
     2. Encapsulamiento: Toda la complejidad del manejo de bytes, buffers y desplazamientos de memoria queda oculta dentro del MicrobitBinaryAdapter.
     3. Independencia de Transporte: El Bridge se encarga de mover los datos de un punto a otro. No le importa si el contenido original era un texto                 separado por tubos (|) o una secuencia de bytes hexadecimales

### 3: Protocolo binario vs ASCII en el mundo real
Preferiría un Protocolo Binario cuando:

1. Limitación de Ancho de Banda: En misiones espaciales o satélites donde cada bit enviado cuesta mucha energía y tiempo.
2. Sistemas Embebidos: En microcontroladores con muy poca memoria RAM donde procesar strings (texto) es muy costoso y lento.
3. Alta Frecuencia: Si el sensor necesita enviar 1000 lecturas por segundo, el ahorro de bytes del binario es crítico.

Preferiría un Protocolo ASCII cuando:

1. Desarrollo Rápido: Cuando se necesita prototipar algo velozmente y poder leer los errores directamente en la pantalla sin herramientas especiales.
2. Interoperabilidad: Cuando el sistema debe ser leído por aplicaciones muy diversas (como un navegador, un Excel o un archivo de texto simple).
3. Configuraciones Humanas: Archivos de configuración que un administrador deba editar manualmente.

### 4: Diagrama actualizado:


