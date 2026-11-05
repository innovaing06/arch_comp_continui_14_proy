# 14 — AUDITORÍA COMPARATIVA: AICF v3 vs 14 PROYECTOS OPEN SOURCE

**Fecha:** 2026-10-31 (Lima) · **Modo:** SOLO LECTURA — sin commits, sin ediciones a AICF, sin llamadas a proveedores reales, sin `/health`, sin cambios de configuración.
**Base auditada:** copia Z.ai `/tmp/my-project` (post-5E-R4, sin `.git`). Hashes verificados: `ContentGenerationService.ts` = `d6d5bad7…`, `route.ts` = `3d146a11…`, `LessonGenerationFailureDiagnostics.ts` = `df271f18…`, `FailedJobReleaseRecord.ts` = `7adaacd7…`, `ContentGenerator.ts` = `af851377…` (excepción Lote 5 intacta). **8/8 archivos protegidos intactos** (7 FROZEN + ContentGenerator).
**Referencia:** `AICF_Investigacion_Rescate_OpenSource_Cierre.zip` — 187 extractos fuente, 980 bloques verbatim verificados, licencias 14/14 verificadas (ver `08_CIERRE`). LiteLLM verificado contra el tag real `v1.84.0` (ver `12_MATRIZ` §2-bis).

---

## 0. RESUMEN EJECUTIVO

| Área | Veredicto de la comparativa | Evidencia clave |
|---|---|---|
| **P1 Integridad de curso (fail-closed)** | ✅ **AICF v3 POST-R3/R4 IGUALA O SUPERA a las referencias.** El anti-patrón nº1 de los 14 proyectos (fallback permisivo: StudyForge lo resuelve, QuizWeaver/Curriculum/OpenShip lo codifican como bug) está ELIMINADO en AICF desde 5E-R3: catch → curso `FAILED`, plantilla nunca cuenta como contenido, `no_lessons_generated` aborta el curso. | §2.1; `CourseWorker` clasificador; tests P1 5E-R3 |
| **P2 Persistencia de diagnóstico** | ⚠️ **AICF implementó en R4 los dos helpers correctos, pero con DOS bugs de ubicación/visibilidad** (no de escritura): bug A (directorio huérfano por doble identidad courseId) y bug B (registro invisible por API + payload null por diseño). Las referencias (StudyForge run_id, LangGraph pending_writes, Promptfoo fila ERROR legible) demuestran el patrón correcto: **la clave de identidad se crea ANTES del trabajo y el error queda legible por el MISMO canal que el éxito.** | §2.2; doc `16_DIAGNOSTICO` completo |
| **P3 Saturación de proveedores** | ❌ **AICF 1.84.0 en configuración actual NO tiene ningún control de presupuesto por deployment activo.** La cadena causal del 429 Groq/Mistral está confirmada de extremo a extremo contra el código real de 1.84.0: `least-busy` solo cuenta peticiones en vuelo → `ModelRateLimitingCheck` existe pero está **OPT-IN y desactivado** → sin `tpm`/`rpm` por deployment → sin pre-flight ITPM/OTPM (el archivo `io_token_rate_limit_check.py` NO existe en 1.84.0). Las referencias LiteLLM 1.109 + vLLM/SGLang/Ollama demuestran el mecanismo que falta. | §2.3; `12_MATRIZ` §2-bis (verificación directa) |
| **P4 Reintentos y enrutado** | 🟡 **AICF acierta la política (UNA capa de reintentos, LOTE 5B) pero la semántica de 1.84.0 tiene un matiz peligroso que AICF no controla:** el router reintenta INSTANTÁNEAMENTE (return 0) tras un 429 si hay otros deployments sanos → ráfaga contra el proveedor. `classifyErrorOrigin` del adaptador es el análogo AICF de `RateLimitErrorCategory` (que NO existe en 1.84.0), con limitación honesta de snippet 200 chars. | §2.4; `12_MATRIZ` §2-bis fila `_time_to_sleep_before_retry` |
| **P5 Calidad y tests reproducibles** | ⚠️ **AICF tiene fail-closed pero no tiene capa de pre-validación determinista de evaluaciones ni suite CI de prompts.** QuizWeaver `pre_validate_questions` (validación estructural por tipo ANTES del juez, sin LLM) y Promptfoo (fila ERROR ≠ PASS, AJV estricto, finish_reason, JUnit, exit codes) son los mecanismos a adaptar. | §2.5; extractos QuizWeaver/Promptfoo |

