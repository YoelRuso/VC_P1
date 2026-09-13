# Práctica 1. Primeros pasos con OpenCV

Visión por Computador — Grado en Ingeniería Informática, ULPGC (curso 2025/2026).

## Autoría

- Joel Morera Apaza

Repositorio: <https://github.com/YoelRuso/VC_P1>

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `VC_P1.ipynb` | Cuaderno con los ejemplos de clase y la resolución de las cuatro tareas |
| `pop-art-gary-grayson.jpg` | Patrón de estrellas usado como fondo en la tarea 4 |
| `popart_01.png` … `popart_04.png` | Capturas del resultado de la tarea 4 |
| `logo_ulpgc_vertical_acronimo_mancheta_azul.png` | Imagen de ejemplo para la lectura desde disco |
| `VermeerFun.mp4` | Vídeo de ejemplo para la lectura de fotogramas |
| `imagen.jpg` | Salida generada por la celda de primitivas de dibujo |
| `spec-list.txt` | Lista de paquetes proporcionada con el enunciado |

## Entorno y ejecución

El cuaderno se ha desarrollado sobre un *environment* de Anaconda con **Python 3.11.14**:

```
conda create --name VC_P1 python=3.11.5
conda activate VC_P1
pip install opencv-python matplotlib
```

**No hace falta ninguna instalación adicional** más allá de eso. La única dependencia opcional es
`Pillow`: la tarea 4 incluye una función `cargar_imagen()` que recurre a PIL únicamente si
`cv2.imread` falla al abrir el patrón de fondo (por ejemplo con un `.webp` en compilaciones de
OpenCV sin ese soporte). Con el `.jpg` que viene en el repositorio nunca se llega a usar.

Las tareas 3 y 4 **necesitan webcam**. Las ventanas se cierran pulsando **ESC**; se abren en una
ventana independiente de OpenCV, no integradas en el cuaderno, por lo que esas celdas no dejan
salida guardada en el `.ipynb`.

---

## Tarea 1 — Tablero de ajedrez

Imagen de 800×800 con casillas de 100 px, resuelta dos veces para poder comparar.

**Versión manual.** 32 llamadas explícitas a `cv2.rectangle` sobre un lienzo `np.zeros`, una por
casilla blanca, escribiendo a mano las coordenadas de cada esquina.

**Versión con IA.** Dos bucles anidados sobre filas y columnas que pintan la casilla cuando
`(fila + columna) % 2 == 0`.

**Comparación.** La versión manual sirvió para entender los parámetros de `cv2.rectangle` y el
sistema de coordenadas del lienzo, pero es larga, repetitiva y está atada a un tamaño concreto: un
tablero de otras dimensiones obliga a reescribirla entera. La versión con bucles es mucho más corta
y parametrizable — cambiando `celda` o el tamaño del lienzo sigue funcionando. En coste de cómputo
son equivalentes: ambas acaban haciendo 32 llamadas a `cv2.rectangle`, así que la diferencia está en
la mantenibilidad, no en el rendimiento. Se conservan las dos en el cuaderno: la manual como parte
del aprendizaje y la del bucle como la versión que se entregaría.

## Tarea 2 — Imagen estilo Mondrian

Resuelta **sin herramientas de IA**, partiendo del ejemplo de primitivas de dibujo del cuaderno.
Sobre un lienzo blanco de 300×200 (`np.full(..., 255)`) se trazan primero las líneas negras que
forman la retícula con `cv2.line` (grosor 3) y después se rellenan los huecos con `cv2.rectangle`
en modo relleno (`-1`) usando los tres colores primarios de Mondrian: rojo, amarillo y azul. Las
coordenadas se ajustaron a mano para reproducir la composición asimétrica característica del autor,
con rectángulos de tamaños desiguales y algunos huecos en blanco.

## Tarea 3 — Píxel más claro y más oscuro

