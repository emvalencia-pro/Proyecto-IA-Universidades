# Práctica 1 — Sistema Híbrido de Inteligencia Artificial

> Enunciado de la práctica (2026-2S). Integración de sistemas expertos, lógica difusa y ontologías.

---

## Objetivo

Diseñar e implementar un sistema híbrido de inteligencia artificial que integre tres paradigmas clásicos: **sistemas expertos basados en reglas**, **lógica difusa** y **representación del conocimiento mediante ontologías**, para resolver un problema complejo del mundo real, permitiendo la toma de decisiones contextualizadas y enriquecidas semánticamente.

---

## Enunciado del problema

### 1. Descripción general

El grupo de trabajo deberá seleccionar un dominio de aplicación realista y proponer una solución inteligente basada en la combinación de:

- **Sistema experto** para tomar decisiones lógicas basadas en reglas precisas (utilizando **Experta**).
- **Sistema de lógica difusa** para modelar incertidumbre y valores intermedios del entorno (usando **Scikit-Fuzzy**).
- **Ontología y su razonamiento semántico** para representar e inferir entidades y relaciones del dominio (en RDF/RDFS, usando **RDFLib** y **OWL-RL**).

---

## 2. Componentes del sistema híbrido

### 2.1. Sistema Experto (Experta)

**Clases de hechos**
- Definir al menos unas **5 clases de hechos**.

**Reglas basadas en el conocimiento del dominio**
- Definir al menos **15 reglas** que hagan uso de las clases de hechos definidas y sus instancias.
- Utilizar prioridades y técnicas de control de ejecución:
  - Utilizar al menos **3 niveles de prioridad**.
  - Implementar los **tres mecanismos diferentes** para evitar conflictos entre reglas (**Recencia, Specificity, Salience**) y explicarlos detalladamente en el documento.
  - Implementar al menos un mecanismo para evitar bucles infinitos (**No-Loop** o **Lock-Fact**).

**Base de hechos**
- Definir un mínimo de **40 hechos**.

### 2.2. Sistema de lógica difusa (Scikit-Fuzzy)

- Definir al menos **3 variables difusas** del dominio.
  - Para cada variable: mínimo **3 valores difusos** con sus funciones de pertenencia.
  - Para cada variable: definir su **universo del discurso**.
  - Las funciones de pertenencia deben incluir obligatoriamente, en el total de todos los casos: **triangulares, trapezoidales y Gaussianas**.
  - Hacer uso de **dos casos de modificadores** (ejemplo "ligeramente", "muy", etc.), aplicando la respectiva fórmula detallada en clase.
- Definir al menos **9 reglas difusas** que incluyan casos con los operadores:
  - **Complemento** (negación)
  - **Intersección** (AND)
  - **Unión** (OR)
- Implementar un proceso de **defuzzificación justificado** (centroide, bisector, promedio, etc.).

### 2.3. Ontología y razonamiento semántico (RDFLib y OWL-RL)

#### Implementación de la ontología (RDF y RDFS)

- La serialización de la ontología final será en **Turtle**.
- Implementar al menos **10 clases** (`rdfs:Class`) y **5 relaciones jerárquicas** (`rdfs:subClassOf`).
- Crear al menos **10 propiedades RDF originales** con restricciones: dominios (`rdfs:domain`) y rangos (`rdfs:range`) correctamente definidos.
  - Incluir al menos un caso de jerarquía de propiedad (`rdfs:subPropertyOf`).
- Incluir como rangos valores tanto de **tipo clase (URIs)** como **Literales** con sus respectivos tipos usando **XML-Schema**.
- Incluir el uso de clases o propiedades provenientes de **vocabularios definidos y aceptados en la literatura** (FOAF, Dublin Core, etc.).
- Instanciar al menos **4 individuos por clase**.
- Cada una de las clases y propiedades deberá escribir su significado usando la propiedad `rdfs:label`.

#### Razonamiento sobre la ontología RDFS

Aplicar razonamiento usando `DeductiveClosure(RDFS_Semantics)` de `owlrl`. Completar de ser necesario la ontología desarrollada para lograr lo siguiente:

1. Documentar al menos **dos casos de generación de nuevos hechos asociados a una jerarquía de clases** (caso 1 de las diapositivas).
2. Documentar la **asociación de un caso de miembro de una clase desde el dominio o rango de sus propiedades** (caso 2 de las diapositivas).
3. Documentar un caso de **nuevos hechos desde la relación `subPropertyOf`** (caso 3 de las diapositivas).

Los tres requerimientos anteriores deben ser generados al usar el razonador OWL-RL con `DeductiveClosure(RDFS_Semantics)` sobre la ontología final del dominio.

