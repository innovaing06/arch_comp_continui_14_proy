# AICF V5.2 — WP2.1 — AUDITORÍA FINAL, CIERRE FUNCIONAL Y COMPATIBILIDAD ARQUITECTÓNICA

**Fecha:** 2026-11-06 (UTC) · **Ejecutado en:** sandbox Linux aislado (Bun 1.3.14) · **Presupuesto de inferencia: US$0** (cero inferencias, cero llamadas a proveedores, cero cuotas, cero Docker)
**Snapshot auditado:** copia fiel del estado Windows post-WP2.1-v2 (`aicf-wp21-v2-work/repo/` = base verificada + `WP21-V2-DIFF.patch`, byte-idéntico al paquete entregado)

---

## 1. ESTADO

| Ámbito | Estado |
|---|---|
| **Auditoría funcional WP2.1 (esta tarea)** | **COMPLETADO** — todo lo herméticamente verificable fue ejecutado y verificado; no quedó ningún criterio sin examinar |
| **WP2.1 (producto)** | **APTO PARA ACEPTACIÓN FORMAL** — cumple todos los criterios demostrables con pruebas herméticas (estándar del addendum v2.3.1 §2.2); **cero defectos encontrados**; quedan **2 confirmaciones de cierre exclusivas de Windows** (§10) y **1 evidencia administrativa** (FROZEN 8/8) que no constituyen defectos de WP2.1 |
| Archivos modificados por esta auditoría | **NINGUNO** (regla §7: si ya cumple, no modificar — se cumplió) |
| WP2.2 / WP2.3 / WP8 / modos económicos | **NO iniciados** (prohibición respetada) |

**Recomendación:** aceptación formal de WP2.1 por el operador, condicionada a las confirmaciones de cierre de Windows listadas en §10 (verificaciones de confirmación, no correcciones: no existe defecto conocido que corregir).

---

## 2. TABLA DE CRITERIOS (§9 del mandato)

| # | Criterio de aceptación | Resultado | Evidencia |
|---|---|---|---|
| 1 | AUTO y MANUAL correctos | **PASS** | 79/0 focalizadas re-ejecutadas: contrato T1–T4, planner T5/T11, ruta S6/S7; recomendación = `computeCurriculumShape()` (sin segundo algoritmo — T1) |
| 2 | Cantidad efectiva idéntica a la selección aprobada | **PASS** | Planner T5/T11 (prompt "exactly N" + spec N módulos), worker post-plan fail-closed (`module_count_mismatch`), T12/T12b, hijo worker B |
| 3 | Persistencia y recuperación correctas | **PASS** | T14 (round-trip disco + hash re-verificable) + ad-hoc 18/18 (ciclo de vida del job: create→running→retrying→paused→**resume**→idempotencia; selección byte-idéntica en todos los estados; recuperación de disco re-valida) |
| 4 | `module-selection.json` y `manifest.json` coherentes | **PASS (eslabones)** — composición en curso real = confirmación Windows | Misma proyección `toManifestBlock(config)` en ambos (mismo `configHash` por construcción, verificado ad-hoc); `manifest.moduleCount = curriculumSpec.modules.length` + gate fail-closed previo garantiza `moduleCount === effectiveModuleCount`; rechazo ante discrepancia ejecutado (T12, hijo B, defensa en profundidad worker) |
| 5 | Worker fail-closed | **PASS** | Hijo A/B/C con CourseWorker REAL sin mocks: ausente → `module_selection_missing` (<15 s, cero presupuesto/research/proveedor, job persistido failed); manipulado (hash) → `module_selection_invalid`, sin artefactos; válido → arranca + artefacto durable registrado ANTES de etapas |
| 6 | Dashboard y backend reflejan la misma selección | **PASS (frontera) / SKIP (visual)** | S6/S7/S8: la API resuelve server-side y hace eco; el cliente solo envía la decisión; el worker recibe exactamente lo resuelto. Validación visual del Dashboard = **SKIP** (no ejecutable en sandbox; declarado, no inferido) |
| 7 | Recomendación sin consumo de inferencia | **PASS** | `WP21-Recommendation.test.ts`: fetch global espiado = **0 llamadas de red**; cálculo puro del motor existente |
| 8 | FREE_ONLY y US$0 preservados | **PASS** | T16 (cero superficie de proveedor), S10 (adquisiciones de adaptador idénticas AUTO vs MANUAL), RUN5 302/0 (guards WP1/P3-B1/B2), RUN8 43/43; cero inferencias en toda la auditoría |
| 9 | WP1 y P3-B1/B2 intactos | **PASS** | 29 archivos protegidos SHA-256 **idénticos** base↔repo (§7); coexistencia WP1×WP2.1 W1–W4 ejecutada; WP1-CurriculumTruncation dentro de RUN4: PASS |
| 10 | Archivos FROZEN preservados | **SKIP — `FROZEN_BASELINE_NOT_VERIFIED`** (§8) | Lista autorizada de los 8 archivos NO disponible en el sandbox (busqueda exhaustiva). Verificación parcial: todos los FROZEN-candidatos de evidencia previa están byte-idénticos (ResearchEngine, ContentGenerator, QualityChecks, PremiumQualityGate, rutas legacy `[id]/*`, PremiumRubric16, litellm-config.yaml) |
| 11 | Pruebas y lint satisfactorios | **PASS** | Focalizadas 79/0; regresión 1043 pass / 21 fail — **los 21 son el conjunto preexistente idéntico al baseline WP1** (verificado uno a uno, ninguno nuevo ni ocultado); RUN5 302/0; lint 0 problemas; tsc +8 ambiental (solo tests nuevos, 0 producción) |

