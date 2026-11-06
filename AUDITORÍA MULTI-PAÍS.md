AUDITORÍA MULTI-PAÍS — ¿EXISTE IMPLEMENTACIÓN TEMPLATE-FIRST?
================================================================

### 1. countries[] (ARRAY — multi-país)
  src\app\api\countries\route.ts:23: //   200 — { ok: true, countries: CountryEntry[], count: number }
  src\app\api\countries\route.ts:60: let countries: CountryEntry[]
  src\app\api\countries\route.ts:73: countries = listCountriesByRegion(region)
  src\app\api\countries\route.ts:77: countries = countries.filter((c) => matchCodes.has(c.code))
  src\app\api\countries\route.ts:80: countries = searchCountries(q)
  src\app\api\countries\route.ts:82: countries = COUNTRY_CATALOG_EXTENDED
  src\app\page.tsx:171: const countries = React.useMemo(
  src\lib\runtime\configuration\CourseConfiguration.ts:46: countries: string[]
  src\lib\runtime\configuration\CourseConfiguration.ts:100: geographic: partial?.geographic ?? { countries: ['global'], region: 'Global', localCurrency: 'USD', culturalAdaptation: true, legislation: t
  src\lib\runtime\course\ResearchAcquisitionService.ts:117: function mkProfile(id: string, topic: SourceTopic, includeDomains: string[], country?: string): SourceProfile {
  src\lib\runtime\export\ExportManager.ts:230: countries: countrySets.map((s) => ({ code: s.countryCode, name: s.countryName, slideCount: s.slides.length })),
  src\lib\runtime\export\ExportManager.ts:300: countries: locales.map((l) => l.country.name),
  src\lib\runtime\export\ExportManager.ts:483: countries: locales.map((l) => l.country.name),
  src\lib\runtime\locale\CountryCatalogExtended.ts:1995: export const ADDITIONAL_COUNTRIES: CountryEntry[] = [

### 1b. country (SINGULAR — un solo país)
  src\app\api\course\[id]\pricing\route.ts:103: const manifestCountry = (manifest as CourseManifest & { country?: string }).country
  src\app\api\course\[id]\pricing\route.ts:104: const country = (manifestCountry || 'global').toLowerCase()
  src\app\api\course\blueprint\route.ts:60: const country = url.searchParams.get('country')?.trim().toLowerCase() || ''
  src\app\api\course\blueprint\route.ts:69: `level=${levelRaw ? 'yes' : 'no'}, country=${country ? 'yes' : 'no'}.`,
  src\app\api\courses\route.ts:12: //       country: string | null,        // from manifest.pricing (LOTE F2) — honest null
  src\app\api\courses\route.ts:40: country: string | null
  src\app\api\courses\route.ts:64: pricing?: { country?: string; currency?: string } | null
  src\app\api\courses\route.ts:74: country: typeof manifest.pricing?.country === 'string' && manifest.pricing.country.length > 0
  src\app\api\generate-course\route.ts:208: country: z.string().max(64).default('global'),
  src\app\api\generate-course\route.ts:242: country: z.string().regex(/^[A-Za-z]{2}$/, 'country debe ser un código ISO-3166 alpha-2 de 2 letras (ej: PE, MX, US)'),
  src\app\api\generate-course\route.ts:753: pricing?: { amount: number; currency: string; country: string }
  src\app\api\generate-course\route.ts:759: country: requestData.country,
  src\app\api\generate-course\route.ts:786: country: requestData.pricing.country.toLowerCase(),
  src\app\page.tsx:80: country: string | null
  src\app\page.tsx:182: (countryFilter === 'all' || c.country === countryFilter),
  src\components\course-studio\generate.tsx:807: selected={country === c.code}
  src\components\course-studio\master-productization.tsx:82: country: string
  src\components\course-studio\master-productization.tsx:183: country: string
  src\components\course-studio\master-productization.tsx:989: Country: <span className="font-mono">{result.country}</span> ·
  src\components\course-studio\types.ts:132: country: string

### 2. variantOf / sourceJobId / templateSource
  src\lib\runtime\course\PromptOrchestrator.ts:81: build(templateId: string, variables: PromptVariables): { system: string; user: string } {
  src\lib\runtime\course\PromptOrchestrator.ts:82: const tpl = this.templates.get(templateId)
  src\lib\runtime\course\PromptOrchestrator.ts:83: if (!tpl) throw new Error(`PromptOrchestrator: template "${templateId}" not found`)

### 2b. template / variant (lógica)
  src\lib\runtime\course\LocalizationDerivation.ts:45: isVariant: true
  src\lib\runtime\course\LocalizationDerivation.ts:181: isVariant: true,
  src\lib\runtime\course\LocalizationDerivation.ts:337: if (!variantManifest.isVariant) {
  src\lib\runtime\course\LocalizationDerivation.ts:338: issues.push('Course is not a variant (missing isVariant flag)')
  src\lib\runtime\course\LocalizationDerivation.ts:339: checks.push({ name: 'isVariant', pass: false, detail: 'Course is not marked as a variant' })
  src\lib\runtime\course\LocalizationDerivation.ts:341: checks.push({ name: 'isVariant', pass: true, detail: 'Course has isVariant=true' })

### 3. Archivos CountryCatalog / Localization
  src\lib\runtime\course\LocalizationDerivation.ts (18654 B)
  src\lib\runtime\course\LocalizationProfile.ts (9339 B)
  src\lib\runtime\locale\CountryCatalog.ts (16765 B)
  src\lib\runtime\locale\CountryCatalogExtended.ts (102388 B)
  src\lib\runtime\locale\CountryEngine.ts (2540 B)
  src\lib\runtime\modules\localization\LocalizationModule.ts (11700 B)
  src\lib\runtime\presentation\CountryAdapter.ts (1275 B)
  src\lib\telemetry\LocalOutcomeEmitter.ts (14131 B)

### 4. country en ResearchEngine / ContentGenerator / CurriculumPlanner
  src\app\api\countries\route.ts:4: // Wraps the extended country catalog from
  src\app\api\countries\route.ts:5: // `@/lib/runtime/locale/CountryCatalogExtended`:
  src\app\api\countries\route.ts:6: //   - COUNTRY_CATALOG_EXTENDED  — the full catalog (9 original + 45 extended)
  src\app\api\countries\route.ts:12: //   - q — case-insensitive substring search across country name (es + en),
  src\app\api\countries\route.ts:20: //   - neither given              → COUNTRY_CATALOG_EXTENDED
  src\app\api\countries\route.ts:23: //   200 — { ok: true, countries: CountryEntry[], count: number }
  src\app\api\countries\route.ts:35: COUNTRY_CATALOG_EXTENDED,
  src\app\api\countries\route.ts:38: type CountryEntry,
  src\app\api\countries\route.ts:39: } from '@/lib/runtime/locale/CountryCatalogExtended'
  src\app\api\countries\route.ts:60: let countries: CountryEntry[]
  src\app\api\countries\route.ts:63: if (!VALID_REGIONS.includes(regionRaw as CountryEntry['region'])) {
  src\app\api\countries\route.ts:72: const region = regionRaw as CountryEntry['region']
  src\app\api\countries\route.ts:75: // AND region with search query — intersection by country code.
  src\app\api\countries\route.ts:82: countries = COUNTRY_CATALOG_EXTENDED
  src\app\api\course\[id]\pricing\route.ts:24: //   - country ← manifest.country (or 'global' fallback when absent)
  (si aparece solo como campo de contexto: NO hay adaptación real)

### 5. Moneda / legal / locale por país
  src\app\api\countries\route.ts:5: // `@/lib/runtime/locale/CountryCatalogExtended`:
  src\app\api\countries\route.ts:13: //          ISO code, region, language, currency code, and major companies.
  src\app\api\countries\route.ts:39: } from '@/lib/runtime/locale/CountryCatalogExtended'
  src\app\api\course\[id]\artifact\route.ts:79: requestedRevision = Number.parseInt(revisionHeader, 10)
  src\app\api\course\[id]\artifact\route.ts:155: headers: { 'Content-Type': 'text/plain; charset=utf-8' },
  src\app\api\course\[id]\delivery\preview\route.ts:54: 'Content-Type': 'text/html; charset=utf-8',
  src\app\api\course\[id]\pricing\route.ts:66: function parseBool(raw: string | null): boolean | undefined {
  src\app\api\course\[id]\pricing\route.ts:74: function parseNumber(raw: string | null): number | undefined {
  src\app\api\course\[id]\pricing\route.ts:123: const totalDurationMinutes = parseNumber(url.searchParams.get('totalDurationMinutes'))
  src\app\api\course\[id]\pricing\route.ts:124: const hasCapstone = parseBool(url.searchParams.get('hasCapstone'))
  src\app\api\course\[id]\pricing\route.ts:125: const hasCertification = parseBool(url.searchParams.get('hasCertification'))
  src\app\api\course\[id]\pricing\route.ts:134: const premiumScore = parseNumber(url.searchParams.get('premiumScore'))
  src\app\api\course\[id]\releases\route.ts:14: //         events in append-only chronological order, currentRevision as-is).
  src\app\api\course\[id]\releases\route.ts:24: //       unparseable/structurally invalid. Fail-closed: the corrupt authority
  src\app\api\course\[id]\releases\route.ts:50: //   metadata only (no env vars, no secrets, no ZIP bytes, no absolute paths).
  src\app\api\course\[id]\releases\route.ts:242: // Strict integer parse: '1.5'/'abc'/'0'/'-1' are rejected (Number.parseInt
  src\app\api\course\[id]\releases\route.ts:244: // header parsing on purpose: a query param is user-facing).
  src\app\api\course\[id]\review\route.ts:111: // PUBLISHED — blocked pending real authentication/authorization.
  src\app\api\courses\route.ts:13: //       currency: string | null,       // from manifest.pricing — honest null
  src\app\api\courses\route.ts:41: currency: string | null

### 6. PricingRecommendationEngine / pricing por país
  src\app\api\course\[id]\pricing\route.ts:5: // `@/lib/runtime/course/commercial/PricingRecommendationEngine`.
  src\app\api\course\[id]\pricing\route.ts:27: //   200 — { ok: true, recommendation: PricingRecommendation }
  src\app\api\course\[id]\pricing\route.ts:47: } from '@/lib/runtime/course/commercial/PricingRecommendationEngine'
  src\app\api\generate-course\route.ts:238: // CourseWorker asks the country-aware PricingRecommendationEngine.
  src\app\api\generate-course\route.ts:781: // falls back to the PricingRecommendationEngine.
  src\components\course-studio\generate.tsx:324: // undefined → the worker asks the country-aware PricingRecommendationEngine.
  src\components\course-studio\master-productization.tsx:180: interface PricingRecommendation {
  src\components\course-studio\master-productization.tsx:202: recommendation?: PricingRecommendation
  src\components\course-studio\master-productization.tsx:841: const [result, setResult] = React.useState<PricingRecommendation | null>(null)
  src\components\course-studio\overview.tsx:709: ? `Operador o PricingRecommendationEngine · país ${manifest.pricing.country}`

### 7. Zod schema en generate-course/route.ts
  L43: // Inputs: { title, level, audience, locale, country, premiumLevel, mode, objective }
  L208: country: z.string().max(64).default('global'),
  L237: // ISO-3166 alpha-2 country (2 letters), amount ≥ 0. When absent, the
  L238: // CourseWorker asks the country-aware PricingRecommendationEngine.
  L242: country: z.string().regex(/^[A-Za-z]{2}$/, 'country debe ser un código ISO-3166 alpha-2 de 2 letras (ej: PE, MX, US)'),
  L753: pricing?: { amount: number; currency: string; country: string }
  L759: country: requestData.country,
  L780: // to ISO casing (currency UPPER, country lower). Undefined → the worker
  L786: country: requestData.pricing.country.toLowerCase(),

### 8. CourseContracts.ts — campos country/countries
  L71: country: string

### 9. Tests que mencionan country/countries
  src\lib\runtime\course\blueprint\__tests__\CourseBlueprintEngine.test.ts
  src\lib\runtime\course\commercial\__tests__\PricingRecommendationEngine.test.ts
  src\lib\runtime\course\delivery\__tests__\HTML5LearnerPackageBuilder.test.ts
  src\lib\runtime\course\release\__tests__\CourseReleaseService.test.ts
  src\lib\runtime\course\release\__tests__\F4B10-DeliveryPreview.test.ts
  src\lib\runtime\course\release\__tests__\R3-ReleasesHttpApi.test.ts
  src\lib\runtime\course\__tests__\5ER4-FailureDiagnostics.test.ts
  src\lib\runtime\course\__tests__\5ER4-TransportQuotas.test.ts
  src\lib\runtime\course\__tests__\CourseProductionEngine.test.ts
  src\lib\runtime\course\__tests__\CurriculumPlannerDiagnostics.test.ts
  src\lib\runtime\course\__tests__\F1-S1-DiagnosticIdentity.test.ts
  src\lib\runtime\course\__tests__\S1-Acquisition.test.ts
  src\lib\runtime\course\__tests__\WP1-CurriculumTruncation.test.ts
  src\lib\runtime\course\__tests__\WP21-CurriculumPlanner-Contract.test.ts
  src\lib\runtime\course\__tests__\WP21-ModuleSelectionContract.test.ts

### 10. Artefactos de cursos con country

CourseId             Country                  Countries
--------             -------                  ---------
course-mug4zhpq-sqaj (no country en manifest) (no countries[])
course-mupop49a-purz (no country en manifest) (no countries[])
course-murkhtqw-0v6k (no country en manifest) (no countries[])
course-murktz78-0afa (no country en manifest) (no countries[])



================================================================
  FIN — Interpretación rápida:
  • Sección 1: countries[] vs country → determina si hay multi-país
  • Sección 2: variantOf/sourceJobId → template-first
  • Sección 4-6: adaptación real (moneda/legal/pricing) o solo campo
  • Sección 7-8: qué acepta el contrato real
  • Sección 10: evidencia en cursos reales


=== AUDITORÍA COMPLEMENTARIA — 3 focos de riesgo ===

### Foco 1: ¿Cómo se propaga country al research?
--- ResearchAcquisitionService.ts ---
  L42: locale: string
  L43: country?: string
  L112: country?: string              // ISO-3166 alpha-2 (mayúsculas); undefined = global
  L117: function mkProfile(id: string, topic: SourceTopic, includeDomains: string[], country?: string): SourceProfile {
  L121: ...(country ? { country } : {}),
  L127: // Registro mínimo del spec (LOTE F3 T2). country = undefined → perfil global.
  L174: const COUNTRY_ALIASES: Record<string, string> = {
  L183: export function normalizeCountryCode(country?: string): string | null {
--- ResearchEngine.ts ---
  L48: 1. Identify canonical industry systems, tools, standards, frameworks, or protocols appropriate to the topic. Do NOT mention countries, cities, regions, or regional companies.
  L64: async research(topic: string, locale: string, depth: 'quick' | 'standard' | 'deep' = 'standard'): Promise<ResearchResult> {
  L65: logger.info({ topic, locale, depth }, 'ResearchEngine: starting')
  L76: { role: 'user', content: `Topic: "${topic}"\nLocale: ${locale}\nDepth: ${depth}\n\nReturn JSON only.` },
  L129: return this.deterministicResearch(topic, locale)
  L137: private deterministicResearch(topic: string, locale: string): ResearchResult {
  L151: summary: `This research covers ${topic} comprehensively, addressing key concepts, applications, and learning considerations for the ${locale} locale.`,
--- ResearchSource.ts ---

### Foco 2: ¿PricingRecommendationEngine discrimina por país?
  L2: // AICF V3 — FASE 25B-3 — Pricing Recommendation Engine (country-aware)
  L4: // Country-aware pricing in country currency (NOT simple USD→local conversion).
  L5: // Each country has a base price calibrated to local purchasing power; the
  L17: //   6. Country-aware: each country has a BASE PRICE in local currency,
  L18: //      not a USD conversion. Unknown countries fall back to USD 99 with
  L26: getExtendedCountryByCode,
  L27: type CountryEntry,
  L28: } from '../../locale/CountryCatalogExtended'
  L45: country: string // ISO 3166-1 alpha-2 lowercase
  L68: country: string
  L69: currency: { code: string; symbol: string; formatLocale: string }
  L81: // Country base prices (in LOCAL currency — calibrated to local purchasing power)
  L84: interface CountryBasePricing {
  L86: currency: { code: string; symbol: string; formatLocale: string }
  L96: const COUNTRY_BASE_PRICES: Record<string, CountryBasePricing> = {
  L97: pe: { base: 199, currency: { code: 'PEN', symbol: 'S/', formatLocale: 'es-PE' }, allowDecimalEndings: false, isLatam: true },
  L98: mx: { base: 499, currency: { code: 'MXN', symbol: '$', formatLocale: 'es-MX' }, allowDecimalEndings: false, isLatam: true },
  L99: ar: { base: 8999, currency: { code: 'ARS', symbol: '$', formatLocale: 'es-AR' }, allowDecimalEndings: false, isLatam: true },
  L100: cl: { base: 19900, currency: { code: 'CLP', symbol: '$', formatLocale: 'es-CL' }, allowDecimalEndings: false, isLatam: true },
  L101: co: { base: 99900, currency: { code: 'COP', symbol: '$', formatLocale: 'es-CO' }, allowDecimalEndings: false, isLatam: true },

### Foco 3: ¿CountryCatalog se usa en el pipeline o es decorativo?
  Catálogo: src\lib\runtime\locale\CountryCatalog.ts
  ¿Quién lo importa?
    src\app\api\countries\route.ts
    src\lib\runtime\course\LocalizationProfile.ts
    src\lib\runtime\course\commercial\PricingRecommendationEngine.ts
    src\lib\runtime\course\delivery\HTML5LearnerPackageBuilder.ts
    src\lib\runtime\locale\CountryCatalogExtended.ts
    src\lib\runtime\locale\CountryEngine.ts