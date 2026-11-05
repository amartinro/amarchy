# amarchy

Portabilidad de la personalización Omarchy (Hyprland + Quickshell) entre máquinas.
Un único script interactivo: exporta en la máquina origen, importa en el destino y
audita las diferencias dependientes de cada máquina.

```
~/.config/
├── hypr/          13 archivos Lua/conf propios
└── omarchy/
    ├── shell.json        layout de la barra (5 secciones)
    ├── dock.json         12 entradas del dock
    ├── plugins/          8 plugins Quickshell
    ├── themes/           4 temas
    └── hooks/            hooks *.d
```

## Uso

Máquina origen:

```bash
./amarchy        # menú → `e` exporta → escribe ~/omarchy-personal.tar.gz (+ .manifest)
```

Máquina destino (misma versión de script + payload):

```bash
./amarchy        # menú → `i` importa + audita → aplica, o Enter para aceptar por defecto
```

El modo inicial se elige solo: si existe `~/omarchy-personal.tar.gz` arranca en `import`,
si no, en `export`. `OMARCHY_MIG_PAYLOAD=<ruta>` cambia la ruta del payload.

Dentro del menú: dígitos `1-21` alternan grupo, `a` todos, `n` ninguno, `r` reset,
`e` export, `i` import + auditoría, `s` entorno, `p` payload, `h` ayuda, `q` salir,
Enter aplica el modo inicial. Opción no reconocida → aviso con la lista válida.
Cada fila muestra `●`/`○`, un glifo de categoría, nº de archivos, peso real y rutas
ausentes; la cabecera lleva versión de Omarchy, de Hyprland y el nº de grupos.
Los paneles `s` y `p` son de sólo lectura: `s` imprime Omarchy, Hyprland, tema, iconos,
terminal, estado de `hyprpm`, plugins cargados y monitores; `p`, tamaño, nº de entradas
y el manifiesto del origen.

## Los 21 grupos

| # | Grupo | Contenido |
|---|-------|-----------|
| 1 | Tema activo · tahoe-dark | `omarchy/themes/tahoe-dark` + `state/current/theme` (+ `theme.name`, `theme.generation`, `background`) |
| 2 | Temas extra | `dos-moos`, `hermarchy`, `noir` |
| 3 | Fondo propio | `omarchy/backgrounds/tahoe-dark/rrt1.jpg` |
| 4 | Barra superior | `omarchy/shell.json`, `omarchy/shell.toml` |
| 5 | system-monitor | `omarchy/bar/modules/system-monitor.qml` |
| 6 | Widgets propios | `amartin.clock`, `amartin.kairos`, `amartin.monitor`, `amartin.weather`, `amartin.mouse-mapper`, `amartin.notifications` |
| 7 | Dock | `plugins/dock` + `omarchy/dock.json` |
| 8 | omdrop | `netmojo.omdrop` (AirDrop, v0.15.1) |
| 9 | Hyprland | `hypr/` completo (13 archivos) |
| 10 | Iconos | `icons.theme` (Yaru-blue-dark) + paquete `yaru-icon-theme` |
| 11 | Terminales | ghostty, alacritty, foot, kitty |
| 12 | Prompt/TUI | `starship.toml`, `btop.conf`, `btop/themes` |
| 13 | Dictado | `voxtype/config.toml` |
| 14 | Entrada | `fcitx5/` (conf, profile, 8 conf) |
| 15 | XDG/GTK | `mimeapps.list`, `xdg-terminals.list`, `user-dirs.dirs`, `gtk-3.0/bookmarks` |
| 16 | Metadatos | `omarchy/language`, `branding`, `extensions/omarchy-menu.jsonc`, `defaults` |
| 17 | Hooks | `omarchy/hooks/*.d` |
| 18 | Share picker | `hyprland-preview-share-picker/config.yaml` |
| 19 | Estado runtime | `state/settings/weather.json`, `state/mouse-mapper.json` |
| 20 | Paquetes | `pacman -Qqen` (repo) |
| 21 | AUR | `pacman -Qqm` |

## Auditoría de máquina destino

Tras importar, el script compara el manifiesto del origen con el destino en vivo
(`hyprctl`, `omarchy`, `fc-list`, `pacman`, `curl`, `jq`) y adapta lo que puede:

