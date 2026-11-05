# 16 — DIAGNÓSTICO DE LOS DOS BUGS DE PERSISTENCIA 5E-R4

**Fecha:** 2026-10-31 (Lima) · **Modo:** SOLO LECTURA — sin commits, sin ediciones, sin proveedores reales.
**Base auditada:** copia Z.ai `/tmp/my-project` (post-5E-R4). Hashes verificados: `ContentGenerationService.ts` = `d6d5bad7…`, `route.ts` = `3d146a11…`, `LessonGenerationFailureDiagnostics.ts` = `df271f18…`, `FailedJobReleaseRecord.ts` = `7adaacd7…`. 8/8 protegidos intactos (7 FROZEN + `ContentGenerator.ts` `af851377…`).

---

## 0. RESUMEN EJECUTIVO

| Bug | Síntoma reportado | Causa raíz DEMOSTRADA | El helper funciona? |
|---|---|---|---|
| **A** — `lesson-generation-failure.json` | No aparece en el curso fallido | **Doble identidad de courseId**: el registro se construye con `spec.courseId` (generado por el LLM en `CurriculumPlanner.normalizeSpec`), pero el curso en disco vive en `job.courseId` (generado por `CourseJobStore.create`). El archivo SÍ se escribe — en un **directorio huérfano** con el nombre que inventó el LLM. | SÍ — escribe correctamente (tmp+rename, mkdir incluido). Escribe en el sitio equivocado. |
| **B** — `release.json` "aparece vacío" | El registro de auditoría no se ve | **Doble capa**: (1) el registro BLOCKED existe en disco pero su payload de evaluación es TODO-null por diseño y su única causa es el string `job_status_failed:<reasonCode>`; (2) **el endpoint `GET /api/course/[id]/releases` devuelve 404 "Course not found"** para estos cursos porque exige `manifest.json`, que **nunca se escribe** para fallos `no_lessons_generated` → el registro es invisible por API y "vacío" de contenido al leerlo en crudo. | SÍ — verificado por ejecución aislada: escribe la revisión BLOCKED completa. |

Ninguno de los dos bugs es un fallo de escritura atómica. Son un fallo de **identidad** (A) y de **visibilidad + diseño informativo** (B). La solución duradera NO requiere tocar `CourseWorker.ts` (FROZEN).

---

## 1. BUG A — `LessonGenerationFailureDiagnostics.ts`

### 1.1 Traza completa (origen → escritura efectiva)

```
POST /api/generate-course
 └─ route.ts:641            worker.run(jobRequest)                     [CourseWorker FROZEN]
     └─ CourseWorker:368      projectId = job.courseId                 ← ID REAL (CourseJobStore.create:88
     │                                                                  genera `course-{ts}-{rand}`, p.ej. course-mvvcmol9-0nez)
     ├─ CourseWorker:585      curriculumPlanner.plan({...})            ← SIN courseId en la entrada
     │    └─ CurriculumPlanner.normalizeSpec:493-496
     │         spec.courseId = r.courseId (respuesta del LLM)
     │                          ?? `course-${slug(title)}`             ← ID FANTASMA (inventado por el modelo)
     ├─ CourseWorker:624      contentService.generateFromSpec({ spec }) ← pasa el spec SIN normalizar
     │    └─ ContentGenerationService:379  catch (err) de la lección
     │         ├─ :461      courseId: spec.courseId                    ← EL REGISTRO LLEVA EL ID FANTASMA
     │         ├─ :473      await persistLessonGenerationFailure(record)
     │         │    └─ LessonGenerationFailureDiagnostics:214
     │         │         dir = getCourseDir(spec.courseId)             ← experiments/fase64/courses/<ID FANTASMA>/
     │         │         writeAtomic(dir + '/lesson-generation-failure.json')  ✓ ESCRIBIÓ (mkdir -p + tmp+rename)
     │         │    └─ :228  CourseArtifactRegistry.register({ projectId: spec.courseId, ... })  ← CLAVE EQUIVOCADA
     │         └─ :482      logger.error(... reasonCode:'lesson_generation_failed' ...)  ← ESTE ES EL LOG
     │                     QUE EL OPERADOR SÍ VE (por eso sabe que el catch corrió)
     └─ CourseWorker:638     CourseJobStore.update(jobId, {status:'failed', error:'no_lessons_generated'})
 └─ route.ts:643            courseId = job.courseId                     ← ID REAL
 └─ route.ts:650            allArtifacts = CourseArtifactRegistry.listByProject(job.courseId)
                            ↑ NO incluye el diagnóstico (está registrado bajo spec.courseId)   ← PÉRDIDA 2
 └─ route.ts:668-671        persistArtifact(job.courseId, ...)         → directorio del curso REAL
```

