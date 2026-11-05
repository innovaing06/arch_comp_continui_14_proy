# 17 — ESTRATEGIA DE PROVEEDORES Y CUOTAS (AICF v3 — SOLO DISEÑO)

**Fecha:** 2026-10-31 (Lima) · **Modo:** SOLO LECTURA — nada de lo aquí propuesto se aplicó. `litellm-config.yaml` INTACTO (hash de la config V30 LOTE 5A sin cambios). Sin llamadas a proveedores, sin `/health`, sin consumo de cuota.
**Base de hechos:** LiteLLM **1.84.0** verificado contra el tag oficial (`12_MATRIZ` §2-bis, evidencia `archivo:línea`); config AICF V30 (LOTE 5A) leída directamente; incidente 5E-R4 (Groq TPM 8000 con uso 7977 + petición 478; OTPM 1000 con uso 992 + 200; Mistral HTTP 429 código 1300 en el mismo milisegundo).

---

## 0. RESUMEN EJECUTIVO

| Pregunta | Respuesta corta |
|---|---|
| ¿Puede AICF evitar el 429 sin cambiar LiteLLM de versión? | **SÍ, con configuración:** `ModelRateLimitingCheck` existe en 1.84.0 y está APAGADO. Activarlo + declarar `tpm`/`rpm` por deployment es la mitigación inmediata (Opción 3). No cubre OTPM (ver limitaciones). |
| ¿Puede el adaptador TS reservar tokens por deployment? | **NO honestamente.** El routing ocurre DENTRO del proxy; el adaptador no sabe qué deployment atenderá hasta que la respuesta vuelve. Cualquier "reserva por deployment" en TS sería fingir conocimiento. Lo que SÍ puede hacer el lado AICF: presupuesto a nivel ALIAS (global), clasificación post-hoc vía `x-litellm-model-id`, y gate cliente. |
| ¿Qué opción de routing conviene? | **Opción 3 primero** (pre-call checks, riesgo bajo, reversible), **evaluando la Opción 2** (`usage-based-routing-v2`, disponible en 1.84.0). La Opción 4 (módulo nativo AICF) es el diseño de fondo para lo que la librería no da. La Opción 5 (alias por proveedor) solo si se necesita presupuesto por proveedor desde AICF. |
| ¿Cuál es la regla de oro de reintentos? | **UNA capa de reintentos por responsabilidad** (ya fijada en LOTE 5B). Peligro específico de 1.84.0: tras un 429 el router devuelve `time=0` (reintento INSTANTÁNEO) si hay otros deployments sanos → ráfaga. Contramedidas: gate cliente existente + cooldowns de config. |
| ¿Qué NO se puede saber hoy? | Qué deployment atendió cada intento del incidente (log efímero, sin fila durable → gap G-4/S-4); el significado exacto de Mistral 1300; el `Retry-After` real del incidente. Todo queda registrado como AUSENTE, no adivinado. |

---

## 1. ESTADO ACTUAL VERIFICADO (config V30 LOTE 5A + código 1.84.0)

### 1.1 Pool de deployments (3 funcionales)

| # | Deployment (litellm_params.model) | order/weight | Cuota declarada en config | Capacidades (comentario de config) |
|---|---|---|---|---|
| 1 | `groq/qwen/qwen3.8-27b` | order 1, weight 1 (implícito) | **NINGUNA** (sin `tpm`/`rpm`) | chat, structured_output, json_mode, tool_calls, vision, streaming |
| 2 | `gemini/gemini-3.8-flash` | order 2, **weight 0.5** (mitigación 503) | **NINGUNA** | + long_context |
| 3 | `mistral/mistral-small-latest` | order 3 | **NINGUNA** | + embeddings |

Cohere (`cohere/command-a-03-2025`) está FUERA DE SERVICIO desde LOTE 5A (clave trial: 1000 llamadas/mes, 20/min — evidencia course-mvudo8fd-o7gi). Restaurable solo con clave no-trial.

### 1.2 Router