Partiendo de la celda del manejador de ratón, en cada fotograma se convierte a gris con
`cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)` y se localizan los extremos con `cv2.minMaxLoc`, que
devuelve valor y posición del mínimo y del máximo en una sola pasada. Sobre esas posiciones se
dibuja un círculo azul (más oscuro) y otro rojo (más claro), y se mantiene el texto con los valores
RGB bajo el puntero.

**¿Va fluido o a saltos?** Va fluido. La búsqueda se hace sobre la imagen en gris, es decir sobre un
único plano en lugar de tres, y `cv2.minMaxLoc` está implementado en C: recorre el array una sola
vez sin que el bucle pase por el intérprete de Python. El coste por fotograma es despreciable frente
a la propia captura de la cámara.

**¿Cómo se aceleraría si fuera a saltos?** La solución ingenua —recorrer los píxeles con bucles
`for` en Python— sí se arrastra, y ahí las opciones serían: usar las operaciones vectorizadas de
NumPy/OpenCV en lugar del bucle (que es justo lo que se hace aquí), reducir la resolución antes de
buscar con `cv2.resize` o `cv2.pyrDown` y reescalar después las coordenadas encontradas, o limitar
la búsqueda a una región de interés en vez de al fotograma completo.

## Tarea 4 — Propuesta propia de pop art

Collage 2×2 en directo inspirado en la *Marilyn* de Warhol, pero llevado a una posterización
configurable en lugar de la separación de canales del ejemplo de clase.

Cada fotograma se procesa así:

1. **Espejo y realce.** Se voltea horizontalmente y, opcionalmente, se ecualiza el contraste local
   con **CLAHE** para que los tonos queden bien repartidos con cualquier iluminación.
2. **Aplanado.** `cv2.bilateralFilter` + `cv2.medianBlur` eliminan el ruido conservando los bordes,
   de modo que las zonas de color salen limpias y no moteadas.
3. **Posterización a 4 tonos.** Se calculan tres umbrales —fijos o automáticos por **percentiles**
   de la imagen, lo que adapta el resultado a la luz de la escena— y se construye una **LUT** que
   `cv2.LUT` aplica de golpe. Se usan cuatro paletas distintas, una por cuadrante.
4. **Patrón de fondo.** Uno de los cuatro niveles de tono (elegible) se sustituye por un patrón de
   estrellas en vez de un color plano. El patrón se separa en motivo y fondo con **umbralizado de
   Otsu** y se repinta con dos colores distintos en cada cuadrante; además se voltea en cada uno
   para que no se vea repetido. Existe un segundo modo, `tinte`, que en lugar de repintar desplaza
   el tono en **HSV**, válido para cualquier imagen y no solo para tramas planas.
5. **Contorno.** `cv2.Canny` + `cv2.dilate` añaden una línea negra sobre los bordes, que es lo que
   da el aire de serigrafía.

Los cuatro cuadrantes son **vistas** (*slices*) del array del collage, así que se escriben en su
sitio sin copias intermedias, y los patrones de fondo se recalculan solo al cambiar un ajuste, no en
cada fotograma.

**Controles interactivos:**

| Tecla | Acción |
|---|---|
| `ESC` | Salir |
| `f` | Activa/desactiva el patrón de fondo |
| `0`–`3` | Nivel de tono que muestra el patrón (0 = más oscuro, 3 = más claro) |
| `p` | Rota qué combinación de colores va a cada cuadrante |
| `m` | Alterna el modo de recoloreado: `dos_tonos` / `tinte` |
| `a` | Alterna umbrales automáticos (percentiles) / fijos |
| `c` | Activa/desactiva la línea de contorno |
| `g` | Activa/desactiva el realce de contraste (CLAHE) |
| `+` / `-` | Desplaza los umbrales (más claro / más oscuro) |
| `s` | Guarda un PNG del collage (`popart_NN.png`) |

**Resultados.** Capturas tomadas con la tecla `s` durante la ejecución, variando los ajustes desde
el teclado:

| | |
|:--:|:--:|
| ![Patrón de estrellas sobre el tono medio](popart_01.png) | ![Posterización sin patrón](popart_02.png) |
| Patrón de estrellas sustituyendo al tono medio, con contorno Canny activado. Cada cuadrante recolorea el patrón con su propia pareja de colores y lo voltea. | El mismo encuadre con el patrón desactivado (`f`): los cuatro tonos quedan en color plano y se aprecia mejor la posterización. |
| ![Posterización pura sin contorno](popart_03.png) | ![Patrón sobre otro nivel de tono](popart_04.png) |
| Sin patrón y sin contorno (`c`): posterización limpia a cuatro tonos, más cercana a la serigrafía original. | El patrón movido a otro nivel de tono con las teclas `0`–`3` y otra rotación de paletas (`p`). |

---

## Fuentes consultadas

- Documentación de OpenCV: [funciones de dibujo](https://docs.opencv.org/4.x/dc/da5/tutorial_py_drawing_functions.html),
  [`minMaxLoc`](https://docs.opencv.org/4.x/d2/de8/group__core__array.html#gab473bf2eb6d14ff97e89b355dac20707),
  [`LUT`](https://docs.opencv.org/4.x/d2/de8/group__core__array.html#gab55b8d062b7f5587720ede032d34156f),
  [umbralizado y método de Otsu](https://docs.opencv.org/4.x/d7/d4d/tutorial_py_thresholding.html),
  [detector de bordes de Canny](https://docs.opencv.org/4.x/da/d22/tutorial_py_canny.html),
  [ecualización de histograma y CLAHE](https://docs.opencv.org/4.x/d5/daf/tutorial_py_histogram_equalization.html),
  [filtrado que preserva bordes](https://docs.opencv.org/4.x/d4/d13/tutorial_py_filtering.html)
- [`numpy.digitize`](https://numpy.org/doc/stable/reference/generated/numpy.digitize.html) y
  [`numpy.percentile`](https://numpy.org/doc/stable/reference/generated/numpy.percentile.html)
- Piet Mondrian, *Composición con rojo, amarillo y azul* (1930) — referencia visual de la tarea 2.
  [Descubriendo a Mondrian](https://www3.gobiernodecanarias.org/medusa/ecoescuela/sa/2017/04/17/descubriendo-a-mondrian/)
- Andy Warhol, *Marilyn Diptych* (1962) — referencia visual de la tarea 4.
  [Comentario de la obra](https://temasycomentariosartepaeg.blogspot.com/p/autor-andy-warhol-1928-1987-titulo.html)
- `pop-art-gary-grayson.jpg`: patrón de estrellas de stock descargado de
  https://previews.123rf.com/images/lenalanette/lenalanette1707/lenalanette170700018/82052979-pop-art-background-with-stars-seamless-vector-illustration.jpg.
  Se usa únicamente como textura decorativa de fondo dentro del cuaderno.
- Material y cuaderno base de la asignatura, proporcionados por el profesorado de Visión por
  Computador (ULPGC).

## Uso de herramientas de IA

- **Tarea 1 (versión con IA).** El propio enunciado pide resolver el tablero primero a mano y
  después con un asistente de IA. Se le pidió la versión con bucles; la comparación entre ambas
  está en la sección [Tarea 1](#tarea-1--tablero-de-ajedrez).
- **Tarea 2.** Resuelta sin asistentes de IA, tal y como pedía el enunciado.
- **Tarea 3.** Se consultó cómo obtener las posiciones del píxel más claro y más oscuro, de donde
  salió el uso de `cv2.minMaxLoc`.
  Conversación: <https://claude.ai/share/690d1b84-02da-409f-a86b-91d6bf7fba5b>
- **Tarea 4.** Se usó como apoyo para la propuesta de pop art (posterización con LUT, separación
  del patrón con Otsu y ajuste de los umbrales por percentiles).
  Conversación: <https://claude.ai/share/3c25dee0-2184-4904-a24d-de7b45f457a0>

---

Enunciado original de la práctica bajo licencia Creative Commons Reconocimiento - No Comercial 4.0
Internacional.
