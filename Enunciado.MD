# Contexto del Proyecto — Trabajo Práctico 1: Sistema Híbrido de IA

**Asignatura:** Introducción a la Inteligencia Artificial – 3010476
**Universidad Nacional de Colombia, Sede Medellín**
**Profesor:** Jaime Alberto Guzmán Luna
**Equipo:** 4 integrantes
**Valor:** 10%

---

## 1. Dominio elegido

**Sistema de recomendación de universidades para intercambio académico**, con ofertas nacionales e internacionales, basado en el archivo de convocatorias de la DRE/UNAL (`Ofertas_Específicas_-_Internacionales_-_2027-1.xlsx`).

El dominio es adecuado porque los tres paradigmas conviven de forma natural:

- Hay **restricciones exactas** (fechas de cierre, nivel de estudios, áreas restringidas) → sistema experto.
- Hay **valores inciertos y graduales** (qué tan competitiva es una oferta, qué tan bueno es el promedio, qué tan suficiente es el nivel de idioma) → lógica difusa.
- Hay **entidades y relaciones jerárquicas** (oferta, universidad, país, área de conocimiento, convenio, beca) → ontología.

---

## 2. Fuente de datos: estructura del Excel

El archivo tiene 4 hojas:

1. **Ofertas Internacionales** (~73 filas)
2. **Ofertas Nacionales** (universidades de Colombia)
3. **Instructivo de Búsqueda**
4. Hoja auxiliar de fechas

Las dos hojas de ofertas comparten la misma estructura de columnas.

### Mapeo de columnas a módulos

| Columna del Excel | Se usa en | Cómo |
|---|---|---|
| PAÍS | Ontología | Individuo `Pais`, propiedad `ofreceEnPais` |
| UNIVERSIDAD DESTINO | Ontología | Individuo `Universidad` / `UniversidadDestino` |
| FECHA APERTURA / CIERRE | Experto (regla dura) | `xsd:date`; regla: si hoy > cierre → descartar oferta |
| NIVEL DE ESTUDIOS | Experto (regla dura) | Debe coincidir con el nivel del estudiante (Pregrado) |
| ÁREA / RESTRICCIONES | Experto + Ontología | Descartar si el área del estudiante está restringida o si la oferta es exclusiva de otra área |
| CUPOS DISPONIBLES | Ontología + **Lógica difusa** | `tieneCupos` (`xsd:integer`); entrada `cupos` del difuso (pocos cupos → oferta más competida). Si no hay número, valor neutro 5 |
| MODALIDAD | Experto (hecho) | Presencial / Virtual |
| OBSERVACIONES | Ontología + Experto + **Lógica difusa** | Beca, GPA exigido; el requisito de idioma va en `requiereIdioma` (texto libre, p. ej. `"Francés B1"`) y el difuso lo interpreta |

### Criterio de selección de instancias

No se usan las 73 filas. Se usan **8 ofertas** (4 internacionales + 4 nacionales, ver §6.3). Fechas y cupos pueden ajustarse libremente para que las reglas se disparen; no tienen que coincidir con el Excel.

| Oferta (id) | Tipo | Cupos | Idioma (`requiereIdioma`) | Cierre | Caso que demuestra |
|---|---|---|---|---|---|
| `oferta_montpellier_2027_1` | int | 4 | Francés B1 | 2026-09-29 | idioma distinto al del estudiante |
| `oferta_unam_2027_1` | int | 10 | Español (B2 para no hispanohablantes) | 2026-08-20 | fecha vencida + GPA 8/10 |
| `oferta_freie_berlin_2027_1` | int | 4 | Inglés B2 / Alemán | 2026-09-30 | con beca; varios idiomas aceptados |
| `oferta_stavanger_2027_1` | int | — | — | 2026-09-09 | restringida a Matemáticas (exclusión) |
| `oferta_escuela_colombiana_2027_1` | nac | 5 | — | 2026-10-11 | lista explícitamente Ing. de Sistemas |
| `oferta_eafit_2027_1` | nac | 9 | — | 2026-09-18 | nacional apta |
| `oferta_ufps_2027_1` | nac | 10 | — | 2026-10-25 | nacional apta, muchos cupos |
| `oferta_tolima_2027_1` | nac | — | — | 2026-09-18 | nacional sin cupos definidos |

Esta mezcla demuestra que las reglas discriminan correctamente.

---

## 3. Diseño por módulo

### 3.1 Ontología y razonamiento semántico (RDFLib + OWL-RL) — 25%

**Clases (mínimo 10):**
`Oferta`, `OfertaInternacional`, `OfertaNacional`, `Universidad`, `UniversidadDestino`, `Pais`, `AreaConocimiento`, `NivelEstudios`, `Convenio`, `Beca`, `Estudiante`.

