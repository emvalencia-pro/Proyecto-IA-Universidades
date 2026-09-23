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
| CUPOS DISPONIBLES | **Lógica difusa** | Pocos cupos → alta *competitividad* |
| MODALIDAD | Experto (hecho) | Presencial / Virtual |
| OBSERVACIONES | Ontología + Experto | Beca, GPA exigido, requisitos de idioma |

### Criterio de selección de instancias

No se usan las 73 filas. Se eligen entre **12 y 15 ofertas** con variedad suficiente para que las reglas realmente se disparen:

- Ofertas de "todas las áreas comunes" (aptas para Ingeniería de Sistemas).
- **University of Stavanger** (Noruega) — restringida a Matemáticas y Ciencias Naturales: caso de exclusión.
- Una oferta de **Bellas Artes / Arquitectura** (ej. ENSAPM, Università di Bologna – Architecture): se filtra.
- **Freie Universität Berlin** — caso con beca (€450/mes, seguro médico, exención de matrícula).
- **UNAM** — caso con requisito de GPA (8/10) y avance mínimo del 40%.
- **Universidade Feevale** — caso de modalidad virtual.
- **Escuela Colombiana de Ingeniería Julio Garavito** (nacional) — lista explícitamente Ingeniería de Sistemas.

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

**Variables de entrada (mínimo 3) y salida:**

| Variable | Universo del discurso | Valores difusos | Tipo de función |
|---|---|---|---|
| `promedio_estudiante` | 0 – 5 | bajo, medio, alto | Trapezoidales (bajo/alto) + triangular (medio) |
| `nivel_idioma` | 0 – 100 | básico, intermedio, avanzado | Triangulares |
| `competitividad` (derivada de cupos) | 0 – 10 | baja, media, alta | Gaussianas |
| `idoneidad` (salida) | 0 – 100 | baja, media, alta | Triangulares |

Con este reparto se cubren obligatoriamente las funciones **triangulares, trapezoidales y Gaussianas**.

**Modificadores (2 casos):**
- `muy` → concentración: $\mu'(x) = \mu(x)^2$ → "promedio **muy** alto"
- `ligeramente` → dilatación: $\mu'(x) = \mu(x)^{0.5}$ → oferta "**ligeramente** competitiva"

**Reglas difusas (mínimo 9)**, incluyendo los tres operadores:
- AND (intersección): `promedio alto AND idioma avanzado → idoneidad alta`
- OR (unión): `promedio bajo OR competitividad alta → idoneidad baja`
- NOT (complemento): `NOT competitividad alta AND promedio medio → idoneidad media`

**Defuzzificación:** método del **centroide**, justificado porque toma en cuenta toda la distribución del conjunto difuso resultante y produce una salida estable frente a pequeños cambios en las entradas.

**Entregable gráfico:** gráficas de todas las funciones de pertenencia (requisito explícito del documento de diseño).

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

### 4.2 Tabla de correspondencia: predicado RDF → Fact de Experta

| Predicado RDF          | Rango RDF               | Fact de Experta       | Slots                              |
|------------------------|-------------------------|-----------------------|------------------------------------|
| `rdf:type`             | clase                   | `EsInstanciaDe`       | `individuo: str`, `clase: str`     |
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

Con esta tabla, el diccionario del traductor queda casi trivial:

```python
PREDICATE_TO_FACT = {
    RDF.type:          EsInstanciaDe,
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

El traductor mapea **vocabulario**, no valores. Sigue cumpliendo la regla del enunciado: no contiene nombres ni valores propios de ningún caso específico.

### 4.3 Regla de coherencia

Cualquier cambio en las columnas de esta tabla (agregar predicado, cambiar tipo del rango) **obliga a actualizar simultáneamente**:

1. La declaración de la propiedad en el `.ttl`.
2. La clase de `Fact` correspondiente en el módulo experto.
3. La entrada en `PREDICATE_TO_FACT` del traductor.
4. Esta tabla en el `Context.md`.

Si el cambio queda solo en 1-2 de esos lugares, el sistema o revienta o produce hechos vacíos silenciosamente.

---

## 5. Integración de los tres componentes

Este es el punto que más peso tiene en la evaluación y donde más se falla.

### 5.1 Traductor Ontología → Sistema Experto

El traductor debe **recorrer dinámicamente el grafo RDF ya razonado**, triple por triple, y generar los `Fact` del sistema experto a partir de él (iterando, por ejemplo, sobre `graph.triples((None, None, None))` y despachando por tipo de predicado).

**Prohibido:** declarar hechos a mano con valores fijos, del tipo `engine.declare(Oferta(pais='Francia'))`. El código del traductor no debe contener nombres ni valores de ningún caso concreto.

**Diseño recomendado:** una clase de `Fact` por tipo de relación relevante (`EsInstanciaDe`, `OfreceEnPais`, `TieneCupos`, `FechaCierre`, …), con un diccionario en el traductor que mapee **predicado RDF → clase de `Fact`** (ver tabla de la sección 4.2). Ese diccionario no viola la regla porque mapea vocabulario, no valores de un caso.

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

**2. Clases de `Fact` (Experta), una por relación del contrato §4.2:**
`EsInstanciaDe`, `OfrecidaPor`, `OfreceEnPais`, `TieneArea`, `DirigidaANivel`, `RestringeArea`, `UbicadaEn`, `TieneCupos`, `FechaCierre`, `RequiereIdioma`, `OfreceBeca`. Cada una con sus slots tipados con `Field(...)`. Aparte, hechos **no ontológicos**: `Estudiante` (el perfil, declarado a mano) y `ResultadoDifuso` (que inyecta el módulo difuso).

**3. El único mapa del traductor = vocabulario, no valores.** No contiene ni un nombre de universidad, país u oferta; solo predicados → clase de `Fact` (la tabla de §4.2):

```python
PREDICATE_TO_FACT = { RDF.type: EsInstanciaDe, EX.ofrecidaPor: OfrecidaPor, EX.tieneCupos: TieneCupos, ... }
```

**4. Algoritmo de recorrido (genérico, sin valores de caso):**

```
para cada (s, p, o) en g:
    si p no está en PREDICATE_TO_FACT:            continuar   # ignora el ruido RDFS
    si p == RDF.type y o NO es una clase ex::     continuar   # ignora rdfs:Resource, rdfs:Class, foaf:Person...
    slot_s = nombre_local(s)                                  # quita el namespace -> string estable
    slot_o = nombre_local(o)  si o es URI
             valor_tipado(o)  si o es literal                 # xsd:integer->int, xsd:date->date, xsd:boolean->bool, xsd:string->str
    engine.declare( PREDICATE_TO_FACT[p](slot_s, slot_o) )
```

Convertir el literal según su datatype es clave: así `TieneCupos` recibe un `int` real y `FechaCierre` un `date`, para que las reglas comparen números y fechas (regla "fecha vencida", competitividad difusa, etc.).

**5. Filtro de relevancia.** Solo se traducen los predicados del contrato y los `rdf:type` hacia clases `ex:`. El andamiaje del cierre RDFS (`rdfs:Resource`, `subClassOf` reflexivo, variantes de literal, declaraciones de propiedad) **no** se traduce, porque no es conocimiento del dominio. Esto cumple la exigencia del enunciado ("cada triple **relevante**") y produce ~139 hechos.

**6. Regla de coherencia (§4.3).** Agregar o cambiar un predicado obliga a tocar a la vez: el `.ttl`, la clase `Fact`, la entrada en `PREDICATE_TO_FACT` y la tabla §4.2.

### 5.2 Lógica difusa → Sistema Experto

El resultado de la defuzzificación (puntaje de idoneidad y su categoría) se declara como hecho `ResultadoDifuso` y se usa **como precondición de al menos una regla**. Ejemplo: solo se genera una `Recomendacion` si la categoría difusa es `alta`.

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

1. Celda de instalación: `!pip install rdflib owlrl experta scikit-fuzzy`.
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