**Conclusión general:** AICF v3 post-R4 ya resolvió el problema más difícil (P1: fail-closed con integridad, donde 3 de los 14 proyectos de referencia fallan) y ya escribió los helpers correctos para P2 (con dos bugs de última milla diagnosticados en doc 16). Lo que AICF NO tiene — y las referencias demuestran que es viable y de bajo riesgo — es: control de cuota por deployment activado (P3, cambio de CONFIGURACIÓN sin código), evidencia durable por llamada (LlmCallRepository), pre-validación determinista de evaluaciones y suite CI de prompts (P5), y reanudación selectiva de etapas (FUTURA).

---

## 1. MÉTODO Y NIVELES DE EVIDENCIA

1. Cada afirmación sobre AICF se verificó contra la copia post-5E-R4 con `archivo:línea` (los archivos R4 citados por línea correspondecen a los hashes verificados arriba).
2. Cada afirmación sobre LiteLLM se hace contra la versión **instalada 1.84.0** (verificación directa del tag oficial, `12_MATRIZ` §2-bis), no contra el snapshot 1.109.0 de los extractos. Donde un mecanismo solo existe en 1.109.0 se marca explícitamente.
3. Etiquetas: **[VERIFICADO]** leído en código real; **[INFERIDO]** razonado sin lectura directa; **[AUSENTE]** dato no disponible en este entorno.
4. Los extractos ZIP citan `proyecto/archivo:líneas` del upstream original (bloques verbatim verificados 980/980). Esta auditoría NO copia ninguno de esos bloques a AICF; usa los mecanismos como evidencia para diseñar soluciones originales.

---

## 2. COMPARATIVA POR PROBLEMA (P1–P5)

### 2.1 P1 — Integridad de curso: ¿una lección no generada puede llegar al producto?

| Proyecto | Mecanismo (cita exacta ZIP) | AICF v3 actual |
|---|---|---|
| **QuizWeaver** | ❌ **Anti-patrón codificado:** `src/agents.py:735-743` — si el juez LLM lanza excepción, se acepta el borrador pre-validado; `:792-802` — agotados reintentos, se devuelve el último borrador sin veredicto (`extractos …/QuizWeaver/generation/02_anti_patron_fallback_permisivo.md`) | ✅ **AICF elimina ambas ramas** (5E-R3 P1): el catch de lección devuelve `FAILED` incondicionalmente; el worker aborta el curso con `no_lessons_generated`; una plantilla NUNCA se registra como contenido generado. Test de regresión invertido correspondiente en las suites P1. |
| **Curriculum** | ❌ **Anti-patrón codificado (y consagrado en tests):** `src/generate/formats/questions.py:307-328, 334-335` — tras agotar reintentos con `critical_issues` pendientes, el contenido inválido se devuelve como válido; `tests/test_content_generators.py:245-262` lo consagra (`extractos …/Curriculum/question_generation/01_generate_questions_FAIL_OPEN.md`, `tests/01_test_fail_open_codificado.md`) | ✅ AICF no tiene esta rama: el error de generación se propaga; no hay camino "aceptar el borrador tras N intentos". |
| **OpenShip** | ❌ **Anti-patrón:** `services/llm.py:1103-1125` — contenido rechazado por el juez dos veces se publica igual ("quality signal, not a hard gate"; "Better mediocre content than no content"); `:845-855` pass-through de resultado parcial con warning (`extractos …/OpenShip/antipatrones/`) | ✅ En AICF la etapa de release es `BLOCKED` para cursos fallidos (5E-R4-F: se omite toda la fase de release); el gate premium es decisión dura, no señal. |
| **StudyForge** | ✅ **Patrón correcto de referencia:** `backend/app/generation.py:547-611` — si ninguna lección sobrevive → `raise ValueError` (sin curso vacío); `:485-523` — salida del LLM es input no confiable: ítems malformados se descartan, no se reparan (`extractos …/StudyForge/error_handling/01,04`) | ✅ **AICF alcanza el mismo contrato** con distinta implementación (TS, fail-closed en el worker + P1 B byte-idéntico del mensaje). Paridad verificada. |
| **PydanticAI** | ✅ **Patrón superior de evidencia:** `pydantic_ai/_fallback.py:186-206` `raise_fallback_exception_group` — al agotar la cadena se lanza un grupo con TODAS las causas; el éxito viaja con `failed_attempts` (`:253-257` `AttemptLog.attach`; `tests/models/test_fallback.py:2570-2605`) (`extractos …/PydanticAI/provider_fallbacks/`) | 🟡 **PARCIAL:** AICF registra la causa del fallo en `lesson-generation-failure.json` (R4) con transportClass/origen/proveedor, pero **no agrupa TODOS los intentos** (el registro es por lección fallida terminal, no por cadena de proveedores; LiteLLM es quien hace el failover interno). El análogo AICF del `failed_attempts` aún no existe. Ver §3 gap G-5. |
| **Instructor** | ✅ **Patrón terminal:** taxonomía de errores con estados terminales (`retry/02_error_taxonomy_exhaustion.md`); truncamiento = `IncompleteOutputException` (`truncation/01`) | ✅ AICF clasifica truncamiento (`finish_reason` capturado en `LiteLLMAdapter:705-763` + `truncated_output` en la taxonomía del worker) y no repara salida malformada. |

