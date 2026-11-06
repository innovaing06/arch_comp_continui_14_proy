# Revisión del plan maestro v2.3 + adiciones necesarias

## Respuesta rápida a tu pregunta sobre selección automática de API/modelo

**Sí está mencionada** — Sección 6 ("Modo AUTO") describe descubrimiento, evaluación de capacidades, cuotas, autenticación y elegibilidad económica, con selección por tarea.

**Pero es binaria:** el plan solo contempla `FREE_ONLY` vs "nada". **No contempla tu caso** de "API de pago ya pagada con créditos que quiero usar como una más". Esa es la **primera inconsistencia real** que hay que resolver.

---

## Adiciones al plan maestro v2.3

### 🧬 ADICIÓN 1 — ADN del producto

**Dónde:** Sección 1 ("Definición del producto"), **nuevo apartado 1.1** — debajo de "El producto terminado debe permitir..." y **antes** de Sección 2.

**Contenido a agregar:**

```
### 1.1 ADN del producto (invariantes no negociables)

Estos principios prevalecen sobre cualquier conveniencia de velocidad. Cualquier cambio que los contradiga debe ser rechazado explícitamente por el operador.

1. **Fail-closed**: si un artefacto no pasa sus gates (pedagógico, técnico, comercial), el curso no se publica ni se empaqueta como "vendible". No hay fallback silencioso ni relleno determinista.

2. **FROZEN 8/8**: ocho archivos son intocables salvo autorización expresa y justificada. Cualquier cambio requiere: (a) motivación documentada, (b) nuevo hash autorizado, (c) actualización de pines en tests, (d) revisión del integrador.

3. **El modelo genera; AICF verifica y decide**: ningún LLM tiene autoridad final sobre gates, presupuesto, elegibilidad ni release. Su output es propuesta, no decisión.

4. **Wikipedia prohibida**: research usa fuentes primarias o secundarias verificables. Wikipedia queda excluida sin excepción.

5. **V32-6 = 0.5**: peso histórico de esta rúbrica. No se modifica sin auditoría específica.

6. **Política económica configurable, no dogmática** (ver Sección 6.4): el desarrollo opera en `FREE_ONLY`. El operador puede activar `FREE_PLUS_PAID_CONFIGURED` (con presupuesto explícito) o `PAID_ALLOWED` (post-monetización). Nunca hay fallback silencioso free → paid.

7. **Reutilización sobre reconstrucción**: si un componente existe y pasa sus pruebas, se extiende. No se reconstruye sin evidencia de insuficiencia.

8. **Trazabilidad de identidad**: todo artefacto (brief, objectives, rubric, curriculum, lesson, assessment, variant) tiene identidad canónica, versión, hash y predecesores. Nada se reescribe sin rastro.

9. **Multi-país = contenido, no estructura** (ver Sección 4.7): un mismo diseño pedagógico produce variantes nacionales; la estructura se reutiliza; solo el contenido se adapta.

10. **Protocolo de sincronización Z.ai ↔ Windows** (Sección 11): ningún agente escribe sobre Windows sin huella verificable de entrada/salida.
```

---

### 🌎 ADICIÓN 2 — Generación multi-país (template-first)

#### 2.1 Nueva Sección 4.7

**Dónde:** Sección 4, **nuevo apartado 4.7** — justo **después de 4.6** (Vista previa y aprobación de currículo) y **antes** de Sección 5.

**Contenido a agregar:**

