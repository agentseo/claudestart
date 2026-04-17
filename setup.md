# Claude Code — Установка и настройка

> Если ты Claude — выполняй шаги последовательно, проверяй результат на каждом шаге.

---

## 0. Замени путь под себя

Во всех файлах ниже путь `/home/user` — замени на свой `$HOME`:

```bash
echo $HOME   # узнай свой путь
```

После создания каждого файла применяй:
```bash
sed -i "s|/home/user|$HOME|g" <файл>
```

---

## 1. Зависимости

```bash
# Обязательно
sudo apt update && sudo apt install -y jq tmux

# Claude Code (если не установлен)
npm install -g @anthropic-ai/claude-code

# RTK
cargo install rtk
# Или скачай бинарник: https://github.com/rtk-ai/rtk/releases

# Форматтеры (опционально)
npm install -g prettier   # JS, TS, JSON, CSS, HTML, MD
pip install ruff          # Python (быстрый)
pip install black         # Python (fallback)
# rustfmt — входит в Rust toolchain
# gofmt — входит в Go
```

Проверь:
```bash
jq --version && rtk --version && claude --version
```

---

## 2. Алиасы

Добавь в конец `~/.bashrc`:

```bash
# Claude Code — два аккаунта
alias claude1="claude"
alias cc1='claude1 --dangerously-skip-permissions'

alias claude2="CLAUDE_CONFIG_DIR=~/.claude-work claude"
alias cc2='claude2 --dangerously-skip-permissions'
```

```bash
source ~/.bashrc
```

---

## 3. Папки

```bash
mkdir -p ~/.claude/hooks
mkdir -p ~/.claude-work
```

---

## 4. `~/.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(docker *)",
      "Bash(docker-compose *)",
      "Bash(poetry run *)",
      "Bash(python *)",
      "Bash(python3 *)",
      "Bash(pip *)",
      "Bash(npm run *)",
      "Bash(npx *)",
      "Bash(bun *)",
      "Bash(node *)",
      "Bash(ls *)",
      "Bash(rtk *)",
      "Bash(cat *)",
      "Bash(find *)",
      "Bash(grep *)",
      "Bash(rg *)",
      "Bash(tree *)",
      "Bash(pwd)",
      "Bash(echo *)",
      "Bash(wc *)",
      "Bash(head *)",
      "Bash(tail *)",
      "Bash(which *)",
      "Bash(whoami)",
      "Bash(env)",
      "Bash(printenv *)",
      "Bash(ps *)",
      "Bash(kill *)",
      "Bash(mkdir *)",
      "Bash(cp *)",
      "Bash(mv *)"
    ]
  },
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe -Command \"[console]::beep(600,200); [console]::beep(900,200)\"",
            "async": true
          },
          {
            "type": "command",
            "command": "tmux set-window-option -t $TMUX_PANE window-status-style 'bg=#f38ba8,fg=#1e1e2e,bold' 2>/dev/null || true",
            "async": true
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "tmux set-window-option -u -t $TMUX_PANE window-status-style 2>/dev/null || true",
            "async": true
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe -Command \"[console]::beep(800,300)\"",
            "async": true
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path // empty' | { read -r f || exit 0; bash /home/user/.claude/hooks/format-file.sh \"$f\"; }",
            "async": true
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/.claude/hooks/rtk-rewrite.sh"
          }
        ]
      }
    ]
  },
  "statusLine": {
    "type": "command",
    "command": "bash /home/user/.claude/statusline-command.sh"
  },
  "skipDangerousModePermissionPrompt": true
}
```

```bash
sed -i "s|/home/user|$HOME|g" ~/.claude/settings.json
```

---

## 5. `~/.claude/statusline-command.sh`

Нижняя строка Claude Code: аккаунт, модель, папка/ветка (`*` если есть незакоммиченные изменения), контекст, rate limits.

```bash
#!/bin/bash
input=$(cat)
model=$(echo "$input" | jq -r '.model.display_name // "unknown"')
used=$(echo "$input" | jq -r '.context_window.used_percentage // empty')
workdir=$(echo "$input" | jq -r '.workspace.current_dir')
branch=$(git --no-optional-locks -C "$workdir" rev-parse --abbrev-ref HEAD 2>/dev/null)
dirty=$(git --no-optional-locks -C "$workdir" status --porcelain 2>/dev/null | head -1)
[ -n "$dirty" ] && branch="${branch}*"

if [ "${CLAUDE_CONFIG_DIR}" = "/home/user/.claude-work" ]; then
  instance="claude2"
else
  instance="claude1"
fi

printf "\033[35m%s\033[0m" "$instance"
printf "  \033[36m%s\033[0m" "$model"

dir=$(basename "$workdir")
if [ -n "$branch" ]; then
  printf "  \033[33m%s/%s\033[0m" "$dir" "$branch"
else
  printf "  \033[33m%s\033[0m" "$dir"
fi

