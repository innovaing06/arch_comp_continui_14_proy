# AICF v3 — Auditoría Comparativa con 14 Proyectos Open Source (Cierre)

**Fecha:** 2026-10-31 (Lima) · **Modo:** SOLO LECTURA — investigación y diseño. Sin implementación, sin commits, sin cambios a AICF, sin llamadas a proveedores.

## Contenido

| Archivo | Contenido |
|---|---|
| `14_AUDITORIA_COMPARATIVA_AICF.md` | Comparativa AICF v3 vs 14 proyectos por problema P1–P5, con citas exactas de extractos (archivo:línea), veredicto por área, 10 anti-patrones auditados, 10 gaps (G-1..G-10), lo que AICF ya hace mejor, y hechos verificados vs inferidos vs ausentes |
| `15_PLAN_PRIORIZADO_DE_SOLUCIONES.md` | Matriz S-1..S-12 (problema | cita ZIP | mecanismo | adaptación original | archivo destino | riesgos | prioridad), orden de implementación F1–F4, contratos de test T-A/T-B, inventario de archivos que requieren autorización |
| `16_DIAGNOSTICO_PERSISTENCIA_5E_R4.md` | Diagnóstico end-to-end de los 2 bugs 5E-R4: bug A (doble identidad courseId → directorio huérfano) y bug B (registro BLOCKED invisible por API + payload null). Con traza archivo:línea, demostración experimental, comando de recuperación y soluciones sin tocar CourseWorker.ts |
| `17_ESTRATEGIA_PROVEEDORES_Y_CUOTAS.md` | Estado verificado de config V30 + LiteLLM 1.84.0 real, cuotas del incidente (TPM/OTPM/ITPM/RPM), comparativa de las 5 opciones de control de proveedores, cuotas recomendadas (método), mapa de capas |

## Verificación previa (preflight)

- Base: copia Z.ai `/tmp/my-project` post-5E-R4 (sin .git). Hashes: CGS `d6d5bad7…`, route `3d146a11…`, LGFD `df271f18…`, FJRR `7adaacd7…`, ContentGenerator `af851377…` (excepción Lote 5 intacta). 8/8 protegidos intactos.
- LiteLLM verificado contra el tag oficial v1.84.0 (el instalado), no contra el snapshot 1.109.0 de los extractos.
- Licencias 14/14 verificadas (08_CIERRE); Curriculum CC BY-NC-ND VETADO; cero código de terceros copiado.

## Reglas respetadas

7 FROZEN + ContentGenerator intactos · litellm-config.yaml / docker-compose / .env intactos · sin gates/thresholds/scoring modificados · sin proveedores reales · sin /health · sin instalación de dependencias · sin P2 ni Lote 6 iniciados · este paquete es SOLO DISEÑO y requiere autorización explícita del operador para cualquier implementación.
