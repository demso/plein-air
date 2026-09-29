# agent-config — скелет домашних каталогов пользователей

Всё, что лежит здесь, копируется в `/home/<user>/` при создании пользователя
(роль `ompweb`, таск «Seed agent config skeletons»).

```
agent-config/
  .agents/        → /home/<user>/.agents   (AGENTS.md, скилы, глобальные правила)
  .omp/           → /home/<user>/.omp      (конфиг omp: agent/config.yml, keybindings.yml, …)
```

Правила:

- **Существующие файлы не перезатираются** (`force: false`): обновлённый скелет
  доезжает только до новых пользователей. У существующих сессии и правки в
  безопасности. Подправить всем — правь вручную или отдельным таском.
- **Не клади сюда `agent/web-service.env`** — его генерирует ansible из vault.
- **Не коммить секреты** (API-ключи в config.yml): складывай их в vault.yml
  и раскатывай шаблоном, либо пусть юзеры вписывают сами через ompweb.
- Права при копировании: `u=rwX,go=` — только владелец (в .omp лежат ключи).

Быстрый способ заполнить из своей машины:

```bash
cp -r ~/.agents .
cp -r ~/.omp/agent/config.yml ~/.omp/agent/keybindings.yml .omp/agent/ 2>/dev/null
```
