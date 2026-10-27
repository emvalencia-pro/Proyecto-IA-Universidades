# Trabajo Práctico 2 – Aprendizaje de Máquinas

**Universidad Nacional de Colombia – Sede Medellín**
**Introducción a la Inteligencia Artificial – 3010476**
**Valor:** 12%
**Profesor:** Jaime Alberto Guzmán Luna

- **Fecha de entrega:** 10 de noviembre, hora de cierre 9:50 am.
- **Fecha de sustentación:** 10 – 12 de noviembre en el horario de clase.
- **Componentes:** Aprendizaje de máquinas supervisado y no supervisado.

> Fuente: `practica-2/Practica2-2026-2.pdf`

---

## Objetivos

**Objetivo general:** Desarrollar una aplicación que, a partir de un dataset abierto, implemente y compare diferentes técnicas de aprendizaje de máquinas para la tarea de clasificación y agrupamiento.

**Objetivos específicos:**
- Obtener y preparar un dataset con un dominio elegido por los estudiantes.
- Aplicar técnicas de preprocesamiento como normalización de los datos (escalar los datos y conversión de características categóricas), manejo de outliers y balanceo de clases.
- Implementar algoritmos de aprendizaje supervisado (Árboles de Decisión, KNN, SVM y Redes Neuronales) para la tarea de clasificación.
- Implementar algoritmos de aprendizaje no supervisado (K-Means y DBSCAN) para la tarea de agrupamiento.
- Evaluar los modelos usando las métricas vistas en clase y las curvas ROC/Precision-Recall.
- Comparar resultados con diferentes técnicas y configuraciones.
- Visualizar los resultados y analizar el impacto de los métodos usados y sus configuraciones.

---

## 1. Descripción general

Los estudiantes deben seleccionar un dataset de acceso libre, con **al menos 4 clases** para la variable objetivo. Se abordará la clasificación supervisada usando múltiples técnicas y luego se retirará la etiqueta objetivo para realizar agrupamiento no supervisado.

---

## 2. Aprendizaje supervisado

### 2.1. Selección del dataset y preprocesado

- **Selección y justificación** del dataset de acceso libre, con al menos 4 clases para la variable objetivo. Debe tener **al menos 5000 registros**, con **características numéricas y categóricas**, y cumplir:

  > **Cada equipo deberá elegir un dominio de dataset diferente y generar un subconjunto único utilizando 5000 registros (como mínimo), más el último dígito diferente de cero de la cédula de un integrante del equipo multiplicado por 150**; por ejemplo, si la cédula del integrante elegido termina en 6, el grupo deberá utilizar 5900 registros. Asimismo, cada grupo deberá seleccionar un **valor de semilla (random state)** correspondiente a los **dos últimos dígitos diferentes de cero de la misma cédula, evitando el uso del valor 42**. **Es obligatorio indicar en la entrega la cédula utilizada.**

- **Preprocesamiento:** generar las diferentes versiones del dataset detalladas en la Figura 1 con los siguientes procedimientos:
  - **Normalización de los datos:** (i) conversión de características categóricas (CC) y (ii) escalado de los datos (ED).
    - **Conversión de categóricas (CC):** One-Hot Encoding para variables nominales y codificación ordinal para las ordinales, justificando la elección.
    - **Escalado (ED):** StandardScaler (estandarización) o MinMaxScaler, justificando cuál se usa. Se ajusta (`fit`) solo con entrenamiento y se aplica (`transform`) a validación y prueba.
  - **Manejo de valores atípicos (outliers):** (i) dataset sin outliers; (ii) dataset con outliers de tal manera que sean el 5 % de la data.
    - **Detección:** mediante IQR (factor 1.5) o Z-score (|z| > 3), indicando método y variables numéricas usadas.
    - **Sin outliers:** se eliminan o se tratan los detectados.
    - **Con outliers:** los atípicos deben representar el **5 % de los registros de entrenamiento**, ya sean reales o inyectados sintéticamente en variables numéricas; documentar cuántos y cómo se generaron.
  - **Balanceo de clases:** (i) dataset balanceado; (ii) dataset con desbalance en las clases (una clase mucho más mayoritaria que las demás).
    - **Desbalanceado:** la clase mayoritaria con aproximadamente el **70 %** de los registros y las demás repartiéndose el resto en partes iguales; reportar la distribución de clases resultante.
    - **Balanceado:** aplicar una técnica concreta (SMOTE, submuestreo o sobremuestreo aleatorio; o `class_weight` en los modelos que lo permiten), indicando cuál y por qué, y reportar la distribución resultante.