```
### 4.7 Generación multi-país: plantilla fuente y variantes por país

El operador puede seleccionar **uno o varios países** para un mismo curso. La generación multi-país NO crea N cursos independientes; produce **una plantilla fuente** y **N−1 variantes** derivadas de ella.

#### 4.7.1 Modelo conceptual

- **Plantilla fuente**: curso generado para el primer país de la lista (`sourceCountry = countries[0]`). Es la referencia canónica de estructura (módulos, lecciones, objetivos, rúbrica pedagógica, plan de evaluaciones).
- **Variante por país**: curso derivado que reutiliza íntegramente la estructura de la plantilla fuente y adapta únicamente el contenido a la realidad del país destino.

#### 4.7.2 Qué se reutiliza (estructura — INTOCABLE en la variante)

- Objetivo general y resultados de aprendizaje (misma redacción pedagógica; pueden añadir "en el contexto de [país]")
- Cantidad, nombres y orden de módulos
- Cantidad y títulos de lecciones
- IDs estables (`moduleId`, `lessonId`, `objectiveId`)
- Rúbrica pedagógica de logro (criterios, niveles, descriptores)
- Plan de evaluaciones y evidencias

#### 4.7.3 Qué se adapta (contenido — específico del país)

- **Moneda**: PEN, CLP, ARS, BRL, MXN, COP, etc. Cifras, precios, salarios y montos de ejemplo.
- **Marco legal y regulatorio**: SUNAT (Perú), SII (Chile), ARCA/AFIP (Argentina), Receita Federal (Brasil), DIAN (Colombia), etc.
- **Casos prácticos y ejemplos**: empresas, personas y situaciones locales verosímiles.
- **Datos y estadísticas**: INEI, INE, INDEC, IBGE, DANE o fuentes equivalentes.
- **Referencias culturales**: festividades, calendario fiscal, convenciones locales.
- **Idioma/regionalismos** cuando `locale` varíe.
- **Recomendación de pricing comercial**: precio sugerido por país y moneda local.
- **Activos visuales contextualizados**.

#### 4.7.4 Qué NO se adapta sin autorización

- Número de módulos o lecciones.
- Objetivos de aprendizaje (más allá del sufijo contextual).
- Estructura de la rúbrica pedagógica.
- Orden pedagógico.
- Tipos, cantidad o pesos de evaluaciones.

#### 4.7.5 Flujo operativo

1. Operador configura `countries[]` en el brief. **`countries[0]` es el `sourceCountry`**.
2. Pipeline genera el **curso fuente completo** para `countries[0]` (dashboard → research → currículo → lecciones → assessment → resources → QA → HTML5 → release), con aprobación del Curriculum Blueprint (4.6) y elegibilidad económica (6.4).
3. El curso fuente aprobado se congela como **plantilla**: hash + versión + referencia canónica.
4. Para cada `countries[i>0]`:
   a. Job variante (`variantOf: sourceJobId`, `variantCountry: XX`, `sourceVersion: N`).
   b. Carga la plantilla congelada (módulos, lecciones, objetivos, rúbrica).
   c. Etapa de **adaptación de contenido** por país (research local + adaptación, sin re-descomponer la estructura).
   d. Re-ejecuta assessments y QA **contra la estructura reutilizada**.
   e. Valida coincidencia 1-a-1 con la plantilla (`moduleId`, `lessonId`, `objectiveId`).
   f. Persiste la variante con `sourceJobId` + `sourceVersion`.
5. Cada variante puede fallar independientemente. **El fallo de una variante NO bloquea la plantilla ya publicada.**

#### 4.7.6 Idempotencia, reanudación y obsolescencia

- Cada job variante es idempotente por `(sourceJobId, sourceVersion, variantCountry)`.
- Si se regenera la plantilla fuente (nueva versión), las variantes existentes quedan **stale** y se marcan; no se actualizan automáticamente.
- La reanudación de una variante reutiliza los mecanismos de StageJournal (WP6).

#### 4.7.7 Pricing y comercialización por país

El precio y la moneda de comercialización son específicos por país y forman parte del artefacto de variante. `PremiumRubric16` se evalúa por variante: un curso puede ser vendible en Perú y no en Argentina, y ese juicio debe ser trazable. El operador puede rechazar una variante sin afectar a la plantilla ni a otras variantes.
```

#### 2.2 Extender Sección 5 (Contrato conceptual)

**Dónde:** Sección 5, dentro del bloque "Contrato conceptual mínimo" — **debajo** de la línea existente `CourseBrief → CourseDesignSpec → CurriculumBlueprint → GenerationPlan → artefactos.`

**Agregar esta línea:**

```
**Extensión multi-país**: `CourseBrief` incluye `countries: string[]` (≥1) y `sourceCountry = countries[0]`. El `GenerationPlan` produce un `SourceCoursePlan` (para `sourceCountry`) y N−1 `VariantPlans` (uno por `countries[i>0]`). Cada `VariantPlan` referencia `sourceJobId` + `sourceVersion` + `variantCountry`. Antes de crear nuevos tipos, Z.ai debe evaluar si `CourseJobRequest`, `CourseContracts` o `CurriculumSpecification` pueden ampliarse sin duplicar estructura.
```

