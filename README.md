# /watch — Claude puede "ver" vídeos

> Copia del proyecto original [bradautomates/claude-video](https://github.com/bradautomates/claude-video) (licencia MIT, autor: Bradley Bonanno), guardada para no perderla. La documentación original en inglés está en [README.en.md](README.en.md).

## ¿Qué hace?

**Watch** es un plugin (y "skill") para Claude Code, Codex y otros agentes de IA que les permite **analizar vídeos**. Le das un enlace (YouTube, Vimeo, TikTok, etc.) o un archivo de vídeo local, y el agente puede responder preguntas sobre lo que se ve y se oye en él, con marcas de tiempo.

Funciona de dos maneras (motores):

| Motor | Cómo funciona | Qué necesita |
|---|---|---|
| **gemini** (recomendado) | El modelo de vídeo de Google Gemini ve el vídeo completo (imagen y sonido) y devuelve una respuesta con marcas de tiempo. Los vídeos de YouTube se envían directamente a Google; los archivos locales se suben a Google y se borran después. | Una clave gratuita de [Google AI Studio](https://aistudio.google.com/apikey) (`GEMINI_API_KEY`) |
| **local** | Todo se procesa en tu máquina: descarga el vídeo, extrae fotogramas y obtiene la transcripción. Claude mira los fotogramas y lee la transcripción. | `python3` (3.10+), `ffmpeg` y `yt-dlp`. Sin clave. |

Con `auto` (por defecto) usa Gemini si encuentra la clave; si no, usa el motor local. Nunca cambia de motor por su cuenta si algo falla.

## ¿Qué hace cada parte del repositorio?

| Ruta | Para qué sirve |
|---|---|
| `.claude-plugin/` | Manifiesto del marketplace y del plugin para Claude Code (`watch@claude-video`). |
| `.codex-plugin/` | Manifiesto equivalente para Codex. |
| `hooks/hooks.json` + `hooks/scripts/check-setup.sh` | Al iniciar cada sesión comprueba (sin instalar nada) si faltan dependencias y avisa si hay que configurar algo. |
| `skills/watch/SKILL.md` | Las instrucciones que sigue el agente: primera configuración, preguntas al usuario, cómo ejecutar y cómo responder. |
| `skills/watch/scripts/watch.py` | Programa principal: recibe el vídeo y las opciones, y genera el informe con fotogramas y transcripción. |
| `skills/watch/scripts/download.py` | Descarga el vídeo o solo los subtítulos con `yt-dlp` (admite cookies si el vídeo requiere iniciar sesión). |
| `skills/watch/scripts/frames.py` | Extrae fotogramas con `ffmpeg`: por cambios de escena, fotogramas clave o muestreo uniforme; elimina duplicados casi idénticos. |
| `skills/watch/scripts/transcribe.py` | Obtiene la transcripción: primero subtítulos nativos del vídeo; si no hay, usa el método de respaldo elegido. |
| `skills/watch/scripts/whisper.py` | Transcripción en la nube con Groq (`whisper-large-v3`) u OpenAI (`whisper-1`). |
| `skills/watch/scripts/local_whisperx.py` | Transcripción local con WhisperX, sin claves (el audio no sale de la máquina). |
| `skills/watch/scripts/gemini.py` | Motor Gemini: envía/sube el vídeo a Google, hace la pregunta y borra el archivo subido. |
| `skills/watch/scripts/setup.py` | Asistente de configuración y diagnóstico (`--check`, `--json`, `--engine`, `--backend`, `--detail`, `--install-whisperx`). |
| `skills/watch/scripts/config.py`, `runtime.py` | Lectura de la configuración (`~/.config/watch/.env`) y utilidades comunes. |
| `skills/watch/scripts/build-skill.sh` | Empaqueta la skill en `dist/watch.skill`. |
| `tests/` | Pruebas automáticas (pytest), sin llamadas a servicios reales. |
| `docs/` | Imágenes de la guía de instalación. |
| `dev-sync.sh`, `AGENTS.md`, `CHANGELOG.md` | Herramientas y reglas para desarrolladores, e historial de versiones. |

## Instalación

**Claude Code en terminal** (dentro de una sesión de Claude Code). Para instalar desde esta copia, cambia `bradautomates/claude-video` por `TU_USUARIO/claude-video`:

```text
/plugin marketplace add bradautomates/claude-video
/plugin install watch@claude-video
```

Desde una terminal normal:

```bash
claude plugin marketplace add bradautomates/claude-video
claude plugin install watch@claude-video
```

**Claude Desktop (sesión Code):** Customize → Plugins → Add → Add marketplace → Add from a repository → pegar la URL del repositorio → Sync → instalar **Watch**.

**VS Code:** escribe `/plugins` en el panel de Claude Code → Marketplaces → añadir el repositorio → instalar **watch**.

**Codex:** pide en el chat: `Use $skill-installer to install the watch skill from: <url>/tree/main/skills/watch`

**Otros agentes:** `npx skills add bradautomates/claude-video -g --skill watch`

> No funciona en Claude Chat ni en Cowork. En Claude Code en la web funciona, pero la instalación se pierde al cerrar la sesión.

### Dependencias

- **macOS:** `brew install python ffmpeg yt-dlp`
- **Ubuntu/Debian:** `sudo apt install python3 ffmpeg pipx` y luego `pipx install "yt-dlp[default,curl-cffi]"`. Para YouTube, instala también [Deno](https://docs.deno.com/runtime/getting_started/installation/).
- **Windows:** Python 3.10+, y `winget install --id Gyan.FFmpeg --exact`, `winget install --id yt-dlp.yt-dlp --exact`, `winget install --id DenoLand.Deno --exact`.

Mantén `yt-dlp` siempre en su última versión: los sitios de vídeo cambian a menudo y las versiones antiguas dejan de funcionar.

## Uso

Escribe `/watch` (en Claude Code aparece como `/watch:watch`) o pide al agente que use la skill watch:

```text
/watch https://www.youtube.com/watch?v=XXXX ¿de qué trata este vídeo?
/watch video.mp4 --start 2:15 --end 2:45
/watch video.mp4 --detail efficient --max-frames 30
/watch video.mp4 --detail transcript --timestamps 1:05,2:30
/watch https://ejemplo.com/video --cookies-from-browser firefox
```

La primera vez, el asistente pregunta:

1. **Motor:** `gemini` o `local`.
2. **Nivel de detalle** (solo local):
   - `transcript`: solo texto, sin fotogramas.
   - `efficient`: fotogramas clave, máximo 50.
   - `balanced` (recomendado): cambios de escena, máximo 100.
   - `token-burner`: sin límite; consume mucho.
3. **Transcripción de respaldo** para vídeos sin subtítulos:
   - `whisperx`: local, ~1,5 GB de descarga, 8 GB de RAM.
   - `groq`: necesita `GROQ_API_KEY`.
   - `openai`: necesita `OPENAI_API_KEY`.
   - `none`: solo subtítulos.

Opciones útiles:

- `--start` / `--end`: analizar solo un tramo.
- `--resolution 1024`: leer mejor el texto en pantalla.
- `--no-dedup`: conservar fotogramas casi iguales.
- `--sub-lang es`: elegir el idioma de los subtítulos.
- `--engine local`: forzar el motor local.
- `--whisper` / `--no-whisper`: elegir o desactivar la transcripción de respaldo.

Para audio en español con WhisperX, añade `WATCH_WHISPERX_LANGUAGE=es` en `~/.config/watch/.env`.

## Configuración y privacidad

- Ajustes y claves en `~/.config/watch/.env` (permisos 0600). **No subas claves al repositorio.**
- Las claves nunca se escriben en los registros ni en los informes.
- Cada análisis usa una carpeta temporal que se borra al terminar; tus archivos originales no se tocan.
- El contenido del vídeo se trata como información, nunca como instrucciones a ejecutar.

## Actualizar

`/plugin update watch@claude-video` dentro de Claude Code. Diagnóstico: pide al agente que ejecute `setup.py --json`.

## Licencia

MIT — © Bradley Bonanno. Ver [LICENSE](LICENSE).
