# Contexto del proyecto – Práctica 2 (para los bloques 2, 3 y 4)

> **Para quién es este archivo:** para los integrantes del equipo que trabajan en los bloques 2, 3 y 4, **y para la IA que usen**. Si trabajas con una IA, pásale este archivo completo junto con `practica-2.ipynb`, `Enunciado.md` y `bloques.md`.
>
> **Regla principal:** el bloque 1 (secciones 0 a 4.10 del notebook) ya está cerrado y probado. **No lo modifiques.** Si necesitas un cambio ahí, pídeselo al responsable del bloque 1.
>
> **Este archivo se actualiza entre todos:** cada bloque registra su avance en la sección 10.3 y verifica periódicamente la consistencia (sección 10.2).

---

## 1. El proyecto en un minuto

| Dato | Valor |
|---|---|
| Dominio | Educación: predecir el **nivel de desempeño en matemáticas** del Saber 11 de 2022 |
| Variable objetivo | `DESEMP_MATEMATICAS`, con 4 clases: Nivel 1, Nivel 2, Nivel 3 y Nivel 4 |
| Subconjunto | 6350 registros (cédula 1033178679) |
| Semilla | `SEMILLA = 79`. Úsala en **todo** lo que tenga aleatoriedad |
| Distribución real | Nivel 1: 11.9 % · Nivel 2: 36.6 % · Nivel 3: 46.6 % · Nivel 4: 4.9 % |
| Partición | 70 % entrenamiento / 15 % validación / 15 % prueba, estratificada |
| Predictores | 23 variables, que después de la codificación quedan en **65 columnas** |

---

## 2. Cómo ejecutar

1. Abre `practica-2.ipynb` (en local o en Colab).
2. Ejecuta **desde el inicio hasta la sección 4.10** (tarda ~15 s). Con eso quedan en memoria los datos (`casos`) y las funciones del contrato.
   - El CSV se carga solo: primero lo busca en local (junto al notebook o en `practica-2/`) y, si no está, lo descarga de Google Drive.
   - La sección 1.1 (generar el subconjunto desde el original) está desactivada con `GENERAR_SUBCONJUNTO = False`. **Déjala así.**
3. Trabaja en tu sección:

| Sección | Bloque | Contenido |
|---|---|---|
| 0–4.10 | 1 | Datos, preprocesamiento y contrato. **Cerrado** |
| **5** | **2** | Árboles de decisión y KNN |
| **6** | **3** | SVM y redes neuronales |
| 7 | 1 | Evaluación comparativa (se construye con lo que ustedes registren) |
| **8** | **4** | K-Means y DBSCAN |

Recomendado: trabaja en una **copia** del notebook y entrega tus celdas para integrarlas al final. Así nadie pisa el trabajo de otro.

---

## 3. Datos disponibles (contrato de entrada)

### `casos[1..8]`: los 8 casos de la Figura 1
No preprocesen nada: los datos ya vienen imputados, codificados y, según el caso, escalados, sin outliers o con ellos, balanceados o desbalanceados.

```python
datos = casos[3]
datos["X_train"], datos["y_train"]   # entrenamiento del caso
datos["X_val"],   datos["y_val"]     # validación: para elegir hiperparámetros
datos["X_test"],  datos["y_test"]    # prueba: SOLO para la evaluación final (lo hace registrar_resultado)
datos["normalizacion"], datos["outliers"], datos["balanceo"]   # la fila de la Figura 1
datos["scaler"]                      # StandardScaler ajustado (casos 5–8) o None
```

