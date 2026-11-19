# Distribución de bloques del proyecto

## Bloque 1 — Datos, preprocesamiento y evaluación comparativa

Responsable de que el notebook corra de principio a fin.

### Preparación del dataset

- Generar el subconjunto (6350 registros, semilla 79) y dejar documentada la cédula usada.
- Justificar el dataset: fuente, descripción de variables, verificación del mínimo de registros y de clases.
- Eliminar las variables con fuga de información y los identificadores, con su justificación.

### Análisis exploratorio

- Dimensiones, tipos de variables, distribución de clases y estadísticas descriptivas.
- Nulos por variable (imputar con mediana o moda, o eliminar, justificando).
- Matriz de correlación.

### Partición y preprocesamiento

- Partición estratificada 70/15/15 con la semilla. El conjunto de prueba es el mismo en los 8 casos.
- **CC:** One-Hot para nominales y codificación ordinal para ordinales, con justificación.
- **ED:** StandardScaler o MinMaxScaler, justificando cuál. Se ajusta solo con entrenamiento.
- **Outliers:**
  - Detección con IQR o Z-score, indicando el método y las variables.
  - Versión sin outliers (eliminados o tratados).
  - Versión con outliers inyectados al 5 % del entrenamiento, documentando cuántos y cómo se generaron.
- **Balanceo:**
  - Versión desbalanceada (~70 % en la clase mayoritaria), con su distribución resultante.
  - Versión balanceada (SMOTE, sobremuestreo o submuestreo), justificando la técnica, con su distribución resultante.
- Construir las 8 versiones de la Figura 1 como funciones reutilizables.

### Evaluación comparativa

> Cuando lleguen los resultados de los bloques 2 y 3.

- Consolidar la tabla de la Figura 1 con los 4 algoritmos.
- Las 4 gráficas de barras de F1-score: máximo por algoritmo, por normalización, por outliers y por balanceo.
- Curvas ROC y Precision-Recall del caso 8 comparando las 4 técnicas (One-vs-Rest para multiclase).
- **Análisis general:**
  - Sensibilidad a la escala de KNN, SVM y redes frente a árboles.
  - Efecto de los outliers.
  - Efecto del desbalance y por qué el accuracy engaña.
  - Costo FP/FN y métrica priorizada.

**Transversal:** celda de instalación de librerías, semillas fijas, integración final del notebook y armado del ZIP.

---

## Bloque 2 — Árboles de decisión y KNN

Para cada uno de los 8 casos:

- Entrenar y evaluar ambos modelos, con predicción de 3 datos de prueba.
- Calcular accuracy, precisión, recall y F1, en el formato que acuerden con el bloque 1.

### Árboles de decisión

- Importancia de características.
- Gráfica de `max_depth` (y/o `ccp_alpha`) vs. accuracy en entrenamiento y validación, mostrando subajuste y sobreajuste.
- Comparación Gini vs. entropía.
- Gráfico del árbol solución con criterio Gini.
- Interpretación de las reglas y de las variables más importantes en el contexto educativo.

### KNN

- Gráfica de k vs. accuracy con validación cruzada de 5 folds, justificando el k elegido.
- Gráfico de vecinos del modelo entrenado, usando PCA a 2D solo para visualizar.

**Del caso 8:** matriz de confusión y reporte de métricas por clase de ambos modelos.

---

## Bloque 3 — SVM y redes neuronales

Para cada uno de los 8 casos:

- Entrenar y evaluar ambos modelos, con predicción de 3 datos de prueba.
- Calcular accuracy, precisión, recall y F1, en el mismo formato.

### SVM

- Regiones de decisión usando PCA a 2D (solo para visualizar, no para entrenar).
- Justificación de kernel, `C` y `gamma`.

### Redes neuronales

- Documentar la arquitectura (capas, neuronas, activaciones, salida softmax) y el proceso de configuración.
- Regularización vista en clase en cada caso: Dropout, L2 y/o Batch Normalization.
- Callbacks vistos en clase: EarlyStopping, ReduceLROnPlateau y/o ModelCheckpoint.
- Curvas de pérdida y accuracy de entrenamiento y validación.

**Del caso 8:** matriz de confusión, reporte por clase y probabilidades de ambos modelos. El bloque 1 las necesita para las curvas ROC y PR.

---

## Bloque 4 — Aprendizaje no supervisado

### Preparación

- Construir las 8 versiones de la Figura 2 con las funciones del bloque 1 (mismos registros, CC, escalado y outliers), sin la etiqueta.
- Documentar cómo cada versión balanceada o desbalanceada corresponde a un caso supervisado.
- Guardar la etiqueta solo para validación externa.

### K-Means, por caso

- Rango de K de 2 a 10: gráfica de inercia vs. K (codo) y silhouette vs. K. Reportar el K óptimo.
- K-Means++, `n_init` definido y semilla fija. Reportar el tamaño de cada cluster.
- Analizar el efecto del escalado y de los outliers.

### DBSCAN, por caso

- MinPts como 2 veces el número de características (ajustado si hay mucho ruido).
- ε a partir del codo del gráfico de k-distancia.
- Silhouette excluyendo el ruido.
- Discutir la sensibilidad a ε y la dificultad con densidades variables.

### Evaluación y visualización

- Asignar 3 datos de prueba por técnica y caso.
- Tabla de la Figura 3 y las 2 gráficas de silhouette (8 barras cada una).
- Comparación K-Means vs. DBSCAN: forma de los clusters, necesidad de definir K, manejo de ruido y outliers, densidades.
- Visualización 2D con PCA (o t-SNE): centroides en K-Means y ruido en otro color en DBSCAN.
- Interpretación de los clusters en el dominio y validación externa contra los niveles reales de matemáticas.

---

## Tareas compartidas (todos)

| Tarea | Cómo repartirla |
|---|---|
| Documento PDF | Cada uno escribe la sección de su bloque. El bloque 1 escribe la introducción, el dataset y las conclusiones generales, e incluye el enlace al video y al dataset. |
| Video pitch (máx. 5 min) | ~1 min por persona explicando su bloque. El bloque 1 abre con el problema y el dominio; el bloque 4 cierra con las conclusiones. Se sube a YouTube antes de la entrega. |
| Comentarios del código | Cada uno documenta sus funciones y el flujo de su parte. |
| Preparación para preguntas | Todos deben entender todo el proyecto: la sustentación pesa 40 % y pueden preguntarle cualquier cosa a cualquiera. |

## Orden de trabajo sugerido

1. **Primero, bloque 1:** subconjunto, EDA y funciones que generen las 8 versiones. Acuerden desde el inicio un formato común, por ejemplo un diccionario `casos[1..8]` con `X_train`, `X_val`, `X_test`, `y_train`, `y_val`, `y_test`.
2. **En paralelo, bloques 2, 3 y 4,** entrenando sobre esas versiones.
3. **Al final, bloque 1** consolida la tabla, las gráficas y las curvas ROC/PR. Todos escriben el documento y graban el video.

## Punto a aclarar con el profesor

El PDF pide “la evaluación de 3 datos de prueba” por técnica y caso, y no es del todo claro qué significa. La interpretación más probable es mostrar la predicción de 3 registros del conjunto de prueba. Confírmenlo usando la Nota 3 del enunciado.