- **Análisis exploratorio y valores nulos:** reportar dimensiones, tipos de variables, distribución de clases, estadísticas descriptivas, cantidad de nulos por variable (imputar con mediana/moda o eliminar, justificando) y matriz de correlación.

- **Partición de datos:** dividir el subconjunto de forma **estratificada** en entrenamiento (**70 %**), validación (**15 %**) y prueba (**15 %**) usando el random state definido. La conversión de categóricas, el escalado, la inyección de outliers y el balanceo se ajustan **únicamente con el conjunto de entrenamiento**. **El conjunto de prueba es el mismo en los 8 casos** para que la Figura 1 sea comparable.

- **Reproducibilidad:** usar el random state definido en la partición, los modelos, las técnicas de balanceo, la inyección de outliers y las inicializaciones.

- Algunas fuentes de datasets: Kaggle, Hugging Face, UCI Machine Learning.

### 2.2. Entrenamiento con Árboles, KNN, SVM y Redes Neuronales

- Para cada caso de la Figura 1, realizar el entrenamiento y la evaluación de 3 datos de prueba para cada una de las técnicas de aprendizaje supervisado (Árboles, KNN, SVM y Redes Neuronales).
  - En el caso específico de las **redes neuronales**, en cada caso de uso se deben implementar las **técnicas de regularización y callbacks** vistas en clase. Igualmente se debe documentar de manera general la **arquitectura** de la red neuronal y su proceso de configuración.

### 2.3. Evaluación de los modelos generados

- Para cada técnica de aprendizaje supervisado, obtener las métricas: **accuracy, precisión, recall, F1-score** y elaborar la tabla de la Figura 1.
  - Incluir la **matriz de confusión** de cada técnica (como mínimo para el caso 8) y el **reporte de métricas por clase**.

**Figura 1: Casos de uso de las técnicas de aprendizaje frente al preprocesamiento**

Para cada fila se llenan Accuracy / Precision / Recall / F1-Score de cada algoritmo: Árboles de Decisión, KNN, SVM y Redes Neuronales.

| # | Normalización | Outliers | Balanceo | Árboles (Acc/Prec/Rec/F1) | KNN (Acc/Prec/Rec/F1) | SVM (Acc/Prec/Rec/F1) | Redes Neuronales (Acc/Prec/Rec/F1) |
|---|---|---|---|---|---|---|---|
| 1 | CC(SI) y ED(NO) | NO | NO | | | | |
| 2 | CC(SI) y ED(NO) | NO | SI | | | | |
| 3 | CC(SI) y ED(NO) | SI | NO | | | | |
| 4 | CC(SI) y ED(NO) | SI | SI | | | | |
| 5 | CC(SI) y ED(SI) | NO | NO | | | | |
| 6 | CC(SI) y ED(SI) | NO | SI | | | | |
| 7 | CC(SI) y ED(SI) | SI | NO | | | | |
| 8 | CC(SI) y ED(SI) | SI | SI | | | | |

Con base en esta tabla, generar las siguientes gráficas y analizar los resultados:
- **Diagrama de barras con el F1-Score máximo por algoritmo (general):** cuatro barras, una por algoritmo.
- **Diagrama de barras con el F1-Score medio por normalización y algoritmo** (barras agrupadas por algoritmo: "Sin ED (NO)" vs. "Con ED (SI)"; el ejemplo del PDF muestra SVM, Árboles y KNN, pero también deben graficarse las Redes Neuronales).
- **Diagrama de barras con el F1-Score medio por manejo de datos atípicos y algoritmo.**
- **Diagrama de barras con el F1-Score medio por balanceo y algoritmo.**

