# Contexto del proyecto — Sistema Híbrido de IA (Práctica 1)

Este documento describe el estado **real y verificado** de [`ProyectoIA_Final.ipynb`](ProyectoIA_Final.ipynb) frente a los requisitos de [`Enunciado.MD`](Enunciado.MD). Está escrito para que cualquier IA o colaborador que retome el proyecto tenga contexto completo sin tener que releer las ~2500 líneas del notebook desde cero.

**Verificación:** el notebook fue ejecutado de punta a punta (todas las celdas, en orden, kernel limpio) sin errores. Las cifras de este documento (triples, hechos, reglas disparadas) salen de esa ejecución real, no del código sin correr.

---

## 1. Dominio y objetivo

Sistema de recomendación de universidades para intercambio académico (ofertas nacionales e internacionales), que integra los tres paradigmas exigidos por el enunciado:

- **Ontología** (RDFLib + OWL-RL) — modela oferta, universidad, país, área, convenio, beca, estudiante.
- **Lógica difusa** (Scikit-Fuzzy) — evalúa qué tan idónea es una oferta para un estudiante según su promedio, su nivel de idioma y los cupos disponibles.
- **Sistema experto** (Experta) — aplica reglas duras + el resultado difuso para descartar o recomendar cada oferta.

Namespace del proyecto: `ex: <http://intercambios.unal.edu.co/>`.

---

## 2. Estructura del notebook (19 celdas)

| # | Tipo | Contenido |
|---|---|---|
| 0 | md | Portada |
| 1 | code | `%pip install rdflib owlrl` |
| 2 | md | Encabezado "Ontología" |
| 3 | code | `build_ontologia`: construye el grafo con `g.add(...)` y lo serializa a `ontologia.ttl` (script auxiliar, se corre una sola vez; no es parte del sistema en operación) |
| 4 | code | `%%writefile ontologia.ttl` — la ontología canónica en Turtle (esto es lo que el sistema realmente lee; **sobrescribe** el archivo que generó la celda 3) |
| 5 | code | `razonamiento`: carga el `.ttl`, aplica `DeductiveClosure(RDFS_Semantics)`, verifica los 3 casos de inferencia exigidos y compara antes/después |
| 6 | md | Encabezado "Lógica difusa" |
| 7 | code | `%pip install scikit-fuzzy` |
| 8 | code | Módulo difuso completo: variables, reglas, defuzzificación, funciones de integración con el grafo |
| 9 | md | Encabezado "Sistema experto e integración" |
| 10 | code | `%pip install experta` |
| 11 | code | Parche de compatibilidad + **todas las clases de `Fact`** |
| 12 | code | Traductor Ontología → Sistema Experto (`CLASS_TO_FACT`, `PREDICATE_TO_FACT`, `traducir()`) |
| 13 | code | `EstrategiaConflictos`, `AgendaConTraza` y la clase `SistemaExpertoIntercambios` con **25 reglas** |
| 14 | code | `correr_caso()` + 5 escenarios de demo (estudiantes reales de la ontología) |
| 15 | code | Pruebas unitarias de reglas que no se disparan con los datos reales |
| 16 | code | Demostración explícita de los 3 mecanismos de resolución de conflictos, con `assert` |
| 17 | md | Encabezado de la prueba obligatoria |
| 18 | code | Caso de prueba obligatorio del traductor (edita el `.ttl`, no el código) |

**Importante — orden de las celdas 3 y 4:** la celda 3 genera `ontologia.ttl` dinámicamente y la celda 4 lo vuelve a escribir con `%%writefile` (contenido curado a mano, ligeramente distinto: 294 triples la versión dinámica vs. **292** la canónica). El archivo que el resto del notebook usa es siempre el de la celda 4, porque se ejecuta después. Editar la celda 4 (o el archivo `ontologia.ttl` directamente) es la forma correcta de cambiar la ontología; la celda 3 es solo el generador inicial.

---

## 3. Ontología

### 3.1 Clases (12) y jerarquías (5 `rdfs:subClassOf`)

