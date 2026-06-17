# Notificaciones al terminar: Claude Code y Codex

Este Markdown es una receta portable para instalar el mismo feature en otra compu: cuando Claude Code o Codex terminan y vuelve a ser tu turno, aparece un popup y suena un aviso.

Después de instalarlo, este archivo no participa en runtime. Lo que queda funcionando son los scripts y la configuración de hooks.

> Este doc vive en `~/.claude/hooks/instalar-notificaciones-claude-codex.md` y puede estar enlazado desde `~/Documentos/notificaciones-claude-code.md`.

## Elegir qué instalar

- Para Claude Code: seguí la sección "Instalar en Claude Code".
- Para Codex: seguí la sección "Instalar en Codex".
- Para ambos: ejecutá las dos secciones. Los scripts son distintos porque cada herramienta entrega datos distintos al hook.
- Si una IA está instalando esto para un usuario, antes de escribir archivos debe preguntarle qué modo quiere: `dialog`, `banner`, `persistent` o `transient`. Si el usuario no elige, usar `dialog`. Después debe poner ese valor explícitamente en la línea `MODE="..."` de cada script que instale.

## Requisitos

Linux con escritorio:

- `zenity` para el modo default: ventana real con botón OK.
- `yad` para el modo `banner`: ventana grande centrada, con texto enorme y color (verde/naranja), que queda abierta hasta que la cerrás. Pensado para ver de lejos desde otro monitor.
- `notify-send` (`libnotify-bin`) para los modos de notificación `persistent` y `transient`.
- `paplay` o `pw-play` para el sonido.
- `jq` para leer el JSON del hook.
- Sonidos en `/usr/share/sounds/freedesktop/stereo/`.

Antes de instalar nada, verificar si ya está disponible:

```bash
command -v notify-send >/dev/null && echo "notify-send OK" || echo "falta notify-send"
command -v zenity >/dev/null && echo "zenity OK" || echo "falta zenity"
command -v yad >/dev/null && echo "yad OK (modo banner)" || echo "falta yad (solo si querés el modo banner)"
command -v jq >/dev/null && echo "jq OK" || echo "falta jq"
{ command -v paplay >/dev/null || command -v pw-play >/dev/null; } && echo "audio OK" || echo "falta paplay o pw-play"
test -f /usr/share/sounds/freedesktop/stereo/complete.oga && echo "sonidos OK" || echo "faltan sonidos freedesktop"
```

Instalar paquetes solo si alguno de esos chequeos falla. No uses `sudo` ni instales paquetes si las herramientas ya están disponibles.

Debian/Ubuntu, solo si falta algo:

```bash
sudo apt install zenity yad libnotify-bin pulseaudio-utils jq sound-theme-freedesktop
```

Fedora, solo si falta algo:

```bash
sudo dnf install zenity yad libnotify jq
```

Arch, solo si falta algo:

```bash
sudo pacman -S zenity yad libnotify jq sound-theme-freedesktop
```

El modo por defecto es `dialog`: una ventana real con botón OK, usando `zenity`. Las otras opciones quedan disponibles: `banner` usa `yad` para mostrar una ventana grande centrada (verde "TERMINÓ" / naranja "PIDE PERMISO" o "ESPERA INPUT") que queda abierta hasta cerrarla, ideal para ver de lejos; `persistent` usa `notify-send -u critical -t 0`, y `transient` usa `notify-send -u normal -t 8000`.

## Instalar en Claude Code

Claude Code usa `~/.claude/settings.json` para registrar hooks. Esta instalación usa `Stop` para avisar cuando terminó el turno y `Notification` para avisar cuando Claude está esperando atención sin haber terminado, por ejemplo un permiso (`permission_prompt`) o input/revisión después de estar idle (`idle_prompt`). El hook `Stop` recibe un JSON con `cwd` y `transcript_path`; el script usa el transcript para mostrar el título autogenerado del chat cuando existe.

### 1. Crear el script

