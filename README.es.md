# Agentic Radar

> 🌐 [English](README.md) · **Español**

> Un radar semanal con IA que escanea tus empresas objetivo, encuentra las posiciones adecuadas y te dice cuáles merecen una aplicación — construido con slash commands de Claude Code.

## El problema que resuelve

Tienes una lista curada de 80 empresas en las que te encantaría trabajar. Cada lunes, el mismo ritual:

- Abres 80 páginas de empleo. La mayoría no tienen vacantes de Product Manager. Algunas redirigen a enlaces muertos. Unas pocas tienen 47 ofertas pero solo te importa 1.
- Revisas las alertas de LinkedIn. Te muestran la misma oferta 6 veces a través de diferentes agencias.
- Pierdes una hora con una posición que resulta ser US-only sin opción remota desde España.
- Para cuando terminas la lista, las primeras que revisaste ya tienen nuevas ofertas.
- Decides si aplicar a ojo, porque no hay tiempo para investigar de verdad el funding de la empresa, su equipo, runway o estrategia de IA.

**Tres meses así, y acabas aplicando a lo que más grita.** Eso no es estrategia de búsqueda, es gestión del burnout.

Este radar es lo que construí para escapar de ese bucle.

## Qué hace exactamente

Cada lunes a las 8:00, el sistema:

1. **Escanea** tus empresas objetivo a través de sus APIs de job boards (Greenhouse, Lever, Ashby, Workable, Personio). Valida cada URL antes de reportarla — cero enlaces muertos.
2. **Filtra** títulos que realmente quieres (Product Manager / Senior PM, no Principal/Staff/Director) y ubicaciones que realmente puedes aceptar (España, EMEA-remote, worldwide — nunca US-only).
3. **Evalúa** cada oferta superviviente en paralelo — un subagente de IA aislado consulta el funding, headcount, layoffs y cultura de la empresa, y puntúa el rol contra tu perfil en tres dimensiones: fit, salud financiera y resiliencia frente a IA.
4. **Te entrega una lista rankeada** con veredictos (`Aplicar YA`, `Decidir`, `Descartar`) y los tres gaps reales + tres preguntas reales que hacer antes de aplicar.

Inviertes 5 minutos un lunes por la mañana decidiendo a cuáles ir. El radar hizo las 4 horas de trabajo de campo mientras dormías.

## Cómo se ve (ejemplo ficticio)

Después de un run un lunes, abres Claude Code y encuentras esto en el chat:

```
## Radar — Evaluación 2026-05-18

| # | Empresa    | Rol                               | Score  | Lectura de la IA                                                              |
|---|------------|-----------------------------------|--------|-------------------------------------------------------------------------------|
| 1 | Lumora AI  | PM — AI Safety Evaluations        | 89/100 | ⭐ Aplicar YA — "Construye exactamente el artefacto que entregaste el año pasado." |
| 2 | DataForge  | Senior PM — Streaming Ingestion   | 82/100 | Aplicar — "Match técnico fuerte; el equipo necesita tus skills de customer discovery." |
| 3 | Northwind  | PM — Pricing & Monetization       | 71/100 | Decidir — "Pivot a fintech; merece una call con recruiter antes de invertir cover letter." |
| 4 | Kraken DE  | PM — German Energy Market         | 68/100 | Decidir — "España viable vía EOR, pero la JD pide alemán nativo para discovery." |
| 5 | OldCorp    | PM — Internal Tools               | 54/100 | Descartar — "Sin AI, sin impacto público — estrictamente lateral."           |
```

Y debajo, el bloque detallado para cada uno. El top hit se ve así:

```
### Lumora AI · PM — AI Safety Evaluations  ⭐
- Score Ajustado: 89/100
- Veredicto: Aplicar YA
- Desglose: Fit 91 · Salud 88 · Resiliencia 86

Ventajas diferenciales (top 3):
- Construiste un framework de evaluación de LLMs en tu rol anterior —
  exactamente el artefacto que produce este equipo (red-teaming, eval metrics,
  HITL loops).
- 8 años de B2B enterprise SaaS — hablas el idioma de los buyers de Lumora
  (industrias reguladas, sovereign AI niche).
- Multi-cloud + agentic AI en producción — cubre dos requisitos del rol
  de una sola vez.

Gaps reales (top 3):
- Sin background formal en AI safety research (ni papers ni liderazgo
  de red-team).
- Inglés C1 vs. equipo nativo en comunicación research-heavy.
- Sin experiencia previa como PM dentro de un AI lab frontier (has aplicado
  LLMs, no entrenado foundation models).

Preguntas a validar antes de aplicar:
- ¿Se espera co-autoría de papers de safety o solo product translation
  de research findings?
- ¿Contratación directa en España o vía EOR? (Europe está listado pero
  el país concreto no está confirmado.)
- ¿Reporta a Head of Safety Research o a VP Product?
  → Define el día a día research-led vs. roadmap-led.

URL: https://jobs.lumora-ai.com/jobs/91234567  (HEAD-validada 2026-05-18 08:14)
```

Lees cuatro de estos, decides que el `1, 3` van a tu pipeline activo, y se lo dices al radar. Escribe esos dos en tu tracker; el resto se queda en el log del radar para el recheck de la próxima semana.

**Eso es todo. Ese es el workflow.**

## Por qué funciona

Tres decisiones de diseño, cada una aprendida a la mala:

### 1. Hablar con las APIs, no con las webs

Cada plataforma de hiring importante (Greenhouse, Lever, Ashby, Workable, Personio) expone un feed JSON de posiciones abiertas. Leer la web en su lugar es cómo acabas con URLs alucinadas que parecen correctas pero no existen, y enlaces muertos cacheados por Google hace tres semanas.

### 2. Validar cada URL antes de mostrársela a un humano

Un simple HEAD request, pero hecho estrictamente. Greenhouse redirige silenciosamente jobs eliminados a `?error=true`. Ashby reenvía las ofertas cerradas al board root. Sin validación, ~60% de las ofertas "activas" que se te reportan llevan muertas más de cuatro semanas.

### 3. Usar subagentes para el trabajo profundo, mantener al humano en el loop principal

Cada evaluación de oferta implica 15K tokens de research de empresa, parseo de JD y scoring. Hacer 7 inline destroza la calidad de razonamiento del modelo en todo lo que venga después. Mandar cada uno a un subagente aislado que devuelve solo un bloque de veredicto de 400 palabras mantiene tu contexto principal limpio — y los subagentes corren 3 en paralelo, así que el batch completo tarda ~2 minutos.

El humano recibe el ranking y decide qué entra a su pipeline. La IA nunca auto-aplica, nunca auto-promociona. Esa separación es lo que hace al sistema lo suficientemente confiable para correr semanalmente sin supervisión.

## Cómo replicarlo tú mismo — sin necesidad de programar