### Detalle por área funcional (§6 del mandato)

| Área | Sub-criterio | Resultado |
|---|---|---|
| **A. AUTO/MANUAL** | Reutilización real de `CurriculumPlanner` | PASS (T1: `recommendedModuleCount === computeCurriculumShape().moduleCount`) |
| | Respeto de la cantidad recomendada / manual | PASS (T2/S6; T3/T4/S7 — MANUAL prevalece, recomendación queda como referencia) |
| | Ausencia de sustitución silenciosa | PASS (T8/T15, S1–S3, hijos A/B; sin contrato → error explícito, jamás `?? 5`) |
| | Persistencia del modo | PASS (T14 + bloque manifest con modo/versión/hash) |
| **B. Persistencia** | Guardado / recuperación tras reinicio / integridad en reintentos | PASS (T14 + ad-hoc disco/revalidación; T6/T6b/S9 reintento improve/apply preserva byte a byte) |
| | Identidad y hash del contrato | PASS (SHA-256 canónico; `validateModuleSelectionConfig` re-verifica hash vs contenido) |
| | Rechazo de ausentes/manipulados/inconsistentes | PASS (hijo B; validate: hash/modo/versión/rango) |
| **C. Artefactos** | Identificador/cantidad/IDs/orden/trazabilidad coherentes | PASS por eslabones ejecutados (mismo `configHash` y `effectiveModuleCount` en contrato, bloque manifest y artefacto durable — ad-hoc; IDs/orden provienen del único `curriculumSpec` tras el gate) |
| | Rechazo ante discrepancias | PASS (T12/T12b/`module_count_mismatch`) |
| **D. Worker** | Validación antes de etapas posteriores | PASS (hijo A: gate ANTES de presupuesto/research/runtime) |
| | Fail-closed contratos inválidos / sin ejecución parcial | PASS (hijo B: failed sin artefactos del contrato) |
| | Reanudación sin cambiar la selección | PASS (ad-hoc: `update()`/`resume()`/idempotencia preservan `request.moduleSelection` byte a byte — el store protege `request` por diseño) |
| **E. Dashboard** | Reflejo de selección / cantidad mostrada = ejecutada / decisión del operador / recomendación ≠ aprobación | PASS en frontera HTTP (eco S6–S8; cliente envía solo la decisión; estados informativos distinguen "recomendación AUTO: X" de "se generarán exactamente N") — **visual: SKIP** |
| | Sin divergencia frontend/backend/worker | PASS (resolución server-side única; worker re-valida) |
| **F. Costes** | Endpoint sin llamadas externas / sin inferencias / sin cuotas / FREE_ONLY | PASS (fetch=0 ejecutado; WP21 no añade ninguna prueba ni camino que requiera proveedor real) |