Generar en el informe un análisis de los resultados observados, identificando patrones y posibles causas del comportamiento de cada modelo ante las diferentes condiciones, y generar conclusiones.

En el análisis explicar las causas del comportamiento observado:
- Sensibilidad a la escala (KNN, SVM y redes neuronales frente a árboles).
- Efecto de los outliers.
- Efecto del desbalance (por qué el accuracy engaña).
- Costo de los errores (falsos positivos vs. falsos negativos) según el dominio elegido, justificando qué métrica se prioriza.

- Para el **caso 8** de la Figura 1, obtener las **curvas ROC y Precision-Recall** comparando las 4 técnicas. Analizar los resultados y generar conclusiones.

### 2.4. Visualización de resultados

Para cada uno de los casos de la Figura 1:

- **Árboles de decisión**
  - Obtener la importancia de características.
  - Gráfica de `max_depth` (y/o `ccp_alpha`) vs. accuracy en entrenamiento y validación, para evidenciar subajuste y sobreajuste; comparar el criterio **Gini** con **Entropía**.
  - Interpretar las reglas del árbol y las características más importantes en el contexto del dominio.
  - Gráfico del árbol de decisión solución usando el criterio Gini.
- **K-Vecinos más cercanos (KNN)**
  - Gráfico de los vecinos para KNN una vez entrenado el modelo.
  - Gráfica de k vs. accuracy (promedio de validación cruzada de 5 folds) que justifique el valor de k elegido.
- **Máquinas de Vectores de Soporte (SVM)**
  - Las regiones de decisión asociadas a su respuesta.
- Para graficar las regiones de decisión de SVM y el gráfico de vecinos de KNN, reducir el dataset a 2 dimensiones con **PCA**. **PCA se usa solo para visualizar, no para entrenar los modelos.**

---

## 3. Aprendizaje no supervisado

### 3.1. Dataset y preprocesado

- Utilizar el mismo dataset del punto anterior y **eliminar la variable objetivo**.
- **Preprocesamiento:** generar las diferentes versiones del dataset detalladas en la Figura 2.
- **El balanceo no aplica directamente sobre una etiqueta eliminada:** las versiones balanceado / desbalanceado se construyen a partir de los mismos registros usados en cada caso del aprendizaje supervisado y luego se descarta la variable objetivo. Documentar esta correspondencia.
- Conservar la etiqueta original **únicamente para la validación externa** de los clusters (nunca para entrenar), y aplicar el mismo preprocesamiento (CC, escalado, outliers) definido en el punto anterior.

**Figura 2. Datasets para aprendizaje no supervisado**

| # | Normalización | Outliers | Balanceo |
|---|---|---|---|
| 1 | CC(SI) y ED(NO) | NO | NO |
| 2 | CC(SI) y ED(NO) | NO | SI |
| 3 | CC(SI) y ED(NO) | SI | NO |
| 4 | CC(SI) y ED(NO) | SI | SI |
| 5 | CC(SI) y ED(SI) | NO | NO |
| 6 | CC(SI) y ED(SI) | NO | SI |
| 7 | CC(SI) y ED(SI) | SI | NO |
| 8 | CC(SI) y ED(SI) | SI | SI |

### 3.2. Entrenamiento con K-Means y DBSCAN