**Veredicto P1: AICF está EN PARIDAD O MEJOR que las referencias.** Ninguna de las tres rutas fail-open documentadas en los proyectos de referencia existe en AICF v3. El mecanismo restante de alto valor es el **grupo de causas agregadas** (PydanticAI) — ver doc 15, solución S-6.

### 2.2 P2 — Persistencia del diagnóstico: ¿la causa del fallo sobrevive al proceso?

| Proyecto | Mecanismo (cita exacta ZIP) | AICF v3 actual |
|---|---|---|
| **StudyForge** | ✅ Clave de identidad creada ANTES del trabajo: `metering.py:494-527` `run_id` pre-creado; toda fila `LlmCall` cuelga de él; el run fallido es VISIBLE agrupado (`main.py:2188-2266`, `course_id` NULL = run fallido) (`extractos …/StudyForge/persistence/01,02,04`) | ❌ **ESTE ES EXACTAMENTE EL BUG A** (doc 16 §1): el registro R4 se construye con `spec.courseId` (inventado por el LLM en `CurriculumPlanner.normalizeSpec`) mientras el curso real vive en `job.courseId` (creado antes por `CourseJobStore.create:88`). La escritura funciona (tmp+rename, mkdir); el DESTINO es un directorio huérfano. **AICF viola el patrón run_id de StudyForge en la costura worker→servicio.** |
| **LangGraph** | ✅ El error de la tarea queda escrito en el checkpoint JUNTO a las salidas: `pregel/_runner.py:575-615` — writes parciales + `(ERROR, exc)` persistidos al fallir; `SqliteSaver.put` con `INSERT OR IGNORE` idempotente (`extractos …/LangGraph/persistence/03`, `checkpointing/04`) | ❌ **Análogo AICF del gap:** el diagnóstico se escribe en disco pero (a) en directorio equivocado (bug A), y (b) `release.json` BLOCKED existe pero es INVISIBLE por API porque el endpoint exige `manifest.json` que jamás se escribe para `no_lessons_generated` (bug B, doc 16 §2). El error NO queda "junto a las salidas" en el canal de lectura estándar. |
| **Promptfoo** | ✅ La fila de error es persistente y LEGIBLE: `evaluator.ts:1990-2035` + `runEval.test.ts:726-762` — error de proveedor = `failureReason: ERROR`, nunca pass/skip; JUnit XML legible por CI (`output.test.ts:863-982`) (`extractos …/Promptfoo/providers/02`, `tests/03`) | ❌ **Bug B viola la legibilidad:** el registro BLOCKED lleva el payload de evaluación TODO-null por diseño y su única causa es el string `job_status_failed:no_lessons_generated` — un reasonCode, no una causa raíz. La causa raíz (transportClass, proveedor, HTTP, Retry-After) vive en `lesson-generation-failure.json`… que por el bug A no está en el curso. **Doble pérdida de legibilidad: exactamente lo que Promptfoo prohíbe.** |
| **OpenShip** | 🟡 Referencia parcial (persistencia durable del log de uso: `persistence/02_durabilidad_registro.md`) pero con gaps propios: solo registra éxitos, sin `duration_ms`, sin `status/errorCategory` (`INDEX_PARCIAL_OpenShip.md` síntesis) | 🟡 AICF R4 registra la llamada fallida (mejor que OpenShip) pero SOLO la fallida de lección; no hay fila por llamada exitosa, ni coste, ni duración. Ver §3 gap G-4. |
| **LearnHouse** | ✅ Puerta de publicación con versionado (`publishing/01-04`, `tests/01-03`); el hallazgo negativo ("curso sin revisión publicado") es el anti-patrón que el toggle evita (`publishing/05_hallazgo_negativo_sin_revision.md`) | ✅ AICF es MÁS estricto: para `status=failed` R4-F omite release completo y escribe revisión BLOCKED audit-only; el toggle de publicación no relaja gates. Paridad/superioridad local, PERO el registro es invisible (bug B). |