| Caso | Escalado (ED) | Outliers | Balanceo | Registros de entrenamiento | Distribución de entrenamiento |
|---|---|---|---|---|---|
| 1 | NO | NO (eliminados) | NO | 2050 | 10 / 10 / **70** / 10 % |
| 2 | NO | NO (eliminados) | SI (SMOTENC) | 8208 | 25 % cada clase |
| 3 | NO | SI (5 %) | NO | 2170 | 10 / 10 / **70** / 10 % |
| 4 | NO | SI (5 %) | SI (SMOTENC) | 8296 | 25 % cada clase |
| 5 | SI | NO (eliminados) | NO | 2050 | 10 / 10 / **70** / 10 % |
| 6 | SI | NO (eliminados) | SI (SMOTENC) | 8208 | 25 % cada clase |
| 7 | SI | SI (5 %) | NO | 2170 | 10 / 10 / **70** / 10 % |
| 8 | SI | SI (5 %) | SI (SMOTENC) | 8296 | 25 % cada clase |

**Hechos importantes:**
- **Validación (952) y prueba (953) son iguales en los 8 casos** y tienen la distribución real. Solo cambia si están escaladas (casos 5–8).
- **Los casos *k* y *k+4* tienen exactamente los mismos registros de entrenamiento**; solo difieren en el escalado. Por eso un árbol da **las mismas métricas en los casos 1 y 5** (los árboles no dependen de la escala). Eso es correcto, no un error.
- **Etiquetas:** enteros `0, 1, 2, 3`. `CLASES[i]` da el nombre (`CLASES = ["Nivel 1", "Nivel 2", "Nivel 3", "Nivel 4"]`).
- **Columnas (65), iguales en todos los casos:**
  - 5 numéricas: `PUNT_LECTURA_CRITICA`, `PUNT_C_NATURALES`, `PUNT_SOCIALES_CIUDADANAS`, `PUNT_INGLES`, `EDAD`.
  - 5 ordinales codificadas como 0, 1, 2…: `FAMI_EDUCACIONMADRE`, `FAMI_EDUCACIONPADRE`, `FAMI_ESTRATOVIVIENDA`, `FAMI_CUARTOSHOGAR`, `FAMI_PERSONASHOGAR`.
  - 55 dummies One-Hot (0/1) de 13 variables nominales.
  - En los casos 5–8, las numéricas y las ordinales están estandarizadas y las dummies siguen en 0/1.

### Funciones y variables que pueden usar
| Nombre | Para qué |
|---|---|
| `SEMILLA` | `random_state` de todo |
| `CLASES` | Nombres de las clases, en orden |
| `obtener_datos_demo(k)` | Los **3 registros de prueba** de demostración (uno de Nivel 1, uno de Nivel 3 y uno de Nivel 4) en la versión del caso `k` |
| `obtener_datos_no_supervisado(k)` | Bloque 4: `(X, etiqueta)`. `X` son los registros de entrenamiento del caso, sin etiqueta |
| `COLOR_PRINCIPAL`, `COLORES_SERIES`, `COLORES_CLASES`, `CMAP_DIVERGENTE` | Paleta común de las gráficas |
| `graficar_distribucion_clases(y, titulo)` | Barras de distribución de clases |

---

## 4. Reglas de uso de los conjuntos

1. **Entrenamiento (`X_train`, `y_train`):** para entrenar y para la validación cruzada.
2. **Validación (`X_val`, `y_val`):** para **elegir hiperparámetros** (`max_depth`, `ccp_alpha`, `k`, `C`, `gamma`, arquitectura de la red) y como `validation_data` de la red (EarlyStopping, ReduceLROnPlateau y curvas de pérdida).
3. **Prueba (`X_test`, `y_test`):** **no la usen directamente.** La única forma de evaluar en prueba es `registrar_resultado(...)`, una vez por algoritmo y caso, con el modelo final. Ajustar hiperparámetros mirando la prueba infla los resultados.
4. **Validación cruzada de 5 folds (KNN, gráfica de k vs. accuracy):** sobre `X_train` con `StratifiedKFold(n_splits=5, shuffle=True, random_state=SEMILLA)`.
   - Ojo: en los casos con SMOTE (2, 4, 6 y 8), la CV es optimista porque los registros sintéticos se parecen a los originales. Para justificar el `k` elegido, reporten también el accuracy en `X_val`.
