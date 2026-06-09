# Claude Code — Установка и настройка

> Если ты Claude — выполняй шаги последовательно, проверяй результат на каждом шаге.
> Полные конфиги (settings.json, statusline-command.sh, tmux.conf) — в git-истории этого репо.

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

## 2. Функции cc1/cc2 (~/.bashrc)

> Раньше тут были `alias`, но при таком запуске процесс в `/proc/<pid>/comm`
> назывался `claude`, и tmux-resurrect не различал cc1/cc2 при восстановлении
> (плюс терялся `--dangerously-skip-permissions` и `CLAUDE_CONFIG_DIR` для cc2).
> Решение — bash-функции с `exec -a` в subshell, которые задают argv[0].
> Subshell `( ... )` нужен, чтобы `exec` замещал процесс субшелла, а не
> родительского bash — иначе после `exit` в Claude закроется окно tmux:

```bash
alias claude1="claude"
alias claude2="CLAUDE_CONFIG_DIR=$HOME/.claude-work claude"
cc1() { ( exec -a cc1 claude --dangerously-skip-permissions "$@" ); }
cc2() { ( CLAUDE_CONFIG_DIR=$HOME/.claude-work exec -a cc2 claude --dangerously-skip-permissions "$@" ); }
```

Если используется `claude-vps` (через VPS-тоннель) — подставь `claude-vps` вместо `claude`.

Проверка после `source ~/.bashrc` + запуска `cc1`:

```bash
ps -eo pid,comm,args | grep -E "cc1|cc2"
# В колонке COMM должно быть cc1/cc2, а НЕ claude
```

### Алиасы быстрого перехода в проекты

```bash
alias pcl='cd ~/projects/claude && cc1 -c'
alias pseo='cd ~/projects/SEO && cc1 -c'
# ... и т.д. для каждого проекта
```

## 3. Папки

```bash
mkdir -p ~/.claude/hooks ~/.claude-work
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

## 8. claude2 — второй аккаунт (опционально)

Скопировать `~/.claude/settings.json` в `~/.claude-work/settings.json`, заменить пути:

```bash
cp ~/.claude/settings.json ~/.claude-work/settings.json
sed -i "s|/home/user|$HOME|g" ~/.claude-work/settings.json
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
        claude поднимаешь сам нужным вариантом (cch1 / cch1d / …),
        каждый ресюмится из ~/.claude/projects/<cwd>/ через флаг -c
```

Что критично для корректного восстановления:

- `cc1`/`cc2` — bash-функции с `exec -a`, не алиасы (см. §2)
- `tmux-continuum @continuum-restore 'on'` в `~/.tmux.conf`
- `@resurrect-processes 'mc'` — claude НЕ воскрешается автоматически (поднимаешь сам нужным вариантом, см. §9)
- Снапшоты живут в `~/.local/share/tmux/resurrect/`

---

## Проверка

```bash
rtk --version && rtk gain && cc1
```

Внутри Claude Code: нижняя строка показывает `claude1  модель  папка/ветка  ctx: X%`.
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

~/.claude-work/settings.json        # cc2-профиль (work)
~/.tmux.conf
~/.bashrc                            # cc1/cc2 функции, o(), алиасы pcl/pseo/…
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
4. Скопировать `~/.bashrc`, `~/.tmux.conf`, `~/.claude/`, `~/.claude-work/`, `~/.vscode/tasks.json`
5. `git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm`
6. Запустить tmux, нажать `Prefix+I` для установки плагинов
7. `rtk init -g` (если используется RTK)
8. На Windows: установить VS Code + WSL-расширение
9. Положить `settings.json` в `%APPDATA%\Code\User\`
10. Создать ярлык на рабочем столе (см. §12.4)
11. Двойной клик → проверить, что VS Code открылся в WSL-режиме и tmux поднялся