**Jerarquías (mínimo 5 `rdfs:subClassOf`):**
- `OfertaInternacional` ⊑ `Oferta`
- `OfertaNacional` ⊑ `Oferta`
- `UniversidadDestino` ⊑ `Universidad`
- `Estudiante` ⊑ `foaf:Person`
- `Ingenieria` ⊑ `AreaConocimiento` (subclase poblada con individuos de varias ingenierías: `area_ingenieria_sistemas`, `area_ingenieria_civil`, `area_ingenieria_electronica`, `area_ingenieria_industrial`, `area_ingenieria_mecanica`; cada uno se infiere además como `AreaConocimiento` → caso 1 de razonamiento)

**Propiedades (mínimo 10, con `rdfs:domain` y `rdfs:range`):**
`ofrecidaPor`, `ofreceEnPais`, `tieneArea`, `dirigidaANivel`, `tieneCupos` (→ `xsd:integer`), `fechaCierre` (→ `xsd:date`), `restringeArea`, `ofreceBeca`, `requiereIdioma`, `ubicadaEn`.

- **Jerarquía de propiedad (`rdfs:subPropertyOf`):** `ofrecidaPorUniversidadDestino` ⊑ `ofrecidaPor`.
- **Rangos mixtos:** clases (URIs) y literales tipados con XML Schema (`xsd:integer`, `xsd:date`, `xsd:string`).
- **Vocabularios externos:** `foaf:name`, `foaf:Person`, `dct:title`, `dct:description`.
- **`rdfs:label`** en todas las clases y propiedades.
- **Mínimo 4 individuos por clase.**
- **Serialización final en Turtle (`.ttl`).**

**Razonamiento con `DeductiveClosure(RDFS_Semantics)`:**

- *Caso 1 — jerarquía de clases:* se declara `oferta1 a :OfertaInternacional` → se infiere `oferta1 a :Oferta`.
- *Caso 2 — dominio/rango:* `oferta5 :tieneCupos 4` → se infiere `oferta5 a :Oferta` (por domain); `oferta1 :ofreceEnPais :Francia` → se infiere `:Francia a :Pais` (por range).
- *Caso 3 — subproperty:* `oferta1 :ofrecidaPorUniversidadDestino :Montpellier` → se infiere `oferta1 :ofrecidaPor :Montpellier`.

Se compara el conteo de triples antes y después del razonamiento (`len(g)`) para evidenciar **al menos 4 hechos nuevos inferidos**.

### 3.2 Lógica difusa (Scikit-Fuzzy) — 15%

**Implementado en `difuso.py`** (API `skfuzzy.control`, inferencia Mamdani). Calcula la **idoneidad** de cada par (estudiante, oferta).

**Variables (universo y funciones de pertenencia):**

| Variable | Universo | Valores difusos y parámetros | Tipo |
|---|---|---|---|
| `promedio` (entrada) | 0 – 5 | bajo `trapmf[0,0,3.0,3.6]`, medio `trimf[3.2,3.8,4.4]`, alto `trapmf[4.0,4.5,5,5]` | Trapezoidal + triangular |
| `nivel_idioma` (entrada) | 0 – 100 | básico `trimf[0,0,45]`, intermedio `trimf[30,55,80]`, avanzado `trimf[65,100,100]` | Triangular |
| `cupos` (entrada) | 0 – 10 | pocos `gaussmf(1, 1.5)`, moderados `gaussmf(5, 1.5)`, muchos `gaussmf(10, 2.0)` | Gaussiana |
| `idoneidad` (salida) | 0 – 100 | baja `trimf[0,0,50]`, media `trimf[25,50,75]`, alta `trimf[50,100,100]` | Triangular |

Se cubren las funciones **triangulares, trapezoidales y gaussianas**. Cambio respecto al diseño inicial: la variable `competitividad` se reemplazó por **`cupos` directamente**, porque el número sale tal cual de la ontología (`tieneCupos`) y no requiere una fórmula inventada.

**Justificación de parámetros (para el documento y la sustentación):**
- *Promedio:* trapecios en los extremos porque todo un rango es claramente bajo (< 3.0, nota mínima aprobatoria) o alto (≥ 4.5); triángulo en medio porque hay un valor típico (3.8).
- *Idioma:* escala MCER → 0-100 (A1=10, A2=25, B1=45, B2=65, C1=85, C2/nativo=100).
- *Cupos:* centros en el mínimo (1), el medio (5) y el máximo (10) del rango real del Excel. σ = 1.5 hace que pocos/moderados se crucen en 3 cupos (μ ≈ 0.41). "muchos" usa σ = 2.0 (más ancha) porque: la diferencia entre 8 y 10 cupos casi no cambia la competencia; la campana está centrada en el borde del universo; y en ese rango hay pocos datos. Cruce moderados/muchos ≈ 7.1 cupos (μ ≈ 0.36).
- σ es un parámetro de **ancho** (análogo a la desviación estándar, varianza = σ²), no una probabilidad: el pico siempre vale 1.

