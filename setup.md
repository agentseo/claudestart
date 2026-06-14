# Claude Code — Установка и настройка

> Если ты Claude — выполняй шаги последовательно, проверяй результат на каждом шаге.
> Полные конфиги (settings.json, statusline-command.sh, tmux.conf) — в git-истории этого репо.

> **Основной интерфейс — интегрированный терминал VS Code** (§12): одно окно в
> WSL-remote режиме, внутри tmux, а файлы правятся прямо из консоли — Ctrl+Click
> по пути из ответа Claude открывает файл вкладкой рядом. Отдельный терминал не нужен.
>
> Доступ к Anthropic из региона с геоблоком настраивается отдельно и на твой
> выбор — см. §2.1. setup.md ничего конкретного не навязывает.

---

## 0. Замени путь под себя

```bash
echo $HOME   # узнай свой путь
# После создания каждого файла: sed -i "s|/home/user|$HOME|g" <файл>
```

## 1. Зависимости

```bash
sudo apt update && sudo apt install -y jq tmux
npm install -g @anthropic-ai/claude-code
cargo install rtk                          # или бинарник с github.com/rtk-ai/rtk/releases
npm install -g prettier && pip install ruff  # форматтеры (опционально)
jq --version && rtk --version && claude --version
```

## 2. Функция запуска `cc` (~/.bashrc)

Один аккаунт — одна функция-лаунчер. Делаем её bash-функцией, а не `alias`,
потому что:

- `exec -a cc` задаёт `argv[0] = cc`, поэтому в `ps` / `/proc/<pid>/comm` процесс
  виден как `cc`, а не `claude` — tmux-resurrect различает свои панели при
  восстановлении, и `--dangerously-skip-permissions` не теряется.
- Subshell `( ... )` нужен, чтобы `exec` замещал процесс субшелла, а не
  родительского bash — иначе после `/exit` в Claude закроется окно tmux.

```bash
cc() { ( exec -a cc claude --dangerously-skip-permissions "$@" ); }
```

Базовая команда тут — `claude`. Если ты настроил доступ к Anthropic через свой
прокси/тоннель (см. §2.1), подставь имя своего wrapper-скрипта вместо `claude` —
тогда `cc` сам поднимет канал перед стартом.

Проверка после `source ~/.bashrc` + запуска `cc`:

```bash
pgrep -a cc   # процесс должен называться cc, а НЕ claude
```

### Алиасы быстрого перехода в проекты

```bash
alias pcl='cd ~/projects/claude && cc -c'   # cc -c = продолжить последнюю сессию
alias pseo='cd ~/projects/SEO && cc -c'
# ... и т.д. для каждого проекта
```

### 2.1. Доступ к Anthropic API (если регион под геоблоком)

Claude Code ходит на `api.anthropic.com`. Из некоторых регионов этот адрес
заблокирован — запросы падают с 403 / «Could not route your request». Способ
обхода — твой выбор; setup.md его не навязывает. Общая идея: пустить HTTPS-трафик
Claude через канал с «разрешённым» IP. Варианты:

- **HTTPS_PROXY на свой прокси/VPS** — `export HTTPS_PROXY=http://127.0.0.1:PORT`
  в `~/.bashrc`, где порт смотрит на твой прокси: SSH-тоннель (`ssh -L PORT:…`),
  tinyproxy/3proxy на VPS в разрешённом регионе и т.п.
- **VPN** до того же региона (системный или split-tunnel только под Claude).
- **API-ключ** вместо OAuth (`export ANTHROPIC_API_KEY=sk-ant-…`) — убирает
  OAuth-flow, но это pay-per-token, а не подписка.

Удобно завернуть запуск в wrapper-скрипт (`~/.local/bin/claude-proxy`: поднять
канал/тоннель → `exec claude "$@"`) и подставить его имя в функции из §2. Тогда
`cc` сам поднимает канал и работает с Anthropic **напрямую через него** — без
ручного включения VPN/прокси перед каждым стартом.

> ⚠️ Нюанс с OAuth: иногда хостинг прокси режут только на OAuth-эндпоинтах
> (`console.anthropic.com`, `claude.ai`, `auth.anthropic.com`), пропуская при этом
> `api.anthropic.com`. Если ловишь `OAuth error 403` — добавь эти три домена в
> `NO_PROXY`: вход пойдёт мимо прокси, а сам API — через него.

### 2.2. Headroom — компрессия контекста (опционально)

Headroom — прослойка, сжимающая контекст между Claude и API: длинные tool-выводы
и файлы ужимаются, в окно влезает больше. Полезно на больших и долгих сессиях.
Ставится отдельно; запуск удобно завести функцией-двойником `cc`, но через свой
Headroom-wrapper:

```bash
# то же, что cc, но с компрессией контекста
cch() { ( exec -a cc claude-headroom --dangerously-skip-permissions "$@" ); }
```

`claude-headroom` — твой wrapper, поднимающий Headroom-прослойку и запускающий
`claude` через неё. Дальше выбираешь по задаче: `cc` для обычной работы, `cch`
когда контекст большой. `exec -a cc` оставлено таким же, чтобы tmux-resurrect не
путал сессии при восстановлении.

## 3. Папки

```bash
mkdir -p ~/.claude/hooks
```

## 4. ~/.claude/settings.json

Ключевые секции:

- `permissions.allow` — список разрешённых команд (git, docker, python, rtk и др.)
- `hooks.PreToolUse[Bash]` — `rtk hook claude` (RTK rewrite)
- `hooks.PostToolUse[Write|Edit]` — `format-file.sh` (авто-форматирование)
- `hooks.PostToolUse[*]` — tmux tab синий (`#89b4fa`): любой инструмент = агент работает
- `hooks.Stop` — beep + tmux tab красный (`#f38ba8`): Claude закончил ход
- `hooks.Notification` — beep + tmux tab жёлтый (`#f9e2af`): простой ≥60с / нужен пермишен
- `hooks.UserPromptSubmit` — tmux tab синий (`#89b4fa`): агент работает (на отправку промпта)
- `hooks.SessionEnd` — tmux tab сброс цвета при выходе из Claude (`/exit`, Ctrl-D)

Автомат состояний вкладки (цвет виден на **фоновых** табах; активный таб всегда
зелёный `#a6e3a1` как `window-status-current`):
🔵 синий — агент работает · 🔴 красный — ход закончен, твой ход · 🟡 жёлтый — ждёт пермишена

Цвет ставится **по состоянию, а не по краю**: синий красится на каждый tool-call
(`PostToolUse`), поэтому если `Notification` мигнул жёлтым посреди хода, следующий
же инструмент вернёт синий. Жёлтый остаётся только когда агент реально стоит.

- `statusLine` — `bash ~/.claude/statusline-command.sh`
- `skipDangerousModePermissionPrompt: true`

```bash
sed -i "s|/home/user|$HOME|g" ~/.claude/settings.json
```

## 5. ~/.claude/statusline-command.sh

Показывает: аккаунт, модель, папка/ветка (со `*` если есть незакоммиченные), ctx%, rate limits (5h/7d).

```bash
sed -i "s|/home/user|$HOME|g" ~/.claude/statusline-command.sh
chmod +x ~/.claude/statusline-command.sh
```

## 6. ~/.claude/hooks/format-file.sh

Авто-форматирование по расширению: `.py` → ruff/black, `.rs` → rustfmt, `.go` → gofmt, остальное → prettier.

```bash
chmod +x ~/.claude/hooks/format-file.sh
```

## 7. RTK хук

```bash
rtk init -g
```

Регистрирует `rtk hook claude` в PreToolUse. Не редактировать хук вручную.

## 8. Второй аккаунт (опционально — большинству не нужно)

Большинство работают с одним аккаунтом, и §2 этого достаточно. Но если нужен
изолированный второй (например личный + рабочий) — заведи отдельный
`CLAUDE_CONFIG_DIR` и вторую функцию:

```bash
mkdir -p ~/.claude-work
cp ~/.claude/settings.json ~/.claude-work/settings.json
sed -i "s|/home/user|$HOME|g" ~/.claude-work/settings.json
```

```bash
# второй аккаунт со своей конфигурацией/историей/лимитами
ccw() { ( CLAUDE_CONFIG_DIR=$HOME/.claude-work exec -a ccw claude --dangerously-skip-permissions "$@" ); }
```

## 9. ~/.tmux.conf

Тема Catppuccin Mocha, префикс `Ctrl+a`, табы сверху, Mouse on, tmux-resurrect (автосохранение каждые 15 мин).

```bash
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
tmux source-file ~/.tmux.conf
# Внутри tmux: Prefix+I для установки плагинов
```

## 10. Перезапусти Claude Code

Хуки и статус-строка подхватываются только при старте новой сессии.

## 11. Open-helper `o` + wslu (для WSL2)

Открытие файлов/папок из терминала «как в IDE». В VS Code-терминале есть Ctrl+Click,
но для случаев когда руки на клавиатуре — функция `o`:

```bash
sudo apt install -y wslu
```

В `~/.bashrc`:

```bash
# o <file>          → открыть в текущем окне VS Code (новая вкладка)
# o <file>:<line>   → открыть в VS Code на нужной строке
# o <dir>           → открыть в Windows Explorer (через wslview → UNC \\wsl.localhost\Ubuntu\...)
# o <URL>           → открыть в браузере по умолчанию
o() {
    local arg="${1/#\~/$HOME}"
    local path="${arg%%:*}"
    if [[ -d "$arg" ]]; then
        wslview "$arg"
    elif [[ -e "$path" ]]; then
        if [[ "$arg" == *:* ]]; then
            code -g "$arg"
        else
            code "$arg"
        fi
    else
        wslview "$arg"
    fi
}
```

## 12. VS Code + WSL Remote (главный интерфейс)

Идея: одно окно VS Code в WSL-remote режиме, в нём авто-стартует tmux,
Ctrl+Click по путям из ответов Claude открывает файлы как вкладки рядом.

### 12.1. Поставить расширение

`Extensions` → ищи **WSL** от Microsoft (`ms-vscode-remote.remote-wsl`).

### 12.2. Глобальные настройки VS Code (Windows-сторона)

Путь: `C:\Users\<USER>\AppData\Roaming\Code\User\settings.json`

```json
{
  "terminal.integrated.fontFamily": "Cascadia Mono",
  "terminal.integrated.fontSize": 14,
  "terminal.integrated.cursorBlinking": false,
  "terminal.integrated.scrollback": 20000,
  "terminal.integrated.profiles.linux": {
    "tmux": {
      "path": "bash",
      "args": ["-lc", "tmux new-session -A -s 0"],
      "icon": "terminal-tmux"
    },
    "bash": { "path": "bash", "icon": "terminal-bash" }
  },
  "terminal.integrated.defaultProfile.linux": "bash",
  "terminal.integrated.defaultLocation": "editor",
  "workbench.colorCustomizations": {
    "terminal.background": "#002b36",
    "terminal.foreground": "#93a1a1",
    "terminalCursor.foreground": "#93a1a1"
  },
  "editor.fontFamily": "Cascadia Mono, Consolas, 'Courier New', monospace",
  "editor.fontSize": 14,
  "editor.minimap.enabled": false,
  "files.autoSave": "onFocusChange",
  "security.workspace.trust.enabled": false,
  "task.allowAutomaticTasks": "on"
}
```

Ключевое:

- `defaultProfile.linux: "bash"` — Ctrl+Shift+\` даёт чистый bash без tmux
- `defaultLocation: "editor"` — новые терминалы открываются как вкладки редактора
- `task.allowAutomaticTasks: "on"` — разрешает авто-задачу `runOn: folderOpen`
- `security.workspace.trust.enabled: false` — убирает «trust this folder?» плашку

### 12.3. Auto-attach tmux на старте (WSL-сторона)

`~/.vscode/tasks.json` — задача запускается при открытии workspace `~`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Auto-attach tmux",
      "type": "shell",
      "command": "tmux new-session -A -s 0",
      "presentation": {
        "reveal": "always",
        "panel": "dedicated",
        "focus": true,
        "echo": false,
        "showReuseMessage": false,
        "clear": false
      },
      "runOptions": { "runOn": "folderOpen" },
      "problemMatcher": []
    }
  ]
}
```

### 12.4. Ярлык на рабочий стол Windows

Создаётся через PowerShell-interop из WSL:

```bash
powershell.exe -NoProfile -Command "
\$s = (New-Object -ComObject WScript.Shell).CreateShortcut('C:\Users\Alex\Desktop\VS Code (WSL).lnk')
\$s.TargetPath = 'C:\Program Files\Microsoft VS Code\Code.exe'
\$s.Arguments = '--remote wsl+Ubuntu /home/alex'
\$s.IconLocation = 'C:\Program Files\Microsoft VS Code\Code.exe,0'
\$s.WorkingDirectory = 'C:\Program Files\Microsoft VS Code'
\$s.Save()
"
```

Замени `Alex` на своё Windows-имя пользователя, `Ubuntu` — на имя своего WSL-distro
(см. `wsl -l -q`).

## 13. Цепочка запуска (как всё связано)

```
[Двойной клик по ярлыку на рабочем столе]
    │
    ▼
Code.exe --remote wsl+Ubuntu /home/alex
    │
    ▼
VS Code открывается в WSL-remote режиме (зелёный "WSL: Ubuntu" внизу)
    │
    ├─→ читает Windows settings.json    (тема, шрифт, профили терминалов)
    └─→ читает ~/.vscode/tasks.json     (auto-attach tmux)
            │
            ▼
        Task "Auto-attach tmux" запускается:
            bash -c "tmux new-session -A -s 0"
            (-A: прицепиться к сессии 0, если живёт; создать, если нет —
             любой новый терминал садится на ту же сессию 0, без 1/2/3)
            │
            ▼
        Если tmux server жив (после ребута WSL):
            → tmux-continuum auto-restore из ~/.local/share/tmux/resurrect/last
            → панели восстанавливаются с правильными cwd
            → claude НЕ перезапускается автоматически (resurrect-processes='mc',
              см. §9) — панели поднимаются как bash
            │
            ▼
        Tmux отображается в editor-area VS Code как вкладка
            │
            ▼
        claude поднимаешь сам — cc (или cch через Headroom),
        ресюм из ~/.claude/projects/<cwd>/ через флаг -c
```