#### 2.3 Nueva WP8 en Sección 10

**Dónde:** Sección 10 (WP0–WP7), **insertar WP8 después de WP7** y **antes** de Section 11.

**Contenido a agregar:**

```
### WP8 — Variantes multi-país (template-first)

**Objetivo:** generar variantes nacionales de un curso aprobado sin duplicar la estructura pedagógica.

**Entregas:**
- Campo `countries[]` + `sourceCountry` en el brief; validación de ≥1 país.
- Job variante con `variantOf` + `variantCountry` + `sourceVersion`.
- Etapa de adaptación de contenido por país (currency, legal, casos, datos, cultural, pricing).
- Validador de coincidencia estructural (mismos `moduleId`, `lessonId`, `objectiveId` que la plantilla).
- Detección de **"adaptación fantasma"** (contenido genérico no localizado) como gate duro.
- Persistencia durable e idempotencia por `(sourceJobId, sourceVersion, variantCountry)`.
- Detección de plantilla stale cuando la fuente se regenera.
- UI mínima: selector multi-país + preview de variantes.
- Tests: estructura idéntica, contenido diferenciado, idempotencia, fallo aislado de una variante, detección de adaptación fantasma.

**Exclusiones:** no re-descomponer módulos/lecciones; no regenerar objetivos; no tocar la rúbrica pedagógica. Si el operador necesita una estructura distinta por país, crea un curso nuevo (no una variante).

**Gate G8:** cada variante respeta la estructura de la plantilla; el contenido es específico del país con evidencia concreta (moneda local, entidad legal local, casos locales); el fallo de una variante no bloquea la publicación de la plantilla; cada variante es trazable a `sourceVersion`; no hay "adaptación fantasma" verificable por detección de términos/valores del país fuente en la variante.
```

#### 2.4 Extender Sección 13 (Riesgos)

**Dónde:** Sección 13 (tabla), **añadir al final de la tabla**:

```
| Variante con estructura divergente | Validador obligatorio: `moduleId`/`lessonId`/`objectiveId` idénticos a la plantilla; bloqueo G8 si difiere. |
| "Adaptación fantasma" (contenido del país fuente reutilizado sin localizar) | Detección automática de términos y valores del país fuente; gate duro si aparecen en la variante. |
| Variante bloquea publicación de la plantilla | Desacoplamiento: la plantilla se publica en cuanto su propio gate pasa; las variantes son jobs independientes. |
| Plantilla regenerada sin invalidar variantes | `sourceVersion` + `stale` marker; el operador debe revisar y re-aprobar variantes o aceptar explícitamente la obsolescencia. |
| Pricing comercial inadecuado por país | `PremiumRubric16` + pricing recomendado por país; el operador puede rechazar una variante sin afectar a las demás. |
| Multi-país confundido con tier de audiencia | Tiers (4.4) seleccionan estructura por audiencia/edad; multi-país adapta contenido. Son ortogonales; documentarlo explícitamente. |
```

#### 2.5 Extender Sección 14 (DoD)

**Dónde:** Sección 14 (lista), **añadir después** del criterio "El curso supera validaciones pedagógicas, técnicas y comerciales.":

```
- Un curso multi-país produce una plantilla fuente congelada y N−1 variantes que reutilizan exactamente su estructura.
- Cada variante está localizada (moneda, legal, casos, datos, cultural, pricing) y verificada por el gate G8.
- El fallo de una variante no bloquea la publicación de la plantilla.
- Cada variante es trazable a la versión específica de la plantilla que la originó.
- El operador puede rechazar una variante sin afectar a la plantilla ni a otras variantes.
```

---

### 💰 ADICIÓN 3 — Modo económico configurable (free + paid)

#### 3.1 Actualizar Sección 0, regla 6

**Dónde:** Sección 0, **reemplazar la regla 6 completa** (actualmente: *"Ambos modos deben respetar FREE_ONLY. Una selección manual no autoriza un proveedor de pago."*).

**Reemplazar por:**