### 1.2 Evidencia por punto de la checklist del mandato §5

| Pregunta del mandato | Respuesta demostrada | Evidencia (archivo:línea) |
|---|---|---|
| ¿Quién llama al helper? | `ContentGenerationService`, catch per-lección (única llamada real) | CGS:473 |
| ¿En qué momento? | Dentro del catch de la lección fallida, ANTES del `logger.error` y del `return FAILED` (record-then-raise, orden correcto) | CGS:379→473→482 |
| ¿Qué parámetros recibe? | Registro completo con `courseId: spec.courseId`, jobId del correlation ALS | CGS:460-472 |
| ¿Qué ruta física escribe? | `<cwd>/experiments/fase64/courses/<spec.courseId>/lesson-generation-failure.json` | LGFD:214-216 + FCS:32-36 |
| ¿La ruta existe? | `writeAtomic` HACE `mkdirSync(dir, {recursive:true})` — la ruta se crea SIEMPRE. **ENOENT no es la causa** | FCS:161-163 |
| ¿La función es alcanzada? | SÍ — el log `lesson_generation_failed` (CGS:482) está DESPUÉS de la persistencia en el mismo catch; el operador lo ve | CGS:482 |
| ¿Condiciones que impiden escribir? | NINGUNA bloquea la escritura. La escritura OCURRE — en el directorio equivocado | §1.3 experimento |
| ¿Se traga una excepción? | El try/catch de LGFD:217/239 solo actuaría si la escritura fallara; no falla. Si el operador no ve el warning `5E-R4 E: atomic write ... failed` en dev.log, es porque la escritura EXITOSA | LGFD:213-248 |
| ¿Problemas de sincronización? | No. La escritura es síncrona (writeFileSync+renameSync) dentro del await | FCS:161-172 |
| ¿Estado del worker disponible? | correlation ALS sí (jobId, stage: 'course'); NO lleva courseId (solo `jobId?`, `stage?`) | LiteLLMAdapter:96-99 |
| ¿Rutas diferentes de almacenamiento? | SÍ — esta es la causa: `experiments/fase64/courses/<spec.courseId>` (helper) vs `experiments/fase64/courses/<job.courseId>` (ruta) vs `Cursos/<id>/versions/vN` (versionado, L8-REV3) y `AICF_COURSE_DATA_DIR` (override SOLO en ReleaseRegistry:191-197 — el helper NO lo honora) | FCS:32,175-179; ReleaseRegistry:191-197 |
| ¿Escritura posterior sobrescribe? | No — nada más escribe ese filename. La pérdida es por destino, no por sobreescritura | grep proyecto: solo LGFD lo escribe |

### 1.3 Demostración experimental (sin tocar el proyecto)

Los cursos de test R4 presentes en la copia (`course-i1-test`, `course-p1-test`) **SÍ contienen** `lesson-generation-failure.json`. Son exactamente el caso "IDs autoconsistente": `5ER4-FailureDiagnostics.test.ts:42-44` construye `makeSpec(courseId)` con el MISMO courseId que inspecciona. **La escritura funciona; lo que falla en producción es la correspondencia spec.courseId ↔ job.courseId.**

En producción, `spec.courseId` sale del JSON del LLM (o del slug del título) y `job.courseId` es `course-{ts}-{rand}` — **coinciden solo por accidente imposible**. Resultado predecible: existe un directorio huérfano `experiments/fase64/courses/<courseId-del-LLM>/lesson-generation-failure.json` que nadie inspecciona, y el curso fallido real (`course-mvvcmol9-0nez` en el incidente) no contiene el diagnóstico. El registro del registry (memoria) también está bajo la clave del ID fantasma → el bucle de persistencia de la ruta (route.ts:650-677) no lo serializa.