---

## 3. EVIDENCIAS CONCRETAS (identidad y huella del snapshot)

**Cadena de correspondencia verificada (Windows ↔ sandbox):**

1. **Árbol base** (`aicf-wp21-v2-work/base/`) = estado REAL post-WP1 del usuario: **0 diferencias** contra `aicf-wp1-p0/tree/` (entrega WP1 previamente verificada); blob git del `CurriculumPlanner.ts` base = `16242916efff…` — **coincide exactamente** con el index post-WP1 del diff de `ESTADO GIT.md` (`index d04fc6f..1624291`).
2. **Parche auto-suficiente:** `git apply --check` sobre copia FRESCA del base → **exit 0**; aplicación real → exit 0; resultado **byte-idéntico** al árbol probado (`diff -rq` vacío); re-aplicación correctamente RECHAZADA (parche no vacuo).
3. **Diferencia total base↔repo = EXACTAMENTE 20 archivos** (11 modificados + 9 nuevos) — `diff -rq` verificado antes y después de toda la auditoría; ningún otro archivo tocado.
4. **Aritmética del estado Windows:** pre-WP2.1-v2 (`ESTADO GIT.md`): 18 M + 20 ?? → parche añade 4 M nuevas (CourseWorker, CourseJobStore, generate.tsx, types.ts) y 8 entradas ?? nuevas (7 archivos + 1 dir `module-recommendation/`) → **22 M + 28 ?? = 50 entradas** — coincide EXACTAMENTE con el diagnóstico Windows reportado en el addendum v2.3.1 §0.1.
5. **Paquete:** SHA-256 del ZIP `fe27daba3c37…` = `ZIP-SHA256.txt` ✓; `SHA256SUMS.txt` 39/39 ✓; `files/` del paquete ↔ árbol probado 20/20 byte-idénticos ✓.
6. **HEAD `0ec0d2c` / rama `feat/f2c-repair-002`: NO verificables desde sandbox** (sin acceso al `.git` de Windows) — declarado como limitación; la correspondencia se establece por la cadena de evidencia del árbol de trabajo (1–5). No se afirma identidad byte-a-byte sin evidencia.

---

## 4. PRUEBAS EJECUTADAS Y RESULTADOS REALES (re-ejecutadas en esta auditoría)