**Modificadores (2 casos, usados dentro de las reglas):**
- `muy` → concentración $\mu'(x) = \mu(x)^2$ → término `promedio['muy_alto']`
- `ligeramente` → dilatación $\mu'(x) = \mu(x)^{0.5}$ → término `cupos['ligeramente_pocos']`
- ⚠️ Verificar que coincidan con las fórmulas vistas en clase.

**Reglas difusas (10):** AND = mínimo (`&`), OR = máximo (`|`), NOT = 1 − μ (`~`).

| # | Regla | Operadores |
|---|---|---|
| R1 | promedio alto AND idioma avanzado → alta | AND |
| R2 | promedio **muy** alto AND cupos pocos → alta | AND + muy |
| R3 | promedio bajo OR idioma básico → baja | OR |
| R4 | promedio medio AND cupos pocos → baja | AND |
| R5 | promedio medio AND NOT cupos pocos → media | AND + NOT |
| R6 | promedio medio AND idioma intermedio → media | AND |
| R7 | promedio alto AND idioma intermedio AND cupos muchos → alta | AND (3) |
| R8 | idioma avanzado AND cupos **ligeramente** pocos → media | AND + ligeramente |
| R9 | NOT idioma básico AND cupos muchos → alta | NOT + AND |
| R10 | promedio medio OR cupos moderados → media | OR |

**Defuzzificación:** **centroide** ($z^* = \int z\mu(z)dz / \int \mu(z)dz$). Considera todo el conjunto agregado (todas las reglas activas aportan) y varía de forma suave ante cambios pequeños en las entradas. `difuso.py` imprime la comparación con bisector y MOM para el documento (MOM da saltos bruscos porque solo mira el máximo).

**Categoría:** además del puntaje, se devuelve la etiqueta de salida con mayor pertenencia en ese puntaje. En la práctica: baja < 33.3 ≤ media < 66.7 ≤ alta. La categoría decide **qué regla del experto se activa**; el puntaje sirve para **ordenar** ofertas de la misma categoría.

**Interpretación de los datos de la oferta (sin valores fijos):**
- `requiereIdioma` es texto libre; `idiomas_requeridos()` extrae idioma(s) y nivel: `"Inglés B2 / Alemán"` → inglés o alemán (se usa el que mejor domine el estudiante). Sin idioma: nacional → español; internacional → inglés.
- Cupos ausentes en la ontología (Stavanger, Tolima) → valor neutro `CUPOS_DESCONOCIDOS = 5` (centro de "moderados": ni premia ni castiga).

**Ejemplo de ejecución paso a paso (promedio 4.3, idioma 70, 3 cupos):**
1. Fuzzificación: alto 0.60, medio 0.17, muy alto 0.36; intermedio 0.40, avanzado 0.14; pocos 0.41, moderados 0.41, ligeramente pocos 0.64.
2. Reglas: R1 = 0.14 (alta), R2 = 0.36 (alta), R4 = 0.17 (baja), R5 = 0.17, R8 = 0.14, R10 = 0.41 (media).
3. Agregación (máximo por consecuente): baja 0.17, media 0.41, alta 0.36.
4. Centroide → **56.62** → categoría **media**.

**Resultados con las 8 ofertas** (Estudiante A: promedio 4.3, inglés B2, francés A2, español nativo):

| Oferta | nivel_idioma | cupos | Puntaje | Categoría |
|---|---|---|---|---|
| Montpellier | 25 | 4 | 41.93 | media |
| UNAM | 100 | 10 | 76.81 | alta |
| Freie Berlin | 65 | 4 | 50.00 | media |
| Stavanger | 65 | 5* | 50.55 | media |
| Escuela Colombiana | 100 | 5 | 63.51 | media |
| EAFIT | 100 | 9 | 76.56 | alta |
| UFPS | 100 | 10 | 76.81 | alta |
| Tolima | 100 | 5* | 63.51 | media |

\* cupos estimados. UNAM sale "alta" pero su cierre ya venció y Stavanger está restringida por área: **el experto debe descartarlas** con sus reglas duras aunque el difuso las califique bien (buen ejemplo para el pitch).

**Entregable gráfico:** `graficar_todo()` genera `mf_promedio.png`, `mf_idioma.png`, `mf_cupos.png` (con los modificadores en línea punteada), `mf_idoneidad.png` y `salida_centroide.png` (salida agregada con la línea del centroide).

