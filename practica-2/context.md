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

**Entorno de referencia (desde el 23/11/2026).** Las salidas guardadas y todas las cifras del markdown corresponden a la ejecución completa en macOS arm64 con Python 3.14.7, scikit-learn 1.9.1, imbalanced-learn 0.14.2 y TensorFlow 2.22.0rc1. La segunda celda de código del notebook imprime estas versiones.
- En este entorno, el notebook es determinista: dos ejecuciones completas dan exactamente las mismas salidas.
- En otro entorno (por ejemplo, Colab), SMOTENC genera registros apenas distintos y TensorFlow entrena distinto. Cambian entonces los casos balanceados (2, 4, 6 y 8), las redes y algunas decisiones automáticas (profundidad del árbol, k, kernel del SVM, configuración de la red). Antes de citar una cifra, compruébala en este entorno.

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

### 7.2 Referencias: material del curso y práctica de ejemplo
En la medida de lo posible, **las implementaciones deben parecerse a las vistas en clase**. El profesor evalúa contra ese material y en la sustentación es más fácil defender código que sigue el mismo enfoque.

| Ruta | Qué contiene | Cómo usarlo |
|---|---|---|
| `documentos/archivos-curso/*.ipynb` | Notebooks de clase: árboles (`5_01`), KNN (`5_02`), SVM lineal y no lineal (`5_03`, `5_04`), métricas y curvas (`6-01`, `6-02`), K-Means (`6-03`), DBSCAN (`6-04`), redes neuronales y Keras (`7-01` a `7-06`), CNN (`9-*`), LLM (`10-*`) | **Modelo de código.** Usar las mismas librerías, clases, parámetros y estilo de gráficas que estos notebooks. Revisar el notebook del tema antes de implementar |
| `documentos/archivos-curso/*.pdf` | Diapositivas con la teoría de cada tema | Para justificar decisiones con los conceptos y la terminología del curso |
| `documentos/Práctica2___Grupo___02___Equipo_06.pdf` | Práctica 2 ya hecha por otro equipo (ejemplo) | **Inspiración** de estructura, gráficas y tipo de análisis esperado. No copiar texto ni cifras: nuestros datos y resultados son otros |

**Regla para desviarse:** si existe una implementación distinta que sea claramente **mejor** que la del curso o la del ejemplo (más correcta, más eficiente o con mejor justificación), **no la apliquen directamente: pregúntenle primero al equipo / al usuario**, explicando la diferencia y por qué es mejor. Si la IA detecta una alternativa así, debe proponerla y esperar confirmación antes de implementarla.

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

- **"Evaluación de 3 datos de prueba": resuelto sin consultar al profesor, cubriendo las dos lecturas en ambos módulos.**
  - Registros reales de la prueba (`INDICES_DEMO`, `predecir_demo`): secciones 5, 6 y 8.3.
  - 3 estudiantes nuevos construidos a mano (`ESTUDIANTES_HIPOTETICOS`, `obtener_datos_hipoteticos`), como los "nuevos_datos" del notebook `5_02`, el "nuevo vino" del `6-03` y la "nueva flor" del `6-04`: secciones 7.5 (supervisado) y 8.3.1 (no supervisado). Se definen una sola vez en la sección 4.9.

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
- **Estado:** secciones 0–4.10 y sección 7 terminadas.
  - El notebook completo (secciones 0 a 8) corre de principio a fin sin errores en un kernel nuevo, en ~7.6 min en el entorno de referencia.
  - Dos ejecuciones completas dan exactamente las mismas salidas en las 70 celdas de código previas a la sección 7.
- **Presentación del notebook:** se quitaron las referencias internas al equipo que vería el profesor ("(bloque X)" en los títulos, "Resumen del bloque N", "contrato"). Los títulos ahora usan el nombre del contenido (por ejemplo, "5.3 Resumen de árboles y KNN"). Se agregó el encabezado "# Módulo 1 – Aprendizaje supervisado", y la instalación ya no muestra el aviso de versión de pip. Los nombres de funciones y variables (`resumen_bloque2`, `TABLA_BLOQUE3`…) no cambiaron.
- **Sección 0:** nueva celda `versiones_librerias()`, que imprime el sistema, Python y las versiones de las librerías. La nota de reproducibilidad de la sección 0 aclara que otro entorno puede cambiar las cifras de los casos balanceados y de las redes. No se fijaron versiones en `%pip`: TensorFlow 2.22.0rc1 para macOS arm64 no garantiza los mismos resultados en Linux.
- **Sección 7 (evaluación comparativa):**
  - 7.1: tabla de la Figura 1 (`tabla_figura1()`) y el mejor algoritmo de cada caso.
  - 7.2: las 4 gráficas de F1 (máximo por algoritmo; medio por normalización, por outliers y por balanceo), cada una con su análisis. La 7.2.4 explica por qué el accuracy engaña, con el recall por clase, las clases que nunca se predicen y la regla trivial "siempre Nivel 3".
  - 7.3: curvas ROC y Precision-Recall del caso 8, micro-promediadas (One-vs-Rest), con las funciones del notebook `6-02` (`roc_curve`, `precision_recall_curve`, `auc`, `average_precision_score`), y tabla de AUC y AP por clase.
  - 7.4: costo de los errores en un uso de alerta temprana, matrices de confusión del caso 8 y recall de Nivel 1 en los 8 casos.
  - 7.5: predicción de los 3 estudiantes hipotéticos con los 4 algoritmos en los 8 casos (la otra lectura de "3 datos de prueba").
  - 7.6: conclusiones del módulo supervisado.