5. **PCA (solo para visualizar):** ajústenlo con `X_train` del caso. **Nunca entrenen los modelos con los datos reducidos por PCA.**
6. **No modifiquen** `casos`, `X_train` ni ninguna variable del bloque 1. Si necesitan transformar algo, trabajen sobre una copia (`X = casos[k]["X_train"].copy()`).

---

## 5. Formato de resultados (obligatorio)

Toda métrica que vaya a la Figura 1 o a la Figura 3 **se registra con estas funciones**. No calculen las métricas de la tabla a mano: la función garantiza que todos usen la misma prueba, las mismas métricas y el mismo promedio.

### 5.1 Supervisado: `registrar_resultado(algoritmo, caso, modelo, parametros=None)`
- `algoritmo`: **exactamente** uno de `"Árboles"`, `"KNN"`, `"SVM"` o `"Redes Neuronales"`, con tilde y mayúsculas tal cual. Cualquier otro nombre lanza un error.
- `caso`: un entero de 1 a 8.
- `modelo`: el modelo **ya entrenado**.
- `parametros`: un diccionario opcional con los hiperparámetros elegidos (por ejemplo, `{"max_depth": 6, "criterion": "gini"}`).

La función predice sobre `casos[caso]["X_test"]`, calcula **Accuracy, Precision, Recall y F1 con promedio `macro`**, guarda las predicciones y las **probabilidades** (que se usan para las curvas ROC/PR) e imprime el resumen. Si registras dos veces el mismo algoritmo y caso, el segundo registro reemplaza al primero y se muestra un aviso.

**¿Por qué `macro`?** Porque cada clase pesa lo mismo. Con `weighted`, el buen desempeño en Nivel 3 (46.6 %) ocultaría los errores en Nivel 4 (4.9 %), y eso es justo lo que el análisis de desbalance tiene que mostrar.

**Ejemplos:**

```python
from sklearn.tree import DecisionTreeClassifier
from sklearn.neighbors import KNeighborsClassifier
from sklearn.svm import SVC

for k in range(1, 9):
    d = casos[k]

    arbol = DecisionTreeClassifier(criterion="gini", max_depth=6, random_state=SEMILLA)
    arbol.fit(d["X_train"], d["y_train"])
    registrar_resultado("Árboles", k, arbol, {"criterion": "gini", "max_depth": 6})

    knn = KNeighborsClassifier(n_neighbors=15).fit(d["X_train"], d["y_train"])
    registrar_resultado("KNN", k, knn, {"n_neighbors": 15})

    # SVM: probability=True es OBLIGATORIO (sin probabilidades no hay curvas ROC/PR)
    svm = SVC(kernel="rbf", C=1.0, gamma="scale", probability=True, random_state=SEMILLA)
    svm.fit(d["X_train"], d["y_train"])
    registrar_resultado("SVM", k, svm, {"kernel": "rbf", "C": 1.0, "gamma": "scale"})
```

**Red neuronal (Keras):** la función detecta que es un modelo de Keras, usa `model.predict` como probabilidades y toma la clase de mayor probabilidad. Requisitos:
- Capa de salida `Dense(4, activation="softmax")`, con una neurona por clase y en el orden de `CLASES`.
- Pérdida `sparse_categorical_crossentropy`, porque las etiquetas son enteros 0–3.
- No hace falta convertir `X` a `float32` antes de registrar: la función lo hace.