```bash
mkdir -p ~/.claude/hooks
cat > ~/.claude/hooks/notify-stop.sh <<'EOF'
#!/usr/bin/env bash
# Claude Code hooks: popup + sonido cuando Claude termina o necesita atención.

# MODO: "dialog" ventana con OK; "banner" ventana grande centrada (yad) que queda
#       hasta cerrarla; "persistent" queda hasta click; "transient" se autocierra.
MODE="dialog"

# Solo para el modo "banner": tamaño de la ventana en px.
BANNER_WIDTH=900
BANNER_HEIGHT=480

# Aislar Cursor: el "Cursor Hooks Service" también lee ~/.claude/settings.json y
# ejecuta este mismo hook al terminar su agente, generando notificaciones "fantasma"
# aunque el CLI de Claude esté cerrado. El CLI de Claude siempre exporta CLAUDECODE=1;
# Cursor no. Si no nos invocó el CLI de Claude, salimos sin notificar.
if [ -z "${CLAUDECODE:-}" ]; then
  exit 0
fi

input=$(cat)

json_field() {
  [ -n "$input" ] || return 0
  printf '%s' "$input" | jq -r "$1 // empty" 2>/dev/null
}

kind="${CLAUDE_NOTIFY_KIND:-}"
[ -n "$kind" ] || kind=$(json_field '.notification_type')
[ -n "$kind" ] || kind=$(json_field '.notificationType')
[ -n "$kind" ] || kind=$(json_field '.matcher')
[ -n "$kind" ] || kind=$(json_field '.hook_event_name')

cwd=$(json_field '.cwd')
[ -z "$cwd" ] && cwd="$PWD"
folder=$(basename "$cwd")

transcript=$(json_field '.transcript_path')
chat=""
if [ -n "$transcript" ] && [ -f "$transcript" ]; then
  chat=$(jq -r 'select(.type=="ai-title") | .aiTitle' "$transcript" 2>/dev/null | tail -1)
fi

export DISPLAY="${DISPLAY:-:1}"
export XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR:-/run/user/$(id -u)}"

run_detached() {
  if command -v setsid >/dev/null 2>&1; then
    setsid -f "$@" >/dev/null 2>&1 </dev/null
  else
    nohup "$@" >/dev/null 2>&1 </dev/null &
  fi
}

# headline y color se usan solo en el modo "banner".
case "$kind" in
  permission_prompt)
    title="Claude pide permiso"
    headline="PIDE PERMISO"
    color="#b35900"
    body="Revisá la autorización pendiente.
$cwd"
    ;;
  idle_prompt)
    title="Claude espera input"
    headline="ESPERA INPUT"
    color="#b35900"
    body="Hay una pregunta o revisión pendiente.
$cwd"
    ;;
  *)
    headline="TERMINÓ"
    color="#1f7a1f"
    if [ -n "$chat" ]; then
      title="Claude terminó: $chat"
      body="Es tu turno.
$cwd"
    else
      title="Claude terminó: $folder"
      body="Es tu turno.
$cwd"
    fi
    ;;
esac

# Escapar para markup Pango (lo usa yad en el modo banner).
pango_escape() {
  printf '%s' "$1" | sed -e 's/&/\&amp;/g' -e 's/</\&lt;/g' -e 's/>/\&gt;/g'
}

sound="/usr/share/sounds/freedesktop/stereo/complete.oga"
[ -f "$sound" ] || sound="/usr/share/sounds/freedesktop/stereo/alarm-clock-elapsed.oga"

if command -v paplay >/dev/null 2>&1; then
  run_detached paplay "$sound"
elif command -v pw-play >/dev/null 2>&1; then
  run_detached pw-play "$sound"
else
  printf '\a\a' >/dev/null
fi

case "$MODE" in
  banner)
    if command -v yad >/dev/null 2>&1; then
      h=$(pango_escape "$headline")
      b=$(pango_escape "$body")
      # El color va en el TEXTO (no en --back, que el tema GTK suele ignorar dejando fondo blanco).
      text="<span font='64' weight='bold' foreground='$color'>$h</span>
<span font='24' foreground='#222222'>$b</span>"
      # Sin --timeout: la ventana queda hasta que la cierres (botón Cerrar / Esc).
      run_detached yad \
        --width="$BANNER_WIDTH" --height="$BANNER_HEIGHT" --center --on-top \
        --text-align=center --justify=center \
        --button="Cerrar:0" \
        --title="$title" \
        --text="$text"
    else
      run_detached zenity --info --title="$title" --text="$body" --no-wrap
    fi
    ;;
  dialog)
    if command -v zenity >/dev/null 2>&1; then
      run_detached zenity --info --title="$title" --text="$body" --no-wrap
    else
      run_detached notify-send -a "Claude Code" -u critical -t 0 -i dialog-information "$title" "$body"
    fi
    ;;
  persistent)
    run_detached notify-send -a "Claude Code" -u critical -t 0 -i dialog-information "$title" "$body"
    ;;
  transient)
    run_detached notify-send -a "Claude Code" -u normal -t 8000 -i dialog-information "$title" "$body"
    ;;
esac

exit 0
EOF
chmod +x ~/.claude/hooks/notify-stop.sh
```

