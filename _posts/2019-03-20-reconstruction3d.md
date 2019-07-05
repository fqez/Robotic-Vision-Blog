---
layout: post
title: 3D Reconstruction practice
categories: [3d_reconstruction]
tags: [3d_reconstruction, MOVA, robotic_Vision]
fullview: true
comments: false
---

## Introducción

Uno de los algoritmos más famosos en visión artificial es la reconstrucción de una escena 3D a partir de imágenes 2D. Una de las aplicaciones de esto es la construcción de mapas para navegación en robótica, que es una de las piedras angulares de los algoritmos de SLAM, por ejemplo.

Existen diversos escenarios posibles sobre los que se puede realizar una reconstrucción 3D a partir de imágenes 2D, y multitud de algoritmos diferentes para llevarla a cabo, desde filtros de partículas hasta creación de mapas densos de disparidad o cálculo de los puntos 3D a partir de correspondencias entre píxeles, como en este caso.

El elemento esencial de este ejercicio son las cámaras. Para realizar una reconstrucción 3D, se pueden emplear infinidad de configuraciones de cámaras: usar 1, 2 o más cámaras; colocarlas en paralelo, en forma de malla; alineadas, no alineadas, etc. El modelo estándar consiste en un par estéreo de cámaras separadas por una distancia (no demasiado grande) y que no están en paralelo, como se puede ver en la siguiente imagen.

<div align="center"><img src="https://jderobot.org/store/fperez/uploads/images/epipolarNC.png" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="50%" align="middle"></div>
Sin embargo, en el caso de este ejercicio, la configuración es ligeramente diferente. Tenemos lo que se denomina una configuración par estéreo canónica, lo cual quiere decir, que en este caso las cámaras están perfectamente alineadas y en paralelo (ambas mirando al frente). Esta configuración tiene una serie de ventajas comparada con la configuración estándar, como la simplificación de los cálculos necesarios para calcular correspondencias entre ambas imágenes utilizando las líneas epipolares. En la configuración canónica del par estéreo, las líneas epipolares son siempre paralelas a la horizontal (asumiendo que el sistema carece de cualquier tipo de ruido), lo que simplifica mucho la búsqueda de puntos en la recta. En la siguiente imagen se puede observar la configuración del par estéreo canónico con la recta epipolar.

<div align="center"><img src="https://jderobot.org/store/fperez/uploads/images/epipolar.png" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="50%" align="middle"></div>
En esta práctica se pide realizar una reconstrucción de una escena 3D proveniente del simulador Gazebo a través de las imágenes captadas por dos cámaras en configuración par estéreo canónico a bordo de un robot Turtlebot de *Yujin Robots*.

## Solución

La solución propuesta, a diferencia del ejercicio de sigue línea, es más algorítmica. Se ha dividido el algoritmo en una serie de pasos a seguir para la resolución del problema. Estos pasos se enumeran a continuación.

### 1. Obtención de píxeles representativos.

El objetivo de este ejercicio es obtener una reconstrucción de la escena lo más parecido a la escena real posible sin perder de vista la eficiencia del algoritmo. A pesar de que en este algoritmo el factor de tiempo real no es crítico, se ha implementado de forma genérica de tal forma que sea flexible para facilitar cambios en el mismo. Por este motivo, el foco no está en reconstruir todos los píxeles de la imagen ya que tomaría un tiempo excesivo, sino solo aquellos suficientemente representativos de la escena. 

En visión artificial, generalmente la información más valiosa proviene de los bordes y esquinas de los objetos, donde se producen los cambios de intensidad más altos en los píxeles. Es por ello que en este caso interesa obtener dicha información.

Uno de los algoritmos más utilizados para obtener las imágenes de bordes es el algoritmo de Canny, que realiza una convolución sobre la imagen con unos determinados filtros y acto seguido la "limpia" resultando en unos bordes muy definidos y con pocos cortes. Aplicando el algoritmo de Canny para obtener la información de bordes, se obtienen del orden de 6000 puntos a representar, lo cual resulta escaso para obtener una representación fidedigna de la escena. Por ello, se ha realizado una operación morfológica de dilatación con elemento estructurante de 5x5 píxeles sobre la imagen de bordes para hacerlos "engordar" y tener así más puntos que representar (unos 23.000). 

La siguiente imagen muestra la aplicación de Canny con y sin operación de dilatación.

 <div class="row">
  <div class="column">
    <img src="https://jderobot.org/store/fperez/uploads/images/canny3dr1.PNG" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="100%" align="middle">
  </div>
  <div class="column">
    <img src="https://jderobot.org/store/fperez/uploads/images/canny3dr2.PNG" alt="f1_intro" title="" hspace="20" vspace="20" width="100%" align="middle">
  </div>
</div> 

### 2. Retroproyección

Una vez se tienen los puntos más representativos de la escena, se trata de estimar su posición 3D en el mundo a partir de la información de la imagen de bordes. Para ello, cada punto hay que expresarlo en coordenadas solidarias a la escena 3D en la que se va a reconstruir, ya que los puntos de borde de la imagen son eso, puntos 2D en la imagen. Para hacer esto, el API de Unibotics ofrece una función denominada `graficToOptical` que se encarga de expresar coordenadas de píxel en coordenadas de la cámara. Después se obtiene un punto en 3D en el espacio a través de otra instrucción ofrecida por el API de Unibotics denominada `backProject`. Esta función nos ofrecerá un punto perteneciente a la recta que une el centro óptico de la cámara (obtenido con la función `getCameraPosition`) y el píxel que se quiere reconstruir PERO es un punto cualquiera de esa recta, no el que nos interesa de verdad.