```python
import tensorflow as tf

d = casos[8]
modelo = tf.keras.Sequential([
    tf.keras.layers.Input(shape=(d["X_train"].shape[1],)),
    tf.keras.layers.Dense(64, activation="relu", kernel_regularizer=tf.keras.regularizers.l2(1e-4)),
    tf.keras.layers.BatchNormalization(),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(32, activation="relu"),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(4, activation="softmax"),
])
modelo.compile(optimizer="adam", loss="sparse_categorical_crossentropy", metrics=["accuracy"])
historia = modelo.fit(
    d["X_train"].astype("float32"), d["y_train"],
    validation_data=(d["X_val"].astype("float32"), d["y_val"]),
    epochs=200, batch_size=64, verbose=0,
    callbacks=[tf.keras.callbacks.EarlyStopping(patience=15, restore_best_weights=True),
               tf.keras.callbacks.ReduceLROnPlateau(factor=0.5, patience=5)],
)
registrar_resultado("Redes Neuronales", 8, modelo, {"capas": [64, 32], "dropout": 0.3, "l2": 1e-4})
```
(Es solo un ejemplo de formato. La arquitectura la decide y la justifica el bloque 3.)

### 5.2 Funciones de apoyo (úsenlas para que todo tenga el mismo formato)
| Función | Qué devuelve | Uso |
|---|---|---|
| `predecir_demo(algoritmo, caso, modelo)` | Tabla con los 3 registros de demostración: clase real, predicha y probabilidad de cada clase | "Evaluación de 3 datos de prueba" en cada caso y técnica |
| `reporte_por_clase(algoritmo, caso)` | Precision, recall y F1 por clase (de un resultado ya registrado) | Reporte por clase del caso 8 |
| `graficar_matriz_confusion(algoritmo, caso, ax=None)` | Matriz de confusión con los nombres de las clases | Matriz de confusión del caso 8 |
| `tabla_figura1()` | Tabla de la Figura 1 con todo lo registrado | Para revisar el avance; la versión final la arma el bloque 1 |
| `predecir_con_probabilidades(modelo, X)` | `(y_pred, y_proba)` para sklearn o Keras | Si necesitan predicciones sobre otro conjunto, por ejemplo validación |

### 5.3 No supervisado (bloque 4): `registrar_clustering(tecnica, caso, X, etiquetas, modelo=None, parametros=None)`
- `tecnica`: `"K-Means"` o `"DBSCAN"`.
- `X`: los mismos datos con los que agruparon (`X, y_ext = obtener_datos_no_supervisado(k)`).
- `etiquetas`: la etiqueta de cluster de cada registro (`modelo.labels_`). En DBSCAN, `-1` es ruido.
- Calcula el **silhouette excluyendo el ruido**. Si quedan menos de 2 clusters, el silhouette queda en `NaN`. En K-Means toma `modelo.inertia_`.
- `tabla_figura3()` arma la tabla de la Figura 3.

```python
from sklearn.cluster import KMeans, DBSCAN

for k in range(1, 9):
    X, y_ext = obtener_datos_no_supervisado(k)   # y_ext: SOLO para la validación externa
    km = KMeans(n_clusters=K_OPTIMO, init="k-means++", n_init=10, random_state=SEMILLA).fit(X)
    registrar_clustering("K-Means", k, X, km.labels_, km, {"n_clusters": K_OPTIMO})
    db = DBSCAN(eps=EPS, min_samples=MIN_PTS).fit(X)
    registrar_clustering("DBSCAN", k, X, db.labels_, db, {"eps": EPS, "min_samples": MIN_PTS})
```

---

## 6. Qué entrega cada bloque

Todo esto sale de `Enunciado.md` y `bloques.md`; revisen ambos para el detalle.

### Bloque 2: Árboles y KNN (sección 5)
- [ ] Entrenar y registrar **Árboles** y **KNN** en los **8 casos** (`registrar_resultado`).
- [ ] `predecir_demo` en cada caso.
- [ ] **Árboles:**
  - [ ] Importancia de características.
  - [ ] Gráfica de `max_depth` (y/o `ccp_alpha`) vs. accuracy en entrenamiento y validación, mostrando subajuste y sobreajuste.
  - [ ] Comparación Gini vs. entropía.
  - [ ] Gráfico del árbol final con criterio Gini.
  - [ ] Interpretación de las reglas y de las variables más importantes en el contexto educativo.
  - Ojo: los puntajes están muy correlacionados entre sí (ρ 0.69–0.82), así que la importancia se reparte entre ellos.