- Para cada caso de la Figura 2, realizar el entrenamiento y la generación de 3 datos de prueba para cada técnica de aprendizaje no supervisado (K-Means y DBSCAN).
- Para cada configuración del dataset de la Figura 2:
  - **K-Means:** usar la **técnica del codo** y el **análisis del silhouette score** para determinar la cantidad óptima de clusters. Usar las métricas vistas en clase para evaluar la calidad del agrupamiento: **inercia** y **silhouette score**.
    - Evaluar un rango de K (p. ej., de 2 a 10) y graficar inercia (SSE) vs. K para el codo y silhouette vs. K; reportar el K óptimo.
    - Usar inicialización **K-Means++**, definir `n_init` y fijar el random state; reportar el tamaño de cada cluster.
    - Analizar el efecto del escalado y de los outliers, dado que K-Means usa distancia euclidiana y es sensible a ambos.
  - **DBSCAN:** usar el procedimiento visto en clase para determinar el mínimo número de vecinos (**MinPts**) y la distancia máxima entre dos puntos para que se consideren vecinos (**Épsilon ε**).
    - Determinar ε con el **gráfico de K-distancia** (con k = MinPts), tomando el codo de la curva; para MinPts usar la regla general de **2 veces el número de características** (aumentarlo en conjuntos grandes o con mucho ruido).
    - Calcular el silhouette **excluyendo los puntos de ruido** (requiere al menos 2 clusters) y discutir la sensibilidad: ε pequeño genera muchos outliers, ε grande fusiona clusters, y DBSCAN tiene dificultad con densidades variables.

### 3.3. Evaluación de los modelos generados

- Para cada técnica de aprendizaje no supervisado, obtener las métricas **Inertia** y **Silhouette_Score** y elaborar la tabla de la Figura 3.

**Figura 3: Casos de uso de las técnicas de aprendizaje no supervisado frente al preprocesamiento**

| # | Normalización | Outliers | Balanceo | K-Means Inertia | K-Means Silhouette | DBSCAN Silhouette |
|---|---|---|---|---|---|---|
| 1 | CC(SI) y ED(NO) | NO | NO | | | |
| 2 | CC(SI) y ED(NO) | NO | SI | | | |
| 3 | CC(SI) y ED(NO) | SI | NO | | | |
| 4 | CC(SI) y ED(NO) | SI | SI | | | |
| 5 | CC(SI) y ED(SI) | NO | NO | | | |
| 6 | CC(SI) y ED(SI) | NO | SI | | | |
| 7 | CC(SI) y ED(SI) | SI | NO | | | |
| 8 | CC(SI) y ED(SI) | SI | SI | | | |

Con base en la Figura 3, generar las siguientes gráficas y analizar los resultados:
- **Diagrama de barras con el Silhouette_Score para todos los casos por algoritmo (general):** 2 gráficas, una para K-Means y otra para DBSCAN, cada una con 8 barras.

Generar en el informe un análisis de los resultados, identificando patrones y posibles causas del comportamiento de cada modelo ante las diferentes condiciones, y generar conclusiones.

Incluir una **comparación entre K-Means y DBSCAN**: forma de los clusters (esféricos vs. arbitrarios), necesidad de definir K, manejo del ruido y de los outliers, y comportamiento ante densidades variables.

### 3.4. Visualización de resultados

- Visualizar los resultados de los clusters mediante **gráficas en 2D**.
  - Analizar e interpretar los clusters obtenidos, discutiendo si tienen sentido desde el punto de vista del dominio del problema.
  - Para graficar en 2D reducir la dimensionalidad con **PCA (o t-SNE)**; mostrar los **centroides en K-Means** y el **ruido en DBSCAN con un color distinto**.

---

## 4. Entregables

### 4.1. Documento de diseño (PDF)

- Descripción detallada de cada componente: Aprendizaje Supervisado y Aprendizaje No Supervisado.
- Particularmente:
  - Para cada componente, incluir los análisis de resultados solicitados en cada sección y generar conclusiones generales.
  - Diagramas solicitados en el análisis de cada técnica de aprendizaje supervisado y no supervisado.
  - Enlace del vídeo tipo pitch de la sustentación (ver sección 5).
  - Justificación del dataset: fuente, descripción de las variables y verificación de que cumple el mínimo de registros y de clases.

### 4.2. Código fuente documentado en Python (Google Colab)

- Código fuente separado por módulos: aprendizaje supervisado y aprendizaje no supervisado.
- Comentarios por función y explicación del flujo general.
- El notebook debe **ejecutarse de principio a fin sin errores**, con **semillas fijas**, una **celda de instalación de librerías** y **funciones reutilizables y bien documentadas** para preprocesar, entrenar, evaluar y graficar.

