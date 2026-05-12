# Arquitectura

> 🌐 [English](architecture.md) · **Español**

## El problema

Tienes un universo grande y de cambio lento de opciones (empresas, proveedores, RFPs, grants, candidatos a entrevista para product discovery, listings) y necesitas:

1. Descubrir qué opciones tienen **oportunidades activas** ahora mismo.
2. Filtrarlas por **restricciones objetivas** (ubicación, título del rol, deadline).
3. Puntuar a las supervivientes contra un **perfil personal** para decidir con cuáles engancharte.

Hacer esto a mano contra 80+ targets semanalmente quema 4–6 horas y produce data stale — para cuando terminas de scrollear careers pages, las primeras que revisaste ya cambiaron.

Hacerlo con una sola llamada de "agente" LLM tampoco escala: cada evaluación de oportunidad son 5–15K tokens (research de empresa + parseo de JD + scoring), y 7 evaluaciones metidas en un contexto degradan la calidad del razonamiento y revientan la ventana de contexto.

## El patrón

Tres capas, cada una con un contrato claro:

### Capa 1 — Discovery (`scan-targets`)

Un fan-out que, para cada target en el inventario, consulta la fuente más autoritativa disponible — típicamente la API JSON del ATS de la careers page de la empresa. Devuelve una lista estructurada de oportunidades con URLs verbatim. Valida cada URL con un HEAD request antes de reportarla.

Propiedad clave: **la capa de discovery nunca inventa data**. Las URLs vienen de campos JSON copiados verbatim. Las ubicaciones vienen de campos estructurados. La única inferencia de IA ocurre en la clasificación de títulos y la categorización de ubicación, ambas acotadas a matching de keyword-lists.

### Capa 2 — Filtro (dentro de `scan-targets`)

Dos filtros apilados:

1. **Filtro de título** — inclusión acotada (`product manager`) más una block-list explícita (`principal`, `staff`, `lead`, `group`, `marketing`, `technical program`, `intern`, ...). Este es el lugar para afinar qué cuenta como "in scope" para tu role band concreto.
2. **Filtro de ubicación** — un veredicto en {OK, EMEA_CTRY, AMBIG, NO}, calculado por precedencia de keywords (marcadores fuertes → contractor-viable → ambiguo → reject). El veredicto guía el comportamiento downstream: OK y EMEA_CTRY sobreviven a evaluación, AMBIG va a un lane separado "verificar manualmente", NO se descarta.

### Capa 3 — Evaluación (`evaluate-target`, orquestado por `radar`)

Para cada superviviente del filtro, correr una evaluación profunda en un **contexto de subagente separado**:

- Cargar el perfil del usuario
- Reusar cualquier ficha previa de la empresa (grep al archivo de historial)
- Fetch del contenido de la oportunidad (JD)
- Puntuar contra cuatro dimensiones: Fit, Salud, Resiliencia, con pesos explícitos
- Calcular Score Ajustado y un veredicto-banda (Aplicar YA / Aplicar / Decidir / Descartar)
- Devolver un bloque de 400 palabras

El orquestador (`radar`) lanza estos subagentes en **tandas de 3 en paralelo**, espera a que cada tanda complete, luego continúa. El contexto principal solo ve los bloques de 400 palabras devueltos, nunca el research crudo.

## Por qué subagentes y no un loop único

Una alternativa natural: un solo agente que loopea `for opportunity in opportunities: evaluate(opportunity)`. Tiene tres problemas:

1. **Ventana de contexto**. Después de ~5 evaluaciones con JD crudo + resultados de WebSearch, el contexto está saturado y el modelo empieza a perder contexto anterior.
2. **Calidad del razonamiento**. Incluso antes de la saturación, contexto histórico irrelevante (el JD de la oportunidad #1) degrada el razonamiento sobre la oportunidad #5. El modelo se distrae con patrones que no debería generalizar.
3. **Recuperación**. Si una evaluación falla (timeout, 403, JD mal formada), contamina el resto. Con subagentes, los fallos están aislados; el orquestador marca uno como `❓ Error` y continúa.

Tandas de 3 son un sweet spot empírico: paralelismo sin presión de rate-limit sobre WebSearch y APIs de tools.

## Por qué human-in-the-loop para la escritura del pipeline

Leer es reversible: si el scan se pierde una oportunidad, la verás la próxima semana. Escribir a tu pipeline activo **no** lo es: una entrada auto-añadida se vuelve parte de tu contexto de decisión, y un add equivocado sesga silenciosamente tu juicio futuro.

El patrón separa explícitamente los dos:
- `scan-targets` escribe a `active-opportunities.md` automáticamente (la lista de radar, bajo coste de falsos positivos porque la puedes scrollear)
- `radar` escribe al **archivo de pipeline** (las cosas que realmente persigues) solo tras confirmación explícita del usuario

Esto es el inverso del framing "agente totalmente autónomo" común en demos de AI agents. Es lo que hace al sistema lo suficientemente confiable para programar semanalmente sin supervisión.

## Economía de tokens

Empírica, de una semana real de scan:

| Stage | Dónde | Tokens |
|---|---|---|
| Discovery (scan-targets) | contexto principal | ~5K |
| Eval profunda por oportunidad (subagente) | contexto aislado | ~15K × N |
| Bloque de veredicto devuelto al main | contexto principal | ~400 palabras × N ≈ ~3K |

Para N=7 hits: ~5K + 7×~15K + ~3K = **~113K tokens totales**, pero solo **~8K visibles en el contexto principal**. El contexto principal se mantiene limpio para razonamiento de follow-up ("¿cuál es la mejor para aplicar primero?").

Sin subagentes: los mismos ~113K tokens pero todos en un contexto → calidad de razonamiento degradada, especialmente en el último 30% de la cadena.

## Tradeoffs a saber

- **El coste en tokens es similar a inline; la ganancia está en calidad de razonamiento, no en coste.** No te prometas a ti mismo que vas a ahorrar tokens yendo subagente — no es ahí donde está el valor.
- **Las APIs JSON evolucionan.** Workable cambió su endpoint path entre versiones. Planifica re-validación periódica de handlers.
- **El anti-bot es el asesino silencioso.** Workday devuelve 200 OK y luego falla en el siguiente request. Greenhouse redirige silenciosamente a `?error=true`. Construye el gate de validación desde el principio, no como afterthought.
- **Los filtros son específicos de dominio.** La block-list de títulos y las listas de keywords de ubicación en este repo están afinadas para "búsqueda de AI PM basado en España late 2025/early 2026". Se pudrirán. Trátalos como configuración, no framework.
- **El patrón no ayuda en evaluación one-shot.** Si tienes 1 oportunidad para evaluar, hazlo inline. El overhead de orquestación solo paga a N ≥ 5.