- [ ] **KNN:**
  - [ ] Gráfica de k vs. accuracy con CV de 5 folds, justificando el k elegido.
  - [ ] Gráfico de vecinos con PCA a 2D (solo para visualizar).
- [ ] **Caso 8:** matriz de confusión y reporte por clase de ambos modelos.

### Bloque 3: SVM y redes neuronales (sección 6)
- [ ] Entrenar y registrar **SVM** (`probability=True`) y **Redes Neuronales** en los **8 casos**.
- [ ] `predecir_demo` en cada caso.
- [ ] **SVM:**
  - [ ] Regiones de decisión con PCA a 2D (para graficar, entrenen un SVM aparte sobre los 2 componentes; el modelo registrado se entrena con las 65 columnas).
  - [ ] Justificación de kernel, `C` y `gamma`.
- [ ] **Redes:**
  - [ ] Arquitectura documentada (capas, neuronas, activaciones, salida softmax) y proceso de configuración.
  - [ ] Regularización vista en clase **en cada caso** (Dropout, L2 y/o BatchNormalization).
  - [ ] Callbacks (EarlyStopping, ReduceLROnPlateau y/o ModelCheckpoint).
  - [ ] Curvas de pérdida y accuracy de entrenamiento y validación.
- [ ] **Caso 8:** matriz de confusión y reporte por clase de ambos modelos. Las probabilidades ya quedan guardadas al registrar.

### Bloque 4: K-Means y DBSCAN (sección 8)
- [ ] Datos: `obtener_datos_no_supervisado(k)`. Ya corresponden 1 a 1 con los casos supervisados (mismos registros, CC, escalado y outliers, sin la etiqueta). La correspondencia está documentada en la sección 4.9.
- [ ] **K-Means**, por caso:
  - [ ] K de 2 a 10: gráficas de inercia vs. K (codo) y de silhouette vs. K, y el K óptimo.
  - [ ] `init="k-means++"`, `n_init` definido, `random_state=SEMILLA`.
  - [ ] Tamaño de cada cluster.
  - [ ] Efecto del escalado y de los outliers.
- [ ] **DBSCAN**, por caso:
  - [ ] MinPts = 2 × número de características. Con 65 columnas da 130; si deja demasiado ruido, ajústenlo y justifiquen el cambio.
  - [ ] ε tomado del codo del gráfico de k-distancias.
  - [ ] Silhouette sin ruido (lo calcula `registrar_clustering`).
  - [ ] Sensibilidad a ε y a las densidades variables.
- [ ] **3 datos de prueba por técnica y caso:** usen los mismos `INDICES_DEMO`, tomando `casos[k]["X_test"].loc[INDICES_DEMO]`.
  - K-Means: `km.predict(...)`.
  - DBSCAN no tiene `predict`: asignen cada punto al cluster de su *core sample* más cercano si está a menos de ε, o como ruido en otro caso.
- [ ] Tabla de la Figura 3 (`tabla_figura3()`) y las 2 gráficas de silhouette (8 barras cada una).
- [ ] Comparación K-Means vs. DBSCAN; visualización 2D con PCA (centroides en K-Means y ruido en otro color en DBSCAN); interpretación de los clusters y validación externa contra `y_ext`.
- Nota: en los casos 1–4 (sin escalar), K-Means queda dominado por `EDAD` y los puntajes. Eso es parte del análisis del efecto del escalado.

### Bloque 1 (cuando lleguen sus resultados)
- La tabla final de la Figura 1, las 4 gráficas de F1, las curvas ROC/PR del caso 8 y el análisis general. Todo sale de lo registrado, así que **si registran bien, no hay que pasar nada a mano.**

---

## 7. Reglas de código e integración