**Funciones públicas de `difuso.py`:**

| Función | Qué hace |
|---|---|
| `evaluar(prom, idioma, cupos, metodo='centroid')` | Inferencia para un caso; devuelve puntaje, categoría, grados y la simulación |
| `ofertas_desde_grafo(g)` | Lee del grafo razonado todas las `ex:Oferta` con `tieneCupos` y `requiereIdioma` (sin nombres fijos) |
| `resultados_para_experto(estudiante, ofertas)` | Lista `{oferta, puntaje, categoria}` lista para `ResultadoDifuso(**d)` |
| `graficar_todo()` | Genera las gráficas del documento |

### 3.3 Sistema experto (Experta) — 25%

**Clases de hechos (mínimo 5):**
`Estudiante`, `Oferta`, `Universidad`, `ResultadoDifuso`, `Recomendacion`.

Se recomienda declarar los campos con `Field(...)` (tipo, `mandatory`, `default`) para tener validación y documentación, en lugar de dejar los `Fact` como diccionarios abiertos.

**Reglas (mínimo 15) organizadas en 3 niveles de prioridad:**

- **Prioridad alta (salience alto) — filtros duros:**
  fecha de cierre vencida → descartar; nivel de estudios no coincide → descartar; área del estudiante restringida → descartar; sin convenio vigente → descartar.
- **Prioridad media — elegibilidad y evaluación:**
  verificar requisitos de GPA e idioma; incorporar el puntaje difuso como precondición.
- **Prioridad baja — salida:**
  generar la recomendación, ordenar el ranking y producir la explicación de por qué se recomienda cada oferta.

**Mecanismos de resolución de conflictos (los tres, explicados en el documento):**
- **Salience:** prioridad asignada manualmente a cada regla; es la que se usa explícitamente en el código.
- **Specificity:** ante igual salience, se prefiere la regla con mayor número de condiciones (más específica).
- **Recency:** ante igual salience y especificidad, se prefiere la regla que hace match con los hechos declarados más recientemente.

**Control de bucles infinitos:** uso de **No-Loop** en las reglas que redeclaran o modifican hechos que ellas mismas evalúan.

**Base de hechos (mínimo 40):** el conteo mezcla los hechos declarados manualmente (estudiante, resultado difuso) con los que inyecta el traductor desde el grafo razonado. Con ~15 ofertas × varias propiedades, el traductor solo ya supera los 40 con holgura. Experta ignora silenciosamente los duplicados, así que no hace falta control de repetidos en el traductor.

---

## 4. Contrato Ontología ↔ Experto

Esta sección fija los nombres exactos que comparten los módulos. **Se define antes de codear** para evitar el problema clásico de que la ontología declare `tieneCupos` con `xsd:integer` y el experto reciba un string, o que dos integrantes creen individuos con URIs que colisionen.

### 4.1 Namespace y convención de URIs

- **Prefijo del proyecto:** `@prefix ex: <http://intercambios.unal.edu.co/> .`
- **Individuos:** `snake_case`, con prefijo por tipo y sufijo estable:
  - Ofertas: `ex:oferta_<uni>_<periodo>` — ej. `ex:oferta_stavanger_2027_1`
  - Universidades: `ex:uni_<nombre>` — ej. `ex:uni_montpellier`, `ex:uni_freie_berlin`
  - Países: `ex:pais_<nombre>` — ej. `ex:pais_noruega`, `ex:pais_alemania`
  - Áreas: `ex:area_<nombre>` — ej. `ex:area_ingenieria_sistemas`
- **Sin tildes ni ñ en URIs** (evitar problemas con serializadores).
- El nombre humano con tildes va en `foaf:name` o `rdfs:label`, no en la URI.

### 4.2 Correspondencia RDF → Fact de Experta

La traducción usa **dos mapas de vocabulario** (ninguno contiene valores de un caso concreto): uno para los tipos (`rdf:type`) y otro para las propiedades.

**(a) Tipos → `CLASS_TO_FACT`: una clase de `Fact` por cada clase de la ontología.**
Cada triple `individuo rdf:type Clase` se traduce a un hecho simple `Clase(id=individuo)`.

| Clase RDF (objeto de `rdf:type`) | Fact de Experta        | Slot        |
|----------------------------------|------------------------|-------------|
| `ex:Oferta`                      | `Oferta`               | `id: str`   |
| `ex:OfertaInternacional`         | `OfertaInternacional`  | `id: str`   |
| `ex:OfertaNacional`              | `OfertaNacional`       | `id: str`   |
| `ex:Universidad`                 | `Universidad`          | `id: str`   |
| `ex:UniversidadDestino`          | `UniversidadDestino`   | `id: str`   |
| `ex:Pais`                        | `Pais`                 | `id: str`   |
| `ex:AreaConocimiento`            | `AreaConocimiento`     | `id: str`   |
| `ex:Ingenieria`                  | `Ingenieria`           | `id: str`   |
| `ex:NivelEstudios`               | `NivelEstudios`        | `id: str`   |
| `ex:Convenio`                    | `Convenio`             | `id: str`   |
| `ex:Beca`                        | `Beca`                 | `id: str`   |
| `ex:Estudiante`                  | `Estudiante`           | `id: str`   |