- **Decisiones:**
  - **Curvas micro-promediadas** (una curva por algoritmo, como en la práctica de ejemplo), complementadas con AUC y AP por clase. El micro-promedio lo dominan Nivel 3 y Nivel 2, y ocultaría la diferencia entre la red y el SVM en Nivel 4 (AP de 0.633 frente a 0.406).
  - **Métrica priorizada: recall de Nivel 1, sin sacrificar el F1 macro.** El falso negativo de Nivel 1 es el error más costoso en la alerta temprana. El recall solo no basta: la red del caso 3 llega a 0.939 asignando Nivel 1 a 408 de 953 estudiantes.
  - En la sección 7 no se ajusta nada con la prueba. La comparación de la 7.4 se presenta como análisis, no como selección de modelo.
- **Nombres globales nuevos:**
  - Variables: `FIGURA1`, `F1_CASOS`, `ETIQUETAS_FACTOR`, `CASO_CURVAS`, `CURVAS_CASO8`, `AREAS_CASO8`, `MODELOS_SUPERVISADOS`, `PREDICCIONES_HIPOTETICOS`.
  - `ESTUDIANTES_HIPOTETICOS` pasó de la sección 8.3.1 a la 4.9, junto con la nueva `obtener_datos_hipoteticos(caso)`, para que la usen los dos módulos. En la 8.3.1, `preparar_hipoteticos(k)` ahora la reutiliza; los datos y las salidas no cambian.
  - Funciones: `versiones_librerias`, `f1_por_caso`, `mejor_por_caso`, `graficar_f1_maximo`, `f1_medio_por_factor`, `graficar_f1_por_factor`, `recall_por_clase`, `accuracy_frente_a_f1`, `clases_no_predichas`, `referencia_clase_mayoritaria`, `binarizar`, `curvas_roc_pr`, `graficar_roc_pr`, `areas_por_clase`, `errores_por_clase`, `recall_clase_por_caso`, `obtener_datos_hipoteticos`, `predecir_hipoteticos`, `resumen_hipoteticos`.
  - Se importan `roc_curve`, `precision_recall_curve`, `auc` y `average_precision_score`.
- **Tiempo de ejecución de la sección 7:** unos segundos (no entrena modelos).
- **Hallazgos (entorno de referencia):**
  - **F1 macro máximo:** red 0.6357 (caso 8) > SVM 0.6055 (caso 2) > árbol 0.5803 (casos 2 y 6) > KNN 0.5274 (caso 2). En los desbalanceados gana siempre el árbol; en los balanceados, el SVM sin escalado y la red con escalado.
  - **Balanceo:** es el factor dominante (de +0.014 en KNN a +0.190 en la red).
  - **Escalado:** nulo en el árbol, +0.125 en la red, −0.079 en KNN y −0.013 en el SVM (solo en los casos con RBF).
  - **Outliers:** ningún algoritmo cambia su F1 medio en 0.02 o más.
  - **Caso 8:** AUC/AP micro de 0.904/0.727 en la red, 0.894/0.695 en el SVM, 0.809/0.537 en el árbol y 0.657/0.364 en KNN (k = 1, un solo umbral).
- **Cambios respecto de la ejecución anterior, que afectan a otros bloques** (el markdown del notebook y las subsecciones de los bloques 2, 3 y 4 ya están actualizados):
  - Bloque 2:
    - `max_depth` = 10 en los casos 4 y 8 (antes 6), y estos dos casos ya no son idénticos (F1 0.5546 y 0.5533).
    - k = 17, 1, 47, 1, 37, 1, 47 y 1 (el caso 8 pasa de 11 a 1).
    - `PUNT_C_NATURALES` concentra 0.584–0.840 de la importancia.
  - Bloque 3:
    - SVM: lineal C = 1 en los desbalanceados; RBF en los balanceados (C = 10/scale en el 2, C = 1/scale en el 4 y el 6, C = 1/0.1 en el 8).
    - Red final: `Dense(64, tanh, L2 0.01) → Dense(32, tanh, L2 0.01) → Dense(4, softmax)`, con Adam lr = 0.01. Antes era ReLU con Dropout 0.3 y lr = 0.001.
    - La red del caso 8 supera al SVM (F1 0.6357 frente a 0.5876).
  - Bloque 4: solo cambian algunos decimales (silhouette de 0.2085 a 0.2086, inercias de los casos balanceados y el ruido de DBSCAN complementario del caso 8: 82 %).