`Oferta`, `Universidad`, `Pais`, `AreaConocimiento`, `NivelEstudios`, `Convenio`, `Beca`, `Estudiante` (⊑ `foaf:Person`), `OfertaInternacional` (⊑ `Oferta`), `OfertaNacional` (⊑ `Oferta`), `UniversidadDestino` (⊑ `Universidad`), `Ingenieria` (⊑ `AreaConocimiento`).

### 3.2 Propiedades (20, con `rdfs:domain`/`rdfs:range`, 1 `rdfs:subPropertyOf`)

**Sobre `Oferta`:** `ofrecidaPor`→Universidad, `ofreceEnPais`→Pais, `tieneArea`→AreaConocimiento, `dirigidaANivel`→NivelEstudios, `restringeArea`→AreaConocimiento, `tieneCupos`→xsd:integer, `fechaCierre`→xsd:date, `requiereIdioma`→xsd:string, `ofreceBeca`→xsd:boolean, `tieneBeca`→Beca, `amparadaPorConvenio`→Convenio.

**Sobre `Universidad`:** `ubicadaEn`→Pais.

**Sobre `Estudiante`:** `estudiaArea`→AreaConocimiento, `cursaNivel`→NivelEstudios, `tienePromedio`→xsd:decimal, `dominaIdioma`→xsd:string (puede repetirse, uno por idioma).

**Jerarquía de propiedad:** `ofrecidaPorUniversidadDestino` (Oferta→UniversidadDestino) ⊑ `ofrecidaPor`.

Nota de diseño clave: `tieneBeca` y `amparadaPorConvenio` ligan una oferta puntual con una beca/convenio concretos, no solo con el catálogo general de `Beca`/`Convenio`. Esto es lo que permite reglas como `r12_recomendacion_con_beca` y `r15_respaldo_convenio`.

### 3.3 Individuos y estudiantes

4 estudiantes reales en la ontología, cada uno con perfil completo (`estudiaArea`, `cursaNivel`, `tienePromedio`, `dominaIdioma`):

| id | área | nivel | promedio | idiomas |
|---|---|---|---|---|
| `est_juan_sistemas` | Ingeniería de Sistemas | pregrado | 4.3 | español nativo, inglés B2, francés A2 |
| `est_maria_matematicas` | Matemáticas | pregrado | 4.7 | español nativo, inglés C1 |
| `est_carlos_arquitectura` | Arquitectura | maestría | 3.6 | español nativo, inglés A2 |
| `est_ana_humanidades` | Humanidades | pregrado | 3.1 | español nativo, portugués B1 |

8 ofertas (4 internacionales + 4 nacionales): Montpellier, UNAM, Freie Universität Berlin, Stavanger, Escuela Colombiana de Ingeniería, EAFIT, UFPS, Tolima. Cada una tiene su combinación de fecha de cierre, área, nivel, cupos, idioma requerido, beca y convenio — diseñadas a propósito para disparar casos distintos (ver §7).

Algunos individuos se dejan **sin `rdf:type` explícito** a propósito (`pais_noruega`, `pais_colombia`, `uni_unal`) para demostrar que el razonador los tipa por dominio/rango (caso 2).

### 3.4 Razonamiento (verificado)

```
Triples antes de razonar : 292
Triples después de razonar: 609   (+317)
```

Los 3 casos exigidos por el enunciado se verifican con `assert`-like checks en la celda 5 y **los tres pasan**:
- **Caso 1 (jerarquía de clases):** una `OfertaInternacional` también es `Oferta`; un área de `Ingenieria` también es `AreaConocimiento`; una `UniversidadDestino` también es `Universidad`.
- **Caso 2 (dominio/rango):** `pais_noruega` resulta `Pais` (por rango de `ofreceEnPais`); `uni_unal` resulta `Universidad` (por dominio de `ubicadaEn`).
- **Caso 3 (subPropertyOf):** toda oferta ligada por `ofrecidaPorUniversidadDestino` queda también ligada por `ofrecidaPor`.

32 hechos nuevos relevantes del dominio (tipos + propiedades del contrato), muy por encima del mínimo de 4.

---

## 4. Lógica difusa (celda 8)

