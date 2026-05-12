# Agentic Radar

Un radar semanal con IA para buscar trabajo a escala. Construido con Claude Code.

Hecho por [Francisco de la Cortina](https://www.linkedin.com/in/fdelacortina/). Ocho años en Product Management, los últimos cuatro construyendo agentic AI en SaaS enterprise. Actualmente buscando mi siguiente rol.

> 🌐 [English](README.md) · **Español**

---

## El dolor

Tienes 80 empresas objetivo en tu lista. Cada lunes: abres 80 careers pages, cruzas alertas duplicadas de LinkedIn, pierdes una hora con un rol "remoto" que resulta ser US-only, decides a ojo porque no hay tiempo para investigar el funding, equipo, runway o estrategia de IA de cada empresa. Tres meses así y acabas aplicando al ruido. Eso es gestión del burnout, no estrategia de búsqueda.

## Qué hace

Un cron corre tres comandos cada lunes: escanea el inventario vía APIs de job boards, filtra a PM/Senior PM con ubicación viable, evalúa cada superviviente contra tu perfil en un sub-agente de IA aislado. El lunes por la mañana te encuentras una tabla rankeada con veredictos, los gaps reales del fit, y las preguntas que tienes que validar antes de aplicar. Decides en cinco minutos.

## Lo que recibes

```
## Radar — 2026-05-18

| # | Empresa   | Rol                       | Score | Lectura de la IA                                       |
|---|-----------|---------------------------|-------|--------------------------------------------------------|
| 1 | Lumora    | PM — AI Safety Evals      | 89    | ⭐ Construye el artefacto que entregaste el año pasado. |
| 2 | DataForge | Senior PM — Ingestion     | 82    | Match fuerte. El equipo necesita tu discovery.         |
| 3 | Northwind | PM — Pricing              | 71    | Pivot a fintech. Llama a un recruiter antes de aplicar. |
| 4 | OldCorp   | PM — Internal Tools       | 54    | Descartar. Sin AI, movimiento lateral.                 |
```

Debajo de la tabla, cada fila se expande con el desglose: sub-scores de fit / salud / resiliencia, ventajas top, gaps reales, y las tres preguntas a validar antes de aplicar. Eliges cuáles van a tu pipeline activo. El resto se queda en el log del radar para el próximo lunes.

## Un run real (abril–mayo 2026)

Después de cuatro semanas usando esto, audité mi propio tracker. Los números que me hicieron rediseñar la capa de scan entera:

- **21 URLs marcadas como "activas"** de scans anteriores
- **13 estaban muertas** (62%): redirects a `?error=true` de Greenhouse, Ashby reenviando ofertas cerradas al board root, 404s de Lever
- **3 imposibles de verificar** detrás de anti-bot (Revolut, careers JS de Cloudbeds)
- **5 realmente válidas**, de las cuales 2 eran duplicados del mismo rol

Después de pasar de scraping HTML a APIs JSON y añadir HEAD validation como gate: 11/11 URLs reportadas en el siguiente scan devolvieron HTTP 200 con URL efectiva sin marca de error. Cero falsos positivos en el mes siguiente.

Ese delta del 62% → 0% es por lo que la capa de validación es innegociable. Sin ella, estás aplicando a fantasmas.

## Lo que aún no hace

Algunas cosas que no he resuelto. Listadas por cuánto me joden.

- **Los tenants de Workday bloquean la API CXS después del primer request.** Varias empresas enterprise grandes en mi inventario acaban en zona ciega parcial. El fallback con WebFetch funciona pero es inestable y las URLs requieren verificación manual.
- **Las ofertas que solo están en LinkedIn son invisibles.** No hay API pública. Si una empresa solo postea ahí, el radar no las ve.
- **El filtro de títulos es por keyword, no semántico.** Caza "Principal PM" correctamente, pero un título creativo como "Product Strategy Lead" se le escapa. Filtrado por embeddings está en la lista.
- **No detecto cambios en el JD entre scans.** Si una oferta endurece requisitos o cambia ubicación silenciosamente, no me entero hasta que la abro.
- **Sin señal salarial.** Los JDs no suelen publicar bandas. Enriquecer vía Levels.fyi o similar es el siguiente paso obvio.

## Setup

Unos 30 minutos la primera vez. Ver [SETUP.es.md](SETUP.es.md).

## Para los curiosos

- [docs/architecture.es.md](docs/architecture.es.md) — por qué sub-agentes en vez de un loop grande, economía de tokens, la frontera human-in-the-loop
- [docs/lessons-learned.es.md](docs/lessons-learned.es.md) — 8 bugs concretos que cacé construyendo esto (patrones anti-bot, páginas con JS-rendering, URLs alucinadas)
- [commands/](commands/) — las tres definiciones de slash command (en inglés, estándar OSS)

El patrón de orquestación aplica más allá del job search. Evaluación de proveedores, RFPs, grant calls, CFPs de conferencias, listings inmobiliarios: cualquier sitio con una long tail de opciones que necesita triage consistente con decisión humana al final.

## Licencia

MIT.
