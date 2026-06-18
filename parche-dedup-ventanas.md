# Parche: reemplazar ventanas en vez de apilarlas (anti-bombardeo)

Este parche aplica a **Claude Code y Codex**, y **solo a instalaciones previas**
(las hechas antes de que el script soportara el reemplazo por proyecto). Si vas a
instalar desde cero, no lo necesitás: seguí
[instalar-notificaciones-claude-codex.md](./instalar-notificaciones-claude-codex.md),
que ya trae el comportamiento nuevo.

## Cuándo aplicarlo

Aplicalo si se dan las dos condiciones:

1. Ya tenías instalado `~/.claude/hooks/notify-stop.sh` (o el de Codex
   `~/.codex/hooks/notify-stop.sh`) de una versión anterior.
2. El script **no** contiene la función `close_previous_window`. Para chequearlo:

   ```bash
   grep -q 'close_previous_window' ~/.claude/hooks/notify-stop.sh && echo "ya parcheado" || echo "falta el parche"
   grep -q 'close_previous_window' ~/.codex/hooks/notify-stop.sh && echo "codex: ya parcheado" || echo "codex: falta el parche"
   ```

Si dice `ya parcheado`, no hagas nada en ese script.

## Por qué

En flujos automáticos (auto mode, monitors o loops) la IA termina turno muchas
veces seguidas. Cada fin de turno dispara el evento `Stop`, que ejecuta el hook.
La versión vieja **no tenía deduplicación**: cada disparo abría una ventana nueva.
Como `zenity` (modo `dialog`) no puede reemplazar una ventana existente, los
popups se **apilaban** y aparecían 4-5 avisos idénticos del mismo proyecto a la vez.

La versión nueva muestra como máximo **una ventana por carpeta** (`cwd`): antes de
abrir un aviso nuevo, cierra el anterior del mismo proyecto. Para eso guarda el PID
de la ventana en `$XDG_RUNTIME_DIR/claude-notify/` (o `codex-notify/`) y lo mata
antes de abrir la siguiente. Funciona con los modos de ventana real (`dialog` con
`zenity` y `banner` con `yad`). Los modos `persistent`/`transient` usan
`notify-send`, que no deja proceso vivo al que cerrar; reemplazarlos requiere
`notify-send >= 0.8` (`--replace-id`), así que con versiones anteriores esos modos
siguen apilando (no es el caso del modo por defecto `dialog`).

No se toca `~/.claude/settings.json` ni `~/.codex/config.toml`: el cambio vive
entero en el script.

## Cómo aplicarlo

La forma más simple es regenerar el script desde la receta conservando tu `MODE`
(y, en Claude, tu `IDLE_MODE`):

```bash
# 1) Recordar configuración actual antes de sobrescribir.
prev_mode=$(grep -m1 -oP '^MODE="\K[^"]*' ~/.claude/hooks/notify-stop.sh 2>/dev/null)
[ -n "$prev_mode" ] || prev_mode="dialog"
prev_idle=$(grep -m1 -oP '^IDLE_MODE="\K[^"]*' ~/.claude/hooks/notify-stop.sh 2>/dev/null)
[ -n "$prev_idle" ] || prev_idle="off"

# 2) Regenerar el/los script(s): ejecutá el bloque "1. Crear el script" de la
#    sección correspondiente del instalar-notificaciones-claude-codex.md tal cual.
#    Eso sobrescribe ~/.claude/hooks/notify-stop.sh (y/o el de Codex) con la
#    versión nueva, que ya trae close_previous_window y launch_window.

# 3) Restaurar tu configuración previa en Claude.
sed -i "s/^MODE=\"[^\"]*\"/MODE=\"$prev_mode\"/" ~/.claude/hooks/notify-stop.sh
sed -i "s/^IDLE_MODE=\"[^\"]*\"/IDLE_MODE=\"$prev_idle\"/" ~/.claude/hooks/notify-stop.sh
```

Después reiniciá Claude Code / Codex o abrí `/hooks`.

### Alternativa: editar a mano (sin regenerar)

Si preferís no regenerar, alcanza con tres cambios en cada script:

1. Justo después de la definición de `run_detached()`, agregar el estado y las dos
   funciones (en Codex usar `codex-notify` en vez de `claude-notify`):

   ```bash
   notify_state_dir="${XDG_RUNTIME_DIR:-/tmp}/claude-notify"
   mkdir -p "$notify_state_dir" 2>/dev/null
   notify_key=$(printf '%s' "$cwd" | cksum | cut -d' ' -f1)
   notify_pidfile="$notify_state_dir/$notify_key.pid"

   close_previous_window() {
     [ -f "$notify_pidfile" ] || return 0
     local oldpid
     oldpid=$(cat "$notify_pidfile" 2>/dev/null)
     if [ -n "$oldpid" ] && kill -0 "$oldpid" 2>/dev/null; then
       kill "$oldpid" 2>/dev/null
     fi
     rm -f "$notify_pidfile" 2>/dev/null
   }

   launch_window() {
     nohup "$@" >/dev/null 2>&1 </dev/null &
     printf '%s' "$!" > "$notify_pidfile"
   }
   ```

2. En la rama `dialog` del `case`, anteponer `close_previous_window` y cambiar
   `run_detached zenity ...` por `launch_window zenity ...`.

3. En la rama `banner` (si la tenés), anteponer `close_previous_window` y cambiar
   tanto `run_detached yad ...` como su fallback `run_detached zenity ...` por
   `launch_window ...`.

   Las ramas `persistent`/`transient` (`notify-send`) se dejan sin tocar.

## Verificar

```bash
# Dos disparos para la MISMA carpeta -> debe quedar UNA sola ventana.
echo '{"cwd":"/tmp/x","transcript_path":""}' | CLAUDECODE=1 ~/.claude/hooks/notify-stop.sh
echo '{"cwd":"/tmp/x","transcript_path":""}' | CLAUDECODE=1 ~/.claude/hooks/notify-stop.sh
#   -> en pantalla hay 1 ventana 'x' (la primera se cerró sola), no 2.

# Carpetas distintas conviven.
echo '{"cwd":"/tmp/y","transcript_path":""}' | CLAUDECODE=1 ~/.claude/hooks/notify-stop.sh
#   -> ahora hay 2 ventanas: x e y.

# El idle sigue mudo y el permiso sigue funcionando.
echo '{"cwd":"/tmp/x"}' | CLAUDECODE=1 CLAUDE_NOTIFY_KIND=idle_prompt ~/.claude/hooks/notify-stop.sh; echo "exit=$?"
echo '{"cwd":"/tmp/x"}' | CLAUDECODE=1 CLAUDE_NOTIFY_KIND=permission_prompt ~/.claude/hooks/notify-stop.sh
```

Cerrá las ventanas de prueba al terminar.