Para calcular ese punto, hay que realizar lo que se denomina triangulación.

### 3. Triangulación

Para realizar la triangulación es necesario tener al menos dos rectas que se corten que partan de los centros ópticos de las cámaras hacia los mismos puntos de interés de la imagen. En el paso anterior obtuvimos una de las rectas al unir el centro óptico de una de las cámaras con el cálculo de las coordenadas 3D de uno de los píxeles característicos de la imagen que captó. Es necesaria otra recta que parta del centro óptico de la otra cámara hacia ese mismo punto, pero captado por la otra cámara.

Para llevar este paso a cabo, se requiere del uso de la línea epipolar, ya que es en la proyección de esa línea donde se encontrará el punto que se está procesando, en la otra imagen. Para obtener la línea epipolar que permita encontrar el píxel que se está procesando en la imagen contraria, se proyectan el punto 3D del centro óptico de la primera cámara y el punto 3D correspondiente al píxel que se está procesando y que se obtuvo en el paso anterior con la función `backproject`.

De tal manera, que después de proyectar ambos puntos se podrá formar una recta proyectada sobre la imagen contraria calculando la recta que pasa por esos dos puntos proyectados mediante la ecuación punto-pendiente de la recta.

Al estar en una configuración par estéreo canónica, las líneas epipolares siempre serán paralelas a la horizontal, por lo que teniendo la proyección de uno de los puntos sobre la imagen contraria no sería necesario calcularla; aún así se ha realizado el cálculo de la misma.

### 4. Emparejamiento.

Hasta ahora se tiene la siguiente información:

* Punto 3D del centro óptico de una cámara
* Píxel 2D que está siendo procesado
* Punto 3D perteneciente a la recta que une el centro óptico de la primera cámara y el píxel procesado (convertido a coordenadas 3D)
* Línea epipolar proyectada en la imagen contraria.

Con esta información, mediante emparejamiento de bloques, se puede realizar la búsqueda de la correspondencia del píxel que se está procesando en una imagen, en la otra. El emparejamiento de bloques se realiza mediante el método de correlación, mediante el cual, convolucionando una plantilla de la que se quiere conocer su posición más probable sobre una imagen, devolverá otra imagen de niveles de confianza de que la plantilla se encuentre en cada uno de los píxeles. De este modo, el valor más alto de la imagen resultante será la posición donde encaja mejor la plantilla. El siguiente gif muestra este método:

<div align="center"><img src="https://jderobot.org/store/fperez/uploads/images/3dr11.gif" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="50%" align="middle"></div>

Se sabe que ese píxel estará en algún punto de la línea epipolar, por lo que se ahorra muchísimo tiempo de cómputo al no ser necesario buscar ese punto por toda la imagen, sino por una sola línea de esta (se recuerda que la epipolar es una linea horizontal).

Experimentalmente se ha comprobado que en este caso, dada la configuración de las cámaras, las correspondencias entre los mismos píxeles en las distintas imágenes no se alejan más de 10 píxeles. Por esto, se ha limitado el espacio de búsqueda a únicamente 15 píxeles de la línea epipolar.

<div align="center"><img src="https://jderobot.org/store/fperez/uploads/images/3dr1.gif" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="50%" align="middle"></div>

Tras aplicar emparejamiento de bloques, se obtiene el mismo píxel que se está procesando en la otra imagen, por lo que repitiendo el paso 2 sobre este píxel, dispondremos de un nuevo punto 3D perteneciente a la recta donde está el punto que se quiere reconstruir en la escena.

<div align="center"><img src="https://jderobot.org/store/fperez/uploads/images/3dr12.gif" alt="f1_intro" title="F1 intro image" hspace="20" vspace="20" width="50%" align="middle"></div>

### 5. Triangulación (II)

En este paso ya se dispone de todos los elementos necesarios para reconstruir el píxel en el espacio 3D. Disponemos de 2 rectas (4 puntos en 3D) las cuales unen los centros ópticos de ambas cámaras con puntos de una recta donde está el punto a reconstruir (su intersección). Por lo que solamente queda obtener el punto de corte de ambas rectas para obtener el punto reconstruido que se busca. No obstante, la intersección de rectas en 3 dimensiones es muy complicada, ya que es muy probables que las rectas se corten en 2 de los planos, pero no el los 3. Por ello, se aplica un algoritmo que encuentra la distancia mínima entre dos rectas y devuelve el punto medio del segmento que las une, siendo este el resultado de nuestra reconstrucción para un píxel de la imagen. Este algoritmo se ha obtenido de [1].

### 6. Resultados

Aplicando los pasos 1-5 en cada píxel de la imagen de bordes, se obtendrá una reconstrucción completa de la escena original a partir de los píxeles de las imágenes captadas por el par estéreo. El siguiente vídeo muestra el resultado final de una ejecución del algoritmo.

<div align="center"> <iframe width="560" height="315" src="https://www.youtube.com/embed/BNaXtTwZAnA" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></div>



[1] http://paulbourke.net/geometry/pointlineplane/