- **Pendiente:** ninguno del notebook. "3 datos de prueba" quedó cubierto en sus dos lecturas (sección 9).

#### Bloque 2 – Árboles y KNN
> Actualizado por el bloque 1 el 23/11/2026 con la ejecución en el entorno de referencia (sección 2). Las cifras y algunas decisiones automáticas cambiaron respecto de la ejecución anterior; los cambios se marcan con **(cambió)**.

- **Estado:** sección 5 terminada.
  - Árboles y KNN entrenados y registrados en los **8 casos** con `registrar_resultado`, y `predecir_demo` en los 8 casos.
  - Árboles: curvas de `max_depth` contra accuracy (entrenamiento y validación) con Gini y entropía, tabla Gini frente a entropía, importancia de características, gráfico del árbol con Gini e interpretación de reglas.
  - KNN: k contra accuracy con validación cruzada de 5 folds y gráfico de vecinos con PCA, en los 8 casos.
  - Matriz de confusión y reporte por clase del caso 8 de ambos modelos.
  - El notebook completo corre de principio a fin sin errores en el entorno de referencia, y dos ejecuciones dan las mismas cifras.
- **Herramientas:** solo las de las diapositivas y notebooks de `documentos/archivos-curso/`: `DecisionTreeClassifier`, `plot_tree`, `feature_importances_` (`5_01`), `KNeighborsClassifier` (`5_02`), `StratifiedKFold` y `cross_val_score` (`5_03`, `5_04`), `PCA` (`6-03`, `7-04`) y `kneighbors` (`6-04`). No se importa ninguna librería nueva.
- **Decisiones:**
  - **Árboles:** `max_depth` de 1 a 15, elegido con el **accuracy de validación** (notebook `5_01` y diapositiva 5-02); en empate, la menor profundidad. Se registra siempre el árbol con **Gini**, que es el que el enunciado pide graficar; la entropía solo se compara. No se usa `ccp_alpha` (el enunciado pide `max_depth` y/o `ccp_alpha`).
    - Profundidades elegidas: 3 en los casos 1, 3, 5 y 7; 5 en los casos 2 y 6; **10 en los casos 4 y 8 (cambió: antes 6)**.
  - **KNN:** k impares de 1 a 51, elegido con el **accuracy promedio de la validación cruzada de 5 folds** sobre el entrenamiento, como dicen el enunciado y la diapositiva 5-03; en empate, el menor. Distancia euclidiana y voto uniforme (`5_02`). La validación cruzada usa el caso tal como viene (no se reescala por fold).
    - k elegidos: **17, 1, 47, 1, 37, 1, 47 y 1** en los casos 1 a 8 **(cambió: antes 51, 1, 49, 1, 37, 1, 47 y 11)**.
  - **Gráfico de vecinos:** PCA ajustado con el entrenamiento de cada caso, solo para dibujar; los vecinos de los 3 registros de demostración los calcula el modelo registrado con `kneighbors` en las 65 columnas.
  - Sin `class_weight`, igual que el bloque 3.
- **Cómo defender los cambios en la sustentación:**
  - **Profundidad 10 en los casos 4 y 8.** Es la que da el mayor accuracy de validación, el criterio del notebook `5_01`. La curva de validación es casi plana entre las profundidades 4 y 10 (0.612–0.637), y la 10 supera a la 6 por solo 0.001.
    - Pregunta probable: "¿no sobreajusta?" A profundidad 10, el accuracy de entrenamiento es 0.855 y el de validación 0.637.
    - Respuesta: aplicamos el criterio del curso sin tolerancia. Con una tolerancia de 0.01 como la de la red (sección 6.2.1), la elegida sería la profundidad 6.
  - **Los casos 4 y 8 ya no son idénticos** (F1 0.5546 y 0.5533). Las curvas de Gini de *k* y *k+4* difieren en 0.002 como máximo. Interpretamos que son empates entre umbrales equivalentes, y a profundidad 10 bastan para cambiar el árbol.
  - **k = 1 en el caso 8.** Pasa lo mismo que en los otros casos con SMOTENC: la validación cruzada favorece k = 1 por los registros sintéticos. En el caso 8, k = 1 empata prácticamente con k ≥ 9 (0.728). Es un ejemplo de "por qué el accuracy engaña", y está documentado en la sección 5.2.1.