```
6. Ambos modos (AUTO y MANUAL) respetan el **modo económico activo** (ver Sección 6.4): `FREE_ONLY` / `FREE_PLUS_PAID_CONFIGURED` / `PAID_ALLOWED`. Una selección manual nunca autoriza por sí sola un proveedor fuera del modo activo ni eleva el presupuesto aprobado.
```

#### 3.2 Nueva Sección 6.4

**Dónde:** Sección 6, **nuevo apartado 6.4** — **después** del bloque "Catálogo híbrido".

**Contenido a agregar:**

```
### 6.4 Modos de elegibilidad económica

AICF opera en uno de tres modos, seleccionable por el operador y auditable en telemetría. El modo afecta **qué proveedores son elegibles**, no cómo se les llama.

| Modo | Comportamiento | Caso de uso |
|---|---|---|
| **FREE_ONLY** (activo por defecto) | Solo proveedores con elegibilidad gratuita efectiva verificada. Intento de usar proveedor de pago → bloqueo con `PROVIDER_NOT_ELIGIBLE`. | Desarrollo y operación sin coste. |
| **FREE_PLUS_PAID_CONFIGURED** | Proveedores gratuitos se prefieren siempre. Proveedores de pago son elegibles **solo si**: (a) el operador ha añadido explícitamente la credencial al `.env`, (b) ha configurado presupuesto de pago (`AICF_PAID_BUDGET_JOB` y `AICF_PAID_BUDGET_MONTH`), (c) ha activado explícitamente el modo (`AICF_ECONOMIC_MODE=FREE_PLUS_PAID_CONFIGURED`). Cada llamada de pago consume del presupuesto y queda auditada. | Operador tiene API de pago o créditos y quiere usarlos junto con los gratuitos. |
| **PAID_ALLOWED** | Pago permitido sin autorización por job; sigue requiriendo presupuesto configurado. Solo se activa con monetización demostrada y decisión explícita del operador. | Operación comercial post-monetización. |

**Invariantes:**
- El modo activo se registra en `CourseJobRequest` y telemetría. Cambios de modo son eventos auditables.
- `AUTO` y `MANUAL` respetan el modo. `MANUAL` puede seleccionar proveedor de pago **solo si** el modo lo permite y el presupuesto lo autoriza.
- **Nunca hay fallback silencioso free → paid.** El uso de pago es siempre explícito, presupuestado y trazable.
- El presupuesto de pago es **separado** del presupuesto de llamadas (F1) y del presupuesto de tokens (WP4).
- La telemetría (F2) etiqueta cada llamada con `economic_tier: free | paid_configured | paid_unrestricted`.
- El cambio de modo es una operación exclusiva del operador — no de Z.ai, la UI ni la telemetría.
```

#### 3.3 Extender Sección 7 (Presupuesto)

**Dónde:** Sección 7, **añadir al final** (después de "El límite debe ser validado contra el deployment real...").

**Contenido a agregar:**

```
### 7.1 Presupuesto en modo FREE_PLUS_PAID_CONFIGURED

En modo `FREE_PLUS_PAID_CONFIGURED`, el presupuesto se compone de tres capas independientes:

- **Presupuesto de llamadas lógicas** (F1): 200 llamadas lógicas/job. Aplica a todas las llamadas.
- **Presupuesto de tokens** (WP4): por job, por solicitud, por modelo. Aplica a todas las llamadas.
- **Presupuesto de pago explícito** (nuevo): en unidades monetarias, por job y por mes. Se consume SOLO por llamadas a proveedores de pago.

**Regla de bloqueo:** un job no puede iniciar una llamada de pago si `paid_budget_used + estimated_cost > paid_budget_remaining`. La estimación es conservadora; la reconciliación tras respuesta ajusta al consumo real.

**Invariante:** el presupuesto de pago NUNCA se eleva automáticamente por un job, un retry ni un fallback. Solo el operador puede incrementarlo, y cada incremento es un evento auditable.
```

#### 3.4 Extender Sección 13 (Riesgos)

**Dónde:** Sección 13, **añadir al final de la tabla**:

```
| Uso silencioso de pago en FREE_PLUS_PAID_CONFIGURED | Cada llamada de pago etiquetada en telemetría con `economic_tier: paid_configured`; ledger de pago independiente del ledger de llamadas y de tokens; sin fallback silencioso free → paid. |
| Presupuesto de pago elevado por retry o fallback | Presupuesto inmutable por runtime; solo el operador lo cambia; cambio auditable. |
| Cambio de modo no auditado | Cada cambio de `AICF_ECONOMIC_MODE` se registra en telemetría con timestamp, operador y motivo. |
| Credencial de pago presente sin modo configurado | Al detectar `*_API_KEY` de proveedor de pago con `AICF_ECONOMIC_MODE=FREE_ONLY` → fail-closed con `PAID_KEY_FORBIDDEN_IN_FREE_ONLY`. |
```

#### 3.5 Extender Sección 14 (DoD)

**Dónde:** Sección 14, **añadir después** del criterio "AUTO y MANUAL funcionan sin fallback pagado.":

```
- En modo `FREE_PLUS_PAID_CONFIGURED`: cada llamada de pago es explícita, presupuestada por separado, etiquetada en telemetría y bloqueada si el presupuesto se agota. Sin fallback silencioso free → paid.
- En modo `PAID_ALLOWED`: la misma exigencia de presupuesto y trazabilidad aplica; el modo solo se activa con decisión explícita del operador.
```

#### 3.6 Actualizar Sección 12, regla 12

**Dónde:** Sección 12, **reemplazar la regla 12 completa** (actualmente: *"No ejecutar inferencias reales ni consumir cuotas hasta que G3/G4 y la elegibilidad gratuita estén demostrados."*).

**Reemplazar por:**

```
12. No ejecutar inferencias reales ni consumir cuotas hasta que: (a) G3/G4 estén aceptados, (b) el modo económico esté configurado explícitamente, (c) el presupuesto correspondiente al modo (gratuito o de pago) esté aprobado para el job.
```

#### 3.7 Actualizar WP3 en Sección 10

**Dónde:** Sección 10, WP3.

**Reemplazar el título de WP3 por:**

```
### WP3 — Catálogo híbrido y selección AUTO/MANUAL con modo económico configurable
```

**Añadir a las entregas de WP3:**

```
- Implementar los tres modos económicos (ver 6.4): FREE_ONLY / FREE_PLUS_PAID_CONFIGURED / PAID_ALLOWED.
- Detección de `*_API_KEY` de proveedores de pago vs gratuitos; clasificación por tier.
- Bloqueo fail-closed de `*_API_KEY` de pago en modo FREE_ONLY.
```

**Reemplazar el gate G3 por:**

```
**Gate G3:** modelos no elegibles nunca reciben llamadas; un pin manual se respeta o bloquea; AUTO solo selecciona candidatos elegibles; los modelos descubiertos se muestran con evidencia y estado; en `FREE_PLUS_PAID_CONFIGURED`, ninguna llamada de pago ocurre sin presupuesto aprobado y telemetría etiquetada; en `FREE_ONLY`, ninguna credencial de pago es usada aunque esté presente.
```

---

## 🔍 Observaciones e inconsistencias a revisar

### 🔴 Críticas (bloquean coherencia con tu intención)

1. **Regla 6 (Sección 0) es binaria** — Solo `FREE_ONLY`. No contempla tu caso de "API de pago ya pagada". Debe reemplazarse (Adición 3.1).

2. **Sección 3, fila "Pool y FREE_ONLY: BLOCKED por discrepancia económica"** — Correcto **hoy**, pero con `FREE_PLUS_PAID_CONFIGURED` el "BLOCKED" cambia de naturaleza: ya no es un bloqueo económico, es una configuración intencional. **Añadir nota al pie en la celda:** *"El estado BLOCKED aplica al modo FREE_ONLY. En FREE_PLUS_PAID_CONFIGURED con presupuesto explícito, el mismo pool deja de estar bloqueado y pasa a ser consumible bajo trazabilidad (ver 6.4)."*

3. **Sección 4.1 dice "país" (singular)** — Debe ser `countries: string[]` con `sourceCountry = countries[0]`. Ya cubierto en Adición 2.

4. **Sección 10 (WP plan) no tiene WP para multi-país** — Añadir WP8 (Adición 2.3).