### 1.4 Por qué los tests de 5E-R4 no lo detectaron

| Suite | Qué verificó | Cegueras |
|---|---|---|
| `5ER4-FailureDiagnostics` (tests 15-18) | El registro se escribe dado un `makeSpec(courseId)` y ese courseId es el directorio inspeccionado | El test INYECTA la condición que en producción no existe (spec.courseId == dir real). No hay test de la correspondencia con `job.courseId` |
| `5ER4-PostFailure` + child (19-23) | La ruta escribe release.json BLOCKED con worker MOCKEADO | El worker mockeado devuelve un job cuyo courseId ES el del escenario → la ruta CGS/diagnostics real nunca se ejercita a nivel ruta |

**Ambas suites son verdaderas pero no reproducen la unión real worker→CGS→disco.** El bug vive exactamente en la costura que ninguna suite cubre.

### 1.5 Diseño de solución (duradera, atómica, verificable, SIN tocar CourseWorker.ts)

**Principio (patrón StudyForge `MeteredLLM` + `run_id`): la clave de persistencia debe crearse ANTES del trabajo y ser la misma que nombra el artefacto final.** Aquí ya existe: `job.courseId`. Solo hay que RESOLVERLA desde CGS sin modificar al worker.

1. **Resolución del ID real en el catch de CGS** (archivo NO FROZEN — ya fue modificado en R4):
   - El correlation ALS ya lleva `jobId` (lo parchea `CourseWorker:374` — código FROZEN que ya existe y funciona).
   - `CourseJobStore.get(jobId)` (CourseJobStore.ts:101, Map en memoria del MISMO proceso) devuelve el job → `job.courseId` es el ID real del directorio.
   - En el catch: `const storageCourseId = correlation?.jobId ? (await CourseJobStore.get(correlation.jobId))?.courseId ?? spec.courseId : spec.courseId`.
   - El registro escribe en `getCourseDir(storageCourseId)` y registra el artefacto con `projectId: storageCourseId`.
   - Cinturón y tirantes: si `storageCourseId !== spec.courseId`, escribir en AMBOS directorios y añadir al registro el campo `courseIdSource: 'job-store' | 'spec-fallback'` (auditable).
2. **Honrar `AICF_COURSE_DATA_DIR`** si se quiere consistencia con ReleaseRegistry (hoy el helper usa `getCourseDir` directo — divergencia latente documentada; no activa en esta copia porque la variable no está definida en `.env`/`docker-compose.yml`, verificada).
3. **Contrato de verificación (tests a añadir cuando el operador autorice):**
   - T-A1: spec.courseId ≠ job.courseId (worker real o harness que llame a CGS con correlation jobId creado vía `CourseJobStore.create`) → el archivo aparece en `experiments/fase64/courses/<job.courseId>/` (aserción de existencia + jobId + transportClass).
   - T-A2: la ruta persiste el diagnóstico desde el registry (`listByProject(job.courseId)` lo incluye tras el catch).
   - T-A3: job-store inaccesible (jobId sin correlación) → fallback a spec.courseId + campo `courseIdSource:'spec-fallback'` + warning.
   - T-A4 (regresión): el mensaje del error propagado permanece byte-idéntico (la resolución es solo observacional — nunca muta el error, contrato P1 B intacto).
4. **Limpieza del directorio huérfano del incidente**: el operador puede localizar el registro perdido buscando `lesson-generation-failure.json` bajo `experiments/fase64/courses/*` con courseId distinto del curso inspeccionado (comando de búsqueda incluido en §5).

---

## 2. BUG B — `FailedJobReleaseRecord.ts` / `release.json` "vacío"

### 2.1 Traza completa (origen → escritura efectiva)