**Hacer:**
- Usar `random_state=SEMILLA` en todo modelo, partición, búsqueda o muestreo aleatorio.
- Escribir funciones reutilizables con docstring en español y comentarios del flujo. El enunciado lo exige: "funciones reutilizables y bien documentadas".
- Usar la paleta común (`COLOR_PRINCIPAL`, `COLORES_SERIES`, `COLORES_CLASES`) y los nombres de `CLASES` en ejes y leyendas.
- Mantener acotado el tiempo de ejecución, porque el notebook completo debe correr de principio a fin sin errores:
  - Búsquedas de hiperparámetros pequeñas (por ejemplo, un `GridSearchCV` con pocas combinaciones y `n_jobs=-1`).
  - Como referencia, el bloque 1 corre en ~15 s. Apunten a que cada bloque tarde unos pocos minutos.
- Escribir el markdown con el tono de la sección 7.1.

**No hacer:**
- No modificar las secciones 0–4.10.
- No reasignar estos nombres globales: `df`, `df_raw`, `casos`, `CLASES`, `SEMILLA`, `OBJETIVO`, `X_train_0`, `X_val_0`, `X_test_0`, `y_train_0`, `y_val_0`, `y_test_0`, `X_train_base`, `X_val_base`, `X_test_base`, `VARIABLES_NUMERICAS`, `VARIABLES_ORDINALES`, `VARIABLES_NOMINALES`, `LIMITES_IQR`, `ENCODER_OH`, `INDICES_DEMO`, `ALGORITMOS`, `METRICAS`, `RESULTADOS`, `RESULTADOS_CLUSTERING`, ni ninguna función ya definida. Usen nombres propios: `arbol_k`, `modelo_svm`, `historia_red`, etc.
- No volver a preprocesar: no escalar, no hacer One-Hot, no balancear ni quitar outliers. Cada caso ya trae su versión.
- No evaluar en `X_test` por fuera de `registrar_resultado` / `predecir_demo`, ni elegir hiperparámetros con la prueba.
- No entrenar con datos reducidos por PCA.
- No cambiar `GENERAR_SUBCONJUNTO` a `True`.

### 7.1 Tono del markdown
El notebook se dirige al **profesor**: no enseña, **justifica**. Todo el notebook debe sonar igual, así que sigan estos criterios (las secciones 0–4.10 son el modelo):

1. **Primera persona del plural.** "Elegimos", "descartamos", "conservamos". No le hablen al lector ("pongan", "ustedes", "fíjense").
2. **Conciso.** Cada decisión con su motivo y, si existe, el dato que la respalda. Nada de relleno ni de frases de transición.
3. **No definan conceptos del curso.** El profesor sabe qué es SMOTE, cómo funciona KNN o qué mide el F1. Expliquen **por qué** lo usaron, no **qué** es.
4. **Nombren las alternativas descartadas** y el motivo: "Elegimos X frente a Y porque…".
5. **Usen cifras reales del notebook.** Nada de "mejora bastante" o "es mucho mejor": escriban "el F1 pasa de 0.49 a 0.59". Si la cifra cambia al reentrenar, actualicen el texto.
6. **Nada que no se pueda sostener en la sustentación.** Si una afirmación es una suposición que los datos no muestran, quítenla o preséntenla explícitamente como hipótesis.
7. **Español formal pero natural**, sin emojis y sin exclamaciones.

**Ejemplos (antes → después):**

| ❌ Explica / ambiguo | ✅ Justifica |
|---|---|
| "Pearson mide relaciones lineales entre números continuos, mientras que Spearman trabaja con el orden…" | "Usamos Spearman en lugar de Pearson porque 5 de las variables y el objetivo son ordinales, y la relación esperada es monótona, no necesariamente lineal." |
| "KNN clasifica un punto según la clase mayoritaria de sus k vecinos más cercanos. Probamos varios k." | "Elegimos k = 15: maximiza el accuracy de la validación cruzada de 5 folds (0.61) y valores mayores no mejoran el accuracy en validación." |
| "¿Quieren correr todo desde el original? Pongan la bandera en True." | "Con `GENERAR_SUBCONJUNTO = True`, el notebook genera el subconjunto desde el original y lo usa en todas las secciones siguientes." |
| "La red funciona muy bien con Dropout." | "Usamos Dropout de 0.3: sin él, la pérdida de validación empieza a subir desde la época 20 mientras la de entrenamiento sigue bajando (sobreajuste)." |