**Variables:** `promedio` (0–5, trapezoidal bajo/alto + triangular medio), `nivel_idioma` (0–100, 3 triangulares), `cupos` (0–10, 3 gaussianas), `idoneidad` (0–100, salida, 3 triangulares). Cubre los 3 tipos de función exigidos.

**Modificadores:** `muy` (concentración, `μ²`) sobre `promedio_alto`; `ligeramente` (dilatación, `μ^0.5`) sobre `cupos_pocos`.

**10 reglas** (R1–R10), con los 3 operadores (`&`=AND, `|`=OR, `~`=NOT). Defuzzificación por **centroide**.

**Funciones de integración con la ontología** (esto es lo que conecta difuso ↔ ontología ↔ experto):
- `ofertas_desde_grafo(g)` — lee del grafo razonado `tieneCupos`/`requiereIdioma`/tipo de cada `Oferta`.
- `perfil_desde_grafo(g, estudiante_id)` — lee `tienePromedio` y todos los `dominaIdioma` de un `Estudiante` de la ontología. Lanza `ValueError` si el estudiante no tiene `tienePromedio`.
- `interpretar_dominio(texto)` / `cumple_idioma(requisito, dominio)` — parsean el texto libre de idioma (ej. "Inglés B2", "Español nativo") y comparan contra la escala MCER (`_MCER`). `cumple_idioma` la usa el sistema experto directamente en la regla `r17` (ver §5).
- `resultados_para_experto(perfil, ofertas)` — corre la inferencia Mamdani para cada oferta y devuelve `{oferta, puntaje, categoria}`, listo para declarar como `ResultadoDifuso`.
- `graficar_todo()` — genera las 5 gráficas para el documento de diseño (`mf_promedio.png`, `mf_idioma.png`, `mf_cupos.png`, `mf_idoneidad.png`, `salida_centroide.png`); estos PNG y `ontologia.ttl` están en `.gitignore` porque son artefactos regenerables.

---

## 5. Sistema experto (Experta) — celdas 11–16

### 5.1 Compatibilidad

`experta` depende de `frozendict`, que usa `collections.Mapping` (retirado de Python desde 3.10). La celda 11 aplica el parche **antes** de `import experta`:
```python
for _name in ("Mapping", "MutableMapping", "Sequence", "Iterable", "Set"):
    if not hasattr(collections, _name):
        setattr(collections, _name, getattr(collections.abc, _name))
```
Sin esto, `import experta` falla con `AttributeError` en cualquier Colab o entorno local con Python ≥ 3.10. Verificado también en este repo con Python 3.14.

### 5.2 Clases de `Fact` (28 en total)

- **De clases de la ontología** (`CLASS_TO_FACT`, solo slot `id`): `Oferta`, `OfertaInternacional`, `OfertaNacional`, `Universidad`, `UniversidadDestino`, `Pais`, `AreaConocimiento`, `Ingenieria`, `NivelEstudios`, `Convenio`, `Beca`, `Estudiante`.
- **De propiedades** (`PREDICATE_TO_FACT`, slots tipados): `TieneCupos`, `FechaCierre`, `OfrecidaPor`, `OfrecidaPorUniversidadDestino`, `OfreceEnPais`, `TieneArea`, `DirigidaANivel`, `RestringeArea`, `OfreceBeca`, `RequiereIdioma`, `UbicadaEn`, `TieneBeca`, `AmparadaPorConvenio`, `EstudiaArea`, `CursaNivel`, `TienePromedio`, `DominaIdioma`.
- **De anotaciones** (`ANNOTATION_TO_FACT`): `Nombre` (de `foaf:name`, `dct:title`, `rdfs:label`), `Descripcion` (de `dct:description`).
- **Genéricas de esquema/cobertura total:** `EsInstanciaDe` (rdf:type hacia una clase `ex:` que NO está en `CLASS_TO_FACT`), `SubClaseDe` (jerarquía `rdfs:subClassOf` entre clases `ex:`), `Relacion` (cualquier predicado `ex:` que NO está en `PREDICATE_TO_FACT` — ver §5.3, esto es lo que hace el traductor verdaderamente "a prueba de futuro").
- **No ontológicas / de control:** `Hoy`, `Consulta`, `PerfilCandidato`, `ResultadoDifuso`, `CumpleIdioma`, `Descartada`, `Elegible`, `Procesada`, `Afinidad`, `Recomendacion`, `Advertencia`, `Respaldo`, `SugerenciaBeca`, `Nota`.

