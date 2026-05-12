# Agentic Radar

Un radar semanal con IA para buscar trabajo a escala. Construido con Claude Code.

Hecho por [Francisco de la Cortina](https://www.linkedin.com/in/fdelacortina/). Ocho años en Product Management, los últimos cuatro construyendo agentic AI en SaaS enterprise. 

> 🌐 [English](README.md) · **Español**

---

## El dolor

Tienes 80 empresas objetivo en tu lista. Cada lunes: abres 80 careers pages, cruzas alertas duplicadas de LinkedIn, pierdes una hora con un rol "remoto" que resulta ser US-only, decides a ojo porque no hay tiempo para investigar el funding, equipo, runway o estrategia de IA de cada empresa. Tres meses así y acabas aplicando al ruido. Eso es gestión del burnout, no estrategia de búsqueda.

## Qué hace

Un cron corre tres comandos cada lunes: escanea el inventario vía APIs de job boards, filtra a PM/Senior PM con ubicación viable, evalúa cada superviviente contra tu perfil en un sub-agente de IA aislado. El lunes por la mañana te encuentras una tabla rankeada con veredictos, los gaps reales del fit, y las preguntas que tienes que validar antes de aplicar. Decides en cinco minutos.

## Lo que recibes

Un lunes típico, el radar escanea 80 empresas, encuentra 22 con ofertas de PM o Senior PM abiertas, filtra 11 que son viables desde España, valida que 7 estén realmente activas, y te muestra las 4 mejores rankeadas:

```
## Radar — 2026-05-18

| # | Empresa   | Rol                       | Score | Lectura de la IA                                       |
|---|-----------|---------------------------|-------|--------------------------------------------------------|
| 1 | Lumora    | PM — AI Safety Evals      | 89    | ⭐ Construye el artefacto que entregaste el año pasado. |
| 2 | DataForge | Senior PM — Ingestion     | 82    | Match fuerte. El equipo necesita tu discovery.         |
| 3 | Northwind | PM — Pricing              | 71    | Pivot a fintech. Llama a un recruiter antes de aplicar. |
| 4 | OldCorp   | PM — Internal Tools       | 54    | Descartar. Sin AI, movimiento lateral.                 |
```

Cada fila se expande con el desglose completo. Por ejemplo, la posición #1:

```
### Lumora · PM — AI Safety Evaluations  ⭐

Score Ajustado: 89/100  ·  Veredicto: Aplicar YA
Desglose: Fit 91 · Salud financiera 88 · Resiliencia frente a IA 86

Ventajas diferenciales que te separan del resto:
  · Construiste un framework de evaluación de LLMs en tu rol anterior —
    exactamente el artefacto que produce este equipo.
  · 8 años en B2B enterprise SaaS — hablas el idioma de los clientes
    regulados de Lumora.
  · Agentic AI en producción — cubre dos requisitos del rol de una vez.

Gaps reales del fit (no se compensan con cover letter):
  · Sin background formal en AI safety research (no papers ni red-team
    leadership).
  · Inglés C1 frente a equipo nativo en comunicación research-heavy.
  · Sin experiencia previa como PM dentro de un AI lab frontier.

Preguntas a validar antes de aplicar:
  · ¿Se espera co-autoría de papers de safety, o solo product translation
    de findings de research?
  · ¿Contratación directa desde España o vía EOR? El JD dice "Europe"
    pero no especifica país concreto.
  · ¿Reporta a Head of Safety Research o a VP Product? Define el día a
    día research-led vs. roadmap-led.

URL: https://jobs.lumora-ai.com/jobs/91234567
```

Eliges cuáles van a tu pipeline activo. El resto se queda en el log del radar para la próxima semana.

## Funcionando contra el mundo real (abril–mayo 2026)

Después de cuatro semanas usando el sistema, hice limpieza. De 21 ofertas que tenía marcadas como activas en mi tracker, 13 estaban muertas: la empresa ya había cerrado la posición pero su web seguía mostrándola. De las que quedaban, 3 ni siquiera podía verificar porque las webs me bloqueaban el acceso, y 2 eran la misma oferta repetida.

Casi dos tercios de lo que tenía era ruido bien presentado.

Eso es lo que me hizo rediseñar la capa de validación entera. Después del cambio, en el mes siguiente, ni un solo enlace muerto.

Sin esa capa, te pasas la semana aplicando a fantasmas: ofertas que existen en Google pero ya no en la realidad.

## Setup

Unos 30 minutos la primera vez. Ver [SETUP.es.md](SETUP.es.md).

## Para los curiosos

- [docs/architecture.es.md](docs/architecture.es.md) — por qué sub-agentes en vez de un loop grande, economía de tokens, la frontera human-in-the-loop
- [commands/](commands/) — las tres definiciones de slash command (en inglés, estándar OSS)

El patrón de orquestación aplica más allá del job search. Product discovery (priorizar 50 candidatos a entrevista), evaluación de proveedores, RFPs, grant calls, CFPs de conferencias, listings inmobiliarios: cualquier sitio con una long tail de opciones que necesita triage consistente con decisión humana al final.

## Licencia

MIT.