| # | Comando / alcance | Resultado | Comparación |
|---|---|---|---|
| A1 | Focalizadas WP2.1/WP1 — 10 archivos (5 WP21 + 5 WP1) | **79 pass / 0 fail** / 560 expect / 10 archivos (1.05 s) | = Windows: 79/0, 10 archivos (6.15 s) ✓ |
| A2 | Regresión `src/lib/runtime/course/` + `generate-course/` + `module-recommendation/` | **1064 tests / 61 archivos — 1043 pass / 21 fail** | Los 21 = conjunto preexistente idéntico al baseline WP1 (verificado uno a uno: pines FROZEN/hash ×6, P4-A ×3, PremiumRubric16 real-course ×5, FASE 32/32.1/32.2 ×6, FASE 30.5B ×1). **0 nuevos, 0 ocultados** = RUN4 del paquete (1043/21) ✓ |
| A3 | Baseline `src/lib/ai/` + `src/app/api/providers/` | **302 pass / 0 fail** / 23 archivos | = RUN5 (302/0) ✓ — guards WP1/P3-B1/B2 intactos |
| A4 | `bun run lint` (repo completo) | **0 problemas, exit 0** | = RUN6 y Windows lint PASS ✓ |
| A5 | `bunx tsc --noEmit` | **330 errores** — delta +8 vs baseline 322, **exclusivamente** clase ambiental `TS2307 bun:test` (×6) + `TS2867 Bun` (×2) en los 7 archivos de test NUEVOS; **0 errores en producción** | = RUN7b (330) ✓ |
| A6 | Ad-hoc hermético (script FUERA del repo, US$0): ciclo de vida del job + disco + coherencia | **18/18 PASS** | Nueva evidencia de esta auditoría (cubre reanudación/recuperación §6-B/§6-D) |
| A7 | Verificaciones de identidad/huella (diff, parche, SHA-256, blobs) | **Todas PASS** (§3) | = CORRESPONDENCE-REPORT ✓ |

Herramientas: Bun 1.3.14, tsc del lockfile, eslint del repo. Nota de entorno: una segunda ejecución de A2 colgó (flake ambiental del sandbox, posiblemente lock SQLite residual); la primera ejecución fue limpia y completa (9.51 s) y su conjunto de fallos es el documentado; artefactos de test (`db/`, `experiments/`, `tsconfig.tsbuildinfo`) fueron limpiados y el árbol re-verificado prístino.

---

## 5. DEFECTOS Y CORRECCIONES

**Defectos de WP2.1 encontrados: NINGUNO.** No se corrigió nada (§7: si ya cumple, no modificar). Sin refactorizaciones, sin archivos ajenos al alcance.

Clasificación de los 21 fallos de A2: **PREEXISTING FAILURE** (conjunto idéntico al baseline post-WP1, documentado canónicamente en el TEST-REPORT WP1 §4; ninguno relacionado con archivos tocados por WP2.1). Anomalía de A2-rerun: **no reproducible como defecto del producto** (flake de entorno del sandbox).

---

## 6. ARCHIVOS MODIFICADOS Y NUEVOS (por esta auditoría)

**En el repositorio auditado: NINGUNO.** El árbol quedó exactamente en el estado de entrada (base + los 20 archivos WP2.1-v2; única adición temporal `node_modules` git-ignored, instalado para ejecutar las pruebas y retirado después). Script ad-hoc creado FUERA del repo: `/home/z/wp21-audit/adhoc-store-resume.ts` (evidencia A6). Sin commits, sin `reset`/`checkout`/`stash`/`clean`, sin ZIP ni parche nuevos (no hubo cambios).

---

## 7. PRESERVACIÓN WP1 / P3-B1/B2

**Verificación SHA-256 base↔repo: 29/29 archivos IDÉNTICOS** — guards WP1 (GenerationAuthGuard, FreeOnlyRoutingGuard, gates de truncamiento en ContentGenerationService, LessonGenerationFailureDiagnostics, improve/apply, AIAdapter con `isTokenLimitFinishReason`, sus 4 suites, docs), P3-B1/B2 completos (PoolMutationGuard, ProviderRegistrationService, LiteLLMAdminClient/ConfigBuilder/ModelDiscoveryService/ProviderDefinitions, routing/apply + tests, 5 suites P3B1/P3B2, docs), PremiumRubric16 + su test, litellm-config.yaml.

Dentro de `CurriculumPlanner.ts` (modificado por WP2.1): gate terminal de truncamiento WP1 ANTES del gate de cantidad (prioridad WP1), condición de reintento WP1 intacta (extendida, no sustituida), `lastResp = retryResp` conservado, mensaje de truncado byte a byte — **probado por ejecución** (W1–W4 + WP1-CurriculumTruncation dentro de A2). Única excepción declarada: fixture `WP1-RouteGuards.child.ts` (+1 campo `moduleSelectionMode:'AUTO'` en el body del escenario R5; aserciones R1–R10 intactas; suite 10/10 dentro de A1). Suites WP1 completas: **43/43**.

