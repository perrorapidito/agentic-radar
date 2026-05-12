# Lecciones aprendidas

> 🌐 [English](lessons-learned.md) · **Español**

Bugs reales y pivotes de diseño que aparecieron construyendo esto. El tipo de detalle que solo obtienes ejecutando contra APIs en vivo.

## Ashby renderiza con JavaScript

El payload HTML en `jobs.ashbyhq.com/{slug}` es un esqueleto. Los listings reales vienen de `https://api.ashbyhq.com/posting-api/job-board/{slug}?includeCompensation=true`. WebFetch sobre la careers page devuelve nada útil — ve a la API JSON.

## El `locationName` de Ashby suele estar vacío

La ubicación estructurada vive en `workplaceType` (string: "Remote" / "Hybrid" / "On-site") + `secondaryLocations[].location` (lista de ciudades/regiones) + `isRemote` (booleano). Un filtro que solo revisa `locationName` descarta hits remotos legítimos. Construye la cadena de ubicación a partir de los cuatro campos y corre el verdict contra la forma concatenada.

## El endpoint v3 de Workable devuelve 404

El path `/api/v3/accounts/{slug}/jobs` está documentado en tutoriales viejos pero ya no existe. El endpoint que funciona es:

```
GET https://apply.workable.com/api/v1/widget/accounts/{slug}
```

Devuelve `{name, description, jobs: [{title, location, country, shortcode, ...}]}`. La URL del job se construye como `https://apply.workable.com/{slug}/j/{shortcode}`.

## Greenhouse redirige jobs eliminados a `{slug}?error=true`

Un request a un job ID eliminado devuelve 200 OK con content-type `text/html`, pero la URL efectiva cambió a `https://job-boards.greenhouse.io/{slug}?error=true`. HEAD-validar solo el status code acepta jobs muertos como vivos. Valida la *URL efectiva*:

```bash
curl -sL -o /dev/null -w "%{http_code}|%{url_effective}" --max-time 12 "$url"
```

Comprueba que `%{url_effective}` no contiene `?error=true` y no equivale al root del board.

## La API CXS de Workday es agresiva con anti-bot

Observado a través de múltiples tenants de Workday en los dominios `*.myworkdayjobs.com` y `wd3.myworkdaysite.com`:

- El primer POST a `/wday/cxs/{tenant}/{site}/jobs` con body JSON devuelve `jobPostings` válidos.
- El segundo POST devuelve `errorCode: HTTP_400` o `HTTP_422`.
- Adivinar el nombre del site también es poco fiable — valores comunes (`External`, `External_Careers`, `External_Career_Site`, el nombre de la empresa) pueden ser cada uno válidos para algunos tenants y 404 para otros.

Para uso en producción, cae a WebFetch con un prompt estricto en vez de depender del CXS.

## Requests API en paralelo por encima de ~5 disparan 403 en Ashby

Ejecutar 25 slugs de Ashby en paralelo vía `urllib` de Python produce respuestas 403. Los primeros ~5 tienen éxito, el resto se bloquea. Con header `User-Agent: Mozilla/5.0` y `sleep 1` entre requests, los 25 tienen éxito. Las otras ATSes (Greenhouse, Lever, Workable) toleran tandas paralelas de 5+ sin problema.

## Las cadenas de ubicación requieren matching de keywords, no regex

Cadenas de ubicación reales:
- `"Remote, US-Southeast"` — US-only a pesar de la palabra "Remote"
- `"Amsterdam, Netherlands"` — on-site Amsterdam
- `"Netherlands (remote)"` — remoto en cualquier parte de NL
- `"United Kingdom (remote)"` — remoto en cualquier parte de UK
- `"Hybrid | Remote | London | New York | Montreal"` — campos divididos concatenados
- `""` — vacío (Ashby)
- `"Remote | Remote | Europe"` — elegible Europa

Una sola regex no puede manejar esta taxonomía. Mejor: listas explícitas de keywords con precedencia:

1. Marcadores fuertes "elegible en todas partes" (`spain`, `europe`, `emea`, `worldwide`, `anywhere`) → OK
2. País/ciudad EMEA único + keyword `remote` → contractor-viable (EMEA_CTRY)
3. Ciudad EMEA única sin `remote` → solo on-site (NO)
4. Marcadores de ciudades US/Canada/India/APAC → NO
5. `remote` solo sin región → AMBIG (el usuario verifica)

## Alucinación de URLs — el modo de fallo silencioso

Antes de HEAD validation: cuando la IA construye reports a partir de HTML scrapeado, ocasionalmente infiere un patrón URL plausible (e.g. `boards.greenhouse.io/{slug}/jobs/{some-id}`) en vez de extraer el campo `absolute_url` real. ~10% de estas URLs inferidas no existen, y el report manda enlaces muertos.

Dos defensas combinadas eliminan esto:
1. Usa la API JSON y copia el campo `absolute_url` / `hostedUrl` / `jobUrl` / `applyUrl` **verbatim**. Nunca construyas.
2. HEAD-valida cada URL antes de reportarla.

## El retraso de WebSearch es de 1–4 semanas

Un approach naive común: WebSearch `"Empresa X product manager job"` → click en el primer resultado. Parece correcto en el momento, pero Google indexó esa página cuando la oferta estaba activa. Para cuando la fetcheas, la oferta puede llevar cerrada una semana o más.

WebSearch solo es útil para **discovery de slugs** (qué ATS usa la empresa — señal estable), nunca para encontrar URLs específicas de jobs.

## "Senior Technical PM" no es "Senior Program Manager"

Cuando construyes la block-list de títulos, `technical program` (TPM) es un rol distinto a `technical product` (un rol real de PM senior). Un filtro naive que bloquea `technical` descarta títulos legítimos como "Senior Technical Product Manager". Usa matching a nivel de frase, no a nivel de palabra.

## El aislamiento de contexto de subagentes es la victoria real

Correr 7 evaluaciones profundas inline en el contexto principal: ~50K tokens de WebSearch+JD+scoring contaminando razonamiento subsecuente.

Correrlas como 7 subagentes en tandas de 3 en paralelo: ~3K tokens en contexto principal (solo los bloques de veredicto), 50K tokens total spend pero distribuidos en contextos aislados que se garbage-collectean.

El coste en tokens es similar; la *calidad del razonamiento subsecuente* en el thread principal es dramáticamente mejor porque el contexto no está saturado con output crudo de scraping. Ese es el valor real de la orquestación con subagentes para evaluación long-tail.

## Human-in-the-loop para estado irreversible

El orquestador puede automatizar totalmente las operaciones de **lectura** (scan, evaluate, rank). No puede — y no debería — automatizar totalmente la **escritura al pipeline** porque el estado del pipeline es lo que actúa el humano. Una entrada auto-añadida no se puede des-añadir en la cabeza del usuario; un falso negativo es invisible.

Una invariante útil: los subagentes devuelven bloques de veredicto; el orquestador presenta una lista rankeada; el humano dice `1,3,5`; solo entonces se escribe el pipeline. Este es el inverso de cómo la mayoría de pitches de "AI agent" lo enmarcan (autonomía total), y es lo que hace al sistema lo suficientemente confiable para correr semanalmente.