Esto está construido para **[Claude Code](https://claude.com/claude-code)**, una app de escritorio de Anthropic que te deja chatear con Claude *y* hacer que ejecute cosas reales en tu computadora — leer archivos, editarlos, correr tareas programadas. Piénsalo como "Claude con manos".

Si puedes escribir una hoja de cálculo de búsqueda de empleo, puedes montar esto. **Tiempo estimado: 30–45 minutos la primera vez. Después, corre solo cada lunes.**

### Paso 1 — Instala Claude Code (5 min)

Descárgalo desde [claude.com/claude-code](https://claude.com/claude-code) e inicia sesión. Es una app normal de Mac/Windows.

### Paso 2 — Añade los tres "skills" (5 min)

Un "slash command" en Claude Code es una instrucción pre-escrita que disparas escribiendo `/algo` en el chat. Este repo te da tres:

1. Abre Claude Code.
2. En el chat, escribe: `/init` — Claude te crea la estructura de carpetas, incluyendo un directorio `~/.claude/commands/`.
3. Descarga los tres archivos dentro de la carpeta [`commands/`](commands/) de este repo (`scan-targets.md`, `evaluate-target.md`, `radar.md`) y déjalos caer en tu carpeta `~/.claude/commands/`.
4. Reinicia Claude Code. Escribe `/` en el chat — deberías ver ahora `/scan-targets`, `/evaluate-target` y `/radar` en la lista.

Eso es todo. Cero código escrito. Los "skills" son simplemente archivos markdown describiendo qué debe hacer Claude; Claude hace el trabajo real.

### Paso 3 — Construye tu lista de objetivos (10 min)

Haz una lista simple de las empresas que quieres que el radar revise cada semana. El formato está en [`examples/targets-inventory.example.md`](examples/targets-inventory.example.md) — una tabla markdown que puedes editar en cualquier editor de texto (o VS Code, o Notion, o directamente en GitHub).

Cada fila necesita solo **nombre de empresa + URL de la página de empleo**. El radar se encarga del resto en su primer run.

Guárdala en algún sitio privado (tu propia carpeta, un Google Doc privado, un repo privado de GitHub). No la metas en la versión pública de este repo — está en el `.gitignore` por esa razón.

### Paso 4 — Dile a Claude qué tipo de rol quieres (5 min)

Este es el único paso de "configuración". Abre `commands/scan-targets.md` en Claude Code y pregúntale en el chat:

> *"En el archivo `commands/scan-targets.md`, cambia el filtro de títulos para que también acepte Principal PM"*

O:

> *"Cambia el filtro de ubicación para que acepte Berlín y Ámsterdam como ubicaciones válidas"*

No tienes que editar el archivo a mano — simplemente le dices a Claude qué quieres y él edita el archivo por ti. (Este es el workflow real para el que la gente usa Claude Code.)

### Paso 5 — Ejecútalo una vez (2 min)

En el chat de Claude Code, escribe: `/radar`

El primer run tarda más porque el sistema tiene que averiguar qué plataforma de job board usa cada empresa. Cuando termine, verás la tabla rankeada — exactamente como la del ejemplo de arriba.

### Paso 6 — Prográmalo semanalmente (1 min)

Escribe en el chat: `/schedule`

Cuando te pregunte, dale:
- **Nombre**: `radar-semanal`
- **Cron**: `0 8 * * 1` (significa "cada lunes a las 8:00")
- **Comando**: `/radar`

Ahora cada lunes por la mañana encontrarás el output del radar esperándote. Inviertes 5 minutos decidiendo a qué ir. Listo.

### ¿Y si me atasco?

Pregunta a Claude directamente en el chat. Ejemplos que funcionan:

> *"El scan devolvió 0 hits para la Empresa X pero sé que están contratando — ¿qué falla?"*

> *"Re-ejecuta el radar pero solo en estas 3 empresas: Lumora, DataForge, Northwind"*

> *"Añade una empresa nueva a mi lista: Atomic AI, página de empleo https://atomic.ai/jobs"*

Los skills están escritos para que Claude pueda arreglarlos, extenderlos y operarlos vía conversación normal. No necesitas aprender YAML, Python ni ningún framework — solo hablar con él.

## Para los curiosos

Si quieres la profundidad técnica:

- [docs/architecture.es.md](docs/architecture.es.md) — por qué subagentes en vez de un loop grande, la economía de tokens, la frontera human-in-the-loop
- [docs/lessons-learned.es.md](docs/lessons-learned.es.md) — los 8 bugs específicos que cacé construyendo esto: patrones anti-bot, páginas con JS-rendering, URLs alucinadas, edge cases en parseo de ubicación
- [commands/](commands/) — las tres definiciones de slash command, completamente anotadas (en inglés — son estándar OSS)

El patrón generaliza más allá de búsqueda de empleo. Donde sea que tengas una long tail de opciones (proveedores, RFPs, grant calls, conference CFPs, listings inmobiliarios) y necesites un triage consistente basado en evidencia con un paso de decisión human-in-the-loop, esta misma arquitectura aplica. Cambia el filtro de títulos, cambia los pesos de scoring, cambia la fuente de inventario — el orquestador se queda igual.

## Licencia

MIT. Forkéalo. Adáptalo. Haz tu propio ritual un poco menos terrible.