**Veredicto P2: los helpers AICF son correctos y las referencias confirman que la fractura es de IDENTIDAD y VISIBILIDAD, no de escritura** (demostrado con ejecución aislada en doc 16 §2.2). Las soluciones sin tocar `CourseWorker.ts` están diseñadas en doc 16 §1.5/§2.6 con contratos de test T-A1..A4 / T-B1..B6.

### 2.3 P3 — Saturación de proveedores: ¿el sistema sabía que la petición excedía la cuota?

Este es el área con MAYOR brecha verificada. La cadena causal del incidente (Groq TPM 8000, uso 7977 + petición 478; OTPM 1000, uso 992 + 200; Mistral HTTP 429 código 1300 al mismo milisegundo) queda confirmada de extremo a extremo contra el código real de 1.84.0 (`12_MATRIZ` §2-bis):

| Mecanismo de control | En 1.84.0 (instalado) [VERIFICADO] | En 1.109.0 (extracto) | Estado AICF (config V30 LOTE 5A) |
|---|---|---|---|
| **Routing con conocimiento de tokens** | `usage-based-routing-v2` SÍ existe (`lowest_tpm_rpm_v2.py:32`) pero requiere `tpm`/`rpm` por deployment | `lowest_tpm_rpm_v2.py:385-389` descarta deployments sobre límite | ❌ `routing_strategy: least-busy` — `least_busy.py:43-51,71-79` de 1.84.0 cuenta SOLO peticiones en vuelo ±1, sin tokens/rpm. **El router no sabía que la petición excedía el TPM.** |
| **Pre-check TPM/RPM por deployment** | `ModelRateLimitingCheck` SÍ existe pero **OPT-IN**: `enable_pre_call_checks` default False (`router.py:282`) y requiere `optional_pre_call_checks=["enforce_model_rate_limits"]` (`router.py:1701`); cuenta `total_tokens` post-éxito por minuto (`model_rate_limit_check.py:246-290`), pre-check `current_tpm >= tpm_limit` (`:115-118`); **sin separación ITPM/OTPM, sin reserva de salida** | `model_rate_limit_check.py:126-581` con reserva/reconcile/refund | ❌ **NO activado**: la config no declara `tpm`/`rpm` en ningún deployment ni `enable_pre_call_checks`. El mecanismo existe y está apagado. |
| **Pre-flight ITPM/OTPM con reserva** | ❌ **NO EXISTE** — `io_token_rate_limit_check.py` da 404 en el tag v1.84.0 | `io_token_rate_limit_check.py:515-873` reserva antes de llamar; `_rate_limit_error` (`:440-458`); reserva mínima 1 token (`:427-458`) | ❌ No existe en la librería instalada; AICF no tiene equivalente nativo. **El 429 de Groq era estructuralmente inevitable.** |
| **Origen del 429 (vendor vs local)** | ❌ `RateLimitErrorCategory` NO existe — `exceptions.py:323` solo `RateLimitError(openai.RateLimitError)` sin categoría; local y vendor lanzan LA MISMA clase | `exceptions.py:25-116, 513-598` con default engañoso `category=VENDOR_RATE_LIMIT` (`:538`) | 🟡 AICF construyó su propio clasificador: `classifyErrorOrigin` (`LiteLLMAdapter.ts:271-304`, 4 orígenes: `router_cooldown`/`upstream_provider`/`litellm_gateway`/`unknown`) + `classifyTransportError` (8 clases, `:341-380`) + `detectQuotaExhaustion` (`:390-404`, mensual vs por-minuto). **AICF ya hace en TS lo que la librería no ofrece** — con la limitación honesta del snippet ≤200 chars → `unknown`. |
| **Cooldown aislado por deployment** | ✅ `cooldown_cache.py:109-110` — clave `"deployment:"+model_id+":cooldown"` | `CooldownCache` rico en contexto | 🟡 Existe en la librería y es correcto; AICF además mantiene un gate cliente por baseUrl (`LiteLLMAdapter:411-420`) con Retry-After respetado y tope 60 s = 2× `cooldown_time` 30 s de la config. |
| **Admisión/concurrencia (referencias locales)** | — | vLLM `check_admission` + cola con prioridad (`admission_control/01-03`), SGLang `MaxRunningRequests` (`limits_concurrency/01`), Ollama `num_parallel` + cola + eviction (`concurrency/01-03`) | 🟡 **AICF tiene el análogo del lado cliente:** `LLMGovernor` (singleton: concurrencia máx 5, token bucket 10/2-s, presupuesto de reintentos 20/60-s, circuit breaker solo-429, espera acotada 5 s) y `FailoverTracker` (máquina HEALTHY→COOLDOWN→RECOVERY_PROBE por proveedor, 60 s→600 s, QUOTA 300 s, AUTH sin auto-recovery — capa de OBSERVACIÓN: "AICF observes; LiteLLM executes"). Lo que falta es la contraparte del lado ROUTER: presupuesto por deployment. |