---

## 5. Sustentación

- Se realizará de manera virtual y/o presencial mediante un **quiz**.
- Se deberá implementar un **vídeo tipo pitch (duración máx. 5 minutos)** con la demostración del sistema funcional, la explicación clara del problema y la solución híbrida implementada.
  - Todos los integrantes deben participar en el video.
  - El video se presentará durante la sustentación, para lo cual deberán tener un buen equipo con internet para su presentación virtual.
  - El video deberá ser subido a **YouTube** y en el documento de diseño (PDF) incluir el enlace.
  - El video deberá estar disponible desde la fecha de entrega del trabajo.
- Se realizará una sesión de preguntas.
  - Todos los integrantes deben estar presentes en la sustentación y se podrá seleccionar a cualquiera para responder preguntas.

---

## 6. Metodología de evaluación

| Componente | Peso |
|---|---|
| **Código y documentación** | **60 %** |
| – Módulo de aprendizaje supervisado | 40 % |
| – Módulo de aprendizaje no supervisado | 20 % |
| **Sustentación** | **40 %** |
| – Video tipo pitch | 15 % |
| – Atención a preguntas | 25 % |

---

## 7. Material a entregar

Entregar en **Google Classroom** un archivo ZIP con nombre `practica2-grupo-XX–equipo-YY.zip` (XX = número del grupo, 1 o 2; YY = número del equipo) que contenga:

- **Documento de diseño:** descripción detallada de cada módulo (Aprendizaje Supervisado y Redes Neuronales, Aprendizaje No Supervisado), junto con la justificación de las decisiones tomadas en cada parte del programa y **el enlace al video de sustentación tipo pitch y al conjunto de datos original**.
- **Código fuente:** todo el programa en Python, debidamente organizado y documentado.

---

## Anexo 1 – Instrucciones generales para la presentación de los trabajos

- Los trabajos serán presentados en **grupos de 4 personas** definidos al inicio del semestre (ver grupos en el material de la semana 2).
- **La selección del subconjunto del dataset debe cumplir obligatoriamente con las condiciones establecidas:** utilizar 5000 registros más el último dígito diferente de cero de la cédula de un integrante del equipo multiplicado por 150, y definir el random state con los dos últimos dígitos diferentes de cero de la misma cédula (sin usar el valor 42).
- **No se admiten trabajos similares (incluyendo el planteamiento del dominio)**; programas con códigos similares serán anulados. La totalidad del trabajo deberá ser original y propio. El estudiante es responsable de evitar que su material evaluable (código, solución, memorias, etc.) sea accesible a otros grupos. Si se detectan copias, la sanción afectará a todos los involucrados: quienes copien y quienes hayan sido copiados.
- Los integrantes de cada grupo deberán ser capaces de explicar y responder preguntas sobre el trabajo durante la sustentación.
- La persona que no asista a la sustentación se considerará que no presentó la práctica en su totalidad (**"no se tendrá en cuenta el código, ni la documentación, ni el video tipo pitch para la evaluación"**).
- El plazo de entrega es **estricto**; no se admiten trabajos después de la fecha. No se considera entrega aquella que solo contiene código, solo el vídeo, solo el documento de diseño o cualquier forma incompleta.
- El código entregado en el .zip debe ser **el mismo** que se ilustra en el vídeo y se explica en el documento de diseño. De lo contrario, se considera que el código no fue entregado.

**NOTA 1:** Si en la fecha de la sustentación no se presenta el video, el trabajo de la práctica se considerará no entregado.

**NOTA 2:** La sustentación se realizará en la fecha indicada mediante el vídeo entregado y las preguntas orales a los integrantes de cada equipo después de la presentación del pitch.

**NOTA 3:** La redacción de este documento puede presentar errores o la ausencia de algunos requisitos específicos, los cuales deberán ser informados al profesor para ser aclarados de manera colectiva en clase. Estas aclaraciones harán parte de los requisitos de la práctica.
