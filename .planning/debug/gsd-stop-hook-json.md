---
status: resolved
trigger: "Stop hook (failed): hook returned invalid stop hook JSON output"
created: 2026-08-02
updated: 2026-08-02
---

# Debug Session: GSD stop-hook JSON

## Symptoms

- **Expected:** encerrar uma resposta sem erro de hook.
- **Actual:** Codex mostra `hook returned invalid stop hook JSON output`.
- **Timeline:** começou imediatamente após instalar a versão mais recente do GSD.
- **Reproduction:** encerrar uma resposta em um projeto com os hooks GSD instalados.

## Evidence

- timestamp: 2026-08-02 — `.codex/hooks.json` registrava `gsd-context-monitor.cmd` no evento `Stop`.
- timestamp: 2026-08-02 — o monitor produz mensagens no envelope `hookSpecificOutput`, aceito para injeção pós-ferramenta, mas inválido para o evento `Stop`.

## Resolution

- **root_cause:** o monitor de contexto foi registrado em uma superfície de hook incompatível (`Stop`).
- **fix:** removido somente o registro `Stop`; `PostToolUse` continua habilitado para os avisos de contexto.
- **verification:** JSON de hooks válido e nenhuma entrada `Stop` restante para o monitor.
- **files_changed:** `.codex/hooks.json`