**Veredicto P3:** el control de cuota por deployment es una **mitigación de CONFIGURACIÓN** (activar `ModelRateLimitingCheck` + `tpm`/`rpm` + considerar `usage-based-routing-v2`), no de código — pero está bajo la protección del mandato (el `litellm-config.yaml` no se toca en esta fase). La reserva ITPM/OTPM nativa NO es posible en 1.84.0 sin módulo propio; el diseño honesto de dónde puede vivir cada capa está en doc `17_ESTRATEGIA` (el adaptador TS NO puede reservar por deployment porque el ROUTING ocurre en el proxy — reservar allí sería fingir conocimiento que no se tiene).

### 2.4 P4 — Reintentos y enrutado: ¿una sola capa, sin multiplicación, con Retry-After?

| Proyecto | Mecanismo (cita exacta ZIP) | AICF v3 actual |
|---|---|---|
| **LiteLLM** | Router como capa única de reintento: `router.py:9305-9517` `async_function_with_retries` (1.109); 1.84.0 [VERIFICADO] respeta Retry-After (`router.py:6361-6394`) con prioridad `deployment_cooldown > header_cooldown > default` (`:6621-6625`) — **PERO devuelve 0 (reintento instantáneo) si hay otros deployments sanos** | ✅🟡 AICF fijó la política correcta (LOTE 5B): el adaptador emite EXACTAMENTE UN intento HTTP por `chat()` — "LiteLLM is the SOLE transport-retry layer" (`LiteLLMAdapter.ts:316-338`); elimina la multiplicación 9×3 del incidente course-mvudo8fd-o7gi. **Riesgo residual no controlado por AICF:** el return-0 de 1.84.0 puede convertir un 429 en ráfaga instantánea entre deployments; con 3 deployments y `num_retries: 2`, una llamada lógica produce hasta 3 intentos upstream. Es el comportamiento de la librería; AICF solo lo puede frenar con gate cliente (que ya abre tras el 429 TERMINAL) o con cooldowns de config. |
| **PydanticAI** | Orden explícito fallback vs retry: `retry/03_orden_fallback_vs_retry.md` — reintentos DENTRO del proveedor antes de pasar al siguiente; registro de intentos (`provider_fallbacks/03_intent_history_instrumentation.md`) | ✅ AICF delega ambos al router (fallbacks entre deployments del alias + `num_retries`) y mantiene el suyo cliente (gate) — una sola capa efectiva por responsabilidad. La instrumentación de intentos por llamada es el gap G-4 (LlmCallRepository). |
| **Instructor** | Reask con inyección del error en el contexto (`validation/02_reask_message_injection.md`), agotamiento terminal con taxonomía (`retry/02`) | ✅🟡 AICF reserva la regeneración SOLO para errores de contenido/validación (LOTE 5C: `classifyTransportError` → cualquier `AIAdapterError` se propaga SIN regeneración de contenido). El reask por inyección de error NO existe (FUTURA/Opcional — ver doc 15 S-8): hoy un fallo de schema mata la lección (fail-closed) en lugar de reintentar con contexto. |
| **LangGraph** | ⚠️ Advertencia negativa: `default_retry_on` reintenta 5xx PERO NO 429 (`types.py:427-446`, `_internal/_retry.py:1-29`) — el gap que AICF debe evitar | ✅ En AICF el 429 SÍ se reintenta (en el router, `num_retries: 2`) y el gate cliente aplica backoff 2s·2^(n-1) tope 60 s con jitter ±25 % tras 429 terminal (`computeCooldownGateMs`, `LiteLLMAdapter.ts:427-439`). AICF NO hereda el gap de LangGraph. |
| **¿Gemini fue realmente usado como fallback?** | — | **[AUSENTE]** El extracto del incidente solo muestra errores Groq y Mistral (429 a las 19:12:37.621; Mistral 1300 a 19:12:37.597). Con `least-busy` la selección es por carga en vuelo, no por `order` — **sin evidencia de intento a Gemini no se afirma ni se descarta** (5E-R4 §1.1). Con las 3 llamadas upstream por llamada lógica, la ausencia de errores de Gemini en el log no prueba que no se intentara: pudo servir sin error o no ser seleccionada. **Dato que cerraría la pregunta:** `x-litellm-model-id` por llamada en un log durable (gap G-4). |

**Veredicto P4:** la arquitectura de reintentos de AICF es correcta y coincide con el consenso de las referencias (una capa, Retry-After respetado, sin multiplicación). El punto ciego restante es la **observabilidad por llamada** (qué deployment atendió cada intento) y el **matiz return-0** de 1.84.0.

