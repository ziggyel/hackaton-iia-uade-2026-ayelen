# T19 · ¿Bajamos la dificultad de este jefe?

> Hackatón IIA · UADE · 3.4.257 · 3 de noviembre de 2026.

**Integrantes:** [COMPLETAR INTEGRANTES]  
**IA declarada:** ChatGPT  
**Pista:** Reglas  
**Publicación en LinkedIn:** [AGREGAR ENLACE DESPUÉS DE PUBLICAR]  
**Notebook ejecutable:** [AGREGAR ENLACE A COLAB O AL NOTEBOOK DE SU FORK]

## 1. Formulación y PEAS

**La pregunta.** Para cada jugador que está trabado con el jefe final, ¿abandonaría el videojuego si no se baja la dificultad? **SÍ (1)** significa que abandonaría; **NO (0)** significa que no abandonaría. El sistema recomienda bajar la dificultad cuando predice SÍ y mantenerla cuando predice NO.

| PEAS | Nuestro sistema |
| --- | --- |
| **Medida de rendimiento** | Priorizar la sensibilidad (detectar a los jugadores que abandonarían), revisar falsos negativos y controlar los falsos positivos para no arruinar el desafío. También informar exactitud, precisión y especificidad. |
| **Entorno** | Videojuego con un jefe final y jugadores que intentan derrotarlo. La evaluación se hace durante los intentos, con los datos disponibles hasta ese momento. |
| **Actuadores** | Recomendar **bajar** o **mantener** la dificultad del jefe para el jugador actual. |
| **Sensores** | Muertes seguidas contra el jefe, porcentaje de vida que le quedó al jefe en el mejor intento y horas totales jugadas. El CSV trae además un porcentaje global de victorias, que decidimos no usar como entrada. |

**Objetivo y métrica.** El objetivo es prevenir abandonos por frustración sin eliminar innecesariamente el desafío. El estudio sigue el porcentaje global de jugadores que derrotan al jefe, pero aumentar ese porcentaje **no es lo mismo** que reducir abandonos: podría subir solo porque el jefe se volvió demasiado fácil. Para la predicción priorizamos la **sensibilidad** y la cantidad de falsos negativos, ya que dejar sin ayuda a alguien que abandonaría es el error más costoso según la ficha. Estas métricas son aproximaciones y no prueban por sí solas que una intervención realmente retenga jugadores.

## 2. El sistema

**Pista elegida: reglas si-entonces.** Primero analizamos **únicamente los 14 casos de entrenamiento**. En ellos, los siete jugadores con resultado real SÍ tienen como máximo 43 horas jugadas, y los siete con NO tienen como mínimo 47. Elegimos el punto medio, 45 horas, como umbral sencillo de partida.

**Regla operativa:**

```python
def decidir(muertes, vida_restante, horas):
    if horas <= 45:
        return 1  # SÍ: recomendamos bajar la dificultad
    return 0      # NO: recomendamos mantenerla
```

**Variables que usa y por qué.** La versión inicial utiliza solo **horas jugadas**, porque esta variable separa los 14 casos de entrenamiento. **Muertes seguidas** y **vida restante del jefe** se reciben y se validan, pero esta primera regla no las usa para clasificar. Esta es una limitación importante: el tiempo jugado no mide directamente la frustración. No utilizamos el **porcentaje global de jugadores que derrotan al jefe** porque es un indicador agregado del juego, no una medida individual de abandono ni el objetivo que debemos optimizar. Tampoco usamos `resultado_real` como predictor: solo sirve para medir el sistema después.

**Cómo se corre en vivo.** Abrir `modelo_T19.ipynb` en Google Colab, ejecutar las celdas desde el principio y ejecutar la última celda. Reemplazar los valores de las tres variables del caso nuevo (muertes, vida restante del jefe, horas jugadas). La última celda mostrará SÍ o NO y la acción recomendada. El programa descarga el CSV T19 original de la cátedra; si no hay conexión, permite subir el archivo CSV al entorno.

| Caso inventado | Valores (muertes, vida restante, horas) | Salida del sistema |
| --- | --- | --- |
| A | 12, 55 %, 35 | **SÍ**, recomendar bajar la dificultad. |
| B | 17, 50 %, 92 | **NO**, mantener la dificultad. |

## 3. Medición

Los casos **1 al 14** son de entrenamiento y los **15 al 20** son de prueba. La división no se cambió. La regla se fijó a partir del entrenamiento; los casos de prueba solo se usaron para evaluarla, no para cambiar el umbral. Las matrices se pueden reproducir ejecutando el notebook.

**Entrenamiento (14 casos)**

| | Predice SÍ | Predice NO |
| --- | ---: | ---: |
| **Real SÍ** | VP = **7** | FN = **0** |
| **Real NO** | FP = **0** | VN = **7** |

Suma: 7 + 0 + 0 + 7 = **14**.

**Prueba (6 casos)**

| | Predice SÍ | Predice NO |
| --- | ---: | ---: |
| **Real SÍ** | VP = **2** | FN = **2** |
| **Real NO** | FP = **0** | VN = **2** |

Suma: 2 + 2 + 0 + 2 = **6**.

| Métrica | Entrenamiento | Prueba |
| --- | ---: | ---: |
| Exactitud = (VP + VN) / total | **100 %** | **66,7 %** |
| Precisión = VP / (VP + FP) | **100 %** | **100 %** |
| Sensibilidad = VP / (VP + FN) | **100 %** | **50 %** |
| Especificidad = VN / (VN + FP) | **100 %** | **100 %** |