| Parámetro | Valor | Hecho verificado en 1.84.0 |
|---|---|---|
| `routing_strategy` | `least-busy` | Cuenta SOLO peticiones en vuelo ±1 (`least_busy.py:43-51,71-79`); **NO mide tokens ni rpm** → no sabía que la petición excedía TPM |
| `num_retries` | 2 (hasta 3 intentos upstream por llamada lógica) | Retry-After SÍ respetado (`router.py:6361-6394`); prioridad `deployment_cooldown > header_cooldown > default` (`:6621-6625`); **PERO** `_time_to_sleep_before_retry` devuelve 0 si hay deployments sanos → ráfaga instantánea |
| `cooldown_time` | 30 s | Cooldown POR DEPLOYMENT (clave `"deployment:"+model_id`, `cooldown_cache.py:109-110`) — el alias NO comparte cooldown ✓ |
| `allowed_fails` | 5 | Umbral de salud por deployment |
| `enable_pre_call_checks` | **AUSENTE (default False)** | `router.py:282` — `ModelRateLimitingCheck` OPT-IN: solo se registra con `optional_pre_call_checks=["enforce_model_rate_limits"]` (`router.py:1701`) |
| Pre-flight ITPM/OTPM | **NO EXISTE en 1.84.0** | `io_token_rate_limit_check.py` → 404 en el tag. En 1.109.0 existe (`:515-873`) con reserva/reconcile/refund |

### 1.3 Lo que el lado AICF YA tiene (verificado, no requiere cambio)

- **Clasificación de origen** (`classifyErrorOrigin`, LiteLLMAdapter:271-304): `router_cooldown` / `upstream_provider` / `litellm_gateway` / `unknown` — el análogo AICF de `RateLimitErrorCategory` (que NO existe en 1.84.0; en 1.109 tiene default engañoso VENDOR, `exceptions.py:538`).
- **Clases de transporte** (`classifyTransportError`, :341-380) + **cuota mensual vs por-minuto** (`detectQuotaExhaustion`, :390-404).
- **Gate cliente por baseUrl** (:411-439): Retry-After respetado (tope 60 s = 2× cooldown_time, derivado de la config real), backoff 2s·2^(n-1) jitter ±25 %, gated calls SIN HTTP y con evidencia honesta 429.
- **UNA capa de reintentos** (LOTE 5B): el adaptador emite exactamente 1 intento HTTP por `chat()`.
- **Captura por llamada** (:173-182, :705-763): `x-litellm-model-id` (deployment REAL que atendió), `x-litellm-call-id`, headers `retry-after`/`x-ratelimit-*`/`llm_provider-*`, `finish_reason`, usage (prompt/completion/total tokens), snippet ≤200 chars. **Todo lo necesario para atribución post-hoc está ya capturado — falta persistirlo (S-4).**

---

## 2. LAS CUOTAS DEL INCIDENTE, CLARIFICADAS (TPM ≠ OTPM ≠ ITPM ≠ RPM)

| Dimensión | Definición operativa | Dato del incidente | ¿Quién la controla en AICF 1.84.0? |
|---|---|---|---|
| **TPM** | Tokens totales (in+out) por minuto | Groq: límite 8000, uso 7977, petición 478 → la petición NO cabía en el minuto | `ModelRateLimitingCheck` (OPT-IN, cuenta `total_tokens` post-éxito, `model_rate_limit_check.py:246-290`); `usage-based-routing-v2` filtra deployments sobre tpm |
| **OTPM** | Tokens de SALIDA por minuto | Groq: límite 1000, uso 992, petición 200 → excedida | **NADIE** — 1.84.0 no separa I/O (el archivo io_token_rate_limit_check.py no existe); ni pre-checks ni routing v2 la cubren. Solo un módulo AICF (Opción 4) o upgrade (descartado, S-11) |
| **ITPM** | Tokens de ENTRADA por minuto | (no reportado en el incidente) | Ídem OTPM |
| **RPM** | Requests por minuto | (no reportado) | `ModelRateLimitingCheck` (rpm por deployment) + routing v2 |
| **Cuota mensual** | Presupuesto mensual de la clave | Cohere trial: 1000/mes, 20/min (evidencia LOTE 5A) | `detectQuotaExhaustion` → `quota_exhausted_monthly` NO-TRANSIENTE + FailoverTracker cooldown 300 s |

**Nota crítica del incidente:** el prompt de lección mide 11,253 B (~2,814 tokens aprox) — un único request no agota TPM 8000; el fallo fue **consumo acumulado del minuto** bajo carga concurrente. Por eso la mitigación correcta es CONTAR por minuto (pre-checks + routing v2), no achicar prompts (ya acotados: top-3 keyFacts ≤400 chars).

---

## 3. LAS CINCO OPCIONES DE CONTROL DE PROVEEDORES (comparativa obligatoria del mandato)

### Opción 1 — Mantener `least-busy` (status quo)

| Aspecto | Evaluación |
|---|---|
| Qué controla | Balanceo por peticiones EN VUELO (±1). Failover efectivo ante 503 (cooldown 30 s + reintento en otro deployment). |
| Qué NO controla | Tokens (TPM/OTPM/ITPM) ni RPM. **La causa raíz del 429 queda activa.** |
| Riesgo | Repetición estructural del incidente bajo carga concurrente; ráfaga return-0 de 1.84.0. |
| Veredicto | **INSUFICIENTE solo** — pero sus componentes sanos (cooldown por deployment, failover) se conservan en todas las demás opciones. |