5. **Sección 12 regla 12** — Bloquea hasta "gratuidad demostrada". En el futuro `FREE_PLUS_PAID_CONFIGURED`, el criterio debe ser "modo económico configurado" (Adición 3.6).

6. **Sección 6.3 dice "Guard central FREE_ONLY común a ambos modos"** — Debe ser "Guard central **económico** (3 modos)" (Adición 3.2).

### 🟡 Importantes (evitan confusión futura)

7. **Sección 4.4 (tiers)** — Los tiers son por **audiencia/edad/duración**, NO por país. **Es ortogonal a multi-país.** Añadir una línea al final de 4.4: *"Los tiers de esta sección seleccionan estructura por audiencia/edad/duración. No seleccionan por país. La selección de país afecta solo al contenido (ver 4.7)."*

8. **Sección 5 (contrato)** — No incluye extensión multi-país. Cubierto por Adición 2.2.

9. **Sección 6.1 "Modo AUTO"** — Dice que evalúa "elegibilidad económica" pero no especifica **cómo** se determina. Con 3 modos, la elegibilidad es función del modo activo, no una propiedad intrínseca del proveedor. Añadir: *"La elegibilidad económica se evalúa contra el modo activo (ver 6.4); no es una propiedad intrínseca del proveedor."*

10. **Sección 13 (riesgos) no incluye** — Ni multi-país ni modos económicos. Cubierto por Adiciones 2.4 y 3.4.

11. **Sección 14 (DoD) no incluye** — Ni multi-país ni modos económicos. Cubierto por Adiciones 2.5 y 3.5.

### 🟢 Menores (cosméticos o de claridad)

12. **Título del documento dice "v2.2"** pero el archivo es `V2_3` y Sección 16 dice "Cambios v2.2 → v2.3". Actualizar el título a **v2.3** (probablemente leftover del rename).

13. **Sección 0, regla 9** dice "Z.ai programa únicamente el alcance autorizado". Reforzar añadiendo: *"Cambios de modo económico o de presupuesto de pago son exclusivos del operador, no de Z.ai, la UI ni la telemetría."*

14. **Sección 5 (contrato)** — La distinción entre rúbrica pedagógica y `PremiumRubric16` es correcta en 4.3, pero la UI debería exponerlas por separado. Añadir nota: *"El Dashboard no expone un único botón 'Generar rúbrica'; expone por separado 'Rúbrica de logro' (pedagógica) y 'Evaluación premium' (comercial)."*

15. **Sección 3 (tabla) dice "Objetivos, Bloom y validación curricular: IMPLEMENTED parcialmente"** — Correcto, pero con multi-país hay que asegurar que los objetivos y rúbrica sean **reutilizados** en variantes (no regenerados). Añadir una nota: *"En variantes multi-país (4.7), los objetivos y la rúbrica se heredan de la plantilla, no se regeneran."*

---

## Resumen ejecutivo

| Adición | Dónde | Impacto |
|---|---|---|
| **ADN** | Sección 1.1 (nueva) | Fija invariantes permanentes |
| **Multi-país** | Sección 4.7 (nueva) + 5 (extender) + WP8 (nueva) + 13 (riesgos) + 14 (DoD) | Habilita template-first por país |
| **3 modos económicos** | Sección 0 regla 6 (reemplazar) + 6.4 (nueva) + 7.1 (extender) + WP3 (extender) + 12 regla 12 (reemplazar) + 13 (riesgos) + 14 (DoD) | Permite usar API de pago configurada junto con free |

**Observaciones críticas:** 6 (todas bloquean coherencia con tu intención).
**Observaciones importantes:** 5 (evitan confusión futura).
**Observaciones menores:** 4 (cosméticos o de claridad).

Con estas adiciones, el plan queda alineado con:
- Tu ADN (fail-closed, FROZEN, US$0 configurable, multi-país como contenido)
- Tu intención de usar API de pago futura con control explícito
- Tu modelo de "plantilla Perú → variantes país"
- Los avances ya cerrados (F1, F2, R28, F2C, P3-B1/B2)

**No se reescribe nada ya cerrado.** Todas las adiciones son **extensivas** — no tocan F1, F2, R28, F2C, P3-B1/B2 ni WPs ya aceptados.