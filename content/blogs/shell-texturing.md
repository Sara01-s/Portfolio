---
title: "Shell Texturing (Spanish)"
url: /blogs/shell-texturing
date: 2024-07-12T21:20:00-04:00
draft: false
author: "Sara San Martín"
tags:
  - Spanish
  - Personal Project
  - Computer Graphics
  - Shaders
image: /images/blogs/shell-texturing/imgs/gif_hand_result.gif
description: "Learning to implement Shell Texturing"
mathjax: true
toc:
---

*Figura 1. GIF Resultado de la investigación.*

# Shell texturing
---

## Introducción
Navegando por Youtube, tuve el agrado de encontrar el [video sobre Shell Texturing](https://www.youtube.com/watch?v=9dr-tRQzij4) hecho por *Acerola* [1], donde explica lo difícil que es renderizar y simular el pelo, de media, los humanos tenemos entre `90.000` y `150.000` pelos en la cabeza [2], así que hay que imaginarse lo compleja que sería geométricamente una simulación físicamente precisa del pelo.

Esto llamó inmediatamente mi curiosidad, pues me pregunté: *"¿Cómo es que desde hace años existen videojuegos que han logrado renderizar pelo o pelaje en hardware significamente menos eficiente que el actual (2024) pareciendo algo tan complejo de lograr?"*. Por lo que me propuse ver el video, implementar esta *ilusión* de pelaje en el motor de videojuegos *Unity* y realizar este registro sobre mi exploración donde entraré en detalle y diseccionaré paso a paso este efecto y así aportar mi granito de arena, ¡espero sea de tu agrado!.

## Hashing
---
Nuestro primer objetivo será generar una grilla de celdas con valores pseudo-aleatorios (deterministas pero difíciles de predecir) del \\([0, 1]\\) sobre la superficie de nuestro mesh.

Para resolver la generación de valores pseudo-aleatorios  podemos utilizar una función de *hash*.

```hlsl
// Thanks to Hugo Elias
float hash(uint n) {
	n = (n << 13u) ^ n;
	n = n * (n * n * 15731u + 0x789221u) + 0x1376312589u;
	
	return float(n & uint(0x7fffffffu)) / float(0x7fffffff);
}
```

Las primeras dos operaciones de la función de hash escapan el alcance de este blog, sin embargo, si analizamos la operación de retorno, podemos observar que se realiza una operación `and(&)` en `n` y `0x7fffffffu`.

`0x7fffffffu` es un número entero con casi todos sus bits en `1`, la única excepción es el bit de signo, el cual es `0` al ser este un número **positivo**.

Realizamos la operación para analizar su comportamiento.
```
  0111 1111 1111 1111 1111 1111 1111 1111  // 0x7fffffffu
& 1010 1011 1001 1100 1010 1011 1110 0011  // random negative number
‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
  0010 1011 1001 1100 1010 1011 1110 0011  // same number but positive
```

Con este ejemplo, podemos concluir que el objetivo de esta operación es obtener el valor absoluto de `n`.

Luego, al dividir el valor absoluto de `n` por `0x7fffffffu` (el mayor valor posible que `n` puede tomar), estaremos realizando una **normalización** de `n`, por lo que el **rango** de la función `hash` es \\((0 \leq n \leq 1)\\).

## Generando la grilla
---

El objetivo ahora es lograr crear una grilla con celdas que tengan un valor pseudo-aleatorio entre \\([0, 1]\\) asociado, veremos como realizar esta tarea paso por paso.

Si devolvemos `uv.x` en el canal rojo del fragment shader obtendremos este output:
![Figura 2. eje x de las coordenadas uv.](../../images/blogs/shell-texturing/imgs/img_uv_x.png)
*Figura 2. eje x de las coordenadas uv.*

Luego, dividimos el espacio en valores enteros calculando el `floor()` de `uv.x`, lo cual cual nos otorga el siguiente resultado:

![Figura 3. "floor" del eje x de las coordenadas uv.](../../images/blogs/shell-texturing/imgs/img_floor_uv_x.png)
*Figura 3. "floor" del eje x de las coordenadas uv.*

Lo cual no es muy interesante... ¡Pero tiene todo el sentido del mundo!, ya que `uv.x` \\(\in [0, 1]\\), `floor(uv.x)` nos devolverá `0` (negro) en todos los pixeles. Para solucionar esto, escalaremos las coordenadas `uv` por un valor que llamaremos `numCells`. 

Veamos cual es el resultado de multiplicar `uv` por `numCells = 10`.

![Figura 4. "floor" de uv.x multiplicado por 10."](../../images/blogs/shell-texturing/imgs/img_floor_uv_x_10.png)
*Figura 4. "floor" de uv.x multiplicado por 10".*

Vemos que ahora \\(\frac{1}{10}\\) de la imagen final es negra, esto es debido a que la franja negra tiene el valor `uv.x = 0`, a su lado, se encuentra una franja roja del mismo ancho y valor `uv.x = 1` y así sucesivamente, como no existe un valor más rojo que `r = 1.0` en colores LDR (Low dynamic range), el efecto no es tan obvio. Sin embargo si ponemos nuestra fórmula como input de hash, la situación será evidente.

![Figura 5. función hash aplicada en uv.x.](../../images/blogs/shell-texturing/imgs/img_hash_uv_x.png)
*Figura 5. función hash aplicada en uv.x.*

¡Ta-da! Recordemos que comprobamos que la función `hash()` nos devuelve un valor \\(\in [0, 1]\\), por lo que la franja negra con el valor `0` que veíamos antes ha sido asociada a un pseudo-aleatorio propio. Vemos que esto ha ocurrido también para `uv.x = 1`, `uv.x = 2`, `uv.x = 3`, y así hasta `uv.x = numCells - 1`.

Sin embargo, nuestras `cells` están algo estiradas verticalmente... debemos agregar el valor `floor(uv.y)` a `floor(uv.x)`, así obtendremos el patrón de grilla que estamos buscando.

![Figura 6. función hash aplicada en uv.x + uv.y.](../../images/blogs/shell-texturing/imgs/img_uv_x_plus_uv_y.png)
*Figura 6. función hash aplicada en uv.x + uv.y.*

Ahora si vemos claramente nuestra grilla, si me lo preguntan a mi diría que se ve muy bien pero claramente tiene un patrón predecible, no hay que olvidar que nuestro objetivo es crear una superficie de celdas con valores pseudo-aleatorios \\(\in [0,1]\\) asociados.

Este problema es aún más evidente si multiplicamos `uv` por `numCells = 100`.

![Figura 7. número de "celdas" multiplicado por 100.](../../images/blogs/shell-texturing/imgs/img_cells_100.png)

*Figura 7. número de "celdas" multiplicado por 100.*

Sin embargo, si multiplicamos `uv.y` por `numCells + 1`, obtenemos:
![Figura 8. Magia.](../../images/blogs/shell-texturing/imgs/img_uv_numcellsplus1.png)
*Figura 8. Magia.*

¿Cómo?, ¿que clase de magia es esa?, ¿Acaso esa multiplicación me apareció en un sueño?. Bueno, la realidad es que se debe a como funciona nuestra fórmula (y ayuda de parte de Francisco Aliaga). Podemos expresar la creación de la seed como \\(s(x, y) := x + yt\\) donde \\(t\\) es una constante `uint` que podremos ajustar a gusto.

Queremos que la `seed` no se reptia para dos input `x, y`, para esto, debemos analizar que significa que dos `seed` sean iguales expresándo tal igualdad utilizando la definición de \\(s(x, y)\\).

$$x_1 + y_1t = x_2 + y_2t.$$

Si restamos \\(x_2\\) en ambos lados y factorizamos por \\(t\\) en el lado derecho nos queda que

$$x_1 - x_2 = t(y_2 - y_1). \tag{1}$$

Ahora, analizando el caso de que \\(y_1 = y_2\\), notamos que

$$x_1 - x_2 = t(y_2 - y_1) = 0 \implies x_1 = x_2.$$

Y si \\(x_1 = x_2\\) e \\(y_1 = y_2\\), significa que estamos hablando del mismo punto \\((x, y)\\), lo cual demuestra la unicidad de un seed en una fila de la grilla.

Ya que el caso \\(y_1 = y_2\\) está abordado, prosigamos con el caso cuando \\(y_2 \ne y_1\\) (celdas en distintas filas), tenemos que el **mínimo valor** posible del *valor absoluto* **del lado derecho** de la ecuación \\((1)\\) **es \\(t\\)** (en el caso \\(y_1 = 0, y_2 = 1\\)). Concluimos que:

$$t \le |x_1 - x_2|.$$

Nos viene bien recordar que \\(x_1, x_2 \le\\) `numcells` por lo que *necesariamente* \\(x_1 - x_2 \le\\) `numCells`. También podemos concluir que,

$$|x_1 - x_2| \le \mathtt{numCells}.$$

Por lo tanto hemos probado que si las semillas son iguales en celdas de  distintas filas entonces \\(t \le\\) `numCells`.

Aquí notamos que podemos elegir un valor de \\(t\\) que sea estrictamente mayor que `numCells` y eso obligaría necesariamente que las semillas sean **distintas**.


## Creando los shells
---

Ahora que tenemos nuestra grilla de números pseudo-aleatorios en la superficie de nuestro mesh, podemos crear nuestro primer `shell`, para esto definiremos el color del pixel dada la siguiente condición \\((2)\\) :
```hlsl
if (rand > 0.005) {
    pixelColor = float3(1.0, 0.0, 0.0); // rojo
}
else {
    pixelColor = float3(0.0, 0.0, 0.0) // negro
}
```

Vemos que esta condición produce el siguiente resultado:

![Figura 9. puntos negros en la grilla.](../../images/blogs/shell-texturing/imgs/img_black_dots.png)
*Figura 9. puntos negros en la grilla.*

Es esperable que si `rand` \\(\in [0, 1]\\), la mayoría de celdas tengan un valor mayor que \\(0.005\\) asociado. Podemos ver esto reflejado como muchas `cells` rojas y pocas negras.

Ahora, como el título de esta sección indica, debemos crear múltiples shells (planos con el shader) para conseguir el efecto final, para esto definiremos una variable llamada `_NumShells`, la cual indica cuantas capas de shells se deben crear.

Ejemplo de 16 shellls instanciados:
![Figura 10. 16 shells instanciados.](../../images/blogs/shell-texturing/imgs/img_shells_no_height.png)

*Figura 10. 16 shells instanciado.*

Bueno... no es el resultando que uno espera, vemos aún parece que es un único shell. Esto se debe a que todos comparten el mismo punto de origen y por lo tanto, sus vértices son iguales. Es necesario extruir cada vértice del shell por un valor que llamaremos `height`.

Para definir el valor de `height` primero es necesario indexar cada shell con un valor `_ShellIndex`, el que será divido por `_NumShells` para así obtener un valor `height` **normalizado**.

![Figura 11. shells con altura variante.](../../images/blogs/shell-texturing/imgs/img_shells_with_height.png)
*Figura 11. shells con altura variante.*

*posición y de los vértices + height*

La magia comienza a aparecer cuando cambiamos el valor arbitrario `0.005` que habíamos establecido en la condición \\((2)\\) por `height`.

![Figura 12. mostrar pixeles negros según valor random influenciado por altura.](../../images/blogs/shell-texturing/imgs/img_rand_lt_height.png)
*Figura 12. mostrar pixeles negros según valor random influenciado por altura.*

El valor `rand` al estar en el rango \\([0, 1]\\), tiene menos probabilidades de ser mayor que `height`, si este es cercano a \\(1\\), lo cual es el caso de las capas superiores.

Aúnque los pixeles negros de cada shell son fantásticos para visualizar que está ocurriendo detrás de cámaras, claramente no nos dejan ver el verdadero efecto que estamos creando, por lo que utilizaremos la palabra clave de HLSL `discard`, para eliminar dichos pixeles y poder ver a través de ellos.

![Figura 13. pixeles negros descartados.](../../images/blogs/shell-texturing/imgs/img_black_pixels_discarded.png)

¡Excelente, los pixeles negros se han ido!, sin embargo no existe suficiente contraste para discernir la profundidad entre los pixeles, ¡pero no pasa nada!, podemos implementar un modelo de iluminación *naive* simplemente multiplicando el color resultante del pixel por el `height` del shell.

![Figura 14. color del pixel multiplicador por la altura del shell.](../../images/blogs/shell-texturing/imgs/img_color_times_height.png) 
*Figura 14. color del pixel multiplicador por la altura del shell.*

Ya que la altura es \\(0\\) en la base y \\(1\\) en la superficie de nuestro shell, el color incrementará su intensidad linearmente. De forma adicional, dentro de este cálculo de iluminación podemos elevar `height` a un exponente `_LightAttenuation`, para romper el comportamiento linear.

![Figura 15. atenuación de la luz variando de 0 a 10.](../../images/blogs/shell-texturing/imgs/gif_light_attenuation.gif)
*Figura 15. atenuación de la luz variando de 0 a 10.*

Los resultados que tenemos actualmente son muy decentes, sin embargo es muy fácil romper la ilusión simplemente bajando un poco la cámara.

![Figura 16. el efecto se rompe al bajar la cámara.](../../images/blogs/shell-texturing/imgs/img_broken_illusion.png)
*Figura 16. el efecto se rompe al bajar la cámara.*

Aumentaremos o disminuiremos la separación de los shells multiplicando el valor `height` por un valor `_ShellsSeparation` \\(\in [0, 10]\\) dentro del vertex shader.

![Figura 17. variación en la separación de los shells.](../../images/blogs/shell-texturing/imgs/gif_shell_separation.gif)
*Figura 17. variación en la separación de los shells.*

*shell separation en funcionamiento*

Si establecemos `_NumShells` y `_NumCells` a \\(128\\) y ajustamos la separación entre capas a \\(0.75\\), podremos obtener un efecto de mejor resolución.

![Figura 18. aumento en la resolución del efecto.](../../images/blogs/shell-texturing/imgs/img_highres_zoomed.png)
*Figura 18. aumento en la resolución del efecto.*


## Césped
---

Si bien con la implementación actual podemos conseguir algo parecido a lo logrado en *Viva Piñata (2007)*, nuestro Shell Texturing tiene una gran desventaja y es que solo funciona verticalmente, si añadimos el shader a un mesh esférico obtendremos esto.

![Figura 20. los shells solo crecen hacia arriba.](../../images/blogs/shell-texturing/imgs/img_shells_in_sphere_bad.png)

*Figura 20. los shells solo crecen hacia arriba.*

Uno esperaría que los shells se extruyeran de forma ortogonal a las caras del mesh, para lograr eso debemos desplazar la posición del vector en dirección a la *normal* (no olvidemos escalar la normal por `height` también).

![Figura 21. los shells ahora crecen en dirección a la normal.](../../images/blogs/shell-texturing/imgs/img_sphere_normals.png)

*Figura 21. los shells ahora crecen en dirección a la normal.*

Nuestra esféra se ve mucho mejor ahora, aunque algo cuadrada, con un un estilo tipo Cube World. Podemos mejorar el efecto para conseguir una ilusión de hebras de césped.

Para lograr un efecto más parecido al pasto debemos dividir el espacio en segmentos de coordenadas `uv` para cada hoja de pasto. Este es el caso idóneo para utilizar la función `frac()`, la cual nos devuelve el valor decimal de un número flotante.

![Figura 22. graficación de la función fract(x) hecha en graphtoy.](../../images/blogs/shell-texturing/imgs/img_fract.png)
*Figura 22. graficación de la función fract(x) hecha en [graphtoy.com](https://graphtoy.com/?f1(x,t)=frac(x)&v1=true&f2(x,t)=&v2=false&f3(x,t)=&v3=false&f4(x,t)=&v4=false&f5(x,t)=&v5=false&f6(x,t)=&v6=false&grid=1&coords=1.921175863793418,0.21346398486593535,2.3741360268016223).*

Si volvemos atrás un momento y retornamos las coordenadas `uv` en un plano, notamos como en la esquina inferior izquierda se ve el espacio `uv` original y luego simplemente se escala por `_NumCells`.

![Figura 23. visualización de las coordeandas uv.](../../images/blogs/shell-texturing/imgs/img_uv_pre_frac.png)
*Figura 23. visualización de las coordeandas uv.*

La idea es repetir las coordenadas `uv` normalizadas en todas las celdas de la grilla, es por eso eso que utilizaremos la función `frac()` en las coordenadas `uv`, así sus componentes no podrán escalar más allá de \\(1\\) y serán encerrados en el rango \\([0, 1]\\).

![Figura 24. visualización de frac(uv).](../../images/blogs/shell-texturing/imgs/img_uv_frac.png)

*Figura 24. visualización de frac(uv).*

Misión cumplida, aunque es difícil distinguir bien las coordenadas, por lo que haremos zoom para hacer algunos ajustes quirúrjicos.

![Figura 25. zoom para distingir las coordenadas uvs fragmentadas (cell uv).](../../images/blogs/shell-texturing/imgs/img_uv_frac_zoomed.png)

*Figura 25. zoom para distingir las coordenadas uvs fragmentadas (cell uv).*

Primero, podemos verificar que cada celda tiene sus propias coordenadas `uv` normalizadas las cual llamaremos `cellUv`. También notamos que el origen de las coorderdenadas se encuentra en la parte superior izquierda de la celda, lo que no es deseable para la próxima operación que realizaremos, por lo que para desplazar el origen al centro de la celda restaremos \\(0.5\\) a `cellUv`.

![Figura 26. coordenadas cell uv centradas.](../../images/blogs/shell-texturing/imgs/img_centered_frac_uv.png)

*Figura 26. coordenadas cell uv centradas.*

Con tal operación hemos efectivamente centraddo `cellUv`, sin embargo notamos que las celdas perdieron luminosidad, esto es debido a que al restar \\(0.5\\) a ambos elementos de `cellUv` hemos llevado las coordenadas a un rango \\([-0.5, 0.5]\\), lo cuál es algo incómodo... así que multiplicaremos `cellUv` por \\(2\\) para establecer el valor los componentes de `cellUv` dentro del rango \\([-1, 1]\\).

![Figura 27. cell uv multiplicado por dos.](../../images/blogs/shell-texturing/imgs/img_centered_frac_uv_times_2.png)

*Figura 27. cell uv multiplicado por dos.*

Ahora que tenemos el origen `cellUv` podemos dibujar un círculo de forma muy sencilla, simplemente calculamos el `length()` de cada `cellUv` respecto al origen.

![Figura 28. visualización de length(cellUV).](../../images/blogs/shell-texturing/imgs/img_cell_uv_length.png)

*Figura 28. visualización de length(cellUV).*

Luego con nuestro truco anterior, descartamos los pixeles si son mayores que un valor que llamaremos `_CellThickness` \\(\in [0, 10]\\).

![Figura 29. cambio en el grosor de cada cell uv.](../../images/blogs/shell-texturing/imgs/gif_cell_thickness.gif)
*Figura 29. cambio en el grosor de cada cell uv (Cell thickness variando de 0 a 1.5).*

Establecemos `_CellThicknes` en \\(0.6\\) y devolvemos el color para producir el siguiente resultado:

![Figura 30. cell uv cilíndrico.](../../images/blogs/shell-texturing/imgs/img_cell_thickness_no_base.png)

*Figura 30. cell uv cilíndrico.*

El pasto está claramente cilíndrico, lo cual se ve bien pero sabemos que en realidad el pasto tiene una forma más parecida a un *cono*. Por suerte nuestra variable `height` vendrá al rescate una vez más, esta vez para atenuar la contribución de `_CellThickness` en el descarte de pixeles.

![Figura 31. grosor de cell uv atenuado pero al revés.](../../images/blogs/shell-texturing/imgs/img_wrong_height_attenuation.png)

*Figura 31. grosor de cell uv atenuado pero al revés.*

Efectivamente si multiplicamos `_CellThickness` por `height` hemos obtenido una forma reminicente a un cono, pero al revés... para solucionar esto realizaremos la multiplicación `_CellThickness` con `rand - height`

![Figura 32. grosor de cell uv correctamente atenuado.](../../images/blogs/shell-texturing/imgs/img_correct_height_attenuation.png)

*Figura 32. grososr de cell uv correctamente atenuado.*

`rand - height` es la "función" que le da forma al decaimiento del grosor.

Podemos agregar un bloqueo en el descarte de los pixeles del shell cuyo `_ShellIndex = 0` para evitar los huecos en la base.

![Figura 33. base agregada.](../../images/blogs/shell-texturing/imgs/img_base_no_holes.png)
*Figura 33. base agregada.*

Para finalizar, podemos ajustar el color del pasto a un agradable verde, jugar con los parámetros y ajustar el encuadre de la cámara para obtener un resultado así:

![Figura 34. resultado final, un lindo pasto.](../../images/blogs/shell-texturing/imgs/img_result_2.png)
Figura 34. resultado final, un lindo pasto.

## Iluminación
---

Como ejercicio final, se puede mejorar el modelo de iluminación y agregar demás efectos para embellecer un render final de la técnica de shell texturing, para eso me basaré en el trabajo de Adrian Mendez [3] y su implementación del modelo de iluminación de *Genshin Impact*.

Utilizaremos la esféra para probar el modelo de iluminación.

![Figura 35. shells en una esféra con iluminación simple.](../../images/blogs/shell-texturing/imgs/img_shell_sphere_no_light.png)

*Figura 35. shells en una esféra con iluminación simple.*

Vemos la esféra con el modelo de iluminación *naive* que teníamos implementado desde antes. Lo primero que haremos para mejorar la luz es definir el color de cada pixel (\\(P_c\\)) utilizando el clásico *Diffuse Lighting Model*.

El valor de \\(P_c\\) será igual al \\(\cos{\theta}\\) entre la `normal` de la superficie del mesh y un vector *normalizado* que **apunte hacia la luz**. El cálculo se ve así:

$$P_c = N_d \cdot L_d.$$

Notamos que se utiliza el producto punto en lugar de `cos()` en la ecuación, esto es debido a la definición de `dot(x,y)` la cual define como:

$$\vec{u} \cdot \vec{v} = \left|\vec{u}\right| \left|\vec{v}\right| \cos{\theta}.$$

Es ahí donde podemos notar que si ambos vectores están normalizados (lo cual es nuestro caso), la ecuación nos queda como,

$$\vec{u} \cdot \vec{v} = 1 \times 1 \times \cos{\theta}. \\
\vec{u} \cdot \vec{v} = cos \theta. $$

Por lo que,
$$\hat{u} \cdot \hat{v} = \cos{\theta}.$$

En Unity, podemos obtener el vector normalizado que apunta hacia la dirección de la luz si consultamos el *Manual de Built-in Shader Variables [4]*, donde se encuentra estípulado que dicho vector se puede se puede acceder con la variable `_WorldSpaceLight0`. Es importante que esta variable esté en *world space*, ya que si normalizamos este vector (quitarle su magnitud), obtendremos la dirección hacia su posición desde el origen del mundo.

Antes de mostrar el resultado, destacaré el hecho de que el rango de la función \\(\cos{\theta}\\) es \\((-1 \le \cos{\theta} \le 1)\\) y al no existir luz negativa, se *clampea* el resultado al rango \\([0, 1]\\). Por lo que la ecuación final queda:

$$P_c = \max(0, N \cdot \hat{L_p}).$$

Donde \\(N\\) es la normal y \\(L_p\\) es la posición normalizada de la luz en espacio mundial.

![Figura 36. diffuse lightning simple.](../../images/blogs/shell-texturing/imgs/img_ndotl.png)

*Figura 36. diffuse lightning simple.*

Ahora como toque más artístico, controlaremos la transición entre negro a blanco con un `smoothstep` entre \\(0\\) y un valor que llamaremos `_LightSmooth` \\(\in [0, 10]\\).



![Figura 37. diffuse lightning suavizado.](../../images/blogs/shell-texturing/imgs/gif_light_smooth.gif)
*Figura 37 diffuse lightning suavizado. (Light attenuation variando de 0 a 10).*

*light smooth variando de 0 a 10.

Utilizando el valor de nuestro *diffuse* suavizado, generamos un `lerp()` entre un color de sombra a elección y un color base.

![Figura 38. color base y de sombra agregados con una interpolación linear.](../../images/blogs/shell-texturing/imgs/img_shadow_and_base_color.png)

*Figura 38. color base y de sombra agregados con una interpolación linear.*

En mi opinión, la sombra luce muy *dura*, por lo que seguiré los pasos de Acerola en su video sobre Shell Texturing y aplicaré un *Half lambert diffuse [5]*, que no es nada más que modificar la diffusión de Lambert tal que,

$$P_c = \max(0, N \cdot \hat{L_p}) \times 0.5 + 0.5.$$

para así desplazar el rango de \\(P_c\\) de \\([-1, 1]\\) originalmente a un nuevo rango \\([0, 1]\\).

![Figura 39. half lambert aplicado.](../../images/blogs/shell-texturing/imgs/img_half_lambert.png)

*Figura 39. half lambert aplicado.*

Es importante mencionar que el modelo de *Half Lambert Diffuse* es un modelo de iluminación no físicamente realista [5], ya que rompe la Ley de Coseno de Lambert [6].

Adicionalmente, podemos agregar un valor `_ShadowIntensity` \\(\in [0, 1]\\) para ajustar la intensidad de la sombra producida.

![Figura 40. shadow intensity variando de 0 a 1.](../../images/blogs/shell-texturing/imgs/gif_shadow_intensity.gif)
*Figura 40. shadow intensity variando de 0 a 1.*

Para finalizar ajustamos los parámetros creados a lo largo del blog, buscamos un modelo interesante, un skybox acorde a la estética y componemos una bonita escena =).

![Figura 41. resultado final. (utilicé la composición golden triangle).](../../images/blogs/shell-texturing/imgs/gif_hand_result.gif)
*Figura 41. resultado final. (utilicé la composición golden triangle).*


## Bibliografía
---
[1] G. Gunell "Acerola", “How Are Games Rendering Fur?,” YouTube, Oct. 30, 2023. https://youtu.be/9dr-tRQzij4?si=OBngN7l8wAV_BQ98 (accessed Jun. 27, 2024).

[2] M. Bischoff, “The World’s Simplest Theorem Shows That 8,000 People Globally Have the Same Number of Hairs on Their Head,” Scientific American, Mar. 20, 2023. https://www.scientificamerican.com/article/the-worlds-simplest-theorem-shows-that-8-000-people-globally-have-the-same-number-of-hairs-on-their-head/ (accessed Jun. 13, 2024).


[3] A. Mendez, “Genshin Impact Character Shader Breakdown [Unity URP],” adrianmendez.artstation.com, Feb. 27, 2022. https://adrianmendez.artstation.com/projects/wJZ4Gg (accessed Jun. 27, 2024).

[4] Unity Technologies, “Unity - Manual: Built-in Shader Variables,” docs.unity3d.com. https://docs.unity3d.com/Manual/SL-UnityShaderVariables.html (accessed Jun. 27, 2024).

[5] Valve, “Half Lambert - Valve Developer Community,” developer.valvesoftware.com, Jan. 07, 2024. https://developer.valvesoftware.com/wiki/Half_Lambert (accessed Jun. 28, 2024).

[6] Wikipedia Contributors, “Lambert’s Cosine Law,” Wikipedia, Jan. 29, 2021. https://en.wikipedia.org/wiki/Lambert%27s_cosine_law (accessed Jun. 28, 2024).