### Opción 2 — `routing_strategy: usage-based-routing-v2`

| Aspecto | Evaluación |
|---|---|
| Disponibilidad | **SÍ existe en 1.84.0** [VERIFICADO]: `lowest_tpm_rpm_v2.py:32` (clase `LowestTPMLoggingHandler_v2`); incrementa tpm/rpm por minuto y filtra deployments sobre el límite. |
| Qué controla | Selección de deployment con conocimiento de TPM/RPM **acumulados del minuto** → el router elige el deployment con menor uso → evita mandar la petición 478 al deployment que ya va en 7977/8000. |
| Qué NO controla | **NO es un bloqueo duro**: es routing preventivo; un pico entre el último incremento y la llamada puede igualmente 429. **NO cubre OTPM/ITPM** (cuenta total_tokens). Requiere `tpm`/`rpm` declarados por deployment (hoy ausentes). |
| Coste | Cambio de UNA línea de config + datos de cuota por deployment. Sin código. Reversible. |
| Riesgo | Menor que Opción 1: mismo mecanismo de cooldown/failover subyacente. Matiz: prioriza por menor uso → puede concentrar carga en el deployment menos usado (incluido Gemini con weight 0.5 si su tpm declarado es alto). |
| Veredicto | **RECOMENDADA como complemento** de la Opción 3 (juntas: pre-check bloquea duro por TPM/RPM; routing v2 distribuye antes de llegar al bloqueo). |

### Opción 3 — Activar pre-call checks (`ModelRateLimitingCheck` + `tpm`/`rpm` por deployment) ⭐ PRIMERA