> Ejemplo: `ex:oferta_montpellier_2027_1 rdf:type ex:OfertaInternacional` → `OfertaInternacional(id="oferta_montpellier_2027_1")`. Como el razonador infiere además `... rdf:type ex:Oferta`, también se genera `Oferta(id="oferta_montpellier_2027_1")`. Las reglas casan directo, p. ej. `OfertaInternacional(MATCH.o)`, sin comparar strings.

**(b) Propiedades → `PREDICATE_TO_FACT`: una clase de `Fact` por predicado.**

| Predicado RDF          | Rango RDF               | Fact de Experta       | Slots                              |
|------------------------|-------------------------|-----------------------|------------------------------------|
| `ex:tieneCupos`        | `xsd:integer`           | `TieneCupos`          | `oferta: str`, `n: int`            |
| `ex:fechaCierre`       | `xsd:date`              | `FechaCierre`         | `oferta: str`, `fecha: date`       |
| `ex:ofrecidaPor`       | `ex:Universidad`        | `OfrecidaPor`         | `oferta: str`, `uni: str`          |
| `ex:ofreceEnPais`      | `ex:Pais`               | `OfreceEnPais`        | `oferta: str`, `pais: str`         |
| `ex:tieneArea`         | `ex:AreaConocimiento`   | `TieneArea`           | `oferta: str`, `area: str`         |
| `ex:dirigidaANivel`    | `ex:NivelEstudios`      | `DirigidaANivel`      | `oferta: str`, `nivel: str`        |
| `ex:restringeArea`     | `ex:AreaConocimiento`   | `RestringeArea`       | `oferta: str`, `area: str`         |
| `ex:ofreceBeca`        | `xsd:boolean`           | `OfreceBeca`          | `oferta: str`, `tiene: bool`       |
| `ex:requiereIdioma`    | `xsd:string`            | `RequiereIdioma`      | `oferta: str`, `nivel: str`        |
| `ex:ubicadaEn`         | `ex:Pais`               | `UbicadaEn`           | `uni: str`, `pais: str`            |

Con las dos tablas, los diccionarios del traductor quedan casi triviales:

```python
CLASS_TO_FACT = {
    EX.Oferta: Oferta,               EX.OfertaInternacional: OfertaInternacional,
    EX.OfertaNacional: OfertaNacional, EX.Universidad: Universidad,
    EX.UniversidadDestino: UniversidadDestino, EX.Pais: Pais,
    EX.AreaConocimiento: AreaConocimiento, EX.Ingenieria: Ingenieria,
    EX.NivelEstudios: NivelEstudios, EX.Convenio: Convenio,
    EX.Beca: Beca,                   EX.Estudiante: Estudiante,
}

PREDICATE_TO_FACT = {
    EX.tieneCupos:     TieneCupos,
    EX.fechaCierre:    FechaCierre,
    EX.ofrecidaPor:    OfrecidaPor,
    EX.ofreceEnPais:   OfreceEnPais,
    EX.tieneArea:      TieneArea,
    EX.dirigidaANivel: DirigidaANivel,
    EX.restringeArea:  RestringeArea,
    EX.ofreceBeca:     OfreceBeca,
    EX.requiereIdioma: RequiereIdioma,
    EX.ubicadaEn:      UbicadaEn,
}
```

Ambos mapas mapean **vocabulario** (clases y predicados), no valores. El nombre del individuo (`oferta_montpellier_2027_1`) siempre sale del grafo, nunca del código; por eso se cumple la regla del enunciado: el traductor no contiene nombres ni valores de ningún caso específico.

> Nota: `ex:ofrecidaPorUniversidadDestino` (la subpropiedad) no se lista aparte porque el razonador la propaga a `ex:ofrecidaPor`, que es la que el traductor traduce.

### 4.3 Regla de coherencia

Cualquier cambio en las columnas de esta tabla (agregar predicado, cambiar tipo del rango) **obliga a actualizar simultáneamente**:

1. La declaración de la clase o propiedad en el `.ttl`.
2. La clase de `Fact` correspondiente en el módulo experto.
3. La entrada en `CLASS_TO_FACT` (si es una clase) o en `PREDICATE_TO_FACT` (si es una propiedad) del traductor.
4. Esta tabla en el `Context.md`.

