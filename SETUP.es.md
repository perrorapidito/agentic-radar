# Setup

> 🌐 [English](SETUP.md) · **Español**

Construido para [Claude Code](https://claude.com/claude-code). Unos 30 minutos la primera vez. Después, el radar corre solo cada lunes.

Si puedes editar una hoja de cálculo, puedes montar esto. No necesitas saber Python ni ningún framework — le hablas a Claude en el chat y Claude edita los archivos por ti.

## 1. Instala Claude Code

Descárgalo desde [claude.com/claude-code](https://claude.com/claude-code). Inicia sesión. App normal de Mac o Windows.

## 2. Añade los tres skills

Un "slash command" en Claude Code es una instrucción pre-escrita que disparas escribiendo `/algo` en el chat. Este repo trae tres.

1. Abre Claude Code.
2. Descarga los tres archivos de [`commands/`](commands/): `scan-targets.md`, `evaluate-target.md`, `radar.md`.
3. Métenlos en `~/.claude/commands/` (Claude Code crea esa carpeta al primer arranque).
4. Reinicia Claude Code. Escribe `/` en el chat. Deberías ver `/scan-targets`, `/evaluate-target`, `/radar` en la lista.

## 3. Construye tu lista de objetivos

Haz una tabla markdown con las empresas que quieres que el radar revise cada semana. El formato vive en [`examples/targets-inventory.example.md`](examples/targets-inventory.example.md). Cada fila necesita solo el nombre de la empresa y la URL de su careers page. El radar averigua qué plataforma de job board usa cada una en el primer run.

Guarda el archivo en algún sitio privado. No lo metas en tu fork público — para eso `.gitignore` excluye `inventory/` por defecto.

## 4. Afina el filtro a tu rol

Abre el chat en Claude Code y dile a Claude qué tipo de rol quieres. Ejemplos que funcionan:

> *"En `commands/scan-targets.md`, acepta también Principal PM en el filtro de títulos."*

> *"Añade Berlín y Ámsterdam como ubicaciones válidas."*

> *"Rechaza cualquier rol que mencione presencial obligatorio."*

No tienes que abrir el archivo. Claude lo edita por ti.

## 5. Ejecútalo

En el chat, escribe `/radar`.

El primer run es más lento porque el sistema tiene que descubrir la plataforma de job board de cada empresa vía WebSearch. Los runs siguientes reutilizan lo que aprendió. Cuando termina, ves la tabla rankeada.

## 6. Prográmalo semanalmente

En el chat, escribe `/schedule`.

Pásale:
- **Nombre**: `radar-semanal`
- **Cron**: `0 8 * * 1` (lunes 8:00, tu zona horaria)
- **Comando**: `/radar`

Listo. Cada lunes por la mañana el output te está esperando.

## Cuando te atasques

Pregunta a Claude en el chat. Ejemplos que funcionan:

> *"El scan devolvió 0 hits para la Empresa X. ¿Qué falla?"*

> *"Re-ejecuta el radar solo para Lumora, DataForge, Northwind."*

> *"Añade una empresa nueva a mi inventario: Atomic AI, careers en https://atomic.ai/jobs"*

Los skills están diseñados para que Claude pueda arreglarlos, extenderlos y operarlos vía conversación normal.