### 5.3 Traductor (celda 12) — cobertura total, no solo lo previsto

```python
CLASS_TO_FACT     = { EX.Oferta: Oferta, ... }              # 12 entradas
PREDICATE_TO_FACT = { EX.tieneCupos: (TieneCupos,"oferta","n"), ... }  # 16 entradas
ANNOTATION_TO_FACT= { FOAF.name: Nombre, DCTERMS.title: Nombre, ... }
```

`traducir(g, engine)` recorre **cada** triple de `g` (ya razonado) y decide en cascada:
1. `rdf:type` hacia una clase en `CLASS_TO_FACT` → Fact simple `Clase(id=...)`.
2. `rdf:type` hacia otra clase del dominio (`ex:`) no prevista → `EsInstanciaDe(id, clase)` (genérico).
3. predicado en `PREDICATE_TO_FACT` → Fact tipado con conversión XSD real (`valor_tipado`: integer/decimal/date/boolean/string).
4. predicado de anotación (`foaf:name`, `dct:title`, `rdfs:label`, `dct:description`) → `Nombre`/`Descripcion`.
5. `rdfs:subClassOf` entre clases `ex:` → `SubClaseDe`.
6. cualquier otro predicado del dominio (`ex:`) no previsto → `Relacion(sujeto, predicado, objeto)` genérico.
7. cualquier otra cosa (andamiaje RDFS: `rdfs:Resource`, `subClassOf` reflexivo, etc.) → se ignora.

**Por qué esto importa:** el paso 6 significa que si mañana agregan una propiedad nueva a la ontología SIN tocar `PREDICATE_TO_FACT`, el traductor no la descarta silenciosamente — la captura como `Relacion` genérica, y las reglas `r22_clase_nueva`/`r23_relacion_nueva` la reportan. Esto se probó explícitamente en la prueba obligatoria (§7.3) agregando `ex:ofreceAlojamiento` — una propiedad que **no existe** en `PREDICATE_TO_FACT` — y verificando que igual aparece como hecho.

Ningún nombre ni valor de un caso concreto está escrito en el traductor: los dos diccionarios mapean solo vocabulario (clases/predicados), tal como exige el enunciado.

### 5.4 Derivación del perfil del candidato

**El perfil NO se declara a mano.** El único dato de entrada manual es `Consulta(estudiante="est_juan_sistemas")` (o el id que sea). La regla `r00_perfil_desde_ontologia` (salience 200, la de mayor prioridad) deriva `PerfilCandidato(id, nivel_estudios, area)` a partir de los hechos que el traductor ya generó (`CursaNivel`, `EstudiaArea` del estudiante consultado). Si el estudiante no existe en la ontología, `r00b_estudiante_inexistente` lo reporta y el motor simplemente no genera ningún `PerfilCandidato` (probado en celda 15).

### 5.5 Motor de reglas — `SistemaExpertoIntercambios` (25 reglas)

**Niveles de salience usados:** 200 (derivar perfil), 110 (derivar cumplimiento de idioma), 100 (filtros duros de descarte), 50 (elegibilidad vía difuso + afinidad), 10 (generar recomendación), 5 (enriquecer con notas/avisos/becas sugeridas), 0 (reportes). Son más de los 3 niveles mínimos exigidos, agrupables conceptualmente en 3 fases: **derivar → filtrar/evaluar → generar salida**.