---

## 8. ESTADO FROZEN 8/8

**`FROZEN_BASELINE_NOT_VERIFIED`.** Ninguna fuente autorizada disponible en el sandbox enumera los ocho archivos con nombres y hashes (búsqueda exhaustiva: addendum v2.3.1 — que explícitamente no inventa la lista —, auditoría integral, entregas R28/F1/WP1/WP2.1, uploads). Conforme al procedimiento obligatorio (§1.1 del addendum): no se inventó la lista, no se tocó ningún candidato, y se verificó que **todos los FROZEN-candidatos identificados por evidencia previa** (ResearchEngine.ts, ContentGenerator.ts, QualityChecks.ts — hogar de V32-6=0.5 —, PremiumQualityGate.ts, rutas legacy `src/lib/runtime/course/[id]/*` ×6, PremiumRubric16.ts + test, certification.rubrics intacto vía contratos no tocados, litellm-config.yaml) están **byte-idénticos** base↔repo y que el diff WP2.1 (20 archivos) no contiene ninguno. **Acción requerida al operador:** fournir la lista autorizada (nombres + SHA-256 + fuente) para registrar la línea base.

---

## 9. SHA-256 Y ZIP

**Sin cambios → sin ZIP/parche nuevos.** Huella del estado auditado (= estado Windows de entrada):

- ZIP del paquete WP2.1-v2: `fe27daba3c3743751dafe54ee819764742c436ea1c9e7541335ab09fbe8ba1ac` (verificado íntegro)
- Parche `WP21-V2-DIFF.patch`: `git apply --check` exit 0 sobre la huella exacta de entrada (copia fresca del base); resultado byte-idéntico al probado
- Los 20 archivos WP2.1-v2 (fingerprint del estado — extraído del árbol auditado):

```
8caf85db33afef7751e1da3f96f9ee2b8890b5d8c5b0c3eb08233deed2a4dc12  CurriculumPlanner.ts
99e5ed9a243261dd72515fc716cb70e1e8fd3d07f59e8fadb6c6ee8b41dbb0cc  CourseWorker.ts
0a3b71d3462649f6a8146bf939c82cec4e4df7d663404f32dfb825ea691bce92  CourseJobStore.ts
ce917cda414bfada6cec3611dfbd44a931c4da8e96cd10dc959753c12a775e48  generate-course/route.ts
c271716aee48df918d2be457a5f29e7e71d8b301adcb5b52106c9f634aad83c5  generate.tsx
eb91c29abfba8f02ba176b63e425092d2ca3eb0933f0ed98e7ed69dfec2e2437  types.ts
b0c8812e08833a7cfb7b852376946d87072b38ed4ce6923b3925f4074952bab9  F1-C9-PostHocBudget.child.ts
8771430951c39e16168ce0777ce6377925d8ccdfb9ef8e4a6757ed342fec313a  WP1-RouteGuards.child.ts
76133bd8771fe24272aa09cb0a63bfcf9245142d0cc78383f05a4394b6a22cd5  5ER3-P1-RouteQualityReport.child.ts
492f73e0b33e37c78b23d49507aa56ab726c92583edee2e40a0d30634aebd748  5ER4-PostFailure.child.ts
7139e757ec80ccb82fb0f6d18504e6350810027ae2041491c5b299b601d274f7  F1-S2-ReleaseVisibility.child.ts
2fe62bece9bc9e2ddd2310be2624e12e77821a6c8f1195bbe26fc69d865e59ac  ModuleSelectionContract.ts (nuevo)
2aa1bcd1d6b176e855556aaee40f2e209ddc80bd649e55f9d9bbe12bdd82f199  module-recommendation/route.ts (nuevo)
d42ab799095017bfb92e1c8b1efd28e7fc026881ee3d4381daf272f4c3764808  WP21-ModuleSelectionContract.test.ts (nuevo)
7022f12f65414362cc37b6673ccedc3da2f98be0140c08297ec11c26a091dedb  WP21-CurriculumPlanner-Contract.test.ts (nuevo)
232d5679566f8f6dbfe04fd9c6bc0499840eab9fc428721bffe36f00ebd69c17  WP21-WorkerFailClosed.child.ts (nuevo)
31143445eb15e3d3e35a6e5f1ab84b583ccc24fb0fbd80831c96c40168b90b71  WP21-WorkerFailClosed.test.ts (nuevo)
2ca9258b32656caaa61ab1b813b2a442ee7924d8e97442e9ca8c931a7c19bde7  WP21-ModuleSelection.child.ts (nuevo)
4d1f8d6b07b42a9453cbce9949d87e1f1b2c84976a7fc7a9cea33c76965ebc44  WP21-ModuleSelection.test.ts (nuevo)
bd218f95e3d35822a25977c82e5f05d400b40095f959ade5aa266923ca4b48cb  WP21-Recommendation.test.ts (nuevo)
```