- **Comparar el grafo antes y después del razonamiento** para evidenciar nuevas afirmaciones inferidas automáticamente.
- Adicionalmente, se debe evidenciar al menos **4 nuevos hechos inferidos** generados como resultado del razonamiento.

### 2.4. Enfoque propuesto para la integración

#### Ontología → Sistema Experto

El componente de la ontología alimentará al Sistema Experto. Para ello se consultará la ontología RDF y se traducirán las inferencias semánticas obtenidas con el razonador OWL-RL (grafo resultante del razonamiento semántico) como **hechos iniciales** del sistema experto. Se deberá implementar el **traductor** respectivo.

**Traductor Ontología → Sistema Experto**

- El traductor debe **consultar dinámicamente el grafo RDF resultante** (ontología + inferencias del razonador OWL-RL) y generar a partir de él los hechos iniciales del sistema experto.
- **No es válido** declarar manualmente en el código del sistema experto los hechos correspondientes a los datos observados en la ontología, utilizando valores fijos (por ejemplo, `engine.declare(Padre(padre='Juan', hijo='Pedro'))`).
- El código del traductor **no debe contener nombres ni valores propios de un caso específico**.
- Cada grupo es libre de implementar la lectura del grafo con el método que prefiera, siempre que el traductor realmente lea y traduzca la información del grafo, y no se limite a transcribir a mano lo que se observó en la ontología en un momento dado.

**Caso de prueba obligatorio**

1. Agregar un individuo o relación nueva **directamente en el archivo de la ontología**, sin modificar el código del traductor ni del sistema experto.
2. Volver a ejecutar el traductor.
3. Evidenciar que el hecho nuevo se refleja automáticamente como `Fact` en el sistema experto y es procesado correctamente por las reglas correspondientes.

> Si para reflejar el cambio es necesario editar el código Python, el traductor no cumple con lo solicitado.

#### Lógica difusa → Sistema Experto

El componente de lógica difusa igualmente alimentará al Sistema Experto. Los resultados de la lógica difusa se usarán como **condiciones adicionales para activar reglas** en el sistema experto; es decir, se deben tener en cuenta en algunas de las precondiciones de las reglas del Sistema Experto.

#### Inferencia

El Sistema Experto, luego de recibir la información de los anteriores componentes (Ontología y Lógica difusa), procederá a realizar las inferencias respectivas.

#### Nota importante sobre la Ontología y el Sistema Experto

- **Todos los hechos generados en la ontología deben ser traducidos sin excepción** al Sistema Experto como hechos iniciales.
- No se aceptará que se construya una ontología extensa para luego utilizar únicamente una parte reducida en el Sistema Experto.
- Deberán existir **reglas adicionales** que incluyan en sus precondiciones o efectos estos hechos.

La evaluación verificará que:

- El grafo resultante después del razonamiento semántico sea la **fuente completa de hechos** que alimentan el Sistema Experto.
- Cada triple relevante (clases, propiedades, individuos e inferencias) se traduzca a hechos del motor de reglas, **manteniendo la coherencia semántica**.
- El Sistema Experto haga **uso explícito** de dichos hechos en la activación de reglas e inferencias posteriores.

Es decir, la ontología no debe quedar como una base de datos aislada; todo el conocimiento representado y deducido en ella debe reflejarse en el Sistema Experto.

---

## 3. Entregables

### 3.1. Documento de diseño (PDF)

Descripción detallada de cada componente: sistema experto, lógica difusa y ontología. En particular:

- **Gráficas de las funciones de pertenencia** utilizadas en el componente de lógica difusa.
- **Diagrama de clases** y documentación de los **tres casos de generación de nuevo conocimiento** solicitados en el módulo de ontologías.
- **Enlace del vídeo tipo pitch** de sustentación (ver numeral 4).

### 3.2. Código fuente documentado en Python (Google Colab)

- Código fuente **separado por módulos**: experto, difuso y ontología.
- **Comentarios por función** y explicación del flujo general.

---

## 4. Sustentación

- Se realizará de manera **virtual**.
- Se deberá implementar un **vídeo tipo pitch (duración máx. 5 minutos)** con la demostración del sistema funcional, la explicación clara del problema y la solución híbrida implementada.
  - **Todos los integrantes deben participar** en el video.
  - El video se presentará durante la sustentación, para lo cual deberán tener un buen equipo con internet para su presentación virtual.
  - El video deberá ser **subido a YouTube** y, en el documento de diseño (PDF), incluir el enlace (ver numeral 3.1).
  - El video debe estar disponible **desde la fecha de entrega** del trabajo.