### 2. Registrar el hook

Este comando fusiona los hooks en `~/.claude/settings.json` sin borrar otras claves del archivo:

```bash
tmp=$(mktemp)
if [ -f ~/.claude/settings.json ]; then
  jq '
    .hooks.Stop = [{"hooks":[{"type":"command","command":"~/.claude/hooks/notify-stop.sh","timeout":10}]}] |
    .hooks.Notification = [
      {"matcher":"permission_prompt","hooks":[{"type":"command","command":"env CLAUDE_NOTIFY_KIND=permission_prompt ~/.claude/hooks/notify-stop.sh","timeout":10}]},
      {"matcher":"idle_prompt","hooks":[{"type":"command","command":"env CLAUDE_NOTIFY_KIND=idle_prompt ~/.claude/hooks/notify-stop.sh","timeout":10}]}
    ]
  ' \
    ~/.claude/settings.json > "$tmp"
else
  jq -n '{
    "hooks": {
      "Stop": [{"hooks":[{"type":"command","command":"~/.claude/hooks/notify-stop.sh","timeout":10}]}],
      "Notification": [
        {"matcher":"permission_prompt","hooks":[{"type":"command","command":"env CLAUDE_NOTIFY_KIND=permission_prompt ~/.claude/hooks/notify-stop.sh","timeout":10}]},
        {"matcher":"idle_prompt","hooks":[{"type":"command","command":"env CLAUDE_NOTIFY_KIND=idle_prompt ~/.claude/hooks/notify-stop.sh","timeout":10}]}
      ]
    }
  }' > "$tmp"
fi
mv "$tmp" ~/.claude/settings.json
```

### 3. Activar y probar

Abrí `/hooks` en Claude Code o reiniciá Claude Code.

```bash
echo '{"cwd":"/tmp/mi-proyecto","transcript_path":""}' | CLAUDECODE=1 ~/.claude/hooks/notify-stop.sh
echo '{"cwd":"/tmp/mi-proyecto"}' | CLAUDECODE=1 CLAUDE_NOTIFY_KIND=permission_prompt ~/.claude/hooks/notify-stop.sh
echo '{"cwd":"/tmp/mi-proyecto"}' | CLAUDECODE=1 CLAUDE_NOTIFY_KIND=idle_prompt ~/.claude/hooks/notify-stop.sh
```

> La guarda `CLAUDECODE` hace que el script solo notifique cuando lo invoca el CLI de Claude. Por eso las pruebas manuales necesitan `CLAUDECODE=1`; sin esa variable el script sale en silencio (es justamente lo que evita las notificaciones fantasma de Cursor).

## Instalar en Codex

Codex usa `~/.codex/config.toml` o `~/.codex/hooks.json` para hooks. Esta receta usa `config.toml` con `Stop` para fin de turno.

Diferencias importantes:

- Codex pide confiar hooks desde `/hooks` cuando cambia la definición o el hash.
- El hook se ejecuta con el `cwd` de la sesión.
- Codex no usa el mismo `transcript_path` con `ai-title` de Claude. El script intenta varios campos (`thread_name`, `title`, `session_id`) y después cae al nombre de la carpeta.
- No usar `PermissionRequest` para este feature si Codex tiene auto-approve: ese hook dispara cuando se crea la solicitud de permiso, no necesariamente cuando queda esperando al usuario. Con auto-approve produce falsos positivos, porque abre la ventana aunque la acción ya se haya aprobado sola.
- No vi un `idle_prompt` equivalente al de Claude en la documentación local de Codex. Para preguntas normales, `Stop` debería cubrir el caso si el turno queda en manos del usuario.
- Se usa un wrapper estable (`~/.codex/play-finish-sound.sh`) para que el comando registrado en `config.toml` cambie lo menos posible.

### 1. Crear el script real