- **Nombres globales nuevos:**
  - Variables: `PROFUNDIDADES_ARBOL`, `CRITERIOS_ARBOL`, `CURVAS_ARBOL`, `MODELOS_ARBOL`, `importancias`, `top3`, `VALORES_K`, `FOLDS_KNN`, `CURVAS_KNN`, `MODELOS_KNN`, `VOTOS_KNN`, `TABLA_BLOQUE2`, `PROMEDIOS_BLOQUE2`.
  - Funciones: `curva_profundidad`, `mejor_profundidad`, `graficar_profundidad`, `comparar_criterios`, `graficar_importancias`, `graficar_arbol`, `curva_k`, `mejor_k`, `graficar_curvas_k`, `resumen_eleccion_k`, `graficar_vecinos`, `resumen_bloque2`.
  - Se importan `DecisionTreeClassifier`, `plot_tree`, `KNeighborsClassifier`, `StratifiedKFold`, `cross_val_score` y `PCA`.
- **Tiempo de ejecución:** ~1 min (árboles ~15 s con los gráficos; KNN ~45 s, casi todo en la validación cruzada). El notebook completo (secciones 0 a 8) tarda ~7.6 min en el entorno de referencia.
- **Hallazgos:**
  - **F1 macro de prueba:**
    - Árboles: 0.5306–0.5803 (mejor en los casos 2 y 6). Los casos 1 y 5, 2 y 6, y 3 y 7 dan métricas idénticas; los casos 4 y 8 difieren en 0.0013.
    - KNN: 0.4291–0.5274 (mejor en el caso 2).
    - Caso 8: árbol 0.5533 y KNN 0.4335 (SVM 0.5876 y red 0.6357, del bloque 3).
  - **El árbol supera a KNN en los 8 casos.**
  - **Escalado:** no cambia el árbol (0.5506 frente a 0.5503) y **empeora a KNN** (0.5140 sin escalar frente a 0.4354 con escalado). Sin escalar, la distancia la dominan los puntajes, que son las variables más predictivas; con StandardScaler pesan más las 55 dummies, que el árbol considera casi irrelevantes.
    - En la sección 7.2.2, KNN es el algoritmo con el mayor efecto negativo del escalado. El SVM también baja un poco (−0.013), pero solo en los casos donde usa RBF.
  - **Balanceo:** sube el F1 medio del árbol de 0.5338 a 0.5671 y el de KNN de 0.4675 a 0.4819.
  - **Outliers:** sin efecto apreciable en el promedio (árbol −0.016, KNN −0.015). Las caídas mayores por par coinciden con un cambio del hiperparámetro elegido: profundidad de 5 a 10 en el árbol (pares 2 → 4 y 6 → 8) y k de 17 a 47 en KNN (par 1 → 3).
  - **La validación cruzada de KNN es engañosa en dos situaciones.**
    - Con SMOTENC (casos 2, 4, 6 y 8) elige k = 1: el accuracy de la validación cruzada es de 0.816–0.828 en los casos 2 y 4, frente a 0.515–0.588 en validación, porque los registros sintéticos tienen a su "origen" en otro fold.
    - En los casos desbalanceados (1, 3, 5 y 7) elige k = 17–47, porque los folds tienen 70 % de Nivel 3.
    - Con el k elegido, el F1 de validación queda entre 0.008 y 0.065 por debajo del mejor k posible. Se dejó el criterio del enunciado y se documentó como ejemplo de "por qué el accuracy engaña".
  - **Importancia de características:** `PUNT_C_NATURALES` concentra entre 0.584 y 0.840 de la importancia del árbol, seguida por `PUNT_LECTURA_CRITICA`. La raíz del árbol del caso 1 es ciencias ≤ 45.5, y Nivel 4 solo se predice con ciencias > 69.5.
  - El registro de demostración de Nivel 4 se predice como Nivel 3 con los dos modelos en los 8 casos, igual que con SVM y la red.
- **Dudas o cambios solicitados (para el bloque 1):**
  - **Versiones de las librerías:** resuelto por el bloque 1. La sección 0 imprime las versiones del entorno de referencia, y la sección 2 de este archivo explica qué cambia en otro entorno. No se fijaron versiones en `%pip`.

#### Bloque 3 – SVM y redes neuronales
> Actualizado por el bloque 1 el 23/11/2026 con la ejecución en el entorno de referencia (sección 2). La búsqueda y la configuración son automáticas, y en este entorno eligieron otras opciones; los cambios se marcan con **(cambió)**.

- **Estado:** sección 6 terminada.
  - SVM y Redes Neuronales entrenados y registrados en los **8 casos** con `registrar_resultado`, y `predecir_demo` en los 8 casos.
  - Matriz de confusión y reporte por clase del caso 8 de ambos modelos.
  - Regiones de decisión del SVM con PCA en los 8 casos.
  - Curvas de pérdida y accuracy de la red en los 8 casos.
  - El notebook completo corre de principio a fin sin errores en el entorno de referencia, y dos ejecuciones dan las mismas cifras.