### 2.5 P5 — Calidad y tests reproducibles

| Proyecto | Mecanismo (cita exacta ZIP) | AICF v3 actual |
|---|---|---|
| **QuizWeaver** | ✅ **Pre-validación determinista por tipo ANTES del juez:** `critic_validation.py:1-293` — `pre_validate_questions`, validadores por tipo (`_check_mc`, `_check_sa`…), 25 tests reales (`tests/test_critic_validation.py:1-344`); fail-closed en persistencia con estados (`quiz_generator.py:27-206`) | ❌ **GAP:** AICF aplica presupuesto y fail-closed al batching MCQ (Lote 5C) pero NO valida estructura por tipo antes del juez → errores burdos consumen tokens del juez. La lección de QuizWeaver: "saves tokens on obvious structural failures". Propuesta original: doc 15 S-7 (`AssessmentPreValidator` con `objectiveId` obligatorio — que QuizWeaver no tiene y AICF sí exige). |
| **Promptfoo** | ✅ Suite CI de prompts: error de proveedor = fila ERROR nunca PASS (`runEval.test.ts:726-762`); `finish_reason:'length'` = fallo score 0 (`finishReason.test.ts:18-27`); AJV estricto (`json.ts:8-63`); JUnit XML sin fugas (`output.test.ts:863-982`); exit codes (`doEval.ts:1270-1298`); estabilidad por repetición con intervalo de Wilson (`evaluations/03`) | ❌ **GAP:** AICF no tiene suite de regresión de prompts. Cualquier cambio de prompt puede introducir truncamiento o schema-inválido sin detección previa. El adaptador YA captura `finish_reason` y usage por llamada (`LiteLLMAdapter.ts:705-763`) — la base de datos existe; falta la suite. Propuesta: doc 15 S-5. |
| **Outlines** | ✅ Conformidad por construcción (`structured_generation/06_conformidad_por_construccion.md`, logits masking) | ❌ **DESCARTADA para esta fase:** el masking de logits requiere acceso a los pesos/decoding del modelo — imposible vía API de Groq/Gemini/Mistral. La transferencia útil es el PRINCIPIO (validar forma antes de pagar contenido), ya cubierto por S-7. |
| **vLLM/SGLang** (guiado y métricas) | vLLM guided outputs eliminados por complejidad (`guided_outputs/01_removed_guided_fields.md`), métricas Prometheus (`metrics/01`) | 🟡 Referencia negativa útil: vLLM RETIRÓ sus campos guiados por complejidad — refuerza que AICF no intente guiado fino vía API. |
| **StudyForge** (evals) | ✅ `prompt_fingerprint` (sha256 del prompt), grounding/answerability (`evals/metrics.py:268-441`), source_coverage (`:530-584`), veredicto por trials (`evaluations/04`) | ❌ **GAP parcial:** AICF mide los bytes EXACTOS del prompt en el diagnóstico R4 (mejor que nada disponible en las referencias para el fallo), pero no tiene fingerprint por llamada ni métricas de grounding. Se integra en S-4/S-5. |

---

## 3. GAPS DE AICF CONFIRMADOS POR LA COMPARATIVA (síntesis para doc 15)