(rutas completas según MODIFIED-FILES.txt; los 20 coinciden byte a byte con `files/` del paquete y con el resultado del parche sobre el base).

---

## 10. PENDIENTES EXACTOS

1. **Confirmación Windows (E2E, único eslabón no ejecutable en sandbox a US$0):** un run end-to-end con proveedor gratuito del pool FREE_ONLY que materialice en disco el `manifest.json` y `module-selection.json` de un curso completado, verificando `manifest.moduleCount === moduleSelection.effectiveModuleCount` y `configHash` idéntico en ambos. Todos los eslabones individuales están ejecutados y verdes; falta únicamente la composición real.
2. **Verificación visual del Dashboard (Windows):** toggle AUTO/MANUAL, recomendación visible como referencia (no como aprobación), entrada entera 1–20, estados informativos. Declarado SKIP en sandbox.
3. **FROZEN 8/8 (administrativo, afecta al proyecto, no defecto de WP2.1):** el operador debe proporcionar la lista autorizada (nombres + SHA-256 + fuente); mientras tanto rige `FROZEN_BASELINE_NOT_VERIFIED`.
4. **Cadena de sincronización (administrativo):** confirmar en Windows rama/HEAD (`feat/f2c-repair-002` @ `0ec0d2c`) y huella del árbol para cerrar la trazabilidad Z.ai↔Windows (el HEAD no es verificable desde sandbox).
5. Los **21 fallos preexistentes** de la suite course (documentados, ajenos a WP2.1) siguen abiertos como deuda heredada de la línea — sin cambio por esta etapa.

---

## Registro de cumplimiento

- NO se reinstaló WP2.1, NO se reconstruyó la implementación, NO se reaplicó el parche sobre el estado ya parcheado, NO se inició WP2.2/WP2.3/WP8, NO se implementaron modos económicos.
- NO se tocaron `ResearchEngine`, `ResearchAcquisitionService`, `LocalizationDerivation`, `CountryCatalog(Extended)` ni archivos de localización (WP8 no adelantado). WP2.1 no presenta dependencia real con contratos de país (verificado: el contrato `ModuleSelectionContract` no contiene `countries[]`/`sourceCountry`/`variantOf`/`sourceJobId`/`sourceVersion`).
- Requisitos WP8 (template-first, `countries[0]`=sourceCountry, estructura congelada, G8, idempotencia `(sourceJobId, sourceVersion, variantCountry)`, `stale`) quedan registrados como decisiones futuras en el addendum v2.3.1 §4 — sin código creado ahora.
- Sin commits, sin operaciones Git destructivas, sin modificación del repositorio Windows, sin inferencias reales (US$0 preservado en toda la auditoría).

**Fin del informe.**