- **Herramientas:** solo las de los notebooks de `documentos/archivos-curso/` (`5_03`, `5_04`, `6-01`, `7-03` a `7-06`): `SVC` lineal y RBF, `PCA`, `meshgrid`/`contourf`, `tf.keras.Sequential`, `Dense`, `Dropout`, `regularizers.l2`, Adam/SGD y el callback `EarlyStopping`. **No se usan** BatchNormalization, ReduceLROnPlateau, ModelCheckpoint, kernel polinómico ni `joblib`, porque no aparecen en el material del curso.
- **Decisiones:**
  - Hiperparámetros elegidos con el **F1 macro de validación** (misma métrica de la Figura 1). Sin `class_weight`, para no mezclar su efecto con el del balanceo de cada caso.
  - **SVM:**
    - Búsqueda por caso en 12 configuraciones: lineal con C ∈ {0.01, 0.1, 1} y RBF con C ∈ {0.1, 1, 10} × gamma ∈ {scale, 0.01, 0.1}. En empate gana el C menor.
    - Elegidos con el conjunto de validación, como en los experimentos del notebook `5_03`.
    - La búsqueda entrena sin probabilidades; el modelo final usa `probability=True`.
    - Elegidos **(cambió)**: lineal C = 1 en los casos 1, 3, 5 y 7; RBF C = 10, gamma = scale en el 2; RBF C = 1, gamma = scale en el 4 y el 6; RBF C = 1, gamma = 0.1 en el 8. Antes: lineal en 7 de los 8 casos.
  - **Regiones de decisión del SVM:** procedimiento del notebook `7-04`. La malla 2D de PCA se lleva a las 65 columnas con `inverse_transform` y se clasifica con el **modelo registrado**.
  - **Redes:**
    - Configuración en el caso 8, cambiando un aspecto a la vez (activación → optimizador → regularización → arquitectura). Se conserva la opción vigente si la mejora es menor de 0.01 de F1.
    - Red final **(cambió)**, la misma en los 8 casos: `Dense(64, tanh, L2 0.01) → Dense(32, tanh, L2 0.01) → Dense(4, softmax)`, Adam lr = 0.01, lotes de 32, hasta 150 épocas. Antes: ReLU, Dropout 0.3 y Adam lr = 0.001.
    - EarlyStopping sobre `val_loss` con paciencia 10 y `restore_best_weights=True`.
    - Descartadas (F1 de validación): ReLU (0.5766, frente a 0.5953 de tanh), sigmoid (0.5883), Adam lr = 0.001 (0.5953, frente a 0.6077 de lr = 0.01), SGD lr = 0.1 (0.5786), sin regularización (0.6077), Dropout 0.3 (0.5791), L2 + Dropout (0.6153), y las arquitecturas `[32]` (0.6092) y `[128, 64]` (0.6262, frente a 0.6295 de `[64, 32]`).
    - Determinismo con `fijar_semillas()` antes de cada red; no hizo falta `enable_op_determinism`.
- **Cómo defender los cambios en la sustentación:**
  - **RBF en los casos balanceados.** Gana al lineal por menos de 0.01 en los 4 casos (de 0.0005 a 0.0095). Con el umbral de 0.01 que usamos en la red, son equivalentes. La frontera sigue siendo casi lineal: las regiones de decisión son franjas ordenadas a lo largo de PC1.
  - **Red con tanh, L2 y lr = 0.01.** Cada etapa elige la variante con mayor F1 de validación, y solo cambia si mejora en 0.01 o más.
    - tanh supera a ReLU por 0.019.
    - lr = 0.01 alcanza en la primera época una pérdida de validación (0.762) menor que la mejor de lr = 0.001 (0.843).
    - L2 sube el F1 de 0.6077 a 0.6295 y reduce la brecha entre pérdidas (0.093 → 0.064), mientras que Dropout lo baja a 0.5791.
  - Pregunta probable: "¿por qué la red se detiene tan pronto?" La mejor época está entre la 2 y la 12. Respuesta: con lr = 0.01 converge en pocas épocas, EarlyStopping restaura la mejor, y L2 evita que la pérdida de validación se dispare como sin regularización.
  - **La red ahora es el mejor modelo del caso 8** (0.6357 frente a 0.5876 del SVM). Pero el SVM sigue siendo el más robusto (primero o segundo en 6 de los 8 casos) y el que más detecta Nivel 1. Por eso es el preferido para la alerta temprana (sección 7.4).