| Regla | Salience | Qué hace |
|---|---|---|
| `r00_perfil_desde_ontologia` | 200 | Deriva `PerfilCandidato` desde `CursaNivel`+`EstudiaArea` |
| `r00b_estudiante_inexistente` | 200 | Reporta si el id consultado no es un `Estudiante` |
| `r17_cumple_idioma` | 110 | Deriva `CumpleIdioma` comparando `RequiereIdioma` vs `DominaIdioma` (vía `cumple_idioma()` del módulo difuso) |
| `r01_fecha_vencida` | 100 | Descarta si `FechaCierre` < `Hoy` |
| `r02_sin_cupos` | 100 | Descarta si `TieneCupos` = 0 |
| `r03_nivel_no_ofrecido` | 100 | Descarta si el nivel del candidato no está en `DirigidaANivel` |
| `r04_area_restringida_general` | 100 | Descarta si `RestringeArea` ≠ área del candidato |
| `r05_area_restringida_exclusiva` | 100 | Igual que r04 pero cuando el área restringida es la ÚNICA área de la oferta (más específica) |
| `r18_idioma_insuficiente` | 100 | Descarta si hay `RequiereIdioma` y el candidato NO tiene `CumpleIdioma` |
| `r19_ubicacion_inconsistente` | 100 | Control de calidad: advierte si el país de `UbicadaEn` (universidad) y de `OfreceEnPais` (oferta) difieren |
| `r06_elegible_alta` / `r07_elegible_media` | 50 | Elegibilidad según `ResultadoDifuso.categoria` |
| `r08_idoneidad_baja_descarta` | 50 | Descarta si `categoria="baja"` |
| `r09_afinidad_area_exacta` | 50 | Si el área del candidato coincide exacto con `TieneArea` y la categoría era "media", sube a "alta" |
| `r20_afinidad_ingenieria` | 50 | Afinidad parcial cuando ambas áreas son subtipos de `Ingenieria` pero distintas |
| `r10_recomendacion_internacional` / `r11_recomendacion_nacional` | 10 | Generan `Recomendacion` con explicación (usa `Nombre` de universidad y país) |
| `r12_recomendacion_con_beca` | 10 | Recomendación con prioridad `"alta+beca"` cuando `OfreceBeca`+`TieneBeca` aplican |
| `r15_respaldo_convenio` | 5 | Agrega nota de respaldo si la oferta recomendada tiene `AmparadaPorConvenio` |
| `r16_advertencia_sin_convenio` | 5 | Avisa si la oferta recomendada NO tiene convenio registrado |
| `r14_sugerir_beca_general` | 5 | Sugiere una beca del catálogo para ofertas internacionales sin beca propia |
| `r21_nota_descripcion` | 5 | Añade la `Descripcion` de la oferta como nota |
| `r13_informe_descarte` | 0 | Imprime cada descarte con el nombre legible de la oferta |
| `r22_clase_nueva` / `r23_relacion_nueva` | 0 | Reportan cualquier `EsInstanciaDe`/`Relacion` genérica (cobertura total, ver §5.3) |

### 5.6 Los 3 mecanismos de resolución de conflictos — implementados en el motor, no simulados

`EstrategiaConflictos(DepthStrategy)` (celda 13) define la clave de orden de la agenda como una tupla:
```python
(salience, especificidad, recencia) = (activation.rule.salience, len(activation.rule), sorted(factids, reverse=True))
```
Esto es literal, no una aproximación: **salience** es el campo explícito de cada regla; **especificidad** es el número real de condiciones de la regla (`len(activation.rule)`); **recencia** es el orden descendente de los `factid` de los hechos que dispararon la regla (Python ordena tuplas lexicográficamente, así que desempata en cascada exactamente en ese orden).

`AgendaConTraza` registra cada disparo `(regla, salience, especificidad, recencia)` en `engine.agenda.traza`, lo que permite demostrarlo empíricamente. La celda 16 lo hace con `assert`, verificado en la ejecución real:
- **Salience:** `r05` (salience 100) gana sobre `r07` (salience 50) cuando ambas podrían aplicar.
- **Especificidad:** entre `r04` y `r05` (mismo salience 100), gana `r05` porque tiene 6 condiciones contra 5 de `r04` — `assert` confirma que solo `r05` dispara.
- **Recencia:** dos ofertas con `FechaCierre` vencida y mismo salience/especificidad — gana la que se declaró más recientemente (`of_nueva` antes que `of_vieja`, verificado por el orden de sus `factid` de salida).