| # | Gap AICF | Referencia que lo resuelve (cita) | Dónde vive la solución |
|---|---|---|---|
| G-1 | Diagnóstico de lección en directorio huérfano (bug A) | StudyForge `run_id` pre-creado (`metering.py:494-527`) | `ContentGenerationService.ts` catch — resolver `job.courseId` vía `CourseJobStore.get(jobId)` (doc 16 §1.5) |
| G-2 | Registro BLOCKED invisible por API + payload null (bug B) | LangGraph `(ERROR, exc)` junto a salidas (`_runner.py:575-615`); Promptfoo fila ERROR legible | `route.ts` (manifest de fallo) + `FailedJobReleaseRecord.ts` (enriquecer evento) (doc 16 §2.6) |
| G-3 | Sin presupuesto por deployment activado | LiteLLM `ModelRateLimitingCheck` OPT-IN (1.84.0 `router.py:1701,282`) + `usage-based-routing-v2` (`lowest_tpm_rpm_v2.py:32`) | `litellm-config.yaml` — SOLO con autorización (doc 17 §3) |
| G-4 | Sin evidencia durable POR LLAMADA (éxitos incluidos): sin tokens, coste, duración, deployment que atendió | StudyForge `LlmCall` (`models.py:158-190`) + `MeteredLLM` (`metering.py:80-164`); OpenShip `LlmUsageLog` + pricing snapshots; corrección de los gaps de OpenShip (fallidas registradas, `errorCategory` enum, null≠0) | Nuevo `LlmCallRepository` — el adaptador YA captura `x-litellm-model-id`, usage y `finish_reason` por llamada (`LiteLLMAdapter.ts:173-182,705-763`): la atribución post-hoc del deployment REAL es honesta y ya existe |
| G-5 | Sin grupo de causas agregadas por cadena de proveedores | PydanticAI `raise_fallback_exception_group` (`_fallback.py:186-206`) | Evolución de R4-diagnostics cuando el operador la autorice |
| G-6 | Sin pre-validación determinista de evaluaciones | QuizWeaver `pre_validate_questions` (`critic_validation.py:1-293`) | Nuevo `AssessmentPreValidator` (doc 15 S-7) |
| G-7 | Sin suite CI de prompts | Promptfoo (4 reglas inalienables, JUnit, exit codes) | Nuevo `PromptEvaluationSuite` (doc 15 S-5) |
| G-8 | Reanudación selectiva de etapas (re-pago de etapas 1..N-1 al reintentar) | LangGraph journal + `INSERT OR IGNORE` + test `one.calls == 1` (`test_pregel.py:975-976`) | FUTURA (`StageJournal`) — sin ella, un reintento de job repite research+outline pagados |
| G-9 | `usage-based-routing-v2` no controla OTPM ni reserva salida | LiteLLM 1.109 `io_token_rate_limit_check.py:515-873` [AUSENTE en 1.84.0] | Sin solución nativa en 1.84.0; ver doc 17 §5 (proxy callback custom = FUTURA con riesgo medio-alto) |
| G-10 | Tabla de precios/models fresca retiró modelos Groq que pudiera consultar AICF | `12_MATRIZ` §2-bis.1 (`groq/llama-3.3-70b-versatile` retirado del main actual; AICF usa `groq/qwen/qwen3.8-27b` en config V30) | Verificación de 1 minuto por el operador; pin de entradas si se integra tabla fresca |

---

## 4. AUDITORÍA DE ANTI-PATRONES (¿AICF tiene alguno de los 10 documentados?)

| # | Anti-patrón (proyecto) | ¿Presente en AICF v3? | Evidencia |
|---|---|---|---|
| 1 | Fallback permisivo del crítico (QuizWeaver `agents.py:735-743`) | ✅ NO | 5E-R3: catch → `FAILED`; tests P1 lo custodian |
| 2 | Último borrador sin aprobar (QuizWeaver `:792-802`) | ✅ NO | `no_lessons_generated` aborta el curso completo |
| 3 | Proveedor mock por defecto (QuizWeaver `llm_provider.py:370-413`) | ✅ NO | El adaptador productivo es `LiteLLMAdapter` (proxy real); mocks solo en tests, nunca en config por defecto |
| 4 | Fail-open del contenido (Curriculum `questions.py:307-328`) | ✅ NO | No existe rama "aceptar tras agotar reintentos" |
| 5 | Pass-through de resultado parcial (OpenShip `llm.py:845-855`) | ✅ NO | R3: lección fallida → curso fallido; sin publicación parcial silenciosa |
| 6 | Pass-through de rechazo del juez (OpenShip `llm.py:1103-1125`) | ✅ NO | R4-F: `status=failed` omite release entero; gate premium es duro |
| 7 | 429 local etiquetado como vendor (LiteLLM 1.109 `exceptions.py:538`) | 🟡 PARCIALMENTE mitigado | `classifyErrorOrigin` distingue `router_cooldown`/`litellm_gateway`/`upstream_provider`/`unknown` por inspección (la categoría NO existe en 1.84.0) — pero si se activa `ModelRateLimitingCheck` (doc 17), su 429 local (`"Model rate limit exceeded. TPM limit=…"`, `model_rate_limit_check.py:117-144`) NO está hoy en los patrones del clasificador → caería en `unknown` o `litellm_gateway` por "crossed/has hit". **Acción menor requerida al activar pre-checks: añadir el patrón exacto.** |
| 8 | Retry que no reintenta 429 (LangGraph `_retry.py:1-29`) | ✅ NO heredado | Router reintenta 429 (`num_retries: 2`); gate cliente con backoff acotado |
| 9 | Registro de uso solo en éxito (OpenShip síntesis INDEX) | 🟡 MITIGADO a medias | R4 registra la llamada fallida de lección (mejor que OpenShip); PERO éxitos y otras etapas no dejan fila durable → gap G-4 |
| 10 | Coste 0 en lugar de null (OpenShip agregadores) | 🟡 N/A hoy — riesgo latente | AICF no calcula coste; cuando implemente G-4 debe tipar `costUsd: number \| null` desde el diseño (doc 15 S-4) |