if [ -n "$used" ]; then
  printf "  \033[32mctx: %.0f%%\033[0m" "$used"
fi

five_hour=$(echo "$input" | jq -r '.rate_limits.five_hour.used_percentage // empty')
seven_day=$(echo "$input" | jq -r '.rate_limits.seven_day.used_percentage // empty')

if [ -n "$five_hour" ]; then
  if [ "$(echo "$five_hour" | awk '{print ($1 >= 80)}')" = "1" ]; then
    printf "  \033[31m5h: %.0f%%\033[0m" "$five_hour"
  else
    printf "  \033[90m5h: %.0f%%\033[0m" "$five_hour"
  fi
fi

if [ -n "$seven_day" ]; then
  if [ "$(echo "$seven_day" | awk '{print ($1 >= 80)}')" = "1" ]; then
    printf "  \033[31m7d: %.0f%%\033[0m" "$seven_day"
  else
    printf "  \033[90m7d: %.0f%%\033[0m" "$seven_day"
  fi
fi
```

```bash
sed -i "s|/home/user|$HOME|g" ~/.claude/statusline-command.sh
chmod +x ~/.claude/statusline-command.sh
```

---

## 6. `~/.claude/hooks/format-file.sh`

Авто-форматирование файла после записи. Диспетч по расширению.

```bash
#!/usr/bin/env bash
f="$1"
[ -z "$f" ] && exit 0
[ -f "$f" ] || exit 0

case "$f" in
  *.py)
    ruff format "$f" 2>/dev/null || black "$f" 2>/dev/null || true
    ;;
  *.rs)
    rustfmt "$f" 2>/dev/null || true
    ;;
  *.go)
    gofmt -w "$f" 2>/dev/null || true
    ;;
  *)
    prettier --write "$f" 2>/dev/null || npx --no-install prettier --write "$f" 2>/dev/null || true
    ;;
esac
```

```bash
chmod +x ~/.claude/hooks/format-file.sh
```

---

## 7. RTK — хук

RTK сам создаёт хук и обновляет `CLAUDE.md`:

```bash
rtk init -g
```

> Не редактируй `~/.claude/hooks/rtk-rewrite.sh` вручную — RTK проверяет целостность файла через SHA256 и заблокирует выполнение. Для восстановления: `rtk init -g --auto-patch`.

---

## 8. claude2 — второй аккаунт (опционально)

`~/.claude-work/settings.json` — те же хуки, отдельные плагины. Плагины в `enabledPlugins` выбери под себя.

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(docker *)",
      "Bash(docker-compose *)",
      "Bash(poetry run *)",
      "Bash(python *)",
      "Bash(python3 *)",
      "Bash(pip *)",
      "Bash(npm run *)",
      "Bash(npx *)",
      "Bash(bun *)",
      "Bash(node *)",
      "Bash(ls *)",
      "Bash(rtk *)",
      "Bash(cat *)",
      "Bash(find *)",
      "Bash(grep *)",
      "Bash(rg *)",
      "Bash(tree *)",
      "Bash(pwd)",
      "Bash(echo *)",
      "Bash(wc *)",
      "Bash(head *)",
      "Bash(tail *)",
      "Bash(which *)",
      "Bash(whoami)",
      "Bash(env)",
      "Bash(printenv *)",
      "Bash(ps *)",
      "Bash(kill *)",
      "Bash(mkdir *)",
      "Bash(cp *)",
      "Bash(mv *)"
    ]
  },
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe -Command \"[console]::beep(600,200); [console]::beep(900,200)\"",
            "async": true
          },
          {
            "type": "command",
            "command": "tmux set-window-option -t $TMUX_PANE window-status-style 'bg=#f38ba8,fg=#1e1e2e,bold' 2>/dev/null || true",
            "async": true
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "tmux set-window-option -u -t $TMUX_PANE window-status-style 2>/dev/null || true",
            "async": true
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe -Command \"[console]::beep(800,300)\"",
            "async": true
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path // empty' | { read -r f || exit 0; bash /home/user/.claude/hooks/format-file.sh \"$f\"; }",
            "async": true
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/.claude/hooks/rtk-rewrite.sh"
          }
        ]
      }
    ]
  },
  "statusLine": {
    "type": "command",
    "command": "bash /home/user/.claude/statusline-command.sh"
  },
  "skipDangerousModePermissionPrompt": true
}
```

```bash
sed -i "s|/home/user|$HOME|g" ~/.claude-work/settings.json
```

---

## 9. `~/.tmux.conf`

Тема Catppuccin Mocha, удобные шорткаты, сохранение сессий. Активная вкладка — зелёная, ждущая ответа — красная.

```bash
# Установи TPM (менеджер плагинов tmux)
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

```bash
# ~/.tmux.conf
# --- Основные настройки ---
set -g default-terminal "tmux-256color"
set -ag terminal-overrides ",xterm-256color:RGB"
set -g mouse on
set -g history-limit 10000
set -g base-index 1
setw -g pane-base-index 1
set -g renumber-windows on
set -g escape-time 0
set -g focus-events on