Si el cambio queda solo en 1-2 de esos lugares, el sistema o revienta o produce hechos vacíos silenciosamente.

---

## 5. Integración de los tres componentes

Este es el punto que más peso tiene en la evaluación y donde más se falla.

### 5.1 Traductor Ontología → Sistema Experto

El traductor debe **recorrer dinámicamente el grafo RDF ya razonado**, triple por triple, y generar los `Fact` del sistema experto a partir de él (iterando, por ejemplo, sobre `graph.triples((None, None, None))` y despachando por tipo de predicado).

**Prohibido:** declarar hechos a mano con valores fijos, del tipo `engine.declare(Oferta(pais='Francia'))`. El código del traductor no debe contener nombres ni valores de ningún caso concreto.

**Diseño recomendado:** una clase de `Fact` por cada **clase** de la ontología (`Oferta`, `OfertaInternacional`, `Pais`, …) y una por cada **propiedad** relevante (`OfreceEnPais`, `TieneCupos`, `FechaCierre`, …), con dos diccionarios en el traductor que mapeen **clase RDF → `Fact`** (`CLASS_TO_FACT`) y **predicado RDF → `Fact`** (`PREDICATE_TO_FACT`) — ver tablas de la sección 4.2. Esos diccionarios no violan la regla porque mapean vocabulario (clases y predicados), no valores de un caso. Así `ex:oferta_x rdf:type ex:OfertaInternacional` se traduce al hecho simple `OfertaInternacional(id="oferta_x")`.

**Cobertura completa:** hay que traducir *todos* los triples relevantes:

- Los de `rdf:type` (individuo pertenece a una clase).
- Los de propiedades (relaciones entre individuos, o entre individuo y literal).
- **También los triples inferidos por el razonador.** Por eso el orden importa: primero `g.parse(...)`, después `DeductiveClosure(RDFS_Semantics).expand(g)`, después el traductor recorre `g` ya expandido.

**Caso de prueba obligatorio:** agregar una universidad o una relación nueva **únicamente en el archivo `.ttl`**, volver a ejecutar sin tocar el código Python, y evidenciar que el hecho nuevo aparece automáticamente como `Fact` y activa las reglas correspondientes. **Si para reflejar el cambio hay que editar Python, el traductor está mal diseñado.**

### 5.1.1 Ruta de implementación propuesta del traductor

El traductor es una función `traducir(grafo_razonado, engine)` que recorre el grafo y declara hechos. La ruta completa:

**1. Orden del pipeline (obligatorio).** Razonar ANTES de traducir; así las inferencias (p. ej. `oferta a Oferta`, `ofrecidaPor`) también llegan al experto:

```python
g.parse("ontologia.ttl")                       # 1. cargar la fuente canónica
DeductiveClosure(RDFS_Semantics).expand(g)     # 2. razonar (agrega las inferencias en memoria)
traducir(g, engine)                            # 3. recorrer g YA expandido -> Facts
engine.run()                                   # 4. disparar las reglas
```

**2. Clases de `Fact` (Experta):** una por cada **clase** de la ontología
(`Oferta`, `OfertaInternacional`, `OfertaNacional`, `Universidad`, `UniversidadDestino`,
`Pais`, `AreaConocimiento`, `Ingenieria`, `NivelEstudios`, `Convenio`, `Beca`, `Estudiante`)
y una por cada **propiedad** (`OfrecidaPor`, `OfreceEnPais`, `TieneArea`, `DirigidaANivel`,
`RestringeArea`, `UbicadaEn`, `TieneCupos`, `FechaCierre`, `RequiereIdioma`, `OfreceBeca`),
con sus slots tipados con `Field(...)`. Aparte, hechos **no ontológicos**: `Estudiante` (el
perfil, declarado a mano) y `ResultadoDifuso` (que inyecta el módulo difuso).

**3. Los dos mapas del traductor = vocabulario, no valores.** No contienen ni un nombre de universidad, país u oferta; solo clases y predicados → clase de `Fact` (tablas de §4.2):

```python
CLASS_TO_FACT     = { EX.OfertaInternacional: OfertaInternacional, EX.Pais: Pais, ... }
PREDICATE_TO_FACT = { EX.ofrecidaPor: OfrecidaPor, EX.tieneCupos: TieneCupos, ... }
```

**4. Algoritmo de recorrido (genérico, sin valores de caso):**

