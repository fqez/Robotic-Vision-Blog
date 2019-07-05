---
layout: post
title: Follow Line practice
categories: [follow_line]
tags: [follow_line, MOVA, robotic_Vision]
fullview: true
comments: false
identidad: follow_line
lang: en
lang-ref: welcome-to-my-blog
---


<img src="https://jderobot.org/store/fperez/uploads/images/circuito.png" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" align="middle">

<!--src="https://academy.jderobot.org/static/img/notebooks_images/follow_line/follow_line_f1.png"

src="https://academy.jderobot.org/static/img/notebooks_images/follow_line/f1_overview.png"-->


## Introducción

Para esta práctica se dispone de un mundo simulado en el simulador Gazebo que incluye un circuito y un coche de Fórmula 1. La idea detrás de esta práctica es **que el coche sea capaz de completar vueltas en el circuito siguiendo la línea pintada en el asfalto de forma autónoma y lo más robusta posible**.
El coche dispone de **un sensor** (la cámara a bordo) y de **un actuador** (los motores), por lo que el único requisito para solucionar este problema es **hacer uso de la visión artificial** para que el coche en todo momento sepa en qué escenario se encuentra a través de lo que ve mediante su sensor, y pueda actuar en consecuencia.

<img src="https://academy.jderobot.org/static/img/thumbnails/thumbnail_follow_line.png" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" height="300" align="right">

#### Sensores

En cuanto a los sensores del coche, **se dispone de una camára a color** con una resolución espacial de imagen de 640x480 píxeles. Se trata de una imagen con un tamaño relativamente grande que tiene sus ventajas e inconvenientes. Al ser una imagen más grande, el número de píxeles es mayor, por lo que al hacer operaciones a nivel de píxel es computacionalmente más costoso a priori. Por otra parte, a mayor tamaño de imagen, mayor resolución, que implica más información para trabajar; en este caso, el circuito consta de indicadores visuales de curva y distancia a la misma, por lo que quizá con una cámara que sirve imágenes más pequeñas sea más complicado obtener información para segmentar dichos elementos.

#### Actuadores

El coche de fórmula 1 sólo **consta de un actuador**, asociado a los motores. Este actuador **permite dotar al coche de movimiento**. Se trata en este caso de un robot no-holonómico ya que no puede desplazarse en cualquier dirección de una vez, sino que tiene que realizar maniobras previas; por ejemplo, si queremos que el robot se mueva hacia un lateral, debemos girarlo previamente. No obstante, el modelo geométrico que implementa el robot no es de geometría de Ackermann sino geometría diferencial, por lo que el coche es capaz de girar sobre sí mismo, lo que facilita enormemente la tarea del movimiento. 
Con esto, podemos ordenar al robot que avance a una velocidad **v** (en m/s) o que gire a una determinada velocidad **w** (en rad/s) con dos instrucciones sencillas. `motors.sendV(v)` y `motors.sendW(w)` para velocidad linear y angular respectivamente.

## Solución

Como he mencionado antes, para solucionar esta práctica únicamente puede hacerse uso de la visión artificial, por lo que utiliando la cámara a bordo del Fórmula 1 tendremos que completar una vuelta al circuito detectando la línea y actuando en consecuencia según la situación en la que se encuentre el robot. Voy a dividir este apartado en dos subapartados explicando las diferentes partes de la solución (visión y control) para hacerlo más legible y comprensible.

#### Procesamiento visual - primera iteración.

<div align="center"><img src="https://jderobot.org/store/fperez/uploads/images/lineargb.PNG" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="50%" align="middle"></div>


Para resolver el apartado de visión en esta práctica, el algoritmo "esqueleto" que he seguido es el siguiente:

1. Obtener la imagen de la cámara.
2. Convertir la imagen a un espacio de color HSV.
3. Umbralizar la imagen con un filtro de color para rojo. 
4. Calcular el punto central de la línea.
5. Calcular el error entre el centro (en columnas) de la imagen y el punto calculado en 4.

Esta fue mi primera aproximación al algoritmo, la cual voy a desgranar un poco más en detalle.

**(1)**La obtención de la imagen está facilitada por la infraestructura del editor web *Unibotics* mediante la siguiente instrucción del HAL-API: `getImage()`, la cual devuelve una imagen en formato numpy array (API de OpenCV) de 640x480 píxeles en color (3 canales: R, G y B).