# --- Префикс: Ctrl+a (удобнее чем Ctrl+b) ---
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# --- Окна (табы) ---
bind c new-window -c "#{pane_current_path}"
bind n next-window
bind p previous-window
bind -n M-1 select-window -t 1
bind -n M-2 select-window -t 2
bind -n M-3 select-window -t 3
bind -n M-4 select-window -t 4
bind -n M-5 select-window -t 5
bind -n M-6 select-window -t 6
bind -n M-7 select-window -t 7
bind -n M-8 select-window -t 8
bind -n M-9 select-window -t 9

# --- Панели: разделение ---
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
unbind '"'
unbind %

# --- Навигация по панелям: Alt+стрелки ---
bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D

# --- Ресайз панелей: Prefix + стрелки ---
bind -r Left resize-pane -L 5
bind -r Right resize-pane -R 5
bind -r Up resize-pane -U 3
bind -r Down resize-pane -D 3

# --- Быстрая перезагрузка конфига ---
bind r source-file ~/.tmux.conf \; display "Config reloaded!"

# --- Статус-бар (табы сверху) ---
set -g status-position top
set -g status-style "bg=#1e1e2e,fg=#cdd6f4"
set -g status-left-length 30
set -g status-right-length 50

set -g status-left "#[bg=#89b4fa,fg=#1e1e2e,bold] #S #[bg=default] "
set -g status-right "#[fg=#a6adc8] %H:%M  %d/%m "

# Стиль табов (окон)
# Неактивные: серый текст, дефолтный фон
setw -g window-status-style "fg=#6c7086,bg=default"
setw -g window-status-format " #I:#W "
# Активная: чёрный текст на зелёном фоне (единственный зелёный таб)
setw -g window-status-current-style "fg=#1e1e2e,bg=#a6e3a1,bold"
setw -g window-status-current-format " #I:#W "
setw -g window-status-separator ""

# Bell: красный фон когда Claude ждёт ответа
set -g monitor-bell on
set -g bell-action any
setw -g window-status-bell-style "bg=#f38ba8,fg=#1e1e2e,bold"

# --- Панели ---
set -g pane-border-style "fg=#313244"
set -g pane-active-border-style "fg=#89b4fa"

# --- Копирование (vi mode) ---
setw -g mode-keys vi
bind -T copy-mode-vi v send -X begin-selection
bind -T copy-mode-vi y send -X copy-selection-and-cancel

# --- Плагины ---
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'

set -g @resurrect-processes 'mc bash'
set -g @continuum-restore 'on'
set -g @continuum-save-interval '15'

run '~/.tmux/plugins/tpm/tpm'
```

```bash
# Применить конфиг
tmux source-file ~/.tmux.conf
# Установить плагины: внутри tmux нажми Prefix+I (Ctrl+a, затем I)
```

---

## 10. Перезапусти Claude Code

Хуки и статус-строка подхватываются только при старте новой сессии.

---

## Проверка

```bash
rtk --version          # RTK установлен
rtk gain               # Аналитика работает
rtk proxy "git status" # Показывает rewrite
cc1                    # Запускает claude1
```

Внутри Claude Code:
- Нижняя строка: `claude1  claude-sonnet-4-6  папка/ветка  ctx: X%`
- После ответа Claude — вкладка tmux краснеет, двойной бип
- После отправки сообщения — вкладка возвращается в обычный цвет

---

## Справочник

### RTK мета-команды

```bash
rtk gain                   # Сколько токенов сэкономлено
rtk gain --history         # История с детализацией
rtk discover               # Упущенные оптимизации
rtk proxy <cmd>            # Выполнить без RTK (дебаг)
rtk init -g                # Переустановить хук
rtk init -g --auto-patch   # Переустановить после правки хука
```

### tmux шорткаты

| Шорткат | Действие |
|---------|----------|
| `Ctrl+a` | Префикс |
| `Prefix + \|` | Разделить вертикально |
| `Prefix + -` | Разделить горизонтально |
| `Prefix + c` | Новое окно |
| `Alt+1..9` | Переключить окно |
| `Alt+←→↑↓` | Навигация между панелями |
| `Prefix + r` | Перезагрузить конфиг |
| `tmux attach` | Подключиться к существующей сессии |

### Структура файлов

```
~/.claude/
├── CLAUDE.md                  # @RTK.md (создаётся через rtk init -g)
├── RTK.md                     # Документация RTK (создаётся через rtk init -g)
├── settings.json
├── statusline-command.sh
└── hooks/
    ├── rtk-rewrite.sh         # Управляется RTK, не редактировать
    ├── .rtk-hook.sha256
    └── format-file.sh

~/.claude-work/
└── settings.json

~/.tmux.conf
~/.bashrc
```