```
para cada (s, p, o) en g:
    si p == RDF.type:
        si o NO está en CLASS_TO_FACT:            continuar   # ignora rdfs:Resource, rdfs:Class, foaf:Person...
        engine.declare( CLASS_TO_FACT[o]( id=nombre_local(s) ) )   # hecho simple: Clase(id=individuo)
        continuar
    si p NO está en PREDICATE_TO_FACT:            continuar   # ignora el resto del ruido RDFS
    slot_o = nombre_local(o)  si o es URI
             valor_tipado(o)  si o es literal                 # xsd:integer->int, xsd:date->date, xsd:boolean->bool, xsd:string->str
    engine.declare( PREDICATE_TO_FACT[p]( nombre_local(s), slot_o ) )
```

Convertir el literal según su datatype es clave: así `TieneCupos` recibe un `int` real y `FechaCierre` un `date`, para que las reglas comparen números y fechas (regla "fecha vencida", competitividad difusa, etc.).

**5. Filtro de relevancia.** Solo se traducen los predicados del contrato y los `rdf:type` hacia clases `ex:`. El andamiaje del cierre RDFS (`rdfs:Resource`, `subClassOf` reflexivo, variantes de literal, declaraciones de propiedad) **no** se traduce, porque no es conocimiento del dominio. Esto cumple la exigencia del enunciado ("cada triple **relevante**") y produce ~139 hechos.

**6. Regla de coherencia (§4.3).** Agregar o cambiar una clase o un predicado obliga a tocar a la vez: el `.ttl`, la clase `Fact`, la entrada en `CLASS_TO_FACT` o `PREDICATE_TO_FACT`, y la tabla §4.2.

### 5.2 Lógica difusa → Sistema Experto

El módulo difuso **no importa Experta**: solo calcula y devuelve diccionarios. El código de integración los convierte en hechos. Hay **un `ResultadoDifuso` por oferta** (8 con un estudiante), y su campo `oferta` es el **mismo id** que usa la ontología; así las reglas pueden cruzar hechos ontológicos y difusos con `MATCH.o`.

```python
g.parse("ontologia.ttl"); DeductiveClosure(RDFS_Semantics).expand(g)
traducir(g, engine)                                   # hechos de la ontología
ofertas = ofertas_desde_grafo(g)                      # datos que necesita el difuso, leídos del grafo
for d in resultados_para_experto(estudiante, ofertas):
    engine.declare(ResultadoDifuso(**d))              # hechos difusos
engine.run()
```

**Clase de hecho (contrato):**

```python
class ResultadoDifuso(Fact):
    oferta = Field(str, mandatory=True)      # mismo id que en la ontología
    puntaje = Field(float, mandatory=True)   # 0-100
    categoria = Field(str, mandatory=True)   # 'baja' | 'media' | 'alta'
```

**Reglas del experto que lo usan (obligatorio: al menos una; cuentan dentro de las 15):**

```python
@Rule(ResultadoDifuso(oferta=MATCH.o, categoria='alta'), NOT(Descartada(oferta=MATCH.o)))
def recomendar(self, o): ...            # -> Recomendacion(tipo='recomendada')

@Rule(ResultadoDifuso(oferta=MATCH.o, categoria='media'), NOT(Descartada(oferta=MATCH.o)))
def alternativa(self, o): ...           # -> Recomendacion(tipo='alternativa')

@Rule(ResultadoDifuso(oferta=MATCH.o, categoria='baja'))
def no_recomendar(self, o): ...

@Rule(OfreceBeca(oferta=MATCH.o, tiene=True), ResultadoDifuso(oferta=MATCH.o, categoria='alta'))
def prioridad_beca(self, o): ...        # mezcla hecho ontológico + difuso
```

Roles: la **ontología describe** la oferta, el **difuso la califica** para el estudiante, el **experto decide** (los filtros duros de prioridad alta —fecha, área, nivel— se aplican antes y ganan sobre la calificación difusa).

> ⚠️ Experta no funciona tal cual en Python ≥ 3.10 (Colab): falla con `collections.Mapping`. Parche antes de importarla:
> ```python
> import collections, collections.abc
> collections.Mapping = collections.abc.Mapping
> ```

### 5.3 Cobertura completa

Todos los hechos generados en la ontología deben traducirse al sistema experto, sin excepción. Deben existir reglas adicionales que usen esos hechos en sus precondiciones o efectos, para que la ontología no quede como una base de datos aislada.

---

## 6. Decisiones de arquitectura del proyecto

### 6.1 El `.ttl` es un solo archivo, canónico

- **Un solo `.ttl`.** No hay "ttl inicial" y "ttl final".
- El razonador **nunca modifica el archivo `.ttl` en disco.** Solo expande el grafo en memoria (`g`).
- Ese archivo cumple dos funciones:
  1. Fuente de datos del sistema cada vez que corre.
  2. Objeto que el profesor edita para la prueba obligatoria del traductor.