**Control de bucles / duplicados (Lock-Fact):** dos variantes — el propio hecho `Descartada` actúa de candado en el nivel 100 (`NOT(Descartada(oferta=...))` en r02–r05, r18), y el hecho dedicado `Procesada` cumple ese rol en el nivel 50 (r06–r08). Sin esto, dos reglas del mismo nivel podrían declarar conclusiones contradictorias o duplicadas para la misma oferta.

---

## 6. Pipeline de ejecución (celda 14)

```python
g = Graph(); g.parse("ontologia.ttl", format="turtle")
DeductiveClosure(RDFS_Semantics).expand(g)
# ...
engine = SistemaExpertoIntercambios(); engine.reset()
stats = traducir(g, engine)                       # ontología -> Facts
engine.declare(Hoy(fecha=hoy))
engine.declare(Consulta(estudiante=estudiante_id)) # único dato manual
for r in resultados_para_experto(perfil_desde_grafo(g, estudiante_id), ofertas_desde_grafo(g)):
    engine.declare(ResultadoDifuso(**r))           # difuso -> Facts
engine.run()
reporte(engine)                                    # imprime Recomendaciones ordenadas
```

`reporte()` ordena las `Recomendacion` por prioridad (`alta+beca` > `alta` > `media`) y luego por puntaje difuso, y adjunta las notas/avisos/respaldos/afinidades/becas sugeridas de cada oferta.

### Escenarios de demo (con estudiantes reales de la ontología, verificados)

| Escenario | Estudiante | Hoy | Qué demuestra |
|---|---|---|---|
| A | `est_juan_sistemas` | 2026-08-01 | Flujo completo: descarta Montpellier (idioma) y Stavanger (área exclusiva); recomienda 6 ofertas, la mejor con beca |
| B | `est_juan_sistemas` | 2026-09-20 | Convocatorias vencidas (`r01`) descartan 3 ofertas adicionales |
| C | `est_carlos_arquitectura` (maestría) | 2026-08-01 | Nivel no ofrecido (`r03`) |
| D | `est_maria_matematicas` (matemáticas) | 2026-08-01 | La misma oferta Stavanger que se descartaba en A ahora se recomienda (el área sí coincide) |
| E | `est_ana_humanidades` | 2026-08-01 | Sin el idioma exigido (`r18`) y promedio bajo |

Resultado real del Escenario A: 305 hechos en memoria, 30 reglas disparadas, 6 recomendaciones (la primera con `alta+beca`).

---

## 7. Pruebas (celdas 15, 16, 18) — todas verificadas pasando

### 7.1 Unitarias (celda 15)
Con hechos sintéticos, porque las 8 ofertas reales no cubren todos los casos: `r02` (sin cupos), `r04` (área restringida no exclusiva), `r08` (idoneidad baja), `r19` (ubicación inconsistente), `r20` (afinidad entre ingenierías distintas), `r00b` (estudiante inexistente). Las 6 pasan.

### 7.2 Demostración de mecanismos (celda 16)
Ver §5.6 — 3 `assert` (uno por mecanismo) más una traza completa de los primeros 12 disparos del Escenario A.

### 7.3 Prueba obligatoria del traductor (celda 18)
Agrega, **solo en el archivo `.ttl`** (con `open(...).write()`, restaurado al final con `try/finally`):
```turtle
ex:oferta_prueba_2027_1 a ex:OfertaNacional ;
    ex:tieneCupos 3 ; ex:ofreceAlojamiento true ; ... .
```
Nótese `ex:ofreceAlojamiento`: una propiedad que **no existe** en `PREDICATE_TO_FACT`. Sin tocar ni una línea de Python, se verifica que:
- la oferta aparece como `Fact OfertaNacional`,
- la lógica difusa la evaluó (`ResultadoDifuso`),
- la propiedad desconocida se tradujo como `Relacion` genérica,
- las reglas la recomendaron (`Recomendacion`).

Las 4 comprobaciones dan `True` en la ejecución real.

---

## 8. Estado frente al enunciado (checklist)

