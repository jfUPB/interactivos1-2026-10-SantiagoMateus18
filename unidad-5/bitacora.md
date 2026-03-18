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

## Bitácora de reflexión