- **Nombres globales nuevos:**
  - Variables: `REJILLA_SVM`, `BUSQUEDA_SVM`, `MODELOS_SVM`, `PCA_SVM`, `CASO_CONFIGURACION`, `CONFIG_BASE_RED`, `CONFIG_RED`, `COMPARACION_RED`, `VARIANTES_ACTIVACION`, `VARIANTES_OPTIMIZADOR`, `VARIANTES_REGULARIZACION`, `VARIANTES_ARQUITECTURA`, `CONFIG_RED_FINAL`, `MODELOS_RED`, `HISTORIAS_RED`, `TABLA_BLOQUE3`, `PROMEDIOS_BLOQUE3`.
  - Funciones: `tabla_busqueda_svm`, `entrenar_svm_final`, `resumen_eleccion_svm`, `graficar_sensibilidad_svm`, `regiones_decision_svm`, `construir_red`, `entrenar_red`, `comparar_configs_red`, `elegir_variante`, `graficar_curvas_variantes`, `graficar_curvas_red`, `resumen_entrenamiento_red`, `verificar_paciencia`, `resumen_bloque3`.
  - Se importan `time`, `SVC`, `PCA`, `ListedColormap` y `tf`.
- **Tiempo de ejecución en el entorno de referencia:** ~5.5 min.
  - SVM: ~4 min (búsqueda + modelos finales).
  - Redes: ~1.5 min (configuración ~45 s; 8 casos ~22 s; el resto, verificación de paciencia y gráficas).
- **Hallazgos:**
  - **F1 macro de prueba:**
    - SVM: 0.5135–0.6055 (mejor caso 2).
    - Red: 0.2971–0.6357 (mejor caso 8).
    - Caso 8: red 0.6357 y SVM 0.5876 (la red es mejor por 0.048). El SVM detecta más Nivel 1 (recall 0.596 frente a 0.526); la red, más Nivel 3 y Nivel 4 (0.779 y 0.630 frente a 0.698 y 0.565), y es más precisa en las clases extremas.
  - **El balanceo es el factor dominante:** el F1 medio pasa de 0.5157 a 0.5915 en el SVM y de 0.3854 a 0.5751 en la red.
  - **El escalado afecta mucho a la red y poco al SVM.**
    - Red: 0.4175 sin escalar frente a 0.5430 con escalado; la mejora aparece en los 4 pares (de +0.085 a +0.162).
    - Los casos 1 y 3 (sin escalar y desbalanceados) no asignan ningún registro de prueba a Nivel 2 ni a Nivel 4 (F1 0.31 y 0.30), y con paciencia 30 tampoco. El caso 7 tampoco asigna Nivel 2.
    - Con balanceo, la red sin escalar sí aprende las cuatro clases (casos 2 y 4: 0.52 y 0.54), pero oscila entre épocas.
    - SVM: 0.5603 sin escalar frente a 0.5469 con escalado. Con lineal (casos desbalanceados) no hay diferencia; con RBF (balanceados), el escalado baja el F1 entre 0.014 y 0.034.
  - **Outliers:** sin efecto consistente (SVM 0.5526 → 0.5546; red 0.4780 → 0.4825, con pares que cambian en ambos sentidos, de −0.018 a +0.031).
  - **Kernel:** el lineal gana en los 4 casos desbalanceados y el RBF en los 4 balanceados, por menos de 0.01. La frontera entre niveles es casi lineal: las regiones de decisión son franjas ordenadas de Nivel 1 a Nivel 4 a lo largo de PC1 (puntajes).
  - Los errores son casi todos entre niveles vecinos: en el caso 8, 8 de 329 en el SVM y 11 de 295 en la red saltan más de un nivel.
- **Dudas o cambios solicitados:** ninguno.

#### Bloque 4 – K-Means y DBSCAN
> Cifras actualizadas por el bloque 1 el 23/11/2026 con la ejecución en el entorno de referencia (sección 2). En este bloque solo cambiaron algunos decimales.

- **Estado:** sección 8 terminada.
  - K-Means y DBSCAN entrenados y registrados en los **8 casos** con `registrar_clustering`; `tabla_figura3()` completa (el silhouette de DBSCAN con la regla queda en NaN; ver hallazgos).
  - Codo y silhouette (K de 2 a 10), tamaños de los clusters, efecto del escalado y de los outliers.
  - k-distancia, DBSCAN con la regla, sensibilidad a ε y prueba complementaria.
  - 3 datos de prueba por técnica y caso, las 2 gráficas de silhouette (8 barras cada una), comparación K-Means vs. DBSCAN, PCA 2D (centroides y ruido en gris) y validación externa contra `y_ext`.
  - El notebook completo corre de principio a fin sin errores en un kernel reiniciado (secciones 0 a 8).