| Requisito (Enunciado.MD §2) | Mínimo | Real | Estado |
|---|---|---|---|
| Clases de hechos (Experta) | 5 | 28 | ✅ |
| Reglas | 15 | 25 | ✅ |
| Niveles de salience | 3 | 7 valores en 3 fases | ✅ |
| Mecanismos de conflicto (los 3) | Salience, Specificity, Recency | Implementados en `EstrategiaConflictos`, demostrados con `assert` | ✅ |
| Mecanismo anti-bucle (No-Loop o Lock-Fact) | 1 | 2 (Descartada + Procesada) | ✅ |
| Base de hechos | 40 | 305+ (Escenario A) | ✅ |
| Variables difusas | 3 | 3 entradas + 1 salida | ✅ |
| Valores difusos por variable | 3 | 3 (+ 2 con modificador) | ✅ |
| Funciones trapezoidal/triangular/gaussiana | las 3 | las 3 | ✅ |
| Modificadores | 2 | `muy`, `ligeramente` | ✅ |
| Reglas difusas | 9 | 10, con AND/OR/NOT | ✅ |
| Defuzzificación justificada | sí | centroide | ✅ |
| Clases ontología | 10 | 12 | ✅ |
| subClassOf | 5 | 5 | ✅ |
| Propiedades con domain/range | 10 | 20 | ✅ |
| subPropertyOf | 1 | 1 | ✅ |
| Individuos por clase | 4 | ver riesgo abajo | ⚠️ |
| Razonamiento OWL-RL, 3 casos, ≥4 hechos nuevos | sí | 3 casos + 32 hechos nuevos | ✅ |
| Traductor dinámico (sin valores fijos) | sí | verificado con propiedad desconocida | ✅ |
| Caso de prueba obligatorio del traductor | sí | verificado | ✅ |
| Difuso alimenta al experto | sí | `ResultadoDifuso` | ✅ |
| Todos los hechos de la ontología llegan al experto | sí | `Relacion`/`EsInstanciaDe` genéricos | ✅ |

**⚠️ Riesgo pendiente de revisar:** algunas clases "contenedoras" (`Oferta`, `Universidad`) no tienen individuos con `rdf:type` **explícito** directo — solo lo adquieren por inferencia de sus subclases (`OfertaInternacional`/`OfertaNacional`, `UniversidadDestino`). Si el criterio de evaluación cuenta instancias explícitas por clase antes de razonar, conviene mencionarlo en el documento de diseño como decisión de diseño (se demuestra el caso 1 de razonamiento precisamente así).

**Pendiente fuera del código:** documento de diseño (PDF) y video pitch (§3 y §4 del enunciado) — no existen archivos para esto en el repo todavía.

---

## 9. Convenciones para quien edite esto después

- **Un solo `.ttl` canónico** (celda 4, `%%writefile`). El razonador nunca lo modifica en disco. La celda 3 es solo el generador inicial, no se re-ejecuta como parte del flujo normal salvo para regenerar la ontología desde cero.
- **Coherencia clase/propiedad ↔ traductor:** gracias a la cobertura genérica (`EsInstanciaDe`, `Relacion`, `SubClaseDe`), agregar una clase o propiedad nueva al `.ttl` **no rompe** el traductor — como mínimo aparece como hecho genérico. Para que una regla la use explícitamente sí hay que agregarla a `CLASS_TO_FACT`/`PREDICATE_TO_FACT` y escribir la regla.
- **El perfil del candidato siempre se deriva de la ontología** (vía `Consulta` + `r00`). No declarar `PerfilCandidato` a mano salvo en pruebas unitarias aisladas (celda 15).
- **Archivos gitignorados** (regenerables, no se versionan): `ontologia.ttl`, `mf_promedio.png`, `mf_idioma.png`, `mf_cupos.png`, `mf_idoneidad.png`, `salida_centroide.png`.
- **Instalación de dependencias:** todas las celdas de instalación usan `%pip` (no `!pip`), para instalar siempre en el entorno real del kernel. Nombres de paquete vs. import que no coinciden: `scikit-fuzzy` (pip) / `skfuzzy` (import).
- **Python ≥ 3.10:** `experta` requiere el parche de `collections.Mapping` de la celda 11 antes de importarse; no quitarlo.