(Las cifras de los ejemplos 2 y 4 son ilustrativas: usen las suyas.)

---

## 8. Decisiones ya tomadas (no las cambien; sirven para la sustentación)

| Decisión | Justificación corta |
|---|---|
| Se conservan los puntajes de las otras áreas como predictores | No se derivan del puntaje de matemáticas, así que no son fuga |
| Se eliminaron 14 columnas (municipios, constantes, duplicadas, `PERIODO`…) | Cada una se comprobó con datos (sección 2) |
| Nulos imputados (moda y mediana) en vez de borrar filas | Los nulos no son aleatorios; borrarlos sesgaría la muestra |
| Codificación ordinal para 5 variables y One-Hot para 13 | Respetar el orden cuando existe y no inventarlo cuando no |
| StandardScaler sobre numéricas y ordinales | KNN, SVM y redes necesitan escala comparable; se distorsiona menos que MinMax con outliers |
| IQR (1.5) para detectar outliers | `EDAD` es muy asimétrica (Z-score supone normalidad) |
| Sin outliers: se eliminan filas (~3 %) · Con outliers: reales más inyectados hasta el 5 % | No se alteran valores reales y se garantiza el 5 % exacto |
| Desbalanceado: submuestreo 70/10/10/10 · Balanceado: SMOTENC | Solo datos reales en el primero; en el segundo, registros sintéticos sin categorías inexistentes |
| Métricas con promedio `macro` | Que Nivel 4 (4.9 %) pese igual que Nivel 3 |

Cada sección del notebook tiene la justificación completa. **Todos deben entender todo el proyecto:** la sustentación vale 40 % y el profesor puede preguntarle cualquier cosa a cualquiera.

---

## 9. Pendiente

- **"Evaluación de 3 datos de prueba":** la interpretamos como mostrar la predicción de 3 registros fijos de prueba (`predecir_demo`). Falta confirmarlo con el profesor (Nota 3 del enunciado). Si cambia, solo hay que cambiar `INDICES_DEMO` / `predecir_demo` en el bloque 1.

---

## 10. Mantener este contexto actualizado (obligatorio)

Este archivo es la **fuente de verdad del equipo**. Cada bloque debe ir registrando aquí lo que hace, para que los demás (y sus IA) trabajen con información actual y el bloque 1 pueda integrar sin sorpresas.

### 10.1 Qué agregar y cuándo
Cada vez que terminen una parte de su bloque (no solo al final), actualicen **su** subsección en la 10.3 con lo siguiente:
- **Estado:** qué casos están entrenados y registrados, y qué falta.
- **Decisiones y su justificación:** hiperparámetros elegidos y por qué (por ejemplo, "k = 15 por CV de 5 folds y accuracy en validación"), arquitectura de la red, ε y MinPts, etc.
- **Nombres globales nuevos:** funciones y variables que definan y que otros puedan usar o deban evitar.
- **Tiempo de ejecución** aproximado de su sección.
- **Hallazgos relevantes** para el análisis general: patrones, resultados inesperados o problemas.
- **Dudas o cambios que necesiten de otro bloque.**

Reglas:
- Editen **solo su subsección**. Si algo de otra sección está mal o desactualizado, avísenle al responsable en vez de cambiarlo.
- Si una decisión cambia, **actualícenla**, no la dupliquen: el archivo debe reflejar siempre el estado actual.
- No borren las secciones 1 a 9. Si creen que algo del contrato debe cambiar, se acuerda con el bloque 1.