**Entrenamiento contra prueba.** La caída de exactitud del 100 % al 66,7 % y de sensibilidad del 100 % al 50 % muestra que la regla no generaliza perfectamente. En particular, dejó sin detectar **dos** casos de prueba en los que habría abandono (casos 18 y 20). Es una señal de posible sobreajuste a la muestra de entrenamiento: el umbral separa muy bien esos 14 registros, pero no describe por completo el comportamiento de jugadores nuevos. La prueba es muy pequeña (seis casos), así que las cifras no estiman con precisión el rendimiento futuro.

## 4. El error más caro

**El falso negativo es el más costoso:** el sistema responde NO, no recomienda bajar la dificultad y el jugador que realmente iba a abandonar termina yéndose. El falso positivo también importa porque podemos quitarle el desafío a alguien que habría seguido jugando.

**Qué cambiamos por este error.** Tomamos como punto de partida la regla trivial «responder siempre NO», que en entrenamiento dejaría sin detectar **7 abandonos (FN = 7)**. La reemplazamos por una regla basada en la separación observada en el entrenamiento, que reduce los falsos negativos de entrenamiento de 7 a **0**, sin generar falsos positivos en ese conjunto. En prueba todavía quedan **2 falsos negativos**. No reajustamos el umbral usando sus etiquetas porque contaminaríamos la evaluación. Una mejora posterior requeriría más casos de entrenamiento y considerar también muertes y progreso contra el jefe, con nueva evaluación independiente.

## 5. ¿Comparado contra qué?

**Línea base trivial: responder siempre NO.** Se eligió como referencia simple de «no intervenir nunca». En entrenamiento da 7 aciertos de 14 (**50 %** de exactitud), 0 % de sensibilidad y 7 falsos negativos. En prueba da 2 aciertos de 6 (**33,3 %** de exactitud), 0 % de sensibilidad y 4 falsos negativos. Nuestra regla da **100 %** de exactitud y sensibilidad en entrenamiento, y **66,7 %** de exactitud con **50 %** de sensibilidad en prueba. La mejora frente a esa línea base no significa que el modelo sea confiable en situaciones reales. Como las clases de entrenamiento están empatadas (7 SÍ, 7 NO), responder siempre SÍ también sería una línea base trivial válida; tendría 50 % en entrenamiento y 66,7 % en prueba, pero generaría falsos positivos.

| Checklist (Clase 6) | Respuesta |
| --- | --- |
| **¿Qué datos?** | 20 casos ficticios del CSV T19, sin datos de personas reales: 14 para entrenamiento y 6 para prueba. |
| **¿Qué métrica?** | Sensibilidad como prioridad por el costo del falso negativo; también exactitud, precisión, especificidad y conteos de errores. |
| **¿Comparado contra qué?** | Línea base trivial que siempre responde NO; también consideramos siempre SÍ para evitar una comparación engañosa. |
| **¿Quién lo midió?** | El grupo, mediante cálculo reproducible en Python, contrastando las predicciones con `resultado_real` del CSV. [CONFIRMAR POR EL GRUPO DESPUÉS DE EJECUTARLO] |

## 6. Riesgo: el camino completo

**Datos → modelo → decisión → consecuencia:** si el registro indica pocas horas, nuestra regla puede predecir abandono sin considerar que el jugador está a punto de derrotar al jefe. Eso lleva a **bajar la dificultad innecesariamente**, quitándole la sensación de logro. En sentido contrario, un jugador con muchas horas jugadas puede estar muy frustrado y la regla podría recomendar **mantener la dificultad**, provocando un abandono. Por eso no proponemos una intervención irreversible ni tratar la predicción como una certeza: en un videojuego real, convendría ofrecer la ayuda como opción y medir abandono y satisfacción con evaluaciones adicionales.

## 7. Bitácora

**Uso de IA (borrador para verificar y completar con las decisiones reales del grupo).** En las interacciones documentadas hasta el momento se utilizó ChatGPT para comprender la consigna, buscar la ficha T19, revisar el CSV y preparar una propuesta inicial. Antes de entregar, el grupo debe registrar qué propuestas aceptó, rechazó o cambió y por qué. No atribuir al grupo acciones que todavía no realizó.

| # | Qué le pedimos | Qué aceptamos | Qué rechazamos y por qué |
| --- | --- | --- | --- |
| 1 | Explicar las bases de la hackatón y las obligaciones del evaluador. | [COMPLETAR TRAS REVISIÓN DEL GRUPO] | [COMPLETAR TRAS REVISIÓN DEL GRUPO] |
| 2 | Identificar la problemática T19 y revisar los datos. | [COMPLETAR TRAS REVISIÓN DEL GRUPO] | [COMPLETAR TRAS REVISIÓN DEL GRUPO] |
| 3 | Proponer un sistema simple de reglas, sus métricas y una estructura README. | [COMPLETAR TRAS REVISIÓN DEL GRUPO] | [COMPLETAR TRAS REVISIÓN DEL GRUPO] |

**Corridas contra el evaluador.**

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
| --- | --- | --- | --- |
| **Pendiente** | — | Todavía no se consultó al bot. | — |

> Antes de entregar: completar los integrantes, verificar el código en Colab, hacer commits reales en el fork, completar la bitácora con sus decisiones y pegar acá las respuestas reales del bot. Publicar en LinkedIn y agregar el enlace. No inventar evaluaciones.