```bash
mkdir -p ~/.codex/hooks
cat > ~/.codex/hooks/notify-stop.sh <<'EOF'
#!/usr/bin/env bash
# Codex Stop hook: popup + sonido cuando Codex termina y es tu turno.

# MODO: "dialog" ventana con OK; "banner" ventana grande centrada (yad) que queda
#       hasta cerrarla; "persistent" queda hasta click; "transient" se autocierra.
MODE="dialog"

# Solo para el modo "banner": tamaño de la ventana en px.
BANNER_WIDTH=900
BANNER_HEIGHT=480

input=$(cat)

json_field() {
  [ -n "$input" ] || return 0
  printf '%s' "$input" | jq -r "$1 // empty" 2>/dev/null
}

first_nonempty() {
  for value in "$@"; do
    [ -n "$value" ] && printf '%s\n' "$value" && return 0
  done
}

cwd=$(first_nonempty \
  "$(json_field '.cwd')" \
  "$(json_field '.session.cwd')" \
  "$(json_field '.payload.cwd')")
[ -n "$cwd" ] || cwd="$PWD"
folder=$(basename "$cwd")

session_id=$(first_nonempty \
  "$(json_field '.session_id')" \
  "$(json_field '.sessionId')" \
  "$(json_field '.thread_id')" \
  "$(json_field '.threadId')" \
  "$(json_field '.conversation_id')" \
  "$(json_field '.conversationId')" \
  "$(json_field '.payload.session_id')" \
  "$(json_field '.payload.thread_id')" \
  "$(json_field '.payload.threadId')")

chat=$(first_nonempty \
  "$(json_field '.thread_name')" \
  "$(json_field '.threadName')" \
  "$(json_field '.title')" \
  "$(json_field '.payload.thread_name')" \
  "$(json_field '.payload.threadName')" \
  "$(json_field '.payload.title')")

if [ -z "$chat" ] && [ -n "$session_id" ] && [ -f "$HOME/.codex/session_index.jsonl" ]; then
  chat=$(jq -r --arg id "$session_id" 'select(.id == $id) | .thread_name // empty' \
    "$HOME/.codex/session_index.jsonl" 2>/dev/null | tail -1)
fi

transcript=$(first_nonempty \
  "$(json_field '.transcript_path')" \
  "$(json_field '.transcriptPath')" \
  "$(json_field '.session_path')" \
  "$(json_field '.sessionPath')" \
  "$(json_field '.payload.transcript_path')" \
  "$(json_field '.payload.session_path')")

if [ -z "$chat" ] && [ -n "$transcript" ] && [ -f "$transcript" ]; then
  chat=$(jq -r '
    .payload.thread_name? //
    .payload.threadName? //
    .payload.title? //
    .thread_name? //
    .threadName? //
    .title? //
    empty
  ' "$transcript" 2>/dev/null | tail -1)
fi

if [ -z "$chat" ] && [ -n "$session_id" ] && [ -f "$HOME/.codex/history.jsonl" ]; then
  first_prompt=$(jq -r --arg id "$session_id" 'select(.session_id == $id) | .text // empty' \
    "$HOME/.codex/history.jsonl" 2>/dev/null | head -1)
  if [ -n "$first_prompt" ]; then
    chat=$(printf '%s' "$first_prompt" | tr '\n' ' ' | cut -c 1-80)
  fi
fi

export DISPLAY="${DISPLAY:-:1}"
export XDG_RUNTIME_DIR="${XDG_RUNTIME_DIR:-/run/user/$(id -u)}"

run_detached() {
  if command -v setsid >/dev/null 2>&1; then
    setsid -f "$@" >/dev/null 2>&1 </dev/null
  else
    nohup "$@" >/dev/null 2>&1 </dev/null &
  fi
}

# headline y color se usan solo en el modo "banner".
headline="TERMINÓ"
color="#1f7a1f"
if [ -n "$chat" ]; then
  title="Codex terminó: $chat"
  body="Es tu turno.
$cwd"
else
  title="Codex terminó: $folder"
  body="Es tu turno.
$cwd"
fi

# Escapar para markup Pango (lo usa yad en el modo banner).
pango_escape() {
  printf '%s' "$1" | sed -e 's/&/\&amp;/g' -e 's/</\&lt;/g' -e 's/>/\&gt;/g'
}

sound="/usr/share/sounds/freedesktop/stereo/complete.oga"
[ -f "$sound" ] || sound="/usr/share/sounds/freedesktop/stereo/alarm-clock-elapsed.oga"

if command -v paplay >/dev/null 2>&1; then
  run_detached paplay "$sound"
elif command -v pw-play >/dev/null 2>&1; then
  run_detached pw-play "$sound"
else
  printf '\a\a' >/dev/null
fi

case "$MODE" in
  banner)
    if command -v yad >/dev/null 2>&1; then
      h=$(pango_escape "$headline")
      b=$(pango_escape "$body")
      # El color va en el TEXTO (no en --back, que el tema GTK suele ignorar dejando fondo blanco).
      text="<span font='64' weight='bold' foreground='$color'>$h</span>
<span font='24' foreground='#222222'>$b</span>"
      # Sin --timeout: la ventana queda hasta que la cierres (botón Cerrar / Esc).
      run_detached yad \
        --width="$BANNER_WIDTH" --height="$BANNER_HEIGHT" --center --on-top \
        --text-align=center --justify=center \
        --button="Cerrar:0" \
        --title="$title" \
        --text="$text"
    else
      run_detached zenity --info --title="$title" --text="$body" --no-wrap
    fi
    ;;
  dialog)
    if command -v zenity >/dev/null 2>&1; then
      run_detached zenity --info --title="$title" --text="$body" --no-wrap
    else
      run_detached notify-send -a "Codex" -u critical -t 0 -i dialog-information "$title" "$body"
    fi
    ;;
  persistent)
    run_detached notify-send -a "Codex" -u critical -t 0 -i dialog-information "$title" "$body"
    ;;
  transient)
    run_detached notify-send -a "Codex" -u normal -t 8000 -i dialog-information "$title" "$body"
    ;;
esac

exit 0
EOF
chmod +x ~/.codex/hooks/notify-stop.sh
```

