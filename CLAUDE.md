# BlorkoBot

BlorkoBot is a bot for [MeshCore](https://github.com/meshcore-dev/MeshCore), a mesh radio service.

We use [Remote Terminal for MeshCore](https://github.com/nicholasgasior/Remote-Terminal-for-MeshCore) to talk to users. It runs locally and exposes an HTTP API (default `http://127.0.0.1:8000/`, see `/docs` or `/openapi.json`).

`bot.py` is a Python "fanout" plugin for Remote Terminal — it responds to user messages. `server.ts` is a long-running Node.js server that polls external services (USGS, NWS, etc.) and pushes alerts to MeshCore channels via the same API.

If Remote Terminal has HTTP Basic auth enabled (`MESHCORE_BASIC_AUTH_USERNAME` / `MESHCORE_BASIC_AUTH_PASSWORD`), pass credentials on every `curl` with `-u "$RT_USER:$RT_PASS"` (or `curl -u 'user:pass'`). The bot reads the same credentials automatically from those env vars when running inside RT, or set `_API_USER` / `_API_PASS` at the top of `bot.py`.

## Create the fanout (first time)

```sh
export RT_URL="http://127.0.0.1:8000"
export RT_USER="your-username"
export RT_PASS="your-password"

jq -n --rawfile code bot.py '{
  "type": "bot",
  "name": "BlorkoBot",
  "enabled": true,
  "config": {"code": $code},
  "scope": {}
}' | curl -sS -u "$RT_USER:$RT_PASS" -w "\nHTTP %{http_code}\n" \
  -X POST -H "Content-Type: application/json" -d @- \
  "$RT_URL/api/fanout" | tee /tmp/fanout-create.json

jq -r '.id' /tmp/fanout-create.json   # fanout UUID for PATCH below
```

## Updating the fanout config

After editing `bot.py`, deploy it by PATCHing the fanout config (replace `<FANOUT_ID>` with your fanout's UUID):

```sh
jq -n --rawfile code bot.py '{"config": {"code": $code}}' | \
  curl -sS -u "$RT_USER:$RT_PASS" -X PATCH -H "Content-Type: application/json" -d @- \
  "$RT_URL/api/fanout/<FANOUT_ID>"
```

The PATCH triggers an automatic module reload.

## Design notes

- Set `_COMMAND_PREFIX` in `bot.py` (e.g. `"!"`) to require a prefix on commands; leave it empty (`""`) to run commands without a prefix (`weather` instead of `!weather`). Legacy `!cmd` still works when the prefix is empty.
- Per-command settings in `_COMMANDS`: `False` to disable, `True` for default channels (`#bot`, `#test`, plus matching topic channels in `_TOPIC_CHANNELS`), or a list like `["#bot", "#weather"]` to restrict. Aliases follow the canonical command. Chess subcommands (`move`, `board`, `resign`, `elo`) are controlled by `"chess"`.
- DMs to the bot reply with `"<message>" - <path info>` (the path reply). Commands do NOT work in DMs.
- MeshCore has a short message limit (~120 bytes), so responses must be terse.
- Never get into a loop and clog up the mesh with extraneous traffic.
- Never respond in the Public channel.

## API keys

Several commands use external APIs that require keys. The public source has these blanked out — fill them in for your deployment:

- `_N2YO_KEY` in `bot.py` — satellite passes (`!iss <location>`)
- `_BAY511_API_KEY` in `bot.py` — Bay Area traffic (`!511`)
- `_GROQ_KEY` in `bot.py` — AI/LLM commands (`!ai`, `!sonnet`, `!haiku`)
- `_AERODATABOX_KEY` in `bot.py` — flight status (`!flight`)

Commands degrade gracefully when keys are missing.