| Aspecto | Evaluación |
|---|---|
| Disponibilidad | **SÍ existe en 1.84.0** [VERIFICADO]: OPT-IN vía `enable_pre_call_checks: true` + `optional_pre_call_checks: ["enforce_model_rate_limits"]` (`router.py:282,1701`); semántica: `async_log_success_event` incrementa `total_tokens` por minuto (`model_rate_limit_check.py:246-290`) y `pre_call_check` compara `current_tpm >= tpm_limit` (`:115-118`) → **429 LOCAL antes de salir**. |
| Qué controla | **Bloqueo duro pre-call por TPM y RPM por deployment.** El 429 local del 1.84.0 tiene texto exacto conocido: `"Model rate limit exceeded. TPM limit={tpm_limit}, current usage={current_tpm}"` (`:117-144`) — **se puede clasificar honestamente como `litellm_gateway`** añadiendo ese patrón a `classifyErrorOrigin` (hoy NO está → acción menor obligatoria si se activa, doc 14 §4 #7). |
| Qué NO controla | **OTPM/ITPM** (cuenta total, no separa I/O); **no reserva** la salida pedida (cuenta post-éxito → una petición grande puede pasar el pre-check y aun así 429 en el proveedor); no es atómico entre procesos sin Redis (concurrency multi-pod no aplica aquí: proxy único). |
| Coste | Solo config + 1 patrón de clasificación en el adaptador. **Sin código nuevo de lógica.** |
| Riesgo | MEDIO-BAJO y reversible: si los `tpm`/`rpm` declarados son menores que los reales, se bloquea de más (falso 429 local, clasificado honestamente); si mayores, de menos. Requiere staging (mandato: cambios de config solo con autorización). |
| Veredicto | **PRIMERA ACCIÓN RECOMENDADA** — activa el mecanismo que ya existe en la librería instalada, antes de escribir módulos propios. |

### Opción 4 — Módulo nativo AICF `ProviderBudgetGuard` (diseño 09_PLAN §2.1, adaptado con honestidad de dónde vive cada capa)

| Aspecto | Evaluación |
|---|---|
| Qué controla | (a) **Presupuesto a nivel ALIAS desde AICF**: estimación pre-call de tokens de entrada+salida contra un presupuesto global por minuto — el adaptador SÍ puede hacer esto porque el presupuesto es del JOB/PROCESO, no del deployment; (b) reserva con reconcile post-respuesta (usage real ya capturado) y refund en timeout de cola; (c) clasificación de origen SIEMPRE explícita en el punto de creación del error (regla: JAMÁS default vendor). |
| Qué NO controla (honestidad estructural) | **Reserva por deployment**: el routing es interno del proxy; hasta que la respuesta llega con `x-litellm-model-id`, AICF no sabe qué deployment atendió. Cualquier diseño que pretenda reservar por deployment desde el adaptador **fingiría conocimiento** — queda PROHIBIDO en este diseño. La atribución por deployment es POST-HOC (S-4 persiste el model-id real por llamada). |
| Integración | Sobre la instancia 1.84.0 existente: presupuesto del alias en el adaptador ANTES del HTTP + reconcile con usage; cooldown map propio `deploymentId → untilMs` alimentado por la clasificación post-hoc; NO sustituye reintentos del router (regla UNA capa). |
| Coste | Módulo TS nuevo (el mayor del plan); depende de S-4 para la parte durable. |
| Riesgo | Estimación de tokens conservadora serializa de más (mitigación: reserva mínima 1 token si la estimación falla — principio `_reservation_value` del extracto 1.109 `:427-458`); duplicar control con Opción 3 → definir jerarquía: pre-checks de librería = duro por deployment; guard AICF = presupuestos de JOB (USD/tokens por job, `AICF_JOB_TOKEN_BUDGET`). |
| Veredicto | **DISEÑO DE FONDO (F2/F4)** — cierra lo que la librería no da: OTPM aproximado vía presupuesto de job, coste por job/etapa, y el enum de origen en cada error. Es la respuesta AICF al gap "RateLimitErrorCategory no existe en 1.84.0". |

### Opción 5 — Alias separados por proveedor (`aicf-groq`, `aicf-gemini`, `aicf-mistral`)

| Aspecto | Evaluación |
|---|---|
| Qué controla | Hace al routing EXPLÍCITO desde AICF: el adaptador elige proveedor → puede presupuestar, enrutar y hacer cooldown POR PROVEEDOR de verdad (cada alias es un deployment único o pool mono-proveedor). Failover entre alias gestionado por AICF (cadena Groq→Gemini→Mistral configurable por job) con historial de intentos (habilita S-6 de forma natural). |
| Qué NO controla | Los beneficios del pool multi-deployment dentro de un proveedor (si Groq tuviera 2 modelos, habría que gestionarlos por alias o perder el balanceo interno). |
| Coste | Cambio de config (3 alias) + lógica de selección/cadena en el lado AICF (nueva, con su propia capa de failover — riesgo de reintroducir una SEGUNDA capa de reintentos: debe ser failover SIN retry de transporte, respetando LOTE 5B). |
| Riesgo | MEDIO: reordena la arquitectura de routing; el failover AICF debe ser de solo-escalada (si el alias entero está gateado/cooldownizado, pasar al siguiente; JAMÁS reintentar el mismo alias). |
| Veredicto | **SOLO SI** el operador necesita presupuesto/encolado por proveedor DESDE AICF antes de que exista Opción 4, o para hacer S-6 (grupo de causas por proveedor) con visibilidad total. No es necesaria para la mitigación del 429. |

### Resumen de decisión

```
INMEDIATO (con autorización, staging primero):
  Opción 3 (pre-call checks + tpm/rpm) — bloqueo duro TPM/RPM por deployment
  + Opción 2 (usage-based-routing-v2)  — distribución preventiva por TPM/RPM
  + patrón "Model rate limit exceeded" en classifyErrorOrigin (clasificación honesta del 429 local)
FONDO (código AICF, fases F2+):
  Opción 4 (ProviderBudgetGuard con presupuesto de JOB + reconcile + origen explícito)
  + S-4 (LlmCallRepository: fila durable por llamada con x-litellm-model-id)
CONDICIONAL:
  Opción 5 (alias por proveedor) — solo si se exige presupuesto por proveedor desde AICF o S-6 total
PROHIBIDO en cualquier opción:
  fingir reserva por deployment desde el adaptador TS (el routing es del proxy)
  segunda capa de reintentos de transporte (LOTE 5B intacto)
```

---

## 4. CUOTAS RECOMENDADAS A DECLARAR POR DEPLOYMENT (datos a confirmar por el operador)

**Regla de honestidad:** los valores REALES de cada proveedor son dato del operador (consola de Groq/Gemini/Mistral); aquí se declara el MÉTODO, no se inventan números.

| Deployment | `tpm` sugerido | `rpm` sugerido | Método |
|---|---|---|---|
| `groq/qwen/qwen3.8-27b` | **7,200 (90 % de 8,000)** | RPM real de la consola Groq | El incidente muestra TPM 8000; declarar el 90 % deja margen para el error de conteo post-éxito de 1.84.0 (cuenta DESPUÉS de éxito → subestima en vuelo). Si Groq reporta OTPM separado, anotarlo como dato para Opción 4 (presupuesto de job), NO configurable en 1.84.0. |
| `gemini/gemini-3.8-flash` | TPM del tier de la clave (consola) | RPM del tier | Sin datos del incidente (Gemini no aparece en el extracto de errores — AUSENTE, no adivinado). weight 0.5 se mantiene hasta confirmar estabilidad 503. |
| `mistral/mistral-small-latest` | TPM del tier | RPM del tier | El 429 código 1300 del incidente es consistente con rate limit de Mistral (documentado en su API como `too_many_requests`), pero el cuerpo completo NO está disponible → no se clasifica la dimensión exacta (TPM vs RPM). Dato a capturar la próxima vez: body completo + headers (el adaptador YA los captura; faltará persistirlos → S-4). |

**Verificación de 1 minuto ANTES de declarar cuotas (dato del operador):** confirmar en la consola de cada proveedor los límites ACTUALES del tier de cada clave. **Advertencia adicional:** la tabla de modelos de litellm main retiró modelos `groq/*` antiguos (`groq/llama-3.3-70b-versatile` entre ellos — `12_MATRIZ` §2-bis.1); la config V30 usa `groq/qwen/qwen3.8-27b` — el operador debe confirmar que ese modelo sigue vivo en SU cuenta Groq antes de cualquier cambio (dato AUSENTE aquí: no se consultó la API de Groq — regla de no consumo de cuota).

---

## 5. MAPA COMPLETO: QUÉ CUBRE CADA CAPA (evitar solapes y huecos)

| Necesidad | Capa responsable | Mecanismo | Estado |
|---|---|---|---|
| Bloquear pre-call por TPM/RPM del deployment | Librería (proxy) | Opción 3: `ModelRateLimitingCheck` | APAGADO hoy → activar |
| Distribuir carga por uso del minuto | Librería | Opción 2: `usage-based-routing-v2` | Disponible, no configurada |
| Cooldown por deployment aislado | Librería | `cooldown_cache.py` (1.84.0 ✓) | ACTIVO (30 s, Retry-After priorizado) |
| Ráfaga instantánea tras 429 (return 0) | Librería (comportamiento) | — | **Hueco de 1.84.0** → contrarrestar con gate cliente AICF (activo) y cooldowns de config |
| OTPM/ITPM | AICF (Opción 4) | Presupuesto de JOB con estimación + reconcile | DISEÑO (no existe en 1.84.0) |
| Origen del 429 (vendor vs local vs gateway) | AICF | `classifyErrorOrigin` + patrón nuevo de pre-checks | ACTIVO (falta el patrón del 429 local de pre-checks) |
| Retry-After del proveedor | Librería + AICF | Router (`_calculate_retry_after`) + gate cliente tope 60 s | ACTIVO |
| Presupuesto USD por job | AICF (Opción 4 + S-4) | `AICF_JOB_TOKEN_BUDGET` + snapshots de precio | DISEÑO |
| Visibilidad: qué deployment atendió cada llamada | AICF (S-4) | Persistir `x-litellm-model-id` + usage por fila | **DATO YA CAPTURADO, NO PERSISTIDO** |
| Cuota mensual agotada (clave trial) | AICF | `detectQuotaExhaustion` → no-transiente + FailoverTracker 300 s | ACTIVO |
| Concurrency local | AICF | `LLMGovernor` (conc. 5, bucket 10/2-s, presupuesto retry 20/60-s, breaker 429-only) | ACTIVO |

---

## 6. LÍMITES DE ESTA ESTRATEGIA

1. **Nada se aplicó** — `litellm-config.yaml`, `.env`, `docker-compose.yml` intactos (verificado). Toda la Opción 2/3 es un CAMBIO DE CONFIGURACIÓN que el operador debe autorizar y validar en staging (mandato §8: respetar la protección de config en esta fase).
2. **Las cuotas reales son dato del operador** (§4: método, no números inventados). El incidente solo dejó evidencia de Groq TPM 8000 / OTPM 1000.
3. **Mistral 1300 y el intento a Gemini quedan AUSENTES** (sin body completo ni log durable por llamada) — la estrategia está diseñada para que la PRÓXIMA vez la respuesta sea inmediata: Opción 3 clasifica el 429 local, S-4 persiste el model-id real de cada intento.
4. **Sin upgrade de LiteLLM** (S-11 DESCARTADA por ahora): todo lo anterior funciona en 1.84.0; el upgrade solo se reabre con changelog completo + staging + verificación de dependencias.

**Fin del documento 17. — Conjunto entregado: 14 (auditoría comparativa) · 15 (plan priorizado) · 16 (diagnóstico de bugs A/B) · 17 (estrategia de proveedores).**
