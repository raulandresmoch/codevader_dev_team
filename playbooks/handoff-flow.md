# Handoff flow — main → unicorn (Discord #handoffs)

Cómo funciona hoy el ruteo Discord → specialist y cómo replicarlo cuando agreguemos un segundo specialist.

## Resumen

- Tú escribes en `#handoffs` (canal Discord id `1503208090487750746`) → mensaje llega directo al specialist `unicorn`, **no** pasa por `main` (Cerebrin).
- `unicorn` corre **Claude Haiku 4.5 vía claude-cli** → usa la cuota del plan Max, no API tokens.
- Latencia esperada: ~15 s para tickets de codegen pequeños.
- Output: archivos en `~/.openclaw/specialists/unicorn-workspace/`.

Cualquier otro canal/peer sigue con `main`.

## Componentes

```
Discord #handoffs (1503208090487750746)
      │
      ▼
OpenClaw gateway (binding: peer.kind=channel, peer.id=…)
      │
      ▼
agent:unicorn  ──► claude-cli (auth: Claude Code / Max plan)
                        │
                        ▼
                  claude-haiku-4-5
                        │
                        ▼
                 workspace ~/.openclaw/specialists/unicorn-workspace/
```

## Setup paso a paso (para el próximo specialist)

### 1) Crear el agente

```bash
openclaw agents add \
  --id unicorn2 \
  --workspace ~/.openclaw/specialists/unicorn2-workspace \
  --agent-dir ~/.openclaw/specialists/unicorn2-state \
  --model anthropic/claude-haiku-4-5
```

Esto deja el agente listo, ya con el `claude-cli` runtime por default (heredado de `defaults.agentRuntime`).

### 2) Crear el canal Discord dedicado

En el server "Cerebrin & Team", crear un canal de texto (ej. `#handoffs-research`). Anotar el channel id (botón derecho → "Copy Channel ID" con dev mode activo).

### 3) Agregar el binding al routing

⚠️ La CLI `openclaw agents bind --peer ...` actualmente mete el sufijo como `accountId` en vez de `peer.id`. Toca editar `openclaw.json` a mano. Estructura:

```json
{
  "bindings": [
    {
      "type": "route",
      "agentId": "unicorn",
      "match": {
        "channel": "discord",
        "peer": { "kind": "channel", "id": "1503208090487750746" }
      }
    },
    {
      "type": "route",
      "agentId": "unicorn2",
      "match": {
        "channel": "discord",
        "peer": { "kind": "channel", "id": "<NEW_CHANNEL_ID>" }
      }
    }
  ]
}
```

### 4) Reiniciar el gateway

**Esto es obligatorio.** Sin reinicio, los bindings nuevos no se cargan aunque `openclaw agents bindings` los muestre.

```bash
openclaw gateway restart
```

Downtime: ~10 s. Durante esos segundos Discord/Telegram no responden a nadie.

### 5) Validar

Manda un mensaje al canal nuevo. Después de 5–20 s, debe responder. Para verificar que cayó en el specialist (no en main):

```bash
# El último update en sessions.json del specialist debe ser reciente
python3 -c "
import json,datetime
d=json.load(open('/home/cerebro/.openclaw/agents/unicorn2/sessions/sessions.json'))
for k,v in d.items():
    ts=v.get('updatedAt',0)
    dt=datetime.datetime.fromtimestamp(ts/1000, datetime.timezone.utc).isoformat()
    print(f'{k}  updated={dt}')
"

# Y en el gateway log debe aparecer una línea como:
#   cli exec: provider=claude-cli model=claude-haiku-4-5 promptChars=… trigger=user
journalctl --user -u openclaw-gateway -n 50 --no-pager | grep claude-haiku
```

Si en su lugar la respuesta cayó en `agent:main:discord:channel:<id>`, el binding **no se aplicó** — generalmente porque el gateway no se reinició después de editar `openclaw.json`.

## Gotchas observadas

- **Restart obligatorio.** Editar `openclaw.json` y mirar `openclaw agents bindings` muestra el binding en la lista, pero **el gateway no recarga** sin restart. El primer test del `#handoffs` cayó en `main` por esta razón (gateway corría desde antes del binding).
- **Token rotation por canal.** Cada agente mantiene su propia sesión claude-cli por peer (`agent:unicorn:discord:channel:<id>`). Si el agente "olvida" el contexto, revisar que la sesión correcta esté siendo reusada en los logs (`useResume=true session=present`).
- **Filenames colisionan.** Si dos tickets distintos te piden `app.py`, el segundo sobrescribe al primero en el workspace del specialist. Conviene que el ticket incluya en qué subcarpeta trabajar (ej. `tickets/<n>/app.py`).
- **Workspace dirs en claude-cli.** El claude-cli almacena la sesión real en `~/.claude/projects/<workspace-path-mangled>/<sessionId>.jsonl`. Para unicorn: `~/.claude/projects/-home-cerebro--openclaw-specialists-unicorn-workspace/`.

## Verificado el 2026-05-11

- Mensaje a `#handoffs`: "haz una calculadora simple en Python con tests"
- Routing: ✅ fired (después del restart)
- Modelo activo: `claude-haiku-4-5`
- Duración: 14.8 s
- Output: `Calculator` class + 5 pytest tests en el workspace del unicorn
- No consumió API tokens (claude-cli usa auth del plan Max)