**(2)** Para no ser tan sensible ante cambios de iluminación, he transformado la imagen a un espacio de color HSV que tiene más en cuenta la crominancia que la luminancia, de tal forma que los valores de un mimso color ante una iluminación pobre serán similares a los de ese mismo color bajo una iluminación adecuada. Esta operación está incluída dentro del API de OpenCV: `cv2.cvtColor(image_src, cv2.COLOR_RGB2HSV)`

**(3)** Para quedarme sólo con los píxeles que incluyen la línea, realicé una umbralización mediante un filtro de color rojo en la imagen resultante del paso anterior. Esta operación también es sencilla al estar en el API de OpenCV: `cv2.inRange(image_hsv,red_lower,red_upper)`

**(4)** Para el cálculo de este último, obtuve el bounding box de la línea segmentada y me quedé con el punto medio del mismo.

**(5)** Orienté el cálculo del error que alimenta el controlador PID de la siguiente forma: el error será la diferencia (con signo) de píxeles entre el punto central de la imagen (en columnas) y el punto central de la línea. Por lo tanto cuanto más se aleje el coche de la línea por izquierda o por derecha, mayor será esa diferencia en valor absoluto y por lo tanto mayor el error cometido.

#### Procesamiento visual - segunda iteración.

<div align="center"><img src="https://jderobot.org/store/fperez/uploads/images/lineabn.PNG" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="50%" align="middle"></div>


Al realizar pruebas en la primera iteración, observé que a velocidades muy bajas (en torno a los 3 m/s), el algoritmo funcionaba "bien", oscilaba bastante pero conseguía seguir la línea. No obstante, realizando mediciones de tiempo por iteración en el procesamiento de la imagen, resultó que el tiempo de cómputo era notable, por lo que intenté reducirlo en esta iteración con diferentes técnicas. Por lo que el algoritmo quedaba de la siguiente manera (**mejoras en negrita**):

1. Obtener la imagen de la cámara.
2. **Recortar la imagen por arriba para alivio de cómputo.**
3. Convertir la imagen a un espacio de color HSV.
4. Umbralizar la imagen con un filtro de color para rojo. 
5. **Obtener el primer punto donde se detecta la línea en la imagen (horizonte).**
6. **Obtener el punto central de la misma en base al punto anterior calculado y la parte baja de la imagen.**
7. Calcular el error entre el centro (en columnas) de la imagen y el punto calculado en 4.

**(2)** Para no realizar el umbralizado de la imagen completa (recordemos de 640x480 píxeles) en cada iteración, decidí prescindir de información inútil para esta práctica, por lo que recorté la imagen 230 píxeles por arriba, ya que sólo me interesaba la parte baja de la imagen (donde está la línea)

**(5)** Además, en lugar de calcular el bounding box de los píxeles umbralizados de la imagen, y, considerando que en realidad lo único que quería era un punto concreto, opté por seleccionar el punto medio de otra forma más eficiente. Reducí la imagen umbralizada a una sola columna haciendo una suma por filas de toda la imagen umbralizada (operación ultraoptimizada de numpy `numpy.reduce(...)`). De esta forma, me quedaría una única columna donde su mínimo (punto más alto en la imagen) me indicaría donde empieza a verse la línea (horizonte).

**(6)** Con esa información, pude obtener el píxel central de la linea que me interesaba sin necesidad de calcular el bounding box de la línea en cada iteración. haciendo una resta entre el punto obtenido en 5 y el tamaño de la imagen. Con esa información de la fila en la que se encontraba el píxel central de la línea, implementé una función para saber en qué columna se encontraba ese punto (para el cálculo del error). Esta función toma la imagen umbralizada y la fila que se quiere procesar y busca el primer y último píxel blanco de esa fila, obteniendo el punto medio como la media entre los puntos mínimos y máximos.

Con estas mejoras, el procesamiento de imagen se reduce bastante debido a que para cada iteración, la umbralización se realiza sobre 230 píxeles menos, y la detección del centro de la línea ya no utiliza la imagen completa sino una sola fila de la misma y una reducción por columnas de la imagen umbralizada.
 
#### Procesamiento visual - tercera iteración.

 <div class="row">
  <div class="column">
    <img src="https://jderobot.org/store/fperez/uploads/images/lineabnr.PNG" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="100%" align="middle">
  </div>
  <div class="column">
    <img src="https://jderobot.org/store/fperez/uploads/images/lineabnc.PNG" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="100%" align="middle">
  </div>
</div> 

deteccion de curva recta
buffer

En este caso, la primera aproximación y más sencilla para resolver esta práctica es la siguiente:

<div align="center"> <iframe width="560" height="315" src="https://www.youtube.com/embed/3P_WPogZ5rM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></div>

#### Controlador PID