### 2. Crear el wrapper estable

```bash
cat > ~/.codex/play-finish-sound.sh <<'EOF'
#!/usr/bin/env sh

exec "$HOME/.codex/hooks/notify-stop.sh"
EOF
chmod +x ~/.codex/play-finish-sound.sh
```

### 3. Registrar el hook

Este comando agrega el hook a `~/.codex/config.toml` usando la ruta real de tu `$HOME`. No vuelve a agregarlo si ya encuentra `play-finish-sound.sh` en la config:

```bash
mkdir -p ~/.codex
touch ~/.codex/config.toml

if ! grep -Fq ".codex/play-finish-sound.sh" ~/.codex/config.toml; then
  cat >> ~/.codex/config.toml <<EOF

[[hooks.Stop]]
[[hooks.Stop.hooks]]
type = "command"
command = "sh $HOME/.codex/play-finish-sound.sh"
timeout = 10
EOF
fi
```

Los hooks de Codex vienen habilitados por defecto. Si en esa compu los deshabilitaste explícitamente, asegurate de que `hooks = true` quede dentro de la tabla `[features]` existente. Si no existe esa tabla, podés crearla así:

```toml
[features]
hooks = true
```

Abrí `/hooks` en Codex y confiá el hook si aparece como nuevo o cambiado.

### 4. Probar Codex

```bash
echo '{"cwd":"/tmp/mi-proyecto","thread_name":"Prueba Codex"}' | ~/.codex/hooks/notify-stop.sh
sh ~/.codex/play-finish-sound.sh
```

## Documentos opcionales

Si querés llevar este Markdown en `~/Documentos`, podés crear symlinks:

```bash
ln -sf ~/.claude/hooks/instalar-notificaciones-claude-codex.md ~/Documentos/notificaciones-claude-code.md
ln -sf ~/.claude/hooks/instalar-notificaciones-claude-codex.md ~/Documentos/notificaciones-codex.md
```

## Personalizar

- Cambiar modo: `MODE="dialog"`, `MODE="banner"`, `MODE="persistent"` o `MODE="transient"`.
- Cambiar sonido: editá la variable `sound`.
- Sin sonido: borrá el bloque de `paplay`/`pw-play`.
- En modo dialog: requiere `zenity`; si falta, el script cae a `notify-send` persistente.
- En modo banner: requiere `yad`; si falta, cae a `zenity`. Ajustá el tamaño con `BANNER_WIDTH`/`BANNER_HEIGHT` y los tamaños de letra en `font='64'` (titular) y `font='24'` (detalle). El color va en el texto (`foreground`), no en el fondo, porque muchos temas GTK ignoran `--back` y dejarían la ventana blanca con texto invisible. La ventana no tiene timeout: queda hasta que la cerrás (botón Cerrar o Esc).
- En modo persistent: usa `notify-send -u critical -t 0`.
- En modo transient: cambiá `-t 8000` para ajustar la duración en milisegundos.

## Troubleshooting

- Si no aparece nada desde el hook pero sí desde una terminal, revisá `DISPLAY` y `XDG_RUNTIME_DIR` en el script.
- Si aparece `hook timed out`, revisá que el script lance `zenity`, `notify-send` y el audio mediante `run_detached` (`setsid -f` o `nohup`) y que el timeout del hook sea razonable. Una ventana `zenity` sin desacoplar puede dejar al hook esperando hasta que Codex/Claude lo corte.
- En Claude Code, validá `settings.json` con `jq . ~/.claude/settings.json`.
- En Codex, si no corre, abrí `/hooks` y verificá que el hook esté confiado.
- En WSL/headless no hay escritorio donde mostrar `notify-send`.