```
route.ts:1289   if (baseStatus === 'failed')
route.ts:1292     recordFailedJobRelease({ courseId: job.courseId,   ← ID CORRECTO (a diferencia del bug A)
                                  origin:'generate', reasonCode: job.error })
       └─ FailedJobReleaseRecord:74  registry = loadReleaseRegistry(courseId) ?? initReleaseRegistry(courseId)
       │                            (ReleaseRegistry:236-279 — valida schema; CORRUPT si inválido)
       └─ FJRR:82-92      evaluation = { premiumRubric: null, commercialReadiness: null, gate:'BLOCKED',
       │                                gateReasons:['job_status_failed:no_lessons_generated'], ... }
       └─ FJRR:94-116     rev r1 BLOCKED + upsertReleaseRevision
       └─ FJRR:117-123    appendReleaseEvent(GATE_BLOCKED) + saveReleaseRegistry
       │                            └─ ReleaseRegistry:281-284 → writeAtomic(<curso>/release.json)  ✓ ESCRIBE
       └─ route.ts:1298    catch → solo warn (best-effort, nunca cambia el outcome)
```

### 2.2 Ejecución aislada del helper (evidencia determinística)

Se ejecutó `recordFailedJobRelease` real (bun, `AICF_COURSE_DATA_DIR` apuntando a un directorio temporal — **sin modificar el proyecto**) con `reasonCode:'no_lessons_generated'`. Resultado en disco: `release.json` con `revisions:[{r1, status:'BLOCKED', evaluation:{premiumRubric:null, commercialReadiness:null, gate:'BLOCKED', gateReasons:['job_status_failed:no_lessons_generated'], …}}]`, `events:[RELEASE_STARTED, GATE_BLOCKED]`, `currentRevision:null`. **El registro se escribe completo y correcto.**

### 2.3 Entonces, ¿por qué "aparece vacío"? — Tres causas concurrentes demostradas

**(1) Vacío informativo por diseño (payload null).** El registro BLOCKED audit-only lleva TODOS los campos de evaluación y empaquetado en null (`premiumRubric:null`, `commercialReadiness:null`, `manifestPinnedSha256:null`, `eligibleFiles:[]`, `evaluationFailed:false`) y su ÚNICA causa es el string `job_status_failed:<reasonCode>`. **No transporta transportClass, proveedor, HTTP status ni Retry-After** — esa evidencia vive en `lesson-generation-failure.json`… que por el bug A no está. El operador que busca la causa en release.json la encuentra "vacía": `no_lessons_generated` es un reasonCode, no una causa raíz.

**(2) Invisibilidad por API — el mecanismo principal.** El endpoint que expone el registro exige manifest.json:

```typescript
// src/app/api/course/[id]/releases/route.ts:211-213
if (!courseManifestExistsForReleases(id)) {
  return NextResponse.json({ error: 'Course not found' }, { status: 404 })
}
```

Y para un fallo `no_lessons_generated`, **manifest.json no existe NUNCA en disco**:

| Escritor de manifest.json | Actúa para no_lessons_generated? | Evidencia |
|---|---|---|
| Worker (artefacto 'manifest.json') | **NO** — solo lo registra para `curriculum_generation_failed` (failureManifest) y para el camino de éxito | CourseWorker:736 y :1754 (únicos dos registros de manifest) |
| Ruta | **NO** — importa `persistArtifact, readCourseFile, getCourseDir`; nunca persiste un manifest | route.ts:74-78; sin `persistCourseManifest` en el archivo |
| CourseReleaseService (pin B4 + mirrorWorkingManifest) | **NO** — toda la fase de release se OMITE para jobs failed (la propia protección R4-F) | route.ts:1289-1306 |

Consecuencia: **el dashboard/API muestra "sin registro de release"** (404 Course not found) aunque `release.json` exista en disco con la revisión BLOCKED — y el operador que abre el archivo en crudo ve el payload de nulls del punto (1). Ambas lecturas producen "el registro aparece vacío".

**(3) Contexto del manifest incompleto (tercer síntoma reportado).** "El manifest no conserva la causa completa": para el curso fallido PRE-R4 del incidente (`course-mvvcmol9-0nez`), el manifest fue escrito por el mirror del release de la era anterior (solo status failed, sin causa de transporte); para cursos fallidos POST-R4 `no_lessons_generated`, **no hay manifest en absoluto**. En ningún caso el manifest lleva transportClass/proveedor/HTTP.

