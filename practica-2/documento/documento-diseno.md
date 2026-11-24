---
title: "Trabajo Práctico 2 – Aprendizaje de Máquinas"
subtitle: "Documento de diseño"
lang: es
# Datos de la portada (no salen en el cuerpo; los usa el generador del Word/PDF)
universidad: "Universidad Nacional de Colombia – Sede Medellín"
curso: "Introducción a la Inteligencia Artificial"
profesor: "Jaime Alberto Guzmán Luna"
grupo: "Grupo 1 — Equipo 6"
integrantes:
  - "Juan Pablo Mira Cañas – 1033178679"
  - "Emmanuel Valencia Lopera – 1025889015"
  - "José Luis Sánchez Álvarez – 1037121919"
  - "Samuel David Andrade Rivera – 1033179160"
video: "https://youtu.be/T7Bg3Ndw-U4"
fecha: "Medellín, 24 de noviembre de 2026"
---

**Cédula usada:** 1033178679

**Semilla:** 79

**Dataset original:** [Resultados únicos Saber 11 – ICFES](https://drive.google.com/file/d/1gt26zB3NPERbmZ1zwE4fsRLixuNNF6gq/view?usp=sharing)

Cada sección de este documento indica entre paréntesis la sección del notebook donde está el código y el detalle completo.

## Resumen

Predecimos el nivel de desempeño en matemáticas (Nivel 1 a 4) de los estudiantes que presentaron el Saber 11 de 2022, a partir de sus puntajes en las otras áreas y de información de su familia y su colegio. Construimos 8 versiones del entrenamiento combinando escalado, outliers y balanceo, y en cada una entrenamos un árbol de decisión, KNN, una SVM y una red neuronal. El balanceo fue lo que más mejoró los modelos; el escalado ayudó a la red, perjudicó a KNN y no cambió el árbol; y los outliers casi no cambiaron nada. El mejor resultado fue el de la SVM en el caso 2 (F1 macro de 0.601), pero en los casos balanceados el árbol, la SVM y la red quedan prácticamente empatados (a menos de 0.025 entre sí). El árbol fue la técnica más estable entre casos.

Después quitamos la etiqueta y agrupamos los mismos datos con K-Means y DBSCAN. K-Means encontró dos grupos, desempeño bajo y desempeño alto, que separan bien el Nivel 1 del Nivel 4 pero no los niveles intermedios. DBSCAN, con la regla del enunciado, dejó todo en un solo cluster. Ambos resultados indican que el desempeño se comporta como un continuo, y por eso todos los clasificadores se equivocan sobre todo entre niveles vecinos.

<div style="page-break-before: always"></div>

## Decisiones principales

| Decisión | Justificación | Notebook |
|---|---|---|
| Generación verificable desde el original | La huella SHA-256 y la comparación con el entregado demuestran que el subconjunto es reproducible | 1.1 |
| Subconjunto en el ZIP, con copia en Drive | El notebook se ejecuta sin pasos manuales, tanto en local como en Colab | 1.2 |
| Conservar los puntajes de las otras áreas | No se derivan del puntaje de matemáticas, así que no constituyen fuga | 1.3 |
| Eliminar 14 columnas | Cada eliminación está respaldada con datos: no aportan información o la duplican | 2 |
| Imputar en lugar de eliminar filas | Los nulos no son aleatorios; eliminarlos sesgaría la muestra | 3.4 |
| Correlación de Spearman | 5 de las variables analizadas son ordinales | 3.5 |
| Balancear antes de escalar | Los casos *k* y *k+4* conservan los mismos registros | 4 |
| Codificación ordinal para 5 variables y One-Hot para 13 | Se conserva el orden donde existe y no se impone donde no lo hay | 4.3 |
| StandardScaler | KNN, SVM y las redes requieren escala comparable, y se distorsiona menos que MinMax con outliers | 4.4 |
| IQR (1.5) para detectar outliers | No asume normalidad, y `EDAD` es muy asimétrica | 4.5 |
| Sin outliers: eliminar las filas | Son ~3 % del entrenamiento; recortarlas alteraría valores reales | 4.5 |
| Con outliers: reales más inyectados hasta el 5 % | Se conservan los atípicos propios del dominio y se garantiza el 5 % exacto | 4.5 |
| Desbalanceado: submuestreo a 70/10/10/10 | Solo usa registros reales; Nivel 4 determina el tamaño | 4.6 |
| Balanceado: SMOTENC | Genera registros nuevos sin crear categorías inexistentes | 4.6 |
| Curvas ROC y PR micro-promediadas, con AUC y AP por clase | Una curva por algoritmo para comparar las 4 técnicas; el área por clase muestra Nivel 4, que el micro-promedio oculta | 7.3 |
| Métrica priorizada: recall de Nivel 1, sin sacrificar el F1 macro | En un uso de alerta temprana, no detectar a un estudiante de Nivel 1 es el error más costoso | 7.4 |

La columna *Notebook* indica la sección del notebook donde está el código de cada decisión.

# Módulo 1 – Aprendizaje supervisado

## 1. Dataset *(notebook: sección 1)*

### 1.1 Dominio y justificación

Elegimos el dominio de la educación. El desempeño en matemáticas del Saber 11 pesa en el acceso a la educación superior, y predecirlo a partir del contexto del estudiante y de sus puntajes en las otras áreas permite identificar a tiempo a quienes tienen riesgo de quedar en los niveles bajos. Como esos puntajes salen del mismo examen, en la práctica el modelo se usaría con los de un simulacro previo, como herramienta de alerta temprana (sección 5.4). La variable objetivo es `DESEMP_MATEMATICAS`, con los rangos oficiales del ICFES: Nivel 1 (0–35), Nivel 2 (36–50), Nivel 3 (51–70) y Nivel 4 (71–100).

### 1.2 Fuente y generación del subconjunto

Partimos del archivo *Resultados únicos Saber 11* del ICFES (datos abiertos): 1 085 937 registros y 51 columnas de las aplicaciones 2022-1 y 2022-2. Para obtener el subconjunto:

1. Descartamos los registros sin puntaje de matemáticas válido y construimos la etiqueta con los rangos del ICFES.
2. Eliminamos las variables con fuga de información (`PUNT_MATEMATICAS`, de donde sale la etiqueta; `PUNT_GLOBAL`, que la incluye; y los percentiles) y los identificadores.
3. Extrajimos 6350 registros con muestreo estratificado por la etiqueta y `random_state = 79`. Estratificamos para que el Nivel 4 (4.9 %) no quedara sub- o sobrerrepresentado por azar.

El notebook puede regenerar el subconjunto desde el original (`GENERAR_SUBCONJUNTO = True`). Verifica la huella SHA-256 del archivo de partida, porque el muestreo depende del orden de las filas. El CSV va en el ZIP y tiene una copia en Drive, para que el notebook corra en Colab sin pasos manuales.

### 1.3 Verificación de requisitos y variables

| Requisito | Subconjunto |
|---|---|
| ≥ 5000 registros | 6350 |
| ≥ 4 clases | Nivel 1: 11.9 % · Nivel 2: 36.6 % · Nivel 3: 46.6 % · Nivel 4: 4.9 % |
| Numéricas y categóricas | 23 predictores tras la limpieza: 5 numéricos, 5 ordinales y 13 nominales |

La descripción de las 36 variables del subconjunto y el motivo para usar o descartar cada una están en el anexo A.

| Tipo | Variables |
|---|---|
| Numéricas (5) | Puntajes de lectura crítica, ciencias naturales, sociales y ciudadanas e inglés (0–100); `EDAD` (calculada desde la fecha de nacimiento) |
| Ordinales (5) | Estrato, educación de la madre, educación del padre, cuartos del hogar, personas del hogar |
| Nominales (13) | Área (urbano/rural), bilingüe, carácter, departamento, género del colegio, jornada, naturaleza (oficial o no), sede principal, género del estudiante, y si el hogar tiene automóvil, computador, internet y lavadora |

Conservamos los puntajes de las otras áreas como predictores porque son pruebas independientes del mismo examen: no se derivan del puntaje de matemáticas, así que no hay fuga. Además, son prácticamente las únicas variables numéricas disponibles para el tratamiento de outliers.

## 2. Limpieza y análisis exploratorio *(notebook: secciones 2 y 3)*

### 2.1 Limpieza de variables

Además de lo eliminado al generar el subconjunto, quitamos 14 columnas. Cada eliminación está respaldada con un dato:

| Columna | Evidencia | Decisión |
|---|---|---|
| `ESTU_ESTADOINVESTIGACION`, `ESTU_ESTUDIANTE` | Un solo valor en el 100 % de las filas | Eliminar (constantes) |
| `COLE_CALENDARIO`, `ESTU_NACIONALIDAD`, `ESTU_PRIVADO_LIBERTAD` | Un valor ocupa el 98.3 %, 98.7 % y 99.9 % (umbral: 98 %) | Eliminar (casi constantes) |
| `PERIODO` | 98.3 % es 2022-2; su relación con el nivel viene del calendario B, ya reflejado en naturaleza, bilingüe y estrato | Eliminar (administrativa) |
| `ESTU_FECHANACIMIENTO` | Texto con ~1700 valores distintos | Reemplazar por `EDAD` |
| Municipios del colegio, de residencia y de presentación | 453–774 categorías, muchas con menos de 5 registros | Eliminar (se usa el departamento) |
| Departamentos de residencia y de presentación | Coinciden con el del colegio en el 98.9 % y 97.9 % | Eliminar (redundantes) |
| `ESTU_PAIS_RESIDE` | Igual a la nacionalidad en el 100 % | Eliminar (duplicada) |
| `DESEMP_INGLES` | Rangos de `PUNT_INGLES` (ρ de Spearman ≈ 0.94) | Eliminar (redundante) |

`COLE_GENERO` se conserva aunque el 97.4 % sea mixto, porque en los colegios femeninos y masculinos el Nivel 4 llega al 12–14 %, frente al 4.7 % en los mixtos. También calculamos `EDAD` al 1 de septiembre de 2022, pasamos a nulo las edades fuera de 13–80 años (fechas imposibles) y tratamos "No sabe" y "No aplica" en la educación de los padres como faltantes. Sin esta limpieza, el One-Hot habría generado miles de columnas; con ella el dataset queda en **65 columnas**.

### 2.2 Análisis exploratorio

El subconjunto tiene 6350 filas y 37 columnas; después de la limpieza quedan 24 (23 predictores y el objetivo). Estadísticas descriptivas de las variables numéricas (subconjunto completo):

| Variable | Media | Desv. est. | Mín. | Mediana | Máx. |
|---|---|---|---|---|---|
| Lectura crítica | 53.0 | 10.8 | 0 | 53 | 100 |
| Ciencias naturales | 49.0 | 10.5 | 25 | 48 | 100 |
| Sociales y ciudadanas | 47.7 | 12.0 | 19 | 47 | 100 |
| Inglés | 50.2 | 13.4 | 0 | 49 | 100 |
| Edad | 17.3 | 2.9 | 13 | 17 | 67 |

![Distribución de clases y matriz de correlación de Spearman](figuras/eda_clases_correlacion.png)

- **Desbalance natural.** Nivel 3 y Nivel 2 suman el 83 %, y Nivel 4 tiene el 4.9 %. Un clasificador que siempre prediga Nivel 3 tendría un accuracy de ~0.47, por eso priorizamos el **F1 macro** y el recall por clase.
- **Predictores principales.** Los puntajes de las otras áreas tienen ρ entre 0.61 y 0.74 con el objetivo. Les siguen la educación de los padres (ρ ≈ 0.27) y el estrato (ρ ≈ 0.16). La edad tiene correlación negativa (ρ ≈ −0.26).
- **Los puntajes están correlacionados entre sí** (ρ entre 0.69 y 0.82), lo que reparte la importancia entre ellos en el árbol.
- **Hay atípicos reales:** edades de hasta 67 años (jornadas nocturna y sabatina) y puntajes de 0 y de 100.

Usamos Spearman en lugar de Pearson porque 5 de las variables y el objetivo son ordinales.

### 2.3 Valores nulos

El 34 % de las filas (2161) tiene algún nulo, repartido en 13 de las 24 variables. Las que más tienen son `COLE_BILINGUE` (17.7 %), la educación del padre (13.8 %), la de la madre (8.3 %) y el estrato (7.1 %); las demás están por debajo del 6 %, y `PUNT_INGLES` y `EDAD` por debajo del 0.5 %. Los nulos no son aleatorios: las filas con nulos tienen más Nivel 1 (14.8 % frente a 10.4 %) y más colegios rurales (21.9 % frente a 14.4 %). Eliminarlas sesgaría la muestra contra los estudiantes rurales y de bajo desempeño, así que **imputamos**: moda en las categóricas y mediana en `PUNT_INGLES` y `EDAD` (robusta a la cola de edades altas), calculadas solo con el entrenamiento.

## 3. Partición y preprocesamiento: los 8 casos de la Figura 1 *(notebook: sección 4)*

Todo ajuste que aprende de los datos se hace **solo con el entrenamiento**. Validación y prueba solo se transforman, conservan la distribución real y son las mismas en los 8 casos.

| Paso | Qué se hace | Se ajusta con |
|---|---|---|
| 1. Partición | 70 / 15 / 15 estratificada (4445 / 952 / 953), `random_state = 79` | Una sola vez |
| 2. Imputación | Mediana (numéricas) y moda (categóricas) | Entrenamiento base |
| 3. Outliers NO | Se eliminan las filas detectadas con IQR | Límites del entrenamiento base |
| 4. CC ordinal | Cada ordinal se reemplaza por su posición en el orden del dominio | Orden fijo |
| 5. Balanceo | NO: submuestreo a 70/10/10/10 · SI: SMOTENC | Entrenamiento del caso |
| 6. Outliers SI | Se conservan los reales y se inyectan sintéticos hasta el 5 % | Límites del entrenamiento base |
| 7. CC One-Hot | 13 variables nominales | Entrenamiento base |
| 8. ED | StandardScaler sobre numéricas y ordinales (casos 5–8) | Entrenamiento del caso |

**Orden de los pasos.** Los outliers se eliminan antes de balancear, para que SMOTE no genere registros a partir de atípicos. Se balancea antes de escalar, para que los casos *k* y *k+4* tengan los mismos registros y difieran solo en el escalado. Los outliers se inyectan después de balancear, para que sean exactamente el 5 % del entrenamiento final.

- **Conversión de categóricas (CC).** Codificación ordinal para las 5 variables con orden natural (por ejemplo, de *Ninguno* a *Postgrado*). *Sin estrato* va en la posición más baja porque tiene el peor desempeño (31 % en Nivel 1, frente a 13 % en estrato 1). One-Hot para las 13 nominales, porque un número les impondría un orden que no tienen; `drop="if_binary"` deja una columna por variable binaria. El One-Hot se ajusta una sola vez para que los 8 casos tengan las mismas 65 columnas.
- **Escalado (ED).** StandardScaler sobre numéricas y ordinales; las dummies quedan en 0/1. Lo preferimos a MinMaxScaler porque se distorsiona menos con outliers: en MinMax un solo valor extremo fija el máximo, y los casos 7 y 8 tienen outliers.
- **Outliers.** Detección con IQR (factor 1.5) sobre las 5 numéricas; no usamos Z-score porque `EDAD` es muy asimétrica. *Sin outliers*: eliminamos las filas detectadas (137, ~3 % del entrenamiento), sin winsorizar, porque eso alteraría valores reales. *Con outliers*: conservamos los reales e inyectamos sintéticos hasta el 5 %, cambiando 1 o 2 variables numéricas de filas normales (semilla 79) por un valor fuera de los límites IQR pero posible (0–100 en puntajes, 13–80 en edad). Las variables modificadas fueron los cuatro puntajes y la edad.

  | Casos | Outliers reales | Inyectados | Total en entrenamiento |
  |---|---|---|---|
  | 1, 2, 5, 6 (sin outliers) | 137 filas eliminadas | 0 | 0 % |
  | 3 y 7 (desbalanceados) | 55 | 54 | 109 (5.02 %) |
  | 4 y 8 (balanceados) | 368 | 47 | 415 (5.00 %) |

  En los casos balanceados hacen falta menos inyectados, porque SMOTENC ya genera atípicos al sobremuestrear el Nivel 1, que concentra la extraedad.
- **Balanceo.** *Desbalanceado*: submuestreo a 70/10/10/10 con Nivel 3 como mayoritaria; solo usa registros reales y Nivel 4 (205 registros) fija el tamaño. *Balanceado*: SMOTENC hasta igualar la mayoritaria. Lo elegimos frente a duplicar registros (favorece el sobreajuste), al submuestreo (descartaría ~80 % de los datos) y al SMOTE clásico (interpolaría las dummies y crearía categorías inexistentes).

| Caso | ED | Outliers | Balanceo | Registros de entrenamiento | Distribución (N1/N2/N3/N4) |
|---|---|---|---|---|---|
| 1 | NO | NO | NO | 2050 | 10/10/70/10 % |
| 2 | NO | NO | SI | 8208 | 25 % cada uno |
| 3 | NO | SI | NO | 2170 | 10/10/70/10 % |
| 4 | NO | SI | SI | 8296 | 25 % cada uno |
| 5 | SI | NO | NO | 2050 | 10/10/70/10 % |
| 6 | SI | NO | SI | 8208 | 25 % cada uno |
| 7 | SI | SI | NO | 2170 | 10/10/70/10 % |
| 8 | SI | SI | SI | 8296 | 25 % cada uno |

Los casos desbalanceados quedan con ~2100 registros (~32 por columna). Lo consideramos suficiente: la prueba es la misma para todos y el menor desempeño (recall bajo en Nivel 2 y Nivel 4) es justamente el efecto que el enunciado pide analizar.

**Evaluación común.** Todas las métricas de prueba las calcula una sola función (`registrar_resultado`, notebook, sección 4.10), con promedio **macro** para que el Nivel 4 (4.9 %) pese igual que el Nivel 3 (46.6 %). Los "3 datos de prueba" admiten dos lecturas y cubrimos ambas, con los mismos datos para todas las técnicas: 3 registros reales de la prueba (uno de Nivel 1, uno de Nivel 3 y uno de Nivel 4) y 3 estudiantes nuevos construidos a mano (perfiles bajo, medio y alto), como los "nuevos_datos" del notebook de clase 5_02 (notebook, sección 4.9).

## 4. Entrenamiento de los modelos supervisados *(notebook: secciones 5 y 6)*

En las cuatro técnicas los hiperparámetros se eligen sin mirar la prueba, seguimos las herramientas de los notebooks de clase y no usamos `class_weight`, para que la Figura 1 aísle el efecto de la versión del dataset.

### 4.1 Árboles de decisión *(sección 5.1)*

Para cada caso entrenamos árboles con `max_depth` de 1 a 15 con Gini y con entropía, y elegimos la profundidad con mayor accuracy de validación (en empate, la menor). Usamos `max_depth` y no `ccp_alpha` porque la profundidad se lee directamente en el gráfico.

![max_depth frente a accuracy de entrenamiento (continua) y validación (discontinua)](figuras/arbol_profundidad.png){width=90%}

- **Subajuste:** con `max_depth` = 1 el accuracy de validación es 0.508 (caso 1) y menos de 0.2 en los casos balanceados. **Sobreajuste:** con `max_depth` = 15 el entrenamiento llega a 0.98 en los casos desbalanceados (0.94–0.96 en los balanceados) mientras validación se queda cerca de 0.57–0.60.
- **Profundidades elegidas:** 3 en los casos desbalanceados, 5 en los casos 2 y 6 y 6 en los casos 4 y 8. En estos dos, la curva de validación es casi plana entre las profundidades 4 y 10 (alrededor de 0.61–0.64).
- **Gini frente a entropía:** las curvas casi se superponen; Gini gana en 6 de 8 casos. En los casos 3 y 7 la entropía elige profundidad 13 y sobreajusta (0.969 en entrenamiento). Registramos el árbol con Gini.
- **Los casos *k* y *k+4* dan el mismo árbol:** los cortes son de una variable a la vez y escalar no cambia el orden de los valores, así que los pares 1–5, 2–6, 3–7 y 4–8 tienen exactamente las mismas métricas.

![Importancia de características en los 8 casos](figuras/arbol_importancia.png){width=90%}

`PUNT_C_NATURALES` concentra entre 0.747 y 0.840 de la importancia y `PUNT_LECTURA_CRITICA` es la segunda (0.074–0.139). Los otros puntajes casi no aparecen porque están correlacionados con ciencias. Con balanceo entran `EDAD` (0.043–0.057) y `FAMI_TIENECOMPUTADOR_Si`.

![Árbol solución con Gini del caso 1 (los árboles de los 8 casos están en el notebook, sección 5.1.5)](figuras/arbol_caso1.png)

**Interpretación (caso 1).** La raíz es ciencias ≤ 45.5. La regla de riesgo más clara es ciencias ≤ 39.5 y lectura ≤ 46.5: en esa hoja, 128 de 174 estudiantes son de Nivel 1. El Nivel 4 solo se predice con ciencias > 69.5, muy cerca del corte oficial de Nivel 4 en matemáticas (71). El Nivel 2 aparece en hojas impuras, lo que concuerda con su recall de 0.235. Con balanceo (caso 4), la edad y tener computador aparecen en las ramas de puntaje alto, pero el nivel lo deciden los puntajes.

### 4.2 KNN *(sección 5.2)*

Elegimos *k* (impares de 1 a 51) con el accuracy promedio de una validación cruzada estratificada de 5 folds sobre el entrenamiento, con distancia euclidiana y voto uniforme.

![k frente a accuracy de la validación cruzada de 5 folds](figuras/knn_k.png){width=90%}

- **Desbalanceados (k = 37–51):** la curva queda casi plana desde k ≈ 9 (0.76–0.79). Ese accuracy está inflado porque los folds tienen 70 % de Nivel 3; en validación real baja a 0.548–0.591.
- **Balanceados (k = 1 en los 4 casos):** en los casos 2 y 4 la validación cruzada da 0.82–0.83 porque un registro sintético de SMOTENC suele tener su "registro de origen" en el mismo fold. En validación real, k = 1 baja a 0.52–0.59. En los casos 6 y 8 la ventaja de k = 1 es mínima (0.737 y 0.729).
- Dejamos el *k* del criterio del enunciado y lo reportamos como ejemplo de que el accuracy engaña.

![Vecinos de los 3 datos de prueba (estrellas), PCA a 2D](figuras/knn_vecinos.png){width=85%}

Los vecinos los calcula el modelo en las 65 columnas; PCA solo se usa para dibujar. Sin escalar, PC1 explica el 73–87 % de la varianza (los puntajes) y los vecinos aparecen agrupados alrededor de cada estrella. Con escalado, PC1 explica el 28–37 % y los vecinos se ven dispersos.

### 4.3 SVM *(sección 6.1)*

Evaluamos 12 configuraciones por caso con el F1 macro de validación: lineal con C ∈ {0.01, 0.1, 1} y RBF con C ∈ {0.1, 1, 10} × gamma ∈ {scale, 0.01, 0.1}. No usamos `GridSearchCV` porque, con SMOTENC, la validación cruzada resulta optimista.

- **Kernel:** el lineal gana en los 4 casos desbalanceados (por 0.003–0.043 de F1 de validación). En los balanceados la diferencia entre kernels es de solo 0.002–0.013 y el ganador cambia de un caso a otro (RBF en los casos 2 y 6, lineal en los casos 4 y 8), así que ahí son equivalentes. La frontera entre niveles es casi lineal porque los puntajes se mueven juntos y el objetivo es ordinal.
- **C:** casi no cambia el resultado del lineal (F1 de validación entre 0.59 y 0.62 en los casos 4 y 8). En el RBF, un C alto sobreajusta: en el caso 8, con C = 10 y gamma = scale, accuracy de entrenamiento 0.921 y de validación 0.643.
- **gamma:** es lo más sensible a la escala. Con gamma = 0.1 el F1 de validación cae a 0.30–0.42 en el caso 4 (sin escalar), frente a 0.57–0.60 en el caso 8, donde queda entre las mejores configuraciones RBF.

![Regiones de decisión de la SVM registrada (PCA a 2D)](figuras/svm_regiones.png){width=85%}

Para las regiones llevamos una malla del plano PCA a las 65 columnas con `inverse_transform` y la clasificamos con el modelo registrado. En los 8 casos salen franjas ordenadas de Nivel 1 a Nivel 4 a lo largo de PC1; con RBF, la frontera de Nivel 4 se curva en el caso 6. Sin balanceo, la franja de Nivel 2 casi desaparece, lo que concuerda con su recall medio de 0.13.

### 4.4 Redes neuronales *(sección 6.2)*

Configuramos la red en el caso 8 cambiando un aspecto a la vez y usamos la misma red en los 8 casos. Un cambio solo se acepta si mejora el F1 de validación en al menos 0.01.

| Etapa | Variantes (F1 de validación) | Elegida |
|---|---|---|
| Activación | ReLU 0.5981 · tanh 0.6012 · sigmoid 0.6047 | ReLU (ninguna mejora llega a 0.01) |
| Optimizador | Adam lr 0.001: 0.5981 · Adam lr 0.01: 0.5624 · SGD lr 0.1: 0.4884 | Adam lr 0.001 |
| Regularización | Ninguna 0.5981 · L2 0.01: 0.6068 · Dropout 0.3: 0.6231 · ambas 0.6026 | Dropout 0.3 (+0.025) |
| Arquitectura | [32] 0.6114 · [64, 32] 0.6231 · [128, 64] 0.6085 | [64, 32] |

**Arquitectura final:** `Input(65) → Dense(64, relu) → Dropout(0.3) → Dense(32, relu) → Dropout(0.3) → Dense(4, softmax)`, Adam (lr = 0.001), pérdida `sparse_categorical_crossentropy`, lotes de 32 y hasta 150 épocas. **Callback:** EarlyStopping sobre la pérdida de validación, con paciencia 10 y restaurando los mejores pesos. Con lr = 0.01, la pérdida de validación es mínima en la primera época y luego sube, y el F1 cae a 0.562. Con Dropout, la mejor época pasa de la 7 a la 11 y la brecha entre la pérdida de validación y la de entrenamiento baja de 0.276 a 0.174.

![Pérdida de entrenamiento y validación por época](figuras/red_curvas.png){width=90%}

EarlyStopping detiene el entrenamiento entre las épocas 19 y 37, con la mejor época entre la 9 y la 27. En los casos 1 y 3 (sin escalar y desbalanceados) la red no aprende Nivel 2 ni Nivel 4 (F1 de prueba de 0.31 en ambos), ni siquiera con paciencia 30. En los casos 2 y 4, sin escalar pero balanceados, sí aprende las cuatro clases, aunque el accuracy de validación oscila más entre épocas (desviación de 0.014–0.021, frente a 0.008–0.010 en los casos 6 y 8).

### 4.5 Evaluación de los 3 datos de prueba

Los registros de Nivel 1 y Nivel 3 se clasifican bien en casi todos los casos y técnicas; la excepción es el caso 6, donde KNN, la SVM y la red pasan el de Nivel 1 a Nivel 2 o Nivel 3. El de Nivel 4 se predice como Nivel 3 en las cuatro técnicas y los 8 casos. En la SVM, el balanceo sube su probabilidad de Nivel 4 (de 0.054–0.076 en los casos desbalanceados a 0.228–0.290 en los casos 2, 4 y 8), pero no lo suficiente para cambiar la predicción; la red del caso 8 es la que más se acerca (0.369).

Con los **3 estudiantes hipotéticos** (los mismos de la sección 9.2), ninguna técnica invierte el orden de los perfiles en ningún caso. El perfil bajo sale como Nivel 1 en las 32 predicciones y el alto como Nivel 4 en 30 (las otras 2 son de la red en los casos 1 y 3, que no predice Nivel 4). El perfil medio, justo en el corte entre Nivel 2 y Nivel 3, sale como Nivel 3 en 28 y como Nivel 2 en 4, todas de KNN en casos balanceados: es el efecto del balanceo visto en un solo estudiante.

## 5. Evaluación de los modelos supervisados *(notebook: sección 7)*

### 5.1 Figura 1 (prueba, promedio macro)

| # | Norm. | Out. | Bal. | Árbol Acc | Árbol Prec | Árbol Rec | Árbol F1 | KNN Acc | KNN Prec | KNN Rec | KNN F1 | SVM Acc | SVM Prec | SVM Rec | SVM F1 | Red Acc | Red Prec | Red Rec | Red F1 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | ED NO | NO | NO | 0.605 | 0.606 | 0.542 | 0.537 | 0.595 | 0.654 | 0.573 | 0.527 | 0.590 | 0.584 | 0.579 | 0.519 | 0.537 | 0.241 | 0.443 | 0.309 |
| 2 | ED NO | NO | SI | 0.616 | 0.553 | 0.629 | 0.577 | 0.577 | 0.503 | 0.563 | 0.524 | 0.639 | 0.570 | 0.698 | **0.601** | 0.617 | 0.546 | 0.663 | 0.573 |
| 3 | ED NO | SI | NO | 0.617 | 0.618 | 0.525 | 0.531 | 0.572 | 0.646 | 0.561 | 0.486 | 0.589 | 0.601 | 0.564 | 0.516 | 0.528 | 0.265 | 0.384 | 0.307 |
| 4 | ED NO | SI | SI | 0.632 | 0.563 | 0.667 | 0.592 | 0.579 | 0.504 | 0.554 | 0.523 | 0.643 | 0.565 | 0.663 | 0.593 | 0.643 | 0.567 | 0.668 | 0.595 |
| 5 | ED SI | NO | NO | 0.605 | 0.606 | 0.542 | 0.537 | 0.563 | 0.625 | 0.475 | 0.438 | 0.588 | 0.582 | 0.572 | 0.515 | 0.562 | 0.565 | 0.556 | 0.467 |
| 6 | ED SI | NO | SI | 0.616 | 0.553 | 0.629 | 0.577 | 0.502 | 0.430 | 0.499 | 0.444 | 0.643 | 0.558 | 0.624 | 0.580 | 0.636 | 0.556 | 0.637 | 0.579 |
| 7 | ED SI | SI | NO | 0.617 | 0.618 | 0.525 | 0.531 | 0.550 | 0.628 | 0.466 | 0.429 | 0.587 | 0.601 | 0.562 | 0.513 | 0.607 | 0.613 | 0.582 | 0.540 |
| 8 | ED SI | SI | SI | 0.632 | 0.563 | 0.667 | 0.592 | 0.491 | 0.425 | 0.495 | 0.442 | 0.635 | 0.553 | 0.640 | 0.579 | 0.641 | 0.561 | 0.653 | 0.587 |

En todos los casos la normalización incluye CC(SI). Valores con 4 decimales en el notebook.

### 5.2 Gráficas de F1

![F1 macro máximo por algoritmo](figuras/f1_maximo.png){width=60%}

![F1 macro medio por normalización (escalado) y algoritmo](figuras/f1_normalizacion.png){width=70%}

![F1 macro medio por manejo de outliers y algoritmo](figuras/f1_outliers.png){width=70%}

![F1 macro medio por balanceo y algoritmo](figuras/f1_balanceo.png){width=70%}

### 5.3 Análisis

- **El balanceo es el factor dominante.** Sin balanceo, el recall medio de Nivel 2 queda entre 0.05 (red) y 0.25 (árbol), y la red de los casos 1 y 3 no asigna ningún registro a Nivel 2 ni a Nivel 4. Con SMOTENC, el recall medio de Nivel 2 sube a 0.50–0.58 según la técnica, y el F1 medio mejora en las cuatro técnicas: árbol de 0.534 a 0.584, KNN de 0.470 a 0.483, SVM de 0.516 a 0.588 y red de 0.406 a 0.584. Parte del efecto puede venir del tamaño del entrenamiento: sin balanceo se entrena con 2050–2170 registros reales, y con balanceo con todos los reales más los sintéticos (8208–8296). El patrón de errores sí apunta a la distribución de clases, porque sin balanceo casi no se predice Nivel 2.
- **Por qué el accuracy engaña.** En los casos desbalanceados, KNN tiene un accuracy de 0.55–0.60 pero detecta en promedio solo el 6 % del Nivel 2 (recall de 0.065). El accuracy se sostiene porque acierta el 96 % del Nivel 3, que es casi la mitad de la prueba. La red de los casos 1 y 3 nunca predice Nivel 2 ni Nivel 4, y aun así su accuracy (0.537 y 0.528) supera al de la regla trivial "siempre Nivel 3" (0.466).
- **Sensibilidad a la escala.** Árbol: no cambia (0.559 en ambos). KNN: empeora (0.515 a 0.439), porque sin escalar la distancia la deciden los puntajes, y al escalar las 55 dummies y las ordinales ganan peso aunque casi no aportan. SVM: baja un poco (0.557 a 0.547), sobre todo en los pares balanceados (−0.021 y −0.014). Red: mejora (0.446 a 0.543), sobre todo en los casos desbalanceados, donde pasa de 0.31 a 0.47–0.54; en los balanceados casi no cambia. Sin escalar y sin balanceo, su entrenamiento no logra aprender las clases minoritarias.
- **Outliers.** El árbol, KNN y la SVM cambian su F1 medio en menos de 0.015 (árbol +0.004, KNN −0.013, SVM −0.003). La red sube 0.025, pero casi todo viene del par 5 → 7 (+0.073); en los otros tres pares cambia entre −0.002 y +0.022, así que lo atribuimos a la inestabilidad de la red sin balanceo más que a los outliers. El 5 % de atípicos queda amortiguado por los cortes del árbol, el margen suave de la SVM y el Dropout de la red.
- **Comparación.** La SVM del caso 2 tiene el mejor F1 (0.601), seguida por la red del caso 4 (0.595) y el árbol de los casos 4 y 8 (0.592): en los casos balanceados las tres técnicas quedan a menos de 0.025 entre sí. El árbol es el más estable: primero o segundo en 6 de los 8 casos, con F1 entre 0.531 y 0.592, y es el mejor en los casos desbalanceados 1, 3 y 5. La red es la más dependiente del preprocesamiento (0.307–0.595 entre casos). KNN nunca es el mejor.

### 5.4 Costo de los errores y métrica priorizada

![Matrices de confusión del caso 8: árbol, KNN, SVM y red](figuras/matrices_caso8.png)

**Reporte por clase del caso 8** (prueba; cada celda es precisión / recall / F1):

| Clase (registros) | Árbol | KNN | SVM | Red neuronal |
|---|---|---|---|---|
| Nivel 1 (114) | 0.431 / 0.711 / 0.536 | 0.306 / 0.465 / 0.369 | 0.422 / 0.640 / 0.509 | 0.449 / 0.702 / 0.548 |
| Nivel 2 (349) | 0.645 / 0.484 / 0.553 | 0.470 / 0.441 / 0.455 | 0.642 / 0.550 / 0.593 | 0.663 / 0.524 / 0.586 |
| Nivel 3 (444) | 0.755 / 0.714 / 0.734 | 0.659 / 0.532 / 0.589 | 0.776 / 0.696 / 0.734 | 0.764 / 0.714 / 0.738 |
| Nivel 4 (46) | 0.422 / 0.761 / 0.543 | 0.266 / 0.543 / 0.357 | 0.373 / 0.674 / 0.481 | 0.369 / 0.674 / 0.477 |
| **Macro** | 0.563 / 0.667 / 0.592 | 0.425 / 0.495 / 0.442 | 0.553 / 0.640 / 0.579 | 0.561 / 0.653 / 0.587 |

Nivel 3 es la clase mejor detectada por los cuatro modelos (F1 de 0.589–0.738). Las clases minoritarias son las más difíciles: Nivel 4 (F1 de 0.357–0.543) y Nivel 1 (0.369–0.548). En Nivel 4 y en Nivel 1, los cuatro modelos tienen más recall que precisión (Nivel 4: 0.543–0.761 frente a 0.266–0.422; Nivel 1: 0.465–0.711 frente a 0.306–0.449): con el balanceo predicen las clases extremas con más frecuencia, a costa de falsas alarmas entre estudiantes de los niveles vecinos. En Nivel 2 pasa lo contrario: los cuatro modelos son más precisos que exhaustivos.

En un uso de alerta temprana (con los puntajes de un simulacro previo, sección 1.1), el error más costoso es un **falso negativo en Nivel 1**: un estudiante en riesgo que no recibe apoyo. Un falso positivo (dar refuerzo a un estudiante de Nivel 2) cuesta mucho menos. Por eso priorizamos el **recall de Nivel 1**, sin sacrificar el F1 macro: el recall solo no basta, porque la red del caso 1 llega a un recall de Nivel 1 de 0.833, pero con un F1 macro de solo 0.31: marca como Nivel 1 a muchos estudiantes que no lo son. En el caso 8, el árbol detecta más estudiantes de Nivel 1 (0.711, 33 no detectados), seguido de cerca por la red (0.702) y luego por la SVM (0.640) y KNN (0.465). La red tiene menos falsas alarmas (98, frente a 100 de la SVM y 107 del árbol), y en los tres casi todas son estudiantes de Nivel 2. Solo entre 15 y 31 errores por modelo saltan más de un nivel: casi todos son entre niveles vecinos. Para la alerta temprana preferimos el árbol de los casos 4 y 8 (recall de Nivel 1 de 0.711 con F1 macro de 0.592), que además se explica con reglas simples; la red del caso 8 es una alternativa casi equivalente (0.702 y 0.587) con algo menos de falsas alarmas. Como esta comparación usa la prueba, la presentamos como análisis y no como selección de hiperparámetros.

### 5.5 Curvas ROC y Precision-Recall del caso 8

![Curvas ROC y Precision-Recall del caso 8, micro-promediadas (One-vs-Rest)](figuras/roc_pr_caso8.png){width=85%}

Con 4 clases usamos One-vs-Rest. Las gráficas son micro-promediadas (las 4 columnas se tratan como un solo problema binario, con las funciones del notebook 6-02), para tener una curva por algoritmo. Como el micro-promedio lo dominan Nivel 3 y Nivel 2, también reportamos el promedio macro y el área de Nivel 4.

| Modelo | AUC micro | AP micro | AUC macro | AP macro | AUC Nivel 4 | AP Nivel 4 |
|---|---|---|---|---|---|---|
| Red neuronal | 0.887 | 0.674 | 0.875 | 0.636 | 0.946 | 0.558 |
| SVM | 0.884 | 0.671 | 0.866 | 0.632 | 0.936 | 0.577 |
| Árbol | 0.870 | 0.647 | 0.851 | 0.582 | 0.909 | 0.429 |
| KNN | 0.661 | 0.368 | 0.654 | 0.338 | 0.734 | 0.167 |

La red y la SVM quedan prácticamente empatadas y por encima del árbol, aunque por poco (AUC micro de 0.887 y 0.884 frente a 0.870); KNN queda muy por debajo. KNN usa k = 1 en el caso 8, así que solo da probabilidades de 0 o 1 y su curva ROC es un único punto. El árbol tiene el mejor F1, pero solo 101 valores de probabilidad distintos, y por eso ordena a los estudiantes peor que la red y la SVM. ROC y PR no ordenan igual las clases: Nivel 4 tiene el mayor AUC (0.936–0.946 en la SVM y la red), pero su AP es mucho menor (0.558–0.577). La SVM predice Nivel 4 en 83 estudiantes y acierta 31: sus 52 falsas alarmas apenas mueven la tasa de falsos positivos (hay 907 negativos), pero superan a sus aciertos. Por eso le damos más peso a la curva PR. Nivel 2 es la clase más difícil de separar (AUC de 0.80–0.81 en la SVM y la red). En el caso 8, el árbol tiene el mejor F1 y el mayor recall de Nivel 1, y la red y la SVM las mejores curvas; las diferencias son pequeñas (sección 5.4).

# Módulo 2 – Aprendizaje no supervisado

## 6. Datos de la Figura 2 *(notebook: sección 8.0)*

Las versiones de la Figura 2 son los registros de entrenamiento de cada caso de la Figura 1, sin la etiqueta, con el mismo CC, escalado y outliers. Como no se puede balancear sobre una etiqueta eliminada, la correspondencia es por registros: las versiones desbalanceadas son los registros del submuestreo 70/10/10/10, y las balanceadas incluyen los sintéticos de SMOTENC (46–48 % del total). La etiqueta se conserva aparte, solo para la validación externa. Sin escalar, las 5 numéricas aportan el 97.1 % de la varianza (caso 1); con escalado, el 36.5 % (caso 5).

## 7. K-Means *(notebook: sección 8.1)*

Usamos `init="k-means++"`, `n_init=10` y `random_state=79`, con K de 2 a 10. El codo se lee con un criterio fijo: el punto más alejado de la recta entre los extremos de la curva.

![Método del codo (arriba) y silhouette frente a K (abajo)](figuras/kmeans_codo_silhouette.png){width=80%}

- **K óptimo = 2 en los 8 casos**, por silhouette máximo (0.41–0.56 sin escalar y 0.17–0.25 con escalado). El codo marca K = 4 sin escalar y K = 5 con escalado, pero ahí el silhouette baja a 0.26–0.34 y 0.09–0.14. No elegimos K = 4 por ser el número de niveles: eso sería usar la etiqueta.
- **Tamaño de los clusters:** 48–52 % en los casos desbalanceados y 59 % / 41 % en los balanceados (tabla siguiente). Los renumeramos para que C0 sea el de menor puntaje.

| Caso | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| C0 (desempeño bajo) | 989 | 4816 | 1053 | 4898 | 990 | 4832 | 1111 | 4930 |
| C1 (desempeño alto) | 1061 | 3392 | 1117 | 3398 | 1060 | 3376 | 1059 | 3366 |

- **Escalado:** baja el silhouette entre 0.24 y 0.31, pero no cambia los grupos (89.5–94.7 % de registros quedan en el mismo cluster). Sin escalar el agrupamiento ocurre casi en una dimensión, la del puntaje; con escalado entran las dummies, que no tienen estructura de grupos. Un silhouette más alto sin escalar no significa mejores grupos.
- **Outliers:** el silhouette cambia entre −0.019 y +0.004. Los outliers se reparten en los dos clusters (4.0–6.0 % de cada uno) y no forman un grupo propio.

## 8. DBSCAN *(notebook: sección 8.2)*

MinPts = 2 × 65 características = **130** (regla del enunciado y de la diapositiva 6-03), y ε en el codo del gráfico de k-distancia con k = MinPts (ε entre 3.66 y 16.55).

![Gráfico de k-distancia y ε del codo](figuras/kdistancia.png){width=90%}

- **Resultado: un solo cluster en los 8 casos**, con 0.1–3.3 % de ruido, así que el silhouette no está definido. La segunda regla de la diapositiva (MinPts = 66) da lo mismo. La k-distancia sube de forma continua, sin escalones: no hay zonas de baja densidad que separen grupos.
- **Sensibilidad a ε:** al bajar ε desde el codo, el ruido crece antes de que aparezca un segundo cluster. En el caso 5 pasa del 15 % (80 % del codo) al 90 % (70 % del codo) sin separar nada. Con ε grande todo se fusiona y con ε pequeño casi todo es ruido.
- **Prueba complementaria.** Para que la gráfica de la Figura 3 tuviera valores, buscamos el ε más cercano al codo que separa al menos 2 clusters, bajando MinPts a 66 y a 10 cuando hizo falta (la diapositiva lo permite con mucho ruido). Aparecen grupos en todos los casos, pero con entre 7 % y 84 % de ruido, y en los casos 3 y 5 el segundo cluster tiene solo 6–7 registros.
- **Densidades variables.** En los casos balanceados, los clusters más densos son en un 76–82 % registros sintéticos de SMOTENC, frente al 46–48 % del caso completo. SMOTENC crea zonas artificialmente densas, y un solo ε no sirve a la vez para esas zonas y para los datos reales.

## 9. Evaluación y comparación *(notebook: secciones 8.3 a 8.5)*

### 9.1 Figura 3

| # | Normalización | Outliers | Balanceo | K-Means inercia | K-Means silhouette | DBSCAN silhouette (regla) | DBSCAN silhouette (complementario) |
|---|---|---|---|---|---|---|---|
| 1 | CC(SI) y ED(NO) | NO | NO | 519 267.7 | 0.4308 | No definido (1 cluster) | 0.3848 |
| 2 | CC(SI) y ED(NO) | NO | SI | 2 041 823.3 | 0.5559 | No definido (1 cluster) | 0.4333 |
| 3 | CC(SI) y ED(NO) | SI | NO | 701 971.7 | 0.4118 | No definido (1 cluster) | 0.2197 |
| 4 | CC(SI) y ED(NO) | SI | SI | 2 421 023.2 | 0.5473 | No definido (1 cluster) | 0.4768 |
| 5 | CC(SI) y ED(SI) | NO | NO | 22 278.7 | 0.1760 | No definido (1 cluster) | 0.2122 |
| 6 | CC(SI) y ED(SI) | NO | SI | 79 717.7 | 0.2427 | No definido (1 cluster) | 0.3814 |
| 7 | CC(SI) y ED(SI) | SI | NO | 24 075.9 | 0.1694 | No definido (1 cluster) | 0.0249 |
| 8 | CC(SI) y ED(SI) | SI | SI | 81 264.1 | 0.2462 | No definido (1 cluster) | 0.4403 |

La inercia no es comparable entre casos (depende del número de registros y de la escala).

![Silhouette de K-Means y de DBSCAN (prueba complementaria) en los 8 casos](figuras/silhouette_figura3.png){width=85%}

En K-Means, el escalado es el factor que más mueve el silhouette (0.486 sin escalar frente a 0.209 con escalado). El balanceo lo sube de 0.297 a 0.398, porque con 25 % de cada nivel hay más estudiantes en los extremos y la nube queda más alargada. Los outliers casi no influyen (0.351 frente a 0.344). Los valores de la prueba complementaria de DBSCAN no son comparables entre casos, porque cada uno necesitó una configuración y un ruido distintos.

### 9.2 Los 3 datos de prueba

Cubrimos las dos lecturas posibles del enunciado:

- **Registros reales** (los mismos 3 del módulo supervisado): K-Means asigna el de Nivel 1 a C0 y los de Nivel 3 y Nivel 4 a C1 en los 8 casos. DBSCAN con la regla los asigna a su único cluster, excepto al de Nivel 1 en los casos 2, 5 y 6, que queda como ruido por ser un punto extremo. Para asignar en DBSCAN, que no tiene `predict`, usamos el *core sample* más cercano a menos de ε.
- **Estudiantes hipotéticos**, construidos a mano como el "nuevo vino" del notebook 6-03 (los mismos que usa el módulo supervisado, sección 4.5):

| Perfil | Puntajes | Contexto | K-Means (8 casos) |
|---|---|---|---|
| Bajo | 33 a 38 | 19 años, estrato 1, padres con primaria, colegio rural oficial, sin computador ni internet | C0 |
| Medio | 50 a 55 | 17 años, estrato 2, padres bachilleres, colegio urbano oficial | C0 |
| Alto | 72 a 85 | 16 años, estrato 5, madre profesional y padre con posgrado, colegio bilingüe no oficial | C1 |

El perfil medio queda en C0 porque la frontera entre clusters está cerca de los 55 puntos.

### 9.3 Comparación entre K-Means y DBSCAN

| Aspecto | K-Means | DBSCAN |
|---|---|---|
| Forma de los clusters | Esféricos; aquí corta la nube en dos mitades a lo largo de PC1 | Arbitraria; con ε del codo, toda la nube es un cluster |
| Número de clusters | Hay que definir K y siempre entrega K grupos, aunque no haya separación natural | No requiere K; encontrar uno solo indica que no hay estructura de densidad |
| Ruido y outliers | Asigna todos los registros; con K = 2 su efecto es mínimo | Los separa como ruido: 0.1–3.3 % con la regla, hasta 84 % al bajar ε |
| Densidades variables | No depende de la densidad | Es su punto débil: en los casos balanceados solo ve los núcleos de sintéticos |
| Sensibilidad | Escalado e inicialización (controlada con k-means++ y n_init = 10) | ε y MinPts: en el caso 7, del 70 % al 60 % del codo el ruido pasa del 16 % al 89 % |

## 10. Visualización e interpretación de los clusters *(notebook: secciones 8.6 y 8.7)*

![K-Means con centroides (arriba) y DBSCAN complementario con el ruido en gris (abajo), PCA a 2D](figuras/pca_clusters.png){width=80%}

Sin escalar, PC1 explica el 73–87 % de la varianza (el puntaje); con escalado, el 28–37 %. En ninguna proyección hay un espacio vacío entre los clusters de K-Means: es una nube continua cortada por la mitad.

| Cluster | Lectura crítica (media) | Estrato mediano | En estrato 3 o más | Educación de la madre (mediana) | Edad (media) | % Nivel 1 | % Nivel 4 |
|---|---|---|---|---|---|---|---|
| C0 – desempeño bajo | 45–49 | 2 | 15–25 % | Bachillerato completo (incompleto en los casos 6 y 8) | 17.1–18.0 | 19–42 % | 0.0–0.4 % |
| C1 – desempeño alto | 64–67 | 2 | 40–49 % | Técnica o tecnológica completa (6 de 8 casos) | 16.5–16.9 | 0.1–0.4 % | 19–61 % |

El estrato y la educación de la madre son ordinales, así que los resumimos con la mediana y con porcentajes, no con el promedio de sus códigos. El estudiante mediano es de estrato 2 en ambos clusters: la diferencia socioeconómica está en la distribución, porque C1 tiene entre 1.7 y 3.1 veces más estudiantes en estrato 3 o superior.

![Validación externa de K-Means: composición de niveles reales en cada cluster](figuras/validacion_externa.png){width=90%}

**Validación externa.** Los clusters separan bien los extremos (C0 casi no tiene Nivel 4 y C1 casi no tiene Nivel 1), pero no los niveles intermedios. En los casos desbalanceados, Nivel 3 es el 60–62 % de C0 y el 78–79 % de C1. En los balanceados, C0 mezcla Nivel 1 y 2 y C1 mezcla Nivel 3 y 4.

**Interpretación en el dominio.** Los puntajes de las cinco áreas se mueven juntos y acompañan al estrato y a la educación de los padres. Sin la etiqueta, lo que aparece es un eje de "desempeño general", no cuatro niveles. Es el mismo patrón de los modelos supervisados, que aciertan los extremos y confunden niveles vecinos. Los clusters de la prueba complementaria de DBSCAN son muy puros en los casos balanceados (caso 8: 87 % de Nivel 4 en C0), pero como son sobre todo registros sintéticos y dejan el 81 % como ruido, no los interpretamos como grupos reales.

# Conclusiones

1. **El balanceo fue el preprocesamiento más influyente.** Mejoró el F1 macro de las cuatro técnicas (de +0.013 en KNN a +0.178 en la red) y fue la diferencia entre que la red aprendiera las cuatro clases o casi solo la mayoritaria. Parte del efecto puede venir del mayor tamaño del entrenamiento balanceado. Con clases desbalanceadas, el accuracy no sirve para comparar modelos.
2. **El escalado depende del modelo.** No cambia los árboles y ayuda a la red, sobre todo sin balanceo. En KNN empeora el resultado, porque da peso en la distancia a variables que casi no aportan; en la SVM baja un poco el F1 de los casos balanceados.
3. **Los outliers, al 5 %, no tuvieron un efecto apreciable.** En el árbol, KNN, la SVM y K-Means el cambio es menor de 0.02; en la red, el +0.025 medio viene casi todo de un solo par de casos desbalanceados.
4. **No hay un ganador claro en los casos balanceados.** La SVM del caso 2 obtuvo el mejor F1 (0.601), pero el árbol, la SVM y la red quedan a menos de 0.025 entre sí. El árbol fue el más estable (primero o segundo en 6 de los 8 casos), el que más estudiantes de Nivel 1 detecta en el caso 8 (0.711) y el único explicable con reglas simples ("ciencias ≤ 40 y lectura ≤ 46"); por eso es el preferido para la alerta temprana. La red es la más sensible al preprocesamiento, y la red y la SVM son las que mejor ordenan las probabilidades (curvas ROC y PR). KNN fue el más débil.
5. **El aprendizaje no supervisado explica el techo de los clasificadores** (F1 máximo de 0.60). K-Means solo encuentra dos perfiles, bajo y alto, asociados al estrato y a la educación de los padres, y DBSCAN no encuentra separaciones de densidad. El desempeño es un continuo, y los niveles del ICFES son cortes sobre él: por eso los errores son entre niveles vecinos y Nivel 2 es el más difícil.
6. **SMOTENC cambia la geometría de los datos.** Crea zonas densas artificiales que DBSCAN detecta como clusters y que inflan la validación cruzada de KNN. Hay que tenerlo en cuenta al leer métricas calculadas sobre datos balanceados.

# Referencias

- ICFES. *Resultados únicos Saber 11*, aplicaciones 2022-1 y 2022-2. Datos abiertos.
- Guzmán Luna, J. A. Diapositivas y notebooks del curso Introducción a la Inteligencia Artificial (5-01 a 7-06). Universidad Nacional de Colombia, 2026.
- Documentación de scikit-learn, imbalanced-learn (SMOTENC) y TensorFlow/Keras.

# Anexo A. Descripción de las variables

| # | Variable | Descripción | Tipo | ¿Se usa? |
|---|---|---|---|---|
| 1 | `PUNT_LECTURA_CRITICA` | Puntaje en lectura crítica (0–100) | Numérica | Sí |
| 2 | `PUNT_C_NATURALES` | Puntaje en ciencias naturales (0–100) | Numérica | Sí |
| 3 | `PUNT_SOCIALES_CIUDADANAS` | Puntaje en sociales y ciudadanas (0–100) | Numérica | Sí |
| 4 | `PUNT_INGLES` | Puntaje en inglés (0–100) | Numérica | Sí |
| 5 | `ESTU_FECHANACIMIENTO` | Fecha de nacimiento | Fecha | Sí: se transforma en `EDAD` |
| 6 | `FAMI_ESTRATOVIVIENDA` | Estrato de la vivienda (Sin estrato, 1–6) | Ordinal | Sí |
| 7 | `FAMI_EDUCACIONMADRE` | Máximo nivel educativo de la madre | Ordinal | Sí |
| 8 | `FAMI_EDUCACIONPADRE` | Máximo nivel educativo del padre | Ordinal | Sí |
| 9 | `FAMI_CUARTOSHOGAR` | Número de cuartos del hogar | Ordinal | Sí |
| 10 | `FAMI_PERSONASHOGAR` | Número de personas del hogar | Ordinal | Sí |
| 11 | `COLE_AREA_UBICACION` | Colegio urbano o rural | Nominal | Sí |
| 12 | `COLE_BILINGUE` | Colegio bilingüe (S/N) | Nominal | Sí |
| 13 | `COLE_CARACTER` | Académico, técnico o técnico/académico | Nominal | Sí |
| 14 | `COLE_DEPTO_UBICACION` | Departamento del colegio (33) | Nominal | Sí |
| 15 | `COLE_GENERO` | Colegio mixto, femenino o masculino | Nominal | Sí |
| 16 | `COLE_JORNADA` | Mañana, tarde, única, completa, noche o sabatina | Nominal | Sí |
| 17 | `COLE_NATURALEZA` | Oficial o no oficial | Nominal | Sí |
| 18 | `COLE_SEDE_PRINCIPAL` | Si es la sede principal (S/N) | Nominal | Sí |
| 19 | `ESTU_GENERO` | Género del estudiante (F/M) | Nominal | Sí |
| 20 | `FAMI_TIENEAUTOMOVIL` | El hogar tiene automóvil (Sí/No) | Nominal | Sí |
| 21 | `FAMI_TIENECOMPUTADOR` | El hogar tiene computador (Sí/No) | Nominal | Sí |
| 22 | `FAMI_TIENEINTERNET` | El hogar tiene internet (Sí/No) | Nominal | Sí |
| 23 | `FAMI_TIENELAVADORA` | El hogar tiene lavadora (Sí/No) | Nominal | Sí |
| 24 | `COLE_MCPIO_UBICACION` | Municipio del colegio | Nominal | No: ~770 categorías; se usa el departamento |
| 25 | `ESTU_MCPIO_RESIDE` | Municipio de residencia | Nominal | No: ~770 categorías |
| 26 | `ESTU_MCPIO_PRESENTACION` | Municipio donde presentó la prueba | Nominal | No: ~450 categorías |
| 27 | `ESTU_DEPTO_RESIDE` | Departamento de residencia | Nominal | No: Igual al del colegio en ~99 % |
| 28 | `ESTU_DEPTO_PRESENTACION` | Departamento donde presentó la prueba | Nominal | No: Igual al del colegio en ~98 % |
| 29 | `ESTU_PAIS_RESIDE` | País de residencia | Nominal | No: Duplica la nacionalidad |
| 30 | `ESTU_NACIONALIDAD` | Nacionalidad | Nominal | No: 98.7 % colombianos (casi constante) |
| 31 | `COLE_CALENDARIO` | Calendario A, B u otro | Nominal | No: 98.3 % calendario A (casi constante) |
| 32 | `ESTU_PRIVADO_LIBERTAD` | Privado de la libertad (S/N) | Nominal | No: 99.9 % "N" (casi constante) |
| 33 | `ESTU_ESTADOINVESTIGACION` | Estado del resultado | Nominal | No: Constante |
| 34 | `ESTU_ESTUDIANTE` | Tipo de evaluado | Nominal | No: Constante |
| 35 | `PERIODO` | Periodo de aplicación (2022-1 o 2022-2) | Administrativa | No: No describe al estudiante |
| 36 | `DESEMP_INGLES` | Nivel de inglés (A−, A1, …, B+) | Ordinal | No: Es `PUNT_INGLES` en rangos (redundante) |
| — | **`DESEMP_MATEMATICAS`** | **Nivel de desempeño en matemáticas (1–4)** | **Objetivo** | Objetivo (4 clases) |