- **Herramientas:** solo las de las diapositivas 6-02 y 6-03 y los notebooks `6-03` y `6-04`: `KMeans`, `DBSCAN`, `NearestNeighbors`, `silhouette_score` y `PCA` (solo para graficar). La validación externa usa `pd.crosstab` con el formato de la matriz de confusión (diapositiva 6-01). No se usan t-SNE ni ARI, porque no aparecen en el material.
- **Decisiones:**
  - **Datos:** `obtener_datos_no_supervisado(k)`; se agrupa solo el entrenamiento de cada caso. La etiqueta se usa únicamente en 8.7.
  - **K-Means:** `init="k-means++"`, `n_init=10` (valor del notebook `6-03`), `random_state=SEMILLA`. K de 2 a 10, elegido con el **silhouette máximo**: K = 2 en los 8 casos. El codo (leído con el punto más alejado de la recta entre extremos) marca K = 4 sin escalar y K = 5 con escalado, pero ahí el silhouette baja a 0.26–0.34 y 0.09–0.14.
  - Los clusters de K-Means se **renumeran**: C0 = menor puntaje medio en las otras 4 áreas, C1 = mayor. No cambia inercia ni silhouette.
  - **DBSCAN (Figura 3):** MinPts = 2 × 65 = **130** (regla del enunciado y de la diapositiva 6-03) y ε en el codo de la k-distancia con k = MinPts (ε entre 3.64 y 16.55). Resultado: **1 cluster en los 8 casos** (0.1–3.3 % de ruido), silhouette NaN. La segunda regla de la diapositiva (MinPts = 66) da lo mismo.
  - **DBSCAN, prueba complementaria** (acordada con el equipo para que la gráfica de DBSCAN tenga valores): el ε más cercano al codo que separa ≥ 2 clusters, bajando ε en pasos de 2 % hasta el 40 % del codo; si con MinPts = 130 no aparecen, se baja a 66 y luego a 10 (la diapositiva 6-03 permite reducir MinPts con mucho ruido). Se reporta aparte, no reemplaza la regla.
  - **3 datos de prueba:** `INDICES_DEMO`. K-Means con `predict`; DBSCAN asignando al cluster del *core sample* más cercano si está a ≤ ε, o ruido. No se usa el método del notebook `6-04` (reentrenar con el punto nuevo) porque cambia el modelo.
  - **3 estudiantes hipotéticos (8.3.1):** como no se pudo confirmar con el profesor si los "3 datos de prueba" son registros reales o casos nuevos (el enunciado dice "generación" en el módulo no supervisado), cubrimos las dos lecturas: además de `INDICES_DEMO`, tres perfiles construidos a mano (bajo, medio y alto), como el "nuevo vino" del notebook `6-03`, preprocesados con `codificar_ordinales`, `aplicar_one_hot` y `aplicar_escalado` del bloque 1. Desde el 23/11 se definen en la sección 4.9 (`ESTUDIANTES_HIPOTETICOS`, `obtener_datos_hipoteticos`), porque el módulo supervisado también los usa (sección 7.5).
- **Nombres globales nuevos:**
  - Variables: `DATOS_NS`, `COLUMNAS_NUMERICAS_NS`, `RANGO_K`, `N_INIT_KMEANS`, `CURVAS_KMEANS`, `K_CODO`, `K_SILHOUETTE`, `K_OPTIMO`, `ELECCION_K`, `PUNTAJES_NS`, `MODELOS_KMEANS`, `MAPA_KMEANS`, `ETIQUETAS_KMEANS`, `TAMANOS_KMEANS`, `N_CARACTERISTICAS`, `MINPTS_REGLA`, `MINPTS_REGLA_2`, `KDIST`, `EPS_CODO`, `MODELOS_DBSCAN`, `filas_reglas`, `REGLAS_DBSCAN`, `FRACCIONES_EPS`, `SENSIBILIDAD_DBSCAN`, `ESCALERA_MINPTS`, `PASO_EPS`, `FRACCION_MINIMA`, `DBSCAN_COMPLEMENTARIO`, `TABLA_DBSCAN_COMPLEMENTARIO`, `DEMO_CLUSTERING`, `FIGURA3`, `PERFILES_KMEANS`, `PROMEDIOS_BLOQUE4`, `DEMO_HIPOTETICOS` (`ESTUDIANTES_HIPOTETICOS` pasó a la sección 4.9).
  - Funciones: `tabla_figura2`, `participacion_varianza`, `curva_kmeans`, `indice_codo`, `graficar_curvas_kmeans`, `orden_clusters_kmeans`, `predecir_kmeans`, `nombre_cluster`, `tamanos_clusters`, `mascara_outliers_ns`, `efecto_factor_kmeans`, `outliers_por_cluster`, `coincidencia_escalado`, `k_distancias`, `eps_codo`, `graficar_kdistancia`, `resumen_dbscan`, `sensibilidad_eps`, `dbscan_complementario`, `mascara_sinteticos`, `sinteticos_por_cluster`, `predecir_dbscan`, `demo_clustering`, `preparar_hipoteticos`, `demo_hipoteticos`, `graficar_silhouette_figura3`, `graficar_pca_clusters`, `graficar_validacion_externa`, `perfil_clusters`, `resumen_bloque4`.
  - Se importan `KMeans`, `DBSCAN`, `NearestNeighbors` y `PCA`.