### 2.4 Hipótesis alternativa (documentada, no confirmada)

Si en el despliegue del operador `AICF_COURSE_DATA_DIR` estuviera definido (en ESTA copia NO lo está — verificado en `.env`, `.env.example`, `docker-compose.yml`), `release.json` se escribiría bajo el override mientras el curso se inspecciona en `experiments/fase64/courses/` → síntoma idéntico (registro inexistente en la carpeta inspeccionada). **Dato ausente:** el `.env` real del operador. Acción de verificación de 1 minuto: `grep AICF_COURSE_DATA_DIR .env` en el entorno del operador. No puede descartarse desde aquí; el mecanismo (1)+(2) no depende de ella y ya explica el síntoma por completo.

### 2.5 Por qué los tests de 5E-R4 no lo detectaron

El child `5ER4-PostFailure.child.ts:316-325` verificó que `release.json` EN DISCO contiene exactamente una revisión BLOCKED con `job_status_failed:no_lessons_generated` (aserción correcta — y cierta). **Nunca ejercitó `GET /api/course/[id]/releases`** para ese curso (el endpoint habría respondido 404: el escenario del child tampoco crea manifest.json — reproduce fielmente la forma del incidente, incluida la condición que vuelve invisible el registro). El test probó la ESCRITURA; el síntoma del operador está en la LECTURA.

### 2.6 Diseño de solución (duradera, verificable, SIN tocar CourseWorker.ts)

**Objetivo: un curso fallido debe ser (a) visible, (b) con causa completa, (c) legible por la misma API que los cursos sanos.**

1. **Manifest de fallo desde la ruta** (route.ts — NO FROZEN, ya modificado en R4): en la rama `status === 'failed'`, persistir con `persistArtifact(courseId, 'manifest.json', …)` un failure-manifest con el MISMO formato del bloque `failure` del worker (CourseWorker:725-733, patrón LOTE 1 T3): `reasonCode`, `stage`, `jobId`, y un puntero `diagnostics: 'lesson-generation-failure.json'` cuando exista. Esto: (a) hace el curso visible para el endpoint de releases (courseManifestExistsForReleases → true), (b) cierra el síntoma "el manifest no conserva la causa", (c) replica el patrón ya aprobado del worker para curriculum-failures — sin inventar estados.
   - Nota de integridad: la ruta NO debe escribir el manifest para el caso curriculum_generation_failed (el worker YA registra el suyo; escribirlo dos veces es idempotente por nombre pero el del worker es el autorizado — condición `job.error !== 'curriculum_generation_failed'` si se quiere excribir, o simplemente dejar que el artefacto del worker gane porque ya está en el registry).
2. **Enriquecer el evento GATE_BLOCKED del helper** (FailedJobReleaseRecord.ts — archivo R4, NO FROZEN): añadir al `detail` del evento (campo libre `Record<string, unknown>`, compatible con revisiones futuras del servicio real) la causa resumida leída de `job.error` + `reasonCode` ya existente; y — cuando `lesson-generation-failure.json` exista en el directorio del curso — un `failureDiagnostics: { transportClass, provider, httpStatus, retryAfter }` resumido. El registro revision SIGUE audit-only (no se fabrican evaluaciones); la causa viaja en el EVENTO, no en la evaluación → sin conflicto de schema con `ReleaseRegistry`.
3. **NO relajar el gate del endpoint como primera opción**: escribir el manifest de fallo es preferible a debilitar `courseManifestExistsForReleases` (el gate es una protección de consistencia; cambiar el endpoint para aceptar cursos sin manifest amplía la superficie de "legacy courses" en cada consumidor). Si el operador prefiere no escribir manifests de fallos, la alternativa es que el endpoint acepte `release.json` existente como prueba de curso — decisión suya; se lista como opción B con su riesgo.
4. **Contrato de verificación (tests a añadir cuando el operador autorice):**
   - T-B1: flujo failed (child runner existente) → después del POST, `GET /api/course/[id]/releases` responde 200 con la revisión BLOCKED (hoy: 404).
   - T-B2: `release.json` events incluye `GATE_BLOCKED` con `reasonCode` y (cuando existe) el resumen de diagnóstico con `transportClass`.
   - T-B3: el manifest de fallo existe con `failure.reasonCode = job.error` y NO contiene estados de éxito (no `ready_for_review`).
   - T-B4 (regresión): curso completed → el flujo de release normal sigue intacto (suite P23 existente).
   - T-B5 (regresión): `curriculum_generation_failed` → el manifest del worker del escenario sigue siendo el que persiste (no se duplica con datos distintos).
   - T-B6 (idempotencia/reintento): dos POSTs del mismo job failed → `release.json` acumula r1, r2 (append-only, nunca resetea) y el manifest de fallo no se corrompe.

