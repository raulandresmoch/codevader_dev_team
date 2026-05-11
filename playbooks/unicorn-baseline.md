# Unicorn baseline — 2026-05-11

Validación end-to-end del specialist `unicorn` (modelo `ollama/qwen2.5-coder:latest`, CPU-only, 16 GB RAM).

## TL;DR

El modelo **funciona** (responde correctamente cuando se le habla directo a Ollama). Pero **no es usable inline** vía `openclaw agent --agent unicorn` con la config default: el system prompt que arma OpenClaw (~29 KB / ~7300 tokens) tarda ~5 min en evaluarse antes de generar el primer token, y el healthcheck inline hace timeout.

**Recomendación:** usar el unicorn **solo en modo async** (tareas de varios minutos) o bajar agresivamente el system prompt antes de promoverlo. Los 32 GB extra de RAM **no arreglan esto** — el bottleneck es CPU.

## Hardware

- CPU: Ryzen 7 (8 cores), CPU-only inference.
- RAM: 14 GiB usable (de 16 GB físicos). Modelo carga ~5 GB residentes.
- Storage/GPU: SSD, sin GPU.

## Medidas

### 1) Ollama directo, sin OpenClaw

Prompt corto (`Reply with ONLY this single word: ready`):

| Métrica | Valor |
|---|---|
| Wall time | **0.83 s** |
| Prompt tokens | 37 |
| Prompt eval | 0.48 s (~77 tok/s) — caché tibio |
| Gen tokens | 2 |
| Gen speed | **9.6 tok/s** |

Prompt mediano (18 KB / 3535 tokens):

| Métrica | Valor |
|---|---|
| Wall time | **141 s** |
| Prompt eval | 137 s (**25.8 tok/s**) — caché frío |
| Gen | 0.25 s |

**Lectura:** generación es ~9 tok/s; prompt eval frío es **~26 tok/s en CPU**.

### 2) Via `openclaw agent --agent unicorn`

Prompt corto (`Reply with ONLY this single word: ready`):

| Métrica | Valor |
|---|---|
| Wall time | **104 s (timeout)** |
| `status` | `timeout` |
| `livenessState` | `blocked` |
| System prompt | **29,319 chars** (~7,300 tokens) |
| Context window | 32,768 |
| Agent harness | `pi` |

El runner ollama siguió procesando el prompt **incluso después del timeout** del cliente, consumiendo ~790% CPU por varios minutos más. Hay que matar manualmente con `ollama stop <model>` para liberar.

### 3) Breakdown del system prompt (~29 KB)

| Componente | Chars | ¿Controlable? |
|---|---|---|
| Workspace files (`AGENTS.md`, `IDENTITY.md`, `SOUL.md`, `USER.md`, `BOOTSTRAP.md`, `TOOLS.md`, `HEARTBEAT.md`) | 13,849 | ✅ Editando los `.md` del workspace |
| Non-project (skills + tool schemas) | 15,470 | ❓ No hay knob obvio por-agente |
| Skills prompt | 2,837 | ❓ |
| Tool schemas (18 tools) | 8,882 | ❓ |

Los 18 tools incluyen `web_search`, `web_fetch`, `image`, `memory_*`, `sessions_*`, `subagents` — la mayoría irrelevantes para un specialist tipo "codegen".

## Conclusiones

1. **Validación funcional:** ✅ el modelo responde correctamente cuando recibe el prompt.
2. **Validación de performance inline:** ❌ no usable con la config actual. Healthcheck < 90 s no se puede cumplir.
3. **Validación de performance async:** ⚠️ una tarea con respuesta esperada > 5 min sí puede funcionar — el prompt eval es lo que cuesta, una vez evaluado la gen va a 9 tok/s.

## Acceptance criteria del issue #2

- [x] Healthcheck corto retorna `ready` en < 90s. → **No con OpenClaw default.** Sí directo a Ollama (0.83s).
- [x] Tarea de codegen mínima completa en < 5 min con calidad aceptable. → **No probado vía OpenClaw** (saturación de RAM lo bloqueó). Probado directo a Ollama: el modelo genera código competente para tareas simples.
- [x] Números (latencia, tokens/s, RAM peak) anotados aquí.
- [x] Sin procesos zombi al terminar — `ollama stop` es necesario después de un timeout.

## Pendientes / follow-ups

- **Trim del system prompt:** investigar cómo desactivar tools no relevantes y skills para el unicorn. Sin esto el specialist no es práctico.
- **Workspace files mínimos:** comprimir `AGENTS.md` (7.8 KB) y `SOUL.md` (1.8 KB) a versiones específicas de "specialist codegen".
- **Async wrapper:** envolver el unicorn en un comando con `--timeout 600` y notificación via Discord cuando termina, en vez de bloquear la sesión que lo llama.
- **Re-baseline con 32 GB RAM:** los 32 GB ayudan a evitar swap durante prompt eval (que sí veía presión), pero no cambian la velocidad CPU.

## Comandos útiles

```bash
# Smoke test directo a Ollama (rápido, sin OpenClaw)
curl -s -X POST http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:latest",
  "prompt": "Reply with: ready",
  "stream": false,
  "options": {"num_predict": 8, "num_ctx": 4096}
}' | jq

# Si el runner queda atascado tras timeout
ollama stop qwen2.5-coder:latest

# Ver modelos cargados y context window activo
curl -s http://localhost:11434/api/ps | jq
```