### 10.2 Verificación periódica de consistencia
Antes de cada actualización del contexto, y al menos una vez antes de entregar su sección, verifiquen (o pídanle a su IA que verifique):
- [ ] El notebook **corre de principio a fin sin errores** en un kernel reiniciado.
- [ ] `tabla_figura1()` (o `tabla_figura3()` en el bloque 4) muestra sus resultados en los 8 casos, sin celdas vacías en su parte.
- [ ] Todos los resultados se registraron con `registrar_resultado` / `registrar_clustering` y los nombres de algoritmo son los del contrato.
- [ ] No reasignaron ninguno de los nombres reservados de la sección 7 ni modificaron las secciones 0–4.10.
- [ ] Todo lo aleatorio usa `random_state=SEMILLA` y, al correr dos veces, salen los mismos resultados.
- [ ] Lo escrito en este archivo coincide con el notebook: hiperparámetros, nombres, estado y tiempos.
- [ ] Lo escrito en el markdown del notebook coincide con los resultados que realmente se obtienen (sin cifras desactualizadas).
- [ ] El markdown sigue el tono de la sección 7.1: justifica y no explica teoría, está en primera persona del plural y no tiene afirmaciones sin respaldo.
- [ ] Revisaron si otro bloque actualizó este archivo y si eso afecta su trabajo.

Si encuentran una inconsistencia, corríjanla en su sección o avísenle al responsable del bloque afectado.

### 10.3 Registro por bloque

#### Bloque 1 – Datos, preprocesamiento y evaluación comparativa
- **Estado:** secciones 0–4.10 terminadas y verificadas (el notebook corre completo en ~15 s). Las funciones del contrato están probadas con modelos de relleno. Falta la sección 7, que depende de los resultados de los bloques 2 y 3.
- **Nombres globales:** los de la sección 7 de este archivo.
- **Pendiente:** confirmar con el profesor la interpretación de "3 datos de prueba".

#### Bloque 2 – Árboles y KNN
- **Estado:** _sin iniciar_
- **Decisiones:**
- **Nombres globales nuevos:**
- **Tiempo de ejecución:**
- **Hallazgos:**
- **Dudas o cambios solicitados:**

#### Bloque 3 – SVM y redes neuronales
- **Estado:** _sin iniciar_
- **Decisiones:**
- **Nombres globales nuevos:**
- **Tiempo de ejecución:**
- **Hallazgos:**
- **Dudas o cambios solicitados:**

#### Bloque 4 – K-Means y DBSCAN
- **Estado:** _sin iniciar_
- **Decisiones:**
- **Nombres globales nuevos:**
- **Tiempo de ejecución:**
- **Hallazgos:**
- **Dudas o cambios solicitados:**

---

## 11. Instrucción sugerida para tu IA

> Estoy trabajando en el bloque X de un proyecto de aprendizaje de máquinas en un notebook de Jupyter. Te adjunto `context.md` (el contrato del equipo), `Enunciado.md`, `bloques.md` y `practica-2.ipynb`. Las secciones 0 a 4.10 ya están cerradas: no las modifiques ni vuelvas a preprocesar datos. Usa `casos[1..8]`, `SEMILLA` y las funciones del contrato (`registrar_resultado`, `predecir_demo`, `reporte_por_clase`, `graficar_matriz_confusion`, `registrar_clustering`), respetando las reglas de uso de los conjuntos y de código de `context.md`. Escribe mi sección como funciones reutilizables con docstrings en español. Escribe el markdown siguiendo exactamente la sección 7.1 de `context.md`: dirigido al profesor, en primera persona del plural, conciso, justificando cada decisión con cifras reales del notebook y nombrando las alternativas descartadas, sin explicar teoría del curso. Usa como modelo de tono el markdown de las secciones 0–4.10. Antes de escribir código, dime qué vas a hacer en cada paso.
>
> Cada vez que terminemos una parte, actualiza mi subsección en la sección 10.3 de `context.md` (estado, decisiones con su justificación, nombres globales nuevos, tiempo de ejecución, hallazgos y dudas), sin tocar las de los demás bloques. Antes de actualizarla, haz la verificación de consistencia de la sección 10.2 y dime qué encontraste.