- Se realizará una sesión de preguntas (**5 min de pitch y 5 min de preguntas**).
  - Todos los integrantes deben estar presentes en la sustentación y se podrá seleccionar a cualquiera para responder preguntas.

---

## 5. Metodología de evaluación

| Componente | Peso |
|---|---|
| **Código y documentación** | **65 %** |
| — Módulo de Sistemas Expertos | 25 % |
| — Módulo de Lógica Difusa | 15 % |
| — Módulo de Ontología | 25 % |
| **Sustentación** | **35 %** |
| — Video tipo pitch | 20 % |
| — Atención a preguntas | 15 % |

---

## 6. Material a entregar

Se deberá entregar en **Google Classroom** un archivo **ZIP** con el nombre:

```
practica1-grupo-XX-equipo-YY.zip
```

donde `XX` es el número del grupo (1 o 2) y `YY` el número del equipo. Debe contener:

- **Documento de diseño:** descripción detallada de cada módulo (experto, difuso, ontología), junto con la justificación de las decisiones tomadas en cada parte del programa y el enlace al video de sustentación tipo pitch.
- **Código fuente:** todo el programa en Python, debidamente organizado y documentado.

---

## Anexo 1 — Instrucciones generales para la presentación de los trabajos

- Los trabajos serán presentados en **grupos de 4 personas** definidos al inicio del semestre (ver grupos en el material de la clase 2).
- **No se admiten trabajos similares** (incluyendo el planteamiento del dominio). En caso de presentarse programas con códigos similares serán anulados. La totalidad del trabajo deberá ser original y propio del autor que lo presenta. El estudiante es responsable de evitar que su material evaluable (código, solución al problema, memorias, etc.) sea accesible a estudiantes de otros grupos. En caso de que se detecten copias, por acción o inacción, la sanción afectará a todos los estudiantes involucrados: quienes copien y quienes hayan sido copiados.
- Los integrantes de cada grupo deberán ser capaces de **explicar y responder preguntas** sobre el trabajo realizado durante la sustentación.
- La evaluación de la práctica está asociada con la sustentación programada. Quien no asista a la sustentación se considerará que no presentó la práctica en su totalidad (no se tendrá en cuenta el código, ni la documentación, ni el video tipo pitch).
- El **plazo de entrega es estricto** y se indica al inicio del documento original. No se admiten trabajos después de esta fecha. No se considera entrega aquella que solo contiene código, o solo el vídeo, o solo el documento de diseño, o cualquier forma incompleta del trabajo.
- El código entregado en el ZIP en Google Classroom **debe ser el mismo** que se ilustra en el vídeo y se explica en el documento de diseño. En caso de no cumplirse, se considera que el código no fue entregado y, por tanto, no se realizó la práctica por parte del grupo.

**Nota 1:** Si en la fecha de entrega de la práctica no se entrega el video y luego en la sustentación no se presenta el video, el trabajo se considerará no entregado.

**Nota 2:** La sustentación se realizará en la fecha indicada, mediante el vídeo entregado y las preguntas orales a los integrantes de cada equipo después del pitch.

**Nota 3:** La redacción de este documento puede presentar algunos errores o la ausencia de algunos requisitos específicos, los cuales deberán ser informados por los estudiantes al profesor para ser aclarados colectivamente durante la clase. Estas aclaraciones harán parte de los requisitos de la práctica.

---

## Resumen de requisitos mínimos (referencia rápida)

| Módulo | Requisito mínimo |
|---|---|
| Experto | 5 clases de hechos, 15 reglas, 3 niveles de prioridad, Recencia + Specificity + Salience, No-Loop/Lock-Fact, 40 hechos |
| Difuso | 3 variables (≥3 valores c/u), MF triangular + trapezoidal + gaussiana, 2 modificadores, 9 reglas (NOT, AND, OR), defuzzificación justificada |
| Ontología | Turtle, 10 clases, 5 `subClassOf`, 10 propiedades con domain/range, 1 `subPropertyOf`, tipos XSD, vocabulario externo (FOAF/DC), 4 individuos por clase, `rdfs:label` en todo |
| Razonamiento | `DeductiveClosure(RDFS_Semantics)`, 2 casos jerarquía de clases, 1 caso domain/range, 1 caso subPropertyOf, comparación antes/después, ≥4 hechos inferidos |
| Integración | Traductor dinámico Ontología → Experto (sin valores hardcodeados), caso de prueba obligatorio, resultados difusos en precondiciones de reglas |
| Entregables | PDF de diseño (con gráficas de MF, diagrama de clases, link a YouTube), código en Colab por módulos, ZIP `practica1-grupo-XX-equipo-YY.zip` |