Что критично для корректного восстановления:

- `cc` — bash-функция с `exec -a`, не алиас (см. §2)
- `tmux-continuum @continuum-restore 'on'` в `~/.tmux.conf`
- `@resurrect-processes 'mc'` — claude НЕ воскрешается автоматически (поднимаешь сам нужным вариантом, см. §9)
- Снапшоты живут в `~/.local/share/tmux/resurrect/`

## 14. Playwright MCP — headless + reaper (если много параллельных сессий)

Плагин Playwright поднимает свой Chromium на каждую сессию. Когда в tmux крутится
с десяток параллельных Claude, headed-браузеры с GPU-компоновкой иногда уходят в
сотни % CPU. Две защиты:

1. **Форс `--headless`** в args плагина (`.mcp.json` плагина playwright) — браузер
   не рисует окно. ⚠️ Обновление плагина может откатить правку; проверяй после
   апдейтов и возвращай `"--headless"` в массив args.
2. **Cron-reaper** (раз в ~10 мин) — убивает «сбежавшие» и осиротевшие процессы
   Chromium: дерево браузера без живого родителя-`claude` либо дерево с CPU выше
   порога. Node-процесс MCP не трогает — браузер переподнимется по требованию.

Ручная разовая зачистка:

```bash
pkill -f 'playwright-browsers/.*chrome-linux/chrome'
```

---

## Проверка

```bash
rtk --version && rtk gain && cc
```

Внутри Claude Code: нижняя строка показывает `аккаунт  модель  папка/ветка  ctx: X%`.
После ответа — вкладка tmux краснеет + бип. После отправки — синеет (агент работает).

---

## Справочник

### RTK

```bash
rtk gain [--history]   # статистика токенов
rtk discover           # упущенные оптимизации
rtk proxy <cmd>        # выполнить без RTK
rtk init -g            # переустановить хук
```

### tmux шорткаты

| Шорткат       | Действие                 |
| ------------- | ------------------------ |
| `Ctrl+a`      | Префикс                  |
| `Prefix + \|` | Разделить вертикально    |
| `Prefix + -`  | Разделить горизонтально  |
| `Prefix + c`  | Новое окно               |
| `Alt+1..9`    | Переключить окно         |
| `Alt+←→↑↓`    | Навигация между панелями |
| `Prefix + r`  | Перезагрузить конфиг     |
| `tmux attach` | Подключиться к сессии    |

### Структура файлов

WSL-сторона:

```
~/.claude/
├── CLAUDE.md / RTK.md
├── settings.json
├── statusline-command.sh
└── hooks/
    ├── rtk-rewrite.sh   # не редактировать
    └── format-file.sh

~/.claude-work/settings.json        # второй аккаунт (опционально, §8)
~/.tmux.conf
~/.bashrc                            # cc / cch функции, o(), алиасы pcl/pseo/…
~/.vscode/tasks.json                 # auto-attach tmux на старте VS Code
~/.tmux/plugins/                     # tpm + tmux-resurrect + tmux-continuum
~/.local/share/tmux/resurrect/       # снапшоты состояния (auto, каждые 15 мин)
```

Windows-сторона:

```
C:\Users\<USER>\AppData\Roaming\Code\User\settings.json   # VS Code (тема, шрифт, профили)
C:\Users\<USER>\Desktop\VS Code (WSL).lnk                 # ярлык запуска
```

### Чек-лист на новой машине

1. WSL2 Ubuntu установлен (`wsl --install -d Ubuntu`)
2. Node.js + npm в WSL → `npm install -g @anthropic-ai/claude-code`
3. `sudo apt install -y jq tmux wslu`
4. Скопировать `~/.bashrc`, `~/.tmux.conf`, `~/.claude/`, `~/.vscode/tasks.json` (и `~/.claude-work/` — только если нужен второй аккаунт, §8)
5. `git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm`
6. Запустить tmux, нажать `Prefix+I` для установки плагинов
7. `rtk init -g` (если используется RTK)
8. На Windows: установить VS Code + WSL-расширение
9. Положить `settings.json` в `%APPDATA%\Code\User\`
10. Создать ярлык на рабочем столе (см. §12.4)
11. Двойной клик → проверить, что VS Code открылся в WSL-режиме и tmux поднялся
