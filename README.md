# T19 · ¿Bajamos la dificultad de este jefe?

> Hackatón IIA · UADE · 3.4.257 · 3 de noviembre de 2026.

**Integrantes:** Ayelen Mendoza Luque (grupo individual)  
**IA declarada:** ChatGPT  
**Pista:** Reglas  
**Publicación en LinkedIn:** [AGREGAR ENLACE DESPUÉS DE PUBLICAR]  
**Notebook ejecutable:** [Enlace](https://colab.research.google.com/github/ziggyel/hackaton-iia-uade-2026-ayelen/blob/main/T19%20-%20Primera%20versi%C3%B3n%20del%20sistema%20de%20reglas.ipynb#scrollTo=05a2435c)

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

**Cómo se corre en vivo.** Abrir `T19 - Primera versión del sistema de reglas.ipynb` en Google Colab, ejecutar las celdas desde el principio y ejecutar la última celda. Reemplazar los valores de las tres variables del caso nuevo (muertes, vida restante del jefe, horas jugadas). La última celda mostrará SÍ o NO y la acción recomendada. El programa descarga el CSV T19 original de la cátedra; si no hay conexión, permite subir el archivo CSV al entorno.

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

**Línea base trivial: responder siempre NO.** Se eligió como referencia simple de «no intervenir nunca». En entrenamiento da 7 aciertos de 14 (**50 %** de exactitud), 0 % de sensibilidad y 7 falsos negativos. En prueba da 2 aciertos de 6 (**33,3 %** de exactitud), 0 % de sensibilidad y 4 falsos negativos. Nuestra regla da **100 %** de exactitud y sensibilidad en entrenamiento, y **66,7 %** de exactitud con **50 %** de sensibilidad en prueba. La mejora frente a esa línea base no significa que el modelo sea confiable en situaciones reales. Como las clases de entrenamiento están empatadas (7 SÍ, 7 NO), responder siempre SÍ también sería una línea base trivial válida. Tendría 50 % de exactitud en entrenamiento y 66,7 % en prueba. En prueba lograría 100 % de sensibilidad (ningún abandono sin detectar), pero generaría 2 falsos positivos: les bajaríamos la dificultad a dos jugadores que no abandonarían. Nuestra regla tiene la misma exactitud en prueba (66,7 %), evita esos 2 falsos positivos, pero deja sin detectar 2 abandonos. Por eso no afirmamos que una estrategia sea universalmente mejor: el equilibrio depende del costo de perder jugadores frente al costo de quitarles el desafío.

| Checklist (Clase 6) | Respuesta |
| --- | --- |
| **¿Qué datos?** | 20 casos ficticios del CSV T19, sin datos de personas reales: 14 para entrenamiento y 6 para prueba. |
| **¿Qué métrica?** | Sensibilidad como prioridad por el costo del falso negativo; también exactitud, precisión, especificidad y conteos de errores. |
| **¿Comparado contra qué?** | Línea base trivial que siempre responde NO; también consideramos siempre SÍ para evitar una comparación engañosa. |
| **¿Quién lo midió?** | El grupo, mediante cálculo reproducible en Python, contrastando las predicciones con `resultado_real` del CSV. El notebook se ejecutó en Colab y se observaron las métricas de ambos conjuntos. |

## 6. Riesgo: el camino completo

**Datos → modelo → decisión → consecuencia:** si el registro indica pocas horas, nuestra regla puede predecir abandono sin considerar que el jugador está a punto de derrotar al jefe. Eso lleva a **bajar la dificultad innecesariamente**, quitándole la sensación de logro. En sentido contrario, un jugador con muchas horas jugadas puede estar muy frustrado y la regla podría recomendar **mantener la dificultad**, provocando un abandono. Por eso no proponemos una intervención irreversible ni tratar la predicción como una certeza: en un videojuego real, convendría ofrecer la ayuda como opción y medir abandono y satisfacción con evaluaciones adicionales.

## 7. Bitácora

**Registro del uso de IA — 3 de noviembre de 2026.** Usé ChatGPT como apoyo para interpretar la consigna, ubicar mi ficha y los datos, elegir una solución, preparar el código, calcular las métricas y redactar/revisar el README. Abajo se registran las decisiones que quedaron reflejadas en el proyecto; no se atribuyen consultas al evaluador que todavía no ocurrieron.

| # | Qué le pedí a ChatGPT | Qué acepté o utilicé | Qué descarté, cambié o limité, y por qué |
| --- | --- | --- | --- |
| 1 | Explicarme las bases de la hackatón, las pruebas obligatorias y la prohibición de inyección de prompts. | Organicé el proyecto alrededor de la pregunta SÍ/NO, el PEAS, la medición, la comparación y la defensa. | No incluí instrucciones para alterar la corrección del evaluador: está prohibido y no ayuda a demostrar que el sistema funciona. |
| 2 | Buscar la problemática correspondiente al código T19 y explicar los datos. | Trabajé con la ficha «¿Bajamos la dificultad de este jefe?» y el CSV de 20 casos. Respeté los casos 1–14 como entrenamiento y 15–20 como prueba. | No utilicé la columna de resultado real como entrada ni las etiquetas de prueba para diseñar el umbral: hacerlo filtraría información que no corresponde conocer al decidir. |
| 3 | Proponer una IA sencilla para resolver el caso y poder ejecutarla en vivo. | Utilicé un **sistema de reglas** con el umbral de **45 horas**, calculado a partir de los valores 43 y 47 observados en entrenamiento. La salida es SÍ/NO y su acción asociada es bajar/mantener dificultad. | No elegí perceptrón ni agente porque, para esta primera versión con pocos registros, la regla es más fácil de reproducir y defender. No agregué muertes ni vida restante a la decisión porque la regla inicial usa solo horas; reconozco esa limitación. |
| 4 | Preparar las matrices de confusión y las métricas. | Incorporé VP, FN, FP y VN; exactitud, precisión, sensibilidad y especificidad por separado para entrenamiento y prueba. Ejecuté el notebook en Colab y vi los resultados esperados: **14/14 aciertos en entrenamiento y 4/6 en prueba**. | No presenté el 100 % de entrenamiento como prueba de funcionamiento perfecto: en prueba la sensibilidad cae al 50 % y hay dos falsos negativos. |
| 5 | Explicar el error más caro y comparar contra sistemas triviales. | Prioricé discutir los falsos negativos, porque implican no asistir a alguien que abandonaría. Comparé la regla con **siempre NO** y también con **siempre SÍ**, para mostrar el costo de detectar más abandonos a cambio de bajar dificultad de forma innecesaria. | No tomé «porcentaje de jugadores que derrotan al jefe» como objetivo principal: ganar más peleas no demuestra reducir abandonos ni conservar el desafío. Tampoco ajusté la regla según los seis resultados de prueba. |
| 6 | Preparar el README y el programa para GitHub. | Copié un borrador del README a mi fork, subí el notebook y el CSV, y registré cambios mediante commits. | Corregí la referencia inicial al nombre del archivo del notebook para que coincida con el subido; también amplié la explicación de la línea base siempre SÍ para no dar una comparación incompleta. |
| 7 | Guiarme para probar el programa en Google Colab y encontrar las salidas. | Ejecuté las celdas, pude ver las métricas y la demostración de un caso nuevo. El ejemplo **12 muertes, 55 % de vida restante, 35 horas** produjo **SÍ: bajar dificultad**. | No cambié el umbral después de ver la prueba: mantuve la separación entre construcción con entrenamiento y evaluación con prueba. |

**Corridas contra el evaluador.** Se registran los resultados reales del bot. **Corrida 1: 9,5/10; corrida 2: 10/10**, ambas habilitadas para la mesa y sin topes aplicados. En la primera, la bitácora recibió 0,5/1 porque aún no registraba corridas. Tras documentar la primera evaluación y las respuestas a sus observaciones, la segunda obtuvo 1/1 en bitácora y puntaje completo en los siete criterios.

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
| --- | --- | --- | --- |
| **1 (3/11/2026)** | **9,5/10** | (1) Los casos 18 y 20 son falsos negativos: el umbral de 45 horas no considera muertes ni progreso contra el jefe. (2) El umbral deriva de solo 14 ejemplos y puede no generalizar a otro juego o muestra. (3) El bot indicó que el nombre del README no coincide con el registro de T19. La integrante confirmó que trabaja sola; queda verificar el nombre asentado por la cátedra. (4) Faltaba registrar esta corrida. | **Documentación:** registramos esta evaluación y las respuestas a sus preguntas. **Modelo:** no cambiamos la regla usando las etiquetas de prueba, porque eso contaminaría la evaluación. **Integrantes:** se aclaró que T19 es un grupo de una sola integrante. No cambiamos el nombre declarado sin conocer el registro oficial. |
| **2 (3/11/2026)** | **10/10** | El evaluador confirmó todos los criterios completos. Preguntó por el **costo de los dos falsos negativos de prueba** y cómo disminuirlos **sin mirar las etiquetas de prueba**. También recordó verificar el registro oficial de T19. | **Documentación:** registramos esta segunda evaluación. **Defensa:** preparamos una explicación del costo de perder dos jugadores que abandonarían sin ayuda y una propuesta futura para probar reglas con muertes y progreso, ajustadas exclusivamente con datos de entrenamiento adicionales y evaluadas sobre casos nuevos. **Modelo entregado:** se mantiene sin cambios para no contaminar la prueba. El registro oficial de integrantes sigue pendiente de confirmación con el docente. |

**Respuesta al agujero principal (casos 18 y 20).** El caso **18** tiene **70 horas jugadas, 18 muertes seguidas y 44 % de vida restante del jefe** en su mejor intento. El caso **20** tiene **67 horas jugadas, 22 muertes y 83 % de vida restante del jefe**. Como ambos superan las 45 horas, la regla predice NO, aunque su etiqueta real es SÍ. La regla **no aprovecha las muchas muertes ni el progreso frente al jefe**, que podrían señalar frustración. Tener más horas de juego no significa necesariamente que la persona vaya a continuar. Estos casos muestran una limitación, pero no justifican reajustar la regla mirando las etiquetas de prueba.

**Confiabilidad del umbral.** Las 45 horas resultan del punto medio entre 43 (máximo de horas entre los SÍ de entrenamiento) y 47 (mínimo entre los NO). Con solo **14 registros de entrenamiento**, esa separación puede ser casual o específica de la muestra. Si cambian los jugadores o el juego, la distribución podría cambiar y la regla perder rendimiento. Para mejorarla harían falta más datos de entrenamiento, explorar muertes y vida restante y evaluar cualquier nueva versión en datos de prueba independientes.



**Respuesta para la defensa sobre los falsos negativos.** En el conjunto de prueba hay 4 jugadores que abandonarían y el sistema deja sin detectar 2; es decir, no recomienda intervenir en **la mitad de los abandonos reales de prueba**. En un juego, el costo posible es perder esos jugadores y su participación futura, aunque estos 20 casos ficticios no permiten calcular una pérdida económica exacta ni demostrar que bajar dificultad habría evitado el abandono. Para disminuir falsos negativos en una **versión posterior**, reuniríamos más ejemplos de entrenamiento y estudiaríamos muertes seguidas y progreso (vida que le quedó al jefe) junto con las horas. Ajustaríamos una regla nueva solo con entrenamiento, compararíamos su sensibilidad y falsos positivos y la evaluaríamos finalmente con un conjunto nuevo nunca usado para ajustar el sistema. No modificamos el umbral actual mirando las etiquetas de los casos 15–20.

**Pendiente de verificar antes de la mesa:** la estudiante confirmó que es la única integrante de T19. El bot indicó una discrepancia con el nombre registrado; corresponde consultar al profesor si el nombre que figura en el registro oficial está correcto. No se modificó el nombre declarado sin esa comprobación. También falta publicar en LinkedIn y agregar el enlace.