- **Tiempo de ejecución:** ~1.5 min (K-Means de 2 a 10 ~35 s; prueba complementaria de DBSCAN ~30 s; el resto, segundos).
- **Hallazgos:**
  - **K-Means separa desempeño bajo y alto:** C0 con puntajes medios de 45–49 en lectura crítica, estrato 1.7–2.0 y 17.1–18.0 años; C1 con 64–67, estrato 2.4–2.6 y 16.5–16.9 años. C0 casi no tiene Nivel 4 (0.0–0.4 %) y C1 casi no tiene Nivel 1 (0.1–0.4 %), pero los niveles intermedios se mezclan (Nivel 3 es el 60–62 % de C0 en los desbalanceados).
  - **Silhouette de K-Means:** 0.41–0.56 sin escalar y 0.17–0.25 con escalado. Por factor: ED 0.4864 → 0.2086; outliers 0.3513 → 0.3437; balanceo 0.2970 → 0.3980.
  - **El escalado baja el silhouette pero no cambia los grupos** (89.5–94.7 % de registros en el mismo cluster entre k y k+4). Sin escalar, las 5 numéricas aportan el 97.1 % de la varianza.
  - **Outliers:** sin efecto (≤ 0.019 de silhouette); se reparten 4.0–6.0 % en cada cluster.
  - **DBSCAN no encuentra estructura de densidad:** la k-distancia sube de forma continua. Al bajar ε, el ruido crece antes de que aparezca un segundo cluster (caso 5: 15 % → 90 % entre el 80 % y el 70 % del codo).
  - **Prueba complementaria:** silhouette 0.025–0.477, con 7–84 % de ruido; en los casos 3 y 5 el segundo cluster tiene solo 6–7 registros. En los casos balanceados, los clusters densos son 76–85 % registros sintéticos de SMOTENC (frente al 46–48 % del caso): densidad artificial.
  - **3 datos de prueba:** K-Means pone el registro de Nivel 1 en C0 y los de Nivel 3 y Nivel 4 en C1 en los 8 casos. Con los estudiantes hipotéticos, el perfil bajo y el medio van a C0 y el alto a C1 en los 8 casos (la frontera está cerca de 55 puntos).
  - **Para la sección 7 y el documento:** la conclusión del módulo no supervisado es que el desempeño es un continuo; los modelos supervisados confunden niveles vecinos por la misma razón.
- **Dudas o cambios solicitados:** ninguno para otros bloques. `mascara_sinteticos` supone que SMOTENC deja los registros reales al inicio y los sintéticos al final (comportamiento de `imbalanced-learn`); si el bloque 1 cambia el balanceo, hay que revisarla.

---

## 11. Instrucción sugerida para tu IA

> Estoy trabajando en el bloque X de un proyecto de aprendizaje de máquinas en un notebook de Jupyter. Te adjunto `context.md` (el contrato del equipo), `Enunciado.md`, `bloques.md` y `practica-2.ipynb`. Las secciones 0 a 4.10 ya están cerradas: no las modifiques ni vuelvas a preprocesar datos. Usa `casos[1..8]`, `SEMILLA` y las funciones del contrato (`registrar_resultado`, `predecir_demo`, `reporte_por_clase`, `graficar_matriz_confusion`, `registrar_clustering`), respetando las reglas de uso de los conjuntos y de código de `context.md`. Escribe mi sección como funciones reutilizables con docstrings en español. Escribe el markdown siguiendo exactamente la sección 7.1 de `context.md`: dirigido al profesor, en primera persona del plural, conciso, justificando cada decisión con cifras reales del notebook y nombrando las alternativas descartadas, sin explicar teoría del curso. Usa como modelo de tono el markdown de las secciones 0–4.10. Sigue la sección 7.2: implementa de forma parecida a los notebooks y PDFs de `documentos/archivos-curso/` e inspírate en la práctica de ejemplo `documentos/Práctica2___Grupo___02___Equipo_06.pdf`; si ves una implementación diferente y mejor, pregúntame antes de usarla. Antes de escribir código, dime qué vas a hacer en cada paso.
>
> Cada vez que terminemos una parte, actualiza mi subsección en la sección 10.3 de `context.md` (estado, decisiones con su justificación, nombres globales nuevos, tiempo de ejecución, hallazgos y dudas), sin tocar las de los demás bloques. Antes de actualizarla, haz la verificación de consistencia de la sección 10.2 y dime qué encontraste.
