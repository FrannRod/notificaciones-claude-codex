# Parche: apagar el aviso `idle_prompt` de Claude

Este parche es **solo para Claude Code** y **solo para instalaciones previas** de las notificaciones (las hechas antes de que el script soportara la variable `IDLE_MODE`). Si vas a instalar desde cero, no necesitás este parche: seguí [instalar-notificaciones-claude-codex.md](./instalar-notificaciones-claude-codex.md), que ya trae el comportamiento nuevo.

## Cuándo aplicarlo

Aplicalo si se dan las dos condiciones:

1. Ya tenías instalado `~/.claude/hooks/notify-stop.sh` de una versión anterior.
2. El script **no** contiene la línea `IDLE_MODE=`. Para chequearlo:

   ```bash
   grep -q '^IDLE_MODE=' ~/.claude/hooks/notify-stop.sh && echo "ya parcheado" || echo "falta el parche"
   ```

Si dice `ya parcheado`, no hagas nada. Codex no tiene un evento idle equivalente, así que su script no lleva `IDLE_MODE` y este parche no aplica.

## Por qué

En la versión vieja, el matcher `idle_prompt` de `~/.claude/settings.json` disparaba el script cada vez que Claude quedaba esperando tu input tras estar ocioso, y el script abría un popup `dialog` ("Claude espera input") igual que para el fin de turno. Ese evento se dispara seguido, así que el popup aparecía de forma molesta e impredecible (incluso desde sesiones en segundo plano).

La versión nueva mete un único control en el script —la variable `IDLE_MODE`— y lo deja en `off` por defecto: ante un `idle_prompt`, el script sale en silencio (sin popup ni sonido). Los otros avisos (fin de turno `Stop` y permiso `permission_prompt`) no cambian.

Importante: **el arreglo vive entero en el script.** No se toca `~/.claude/settings.json` (el matcher `idle_prompt` sigue registrado; ahora simplemente el script decide qué hacer). Eso mantiene el toggle en un solo lugar.

## Cómo aplicarlo

El parche conserva el `MODE` que tengas configurado, regenera el script con la versión nueva y, opcionalmente, te deja activar un aviso idle no intrusivo.

```bash
# 1) Recordar el MODE actual antes de sobrescribir (default dialog si no se encuentra).
prev_mode=$(grep -m1 -oP '^MODE="\K[^"]*' ~/.claude/hooks/notify-stop.sh 2>/dev/null)
[ -n "$prev_mode" ] || prev_mode="dialog"

# 2) Regenerar el script: ejecutá el bloque "1. Crear el script" de la sección
#    "Instalar en Claude Code" del instalar-notificaciones-claude-codex.md tal cual.
#    Eso sobrescribe ~/.claude/hooks/notify-stop.sh con la versión nueva
#    (que ya trae IDLE_MODE="off" y MODE="dialog").

# 3) Restaurar tu MODE previo.
sed -i "s/^MODE=\"[^\"]*\"/MODE=\"$prev_mode\"/" ~/.claude/hooks/notify-stop.sh

# 4) (Opcional) Si querés que el idle avise de forma no intrusiva (notificación que se autocierra):
#    sed -i 's/^IDLE_MODE="[^"]*"/IDLE_MODE="transient"/' ~/.claude/hooks/notify-stop.sh
```

Después reiniciá Claude Code o abrí `/hooks`.

### Alternativa: editar a mano (sin regenerar)

Si preferís no regenerar el script, alcanza con tres cambios:

1. Agregar la variable cerca de `MODE=`:

   ```bash
   IDLE_MODE="off"
   ```

2. Justo después de resolver `kind` (las líneas `kind=$(json_field ...)`), agregar el corte temprano:

   ```bash
   if [ "$kind" = "idle_prompt" ] && [ "$IDLE_MODE" = "off" ]; then
     exit 0
   fi
   ```

3. Reemplazar `case "$MODE" in` por un `case "$effective_mode" in`, precedido de:

   ```bash
   effective_mode="$MODE"
   if [ "$kind" = "idle_prompt" ] && [ "$IDLE_MODE" = "transient" ]; then
     effective_mode="transient"
   fi
   ```

## Verificar

```bash
echo '{"cwd":"/tmp/x"}' | CLAUDECODE=1 CLAUDE_NOTIFY_KIND=idle_prompt ~/.claude/hooks/notify-stop.sh; echo "exit=$?"
```

No debe abrir ninguna ventana y debe imprimir `exit=0`. El fin de turno y el aviso de permiso siguen funcionando como antes:

```bash
echo '{"cwd":"/tmp/x","transcript_path":""}' | CLAUDECODE=1 ~/.claude/hooks/notify-stop.sh
echo '{"cwd":"/tmp/x"}' | CLAUDECODE=1 CLAUDE_NOTIFY_KIND=permission_prompt ~/.claude/hooks/notify-stop.sh
```