| # | Comprobación | Adaptación |
|---|--------------|------------|
| 1 | Versión Hyprland origen↔destino | si `<0.55`, avisa de que `rounding_power = 2.0` se ignora; recuerda `hyprpm update` para la ABI de `hyprbars`/`hymission` |
| 2 | Versión Omarchy | avisa de defaults posiblemente distintos |
| 3 | Monitores (`hyprctl monitors -j`) | **reescribe `hypr/monitors.lua`** con la topología real: `output`, `mode = WxH`, `scale`, y `transform` solo si ≠0 |
| 4 | Fuente del tema (`font-family`) | `fc-list` → si falta, instala `ttf-jetbrains-mono-nerd-basic`; si existe con otro nombre exacto, lo sugiere |
| 5 | Tema de iconos | si no está `/usr/share/icons/<x>`, instala `yaru-icon-theme` |
| 6 | Terminal XDG | si el `.desktop` del origen no existe, reescribe `xdg-terminals.list` con el primero disponible (ghostty → kitty → foot) |
| 7 | `kb_layout` activo vs origen | muestra la divergencia |
| 8 | Endpoint Kairos | `curl` con `-m 3` al `/api/metrics` del widget `amartin.kairos`; sin respuesta, el widget queda vacío |
| 9 | Ciudad del clima | informativo |
| 10 | ids de la barra en `shell.json` | cada `id` debe resolverse contra los `manifest.json` de `/usr/share/omarchy/shell` y `~/.config/omarchy/plugins` (incluidos los módulos `.qml`); lista los sin plugin |
| 11 | ABI de los plugins | tres estados: sin `.so` en `/var/cache/hyprpm/$USER` (titulos y Mission Control caídos), `Hyprland` más nuevo que los `.so` (reconstruir; si `hyprpm` no está en PATH se indica que los actuales siguen cargando), o al día |

El manifiesto también viaja suelto: `~/omarchy-personal.tar.gz.manifest`
(campos `theme`, `hyprland`, `omarchy`, `font`, `icons`, `terminal`, `kb_layout`,
`kairos`, `weather`, `monitors`, `gen`). `theme` se lee de
`~/.local/state/omarchy/current/theme.name`, así que el script sigue al tema activo
aunque cambie.

El import no usa `tar --pattern`: `tar` no casa nombres exactos con `*` y solapa
prefijos. Se desempaeta una vez en un stage y cada entrada se copia con
`install_entry` → `rm -rf` del destino + `cp -a src/. dst/`, lo que deja el árbol
idéntico al del export, sin anidar y sin ficheros huérfanos.

## Aplicación final

`finalize()` tras el import: `hyprpm update` (si existe) → `omarchy theme set <tema>`
(el leído de `current/theme.name`) → `omarchy restart shell` → `omarchy restart terminal`
→ `hyprctl reload` → `hyprctl configerrors` (sus líneas se imprimen si hay errores).

## Puntos de atención

- **ABI hyprpm.** `hyprbars.so` y `hymission.so` viven en `/var/cache/hyprpm/$USER` (fuera
  de `~/.config`, no viajan). Si Hyprland se actualiza y no se reconstruyen, se pierden
  títulos de ventana y Mission Control sin error explícito; el hook
  `post-update.d/90-hyprpm-plugins.hook` avisa. En builds donde `hyprpm` no está en PATH,
  los `.so` ya presentes siguen cargando desde `hypr/plugins.lua`.
- **`tahoe-dark` es local** (sin `origin`): sólo se recupera de este repo.
- **`rounding` en tres sitios**: `hypr/looknfeel.lua` (8), `hypr/tahoe-overrides.lua`
  (18 + `rounding_power 2.0`) y `themes/tahoe-dark/hyprland.lua` (`P.corner_radius` 18).
  Cambiar los tres a la vez.
- **`system-monitor`** usa `"type": "qml"` → depende de `omarchy/bar/modules/system-monitor.qml`.
- **Kairos** requiere `http://kairos:8787/api/metrics` resolvable.
- **Doble sistema de notificaciones**: `omarchy.notifications` está en `disabledPlugins` y
  manda `amartin.notifications`; no re-habilitar el primero.

## Repos enlazados

| Pieza | Origen |
|-------|--------|
| `amarchy` (este script + payload) | `amartinro/amarchy` |
| Tema Tahoe | `amartinro/omactheme` |
| Índice/config general | `amartinro/amartinomarchy` |
| `dos-moos` | https://github.com/HANCORE-linux/omarchy-dos-moos-theme |
| `hermarchy` | https://github.com/archer-clawbot/omarchy-hermarchy-theme |
| `noir` | https://github.com/tahadx/omarchy-noir-theme |
| `omdrop` | https://github.com/brentkearney/omdrop-plugin |

Los tres temas y `omdrop` son clones git directos en `~/.config/omarchy/...`;
actualizar con `git -C <dir> pull --ff-only` (noir va rebaseado sobre su `origin/main`).