- **Archivo derivado (opcional):** el script de razonamiento exporta además `ontologia_razonada.ttl`, una copia del grafo **después** de razonar (con las tripletas inferidas), solo para inspección. No es la fuente del sistema y nunca sustituye al canónico.

### 6.2 Cómo generar el `.ttl` sin escribir Turtle a mano

- Escribir un **script Python auxiliar** que use RDFLib para construir el grafo con `g.add(...)` (mucho más rápido que escribir Turtle) y al final ejecutar `g.serialize('ontologia.ttl', format='turtle')`.
- Ese script **se corre una sola vez** para generar el archivo canónico. No es parte del sistema en operación.
- A partir de ahí, el `.ttl` queda como la fuente única y el sistema real solo lo lee.
- No está prohibido por el enunciado; es solo un atajo de desarrollo.

### 6.3 Volumen esperado de triples

Los mínimos del enunciado (10 clases, 5 subClassOf, 10 propiedades, ≥ 4 individuos por clase) se refieren a **lo que escribís vos** antes del razonador. Después del razonador el grafo debe crecer **al menos 4** hechos nuevos (mínimo del enunciado).

**Valores reales de esta implementación.** Con **8 ofertas** (4 internacionales + 4 nacionales, el mínimo para cumplir "≥ 4 individuos" en `OfertaInternacional` y `OfertaNacional`):

- `ontologia.ttl` canónico: **235 triples** asertados → **509** tras razonar (**274** inferidos).
- De esos 509, el traductor mapea solo los **~139 hechos relevantes** (tipos hacia clases `ex:` + las 11 propiedades del contrato); el resto es andamiaje RDFS. 139 está muy por encima del mínimo de 40 del enunciado.
- Se eligió reducir de 15 a 8 ofertas para acotar el número de hechos manteniendo todos los mínimos y los casos de discriminación clave (apta, fecha vencida + GPA, con beca, restringida por área, nacional con Ing. de Sistemas).

**Política de anotaciones (para no duplicar tripletas).** `rdfs:label` es obligatorio solo en clases y propiedades. En los individuos se usa una sola anotación legible según su semántica: `foaf:name` para agentes (estudiantes, universidades), `dct:title`/`dct:description` para documentos (ofertas, convenios, becas) y `rdfs:label` para el resto (países, niveles, áreas). Se evita así el par redundante `rdfs:label` + `foaf:name` con el mismo valor.

### 6.4 Flujo en Colab

1. Celda de instalación: `!pip install rdflib owlrl experta scikit-fuzzy` (+ parche de `collections.Mapping` para Experta, ver §5.2).
2. Celda que carga el `.ttl` al entorno de Colab (subida manual con `files.upload()`, o montaje de Google Drive, o `wget` desde GitHub público).
3. Celda del sistema real: `g.parse(...)` → razonador → traductor → módulo difuso → motor experto.
4. El `.ttl` va en el ZIP como archivo separado, **no** embebido como string dentro del notebook (embeberlo rompería la prueba obligatoria, porque "editar la ontología" implicaría editar Python).

---

## 7. Entregables

### 7.1 Documento de diseño (PDF)
- Descripción detallada de cada componente.
- Gráficas de las funciones de pertenencia.
- Diagrama de clases de la ontología.
- Documentación de los tres casos de generación de nuevo conocimiento.
- Comparación del grafo antes y después del razonamiento.
- Explicación de los mecanismos de resolución de conflictos.
- Enlace al video tipo pitch.

### 7.2 Código fuente (Python / Google Colab)
Separado por módulos: `experto/`, `difuso/`, `ontologia/`, más el traductor y el script de integración. Comentado por función, con explicación del flujo general.

### 7.3 Video tipo pitch
Máximo 5 minutos, subido a YouTube, con participación de los 4 integrantes y demostración del sistema funcionando.

### 7.4 Archivo de entrega
ZIP en Google Classroom con el nombre: `practica1-grupo-XX-equipo-YY.zip`

---

## 8. Evaluación

| Componente | Peso |
|---|---|
| Módulo de Sistemas Expertos | 25% |
| Módulo de Ontología | 25% |
| Módulo de Lógica Difusa | 15% |
| Video tipo pitch | 20% |
| Atención a preguntas | 15% |

---

## 9. Reparto sugerido del trabajo (4 personas)

1. **Ontología + traductor** — diseño del `.ttl`, razonador OWL-RL, traductor dinámico al motor de reglas.
2. **Lógica difusa** — variables, funciones de pertenencia, reglas, defuzzificación y gráficas.
3. **Sistema experto** — clases de hechos, las 15+ reglas, prioridades y mecanismos de control.
4. **Integración, documento y pitch** — limpieza del Excel a datos, ensamble de los módulos, documento de diseño y video.

El documento de diseño y el video son responsabilidad transversal de todo el equipo.