---

## 5. LO QUE AICF v3 YA HACE MEJOR QUE LAS REFERENCIAS (honestidad de la comparativa)

1. **Taxonomía de origen de error en TS sin soporte de la librería** — 1.84.0 no tiene `RateLimitErrorCategory`; AICF construyó `classifyErrorOrigin`/`classifyTransportError`/`detectQuotaExhaustion` con la regla "nunca adivinar el proveedor" (`LiteLLMAdapter.ts:260-304` — el caso de evidencia insuficiente cae honestamente en `unknown`). Ninguno de los 14 proyectos audita esta distinción con la misma disciplina en un stack de proxy.
2. **Detección de cuota mensual vs por-minuto** (`detectQuotaExhaustion`) — clasifica la cuota agotada (Cohere trial) como NO-TRANSIENTE por encima del status code; OpenShip (que sufrió el caso real) no lo distingue.
3. **Medición de bytes EXACTOS del prompt en el diagnóstico** (R4) — las referencias estiman tokens; AICF mide bytes reales y declara la estimación de tokens como aproximada (`approx-chars/4`), sin fingir precisión de tokenizer inexistente en runtime.
4. **Gate cliente acotado y derivado de la config real** — tope 60 s = 2× `cooldown_time` 30 s del `litellm-config.yaml` (documentado en el código como derivado, no arbitrario) con jitter ±25 %.
5. **Protección post-failure completa** (R4-F) — ninguno de los proyectos de referencia con pipeline de publicación (LearnHouse, OpenShip) corta la fase de release para jobs fallidos con la misma sistemática.
6. **Fail-closed de curso en el worker** — StudyForge lo logra en `generation.py`; AICF además garantiza que una plantilla no puede colarse como contenido (contrato P1 B byte-idéntico verificado en R3/R4).

---

## 6. HECHOS VERIFICADOS vs INFERENCIAS vs DATOS AUSENTES

**[VERIFICADO] contra código real:** estado P1 (catch→FAILED, R4-F, taxonomía del adaptador, governor, failover tracker); los dos bugs de persistencia (doc 16, con ejecución aislada del helper B); LiteLLM 1.84.0 (ausencia de pre-flight ITPM/OTPM y de categoría de error; existencia OPT-IN de `ModelRateLimitingCheck`, de `usage-based-routing-v2` y del cooldown por deployment; Retry-After respetado con matiz return-0); config V30 (least-busy, 3 deployments, sin tpm/rpm); captura de `x-litellm-model-id` y usage por llamada en el adaptador.

**[INFERIDO]:** que el 429 del incidente fue saturación real acumulada del minuto (TPM) y no límite por-request (el prompt medido ~2,814 tokens no agota TPM 8000 por sí solo) — coherente con la evidencia pero sin cuerpo de error completo.

**[AUSENTE]:** cuerpo/headers completos del 429 del incidente (no se clasifica OTPM vs TPM vs RPM específico ni el significado exacto de Mistral 1300); `Retry-After` real del incidente; evidencia de intento a Gemini; `.env` real del operador (posible `AICF_COURSE_DATA_DIR` — verificación de 1 minuto en doc 16 §2.4); comportamiento en staging de los cambios de config propuestos.

---

## 7. LICENCIAS Y ATRIBUCIÓN

- Esta auditoría NO copia código de ningún proyecto a AICF. Los mecanismos citados funcionan como EVIDENCIA para diseños originales en TypeScript (docs 15/16/17).
- Licencias verificadas 14/14 contra upstream (`08_CIERRE` §2): MIT (LiteLLM, Instructor, LangGraph, PydanticAI, QuizWeaver, OpenShip, StudyForge parcial, Promptfoo, Outlines, Ollama resuelto MIT), Apache-2.0 (vLLM, SGLang), y las de LearnHouse/StudyForge según sus repos. **`Curriculum` está bajo CC BY-NC-ND 4.0 — VETADO para incorporación**; su único uso aquí es como evidencia de anti-patrón (test de regresión invertido), sin copiar una línea.
- Toda implementación derivada requiere: verificación del archivo `LICENSE` real upstream, autorización explícita del operador, y para archivos FROZEN o `ContentGenerator.ts` la autorización específica ya prevista por el proceso AICF.

**Fin del documento 14. → Continúa en `15_PLAN_PRIORIZADO_DE_SOLUCIONES.md` (matriz priorizada) y `17_ESTRATEGIA_PROVEEDORES_Y_CUOTAS.md` (control de proveedores).**