---

## 3. SÍNTESIS: los dos bugs son UNA sola fractura

| | Bug A | Bug B |
|---|---|---|
| Escritura | Sí (correcta) | Sí (correcta) |
| Destino | Directorio huérfano (ID del LLM) | Directorio correcto |
| Visible en el curso | No | En disco sí; por API no (404) |
| Causa raíz | Identidad: `spec.courseId` ≠ `job.courseId` | Visibilidad: manifest inexistente + payload null por diseño |
| Máscara de test | IDs autoconsistentes en las suites | Solo se probó disco, no el endpoint |
| Patrón Open Source violado | StudyForge `run_id` pre-creado antes del trabajo (metering.py:494-527) | Promptfoo fila ERROR persistente y LEGIBLE (evaluator.ts:1990-2035); LangGraph: el error de la tarea queda escrito en el checkpoint junto a las salidas (_runner.py:575-615) |
| Archivo(s) a modificar (con autorización) | `ContentGenerationService.ts` (catch), opcional `LessonGenerationFailureDiagnostics.ts` (resolver + doble escritura) | `route.ts` (manifest de fallo), `FailedJobReleaseRecord.ts` (enriquecer evento) |
| Archivos FROZEN afectados | **NINGUNO** — CourseWorker no se toca | **NINGUNO** |

**Verificación de "se ejecuta" pedida por el mandato:** demostrada — el log del operador (`lesson_generation_failed` + `transportClass: rate_limited`) es la línea CGS:482 que está DESPUÉS de la persistencia (A), y la ejecución aislada del helper produce el registro completo (B). Lo que el mandato pedía no era "el helper está bien escrito" sino "se ejecuta y sus datos son correctos": se ejecuta; sus datos son correctos; su UBICACIÓN (A) y su LEGIBILIDAD (B) son lo que falla.

---

## 4. COMANDO DE RECUPERACIÓN del diagnóstico perdido (operador, lectura de su entorno)

El registro del incidente con `transportClass: rate_limited` está (si se conservó el árbol) en un directorio huérfano:

```bash
# localizar TODOS los lesson-generation-failure.json y ver su courseId:
find experiments/fase64/courses -maxdepth 2 -name 'lesson-generation-failure.json' \
  -exec sh -c 'echo "== $1"; grep -E "\"courseId\"|\"transportClass\"|\"provider\"|\"jobId\"" "$1"' _ {} \;
# el registro del curso fallido course-mvvcmol9-0nez tendrá un courseId DISTINTO (inventado por el LLM)
```

## 5. LÍMITES DE ESTE DIAGNÓSTICO (datos ausentes)

1. No se puede inspeccionar el curso fallido real del operador (vive en su máquina) — la reconstrucción es estático-analítica + experimental y es DETERMINÍSTICA para A (la divergencia spec/job courseId no tiene excepción) y consistente-para-B (mecanismos 1+2 demostrados; la hipótesis override queda listada con su verificación de 1 minuto).
2. El `.env` real del operador (posible `AICF_COURSE_DATA_DIR`) — dato ausente; no afecta el diagnóstico de esta copia (variable no definida aquí, verificada contra `.env`, `.env.example`, `docker-compose.yml`).
3. No se ejecutó ningún proveedor real, ningún `/health`, ningún cambio de configuración (reglas de protección §8).
