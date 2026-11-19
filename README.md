# Clippy

Grabador de repeticiones instantáneas para Windows, ultraliviano: graba siempre en segundo plano
(en RAM) y con una tecla (F9) guarda los últimos segundos como MP4. Todo el video pasa por la GPU
(DXGI Desktop Duplication → D3D11 NV12 → encoder H.264 por hardware de Media Foundation). Si la placa no
tiene encoder de video, usa el H.264 de Windows por procesador, hasta 1080p a 30 fps.

Descarga y novedades en [clpy.vercel.app](https://clpy.vercel.app). Las versiones se publican en
los [Releases](https://github.com/santiremedi/clippy/releases) de este repo, que es donde las busca
el actualizador automático.

## Organización de clips

Los clips nuevos se organizan dentro de la carpeta de destino configurada:

- **Con juego detectado:** una carpeta con el nombre del juego, tanto en “Toda la pantalla”
  como en “Solo juegos”. Se usa ProductName del ejecutable o, si no está disponible, su nombre sin `.exe`.
- **Sin juego detectado:** `DesktopCapture/`, en ambos modos.

El modo de captura decide qué imagen se graba, no cómo se organizan los archivos. Por ejemplo,
un clip de CS2 en “Toda la pantalla” va a la carpeta del juego, no a `DesktopCapture/`.

Ejemplo ilustrativo (los nombres reales dependen de los metadatos del ejecutable):

```text
Clips/
├── Counter-Strike 2/
│   ├── Clippy_2026-11-19_20-00-00.mp4
│   └── Clippy_2026-11-19_20-05-00.mp4
├── Minecraft/
│   └── Clippy_2026-11-19_21-00-00.mp4
└── DesktopCapture/
    └── Clippy_2026-11-19_22-00-00.mp4
```

Los nombres se adaptan a las restricciones de Windows. Las carpetas se crean al guardar y se
reutilizan; los clips anteriores no se mueven. La biblioteca ya permite navegar las subcarpetas.
Se reutiliza el detector de juegos, que mantiene el juego al hacer Alt+Tab, en vez de usar el nombre
de otra aplicación que esté en primer plano al guardar.

## Compilar

```bash
cargo build --release
```

El exe queda en `target/release/clippy.exe`. Para publicar se renombra a `Clippy.exe`
(el actualizador busca ese nombre exacto en el último release de `santiremedi/clippy`).

Para probar mientras desarrollás, con los modos de prueba sin ventana (no van en el exe publicado).
Compila en segundos porque no usa LTO; queda en `target/fast/clippy.exe`:

```bash
cargo build --profile fast --features devtools
```

## Modos del mismo .exe

| Argumentos | Qué hace |
| --- | --- |
| (ninguno) / `--background` | Grabadora con ícono en la bandeja (`--background`: sin abrir la ventana, para el inicio con Windows). |
| `--settings` | Ventana principal / ajustes. |
| `--editor [archivo]` | Editor de clips. |
| `--apply-update <ruta>` | Lo usa el actualizador: reemplaza el exe instalado y lo vuelve a abrir. |

Solo con `--features devtools`:

| Argumentos | Qué hace |
| --- | --- |
| `--test <seg> <carpeta> [game\|desktop\|mic\|juegos\|720\|1080\|1440]` | Graba, guarda un clip y sale (imprime el encoder y el juego detectado; un número elige la resolución del clip). Con la variable `CLIPPY_CPU_ENCODER=1`, codifica con el procesador como en una placa sin encoder. |
| `--test-mic` | Cuánto audio entrega cada micrófono en 3 s. |
| `--test-download` | Descarga la versión publicada como el botón "Actualizar". |
| `--test-export`, `--test-frame`, `--test-strip` | Pruebas del editor (ver `src/editor/devtools.rs`). |

## Mapa del código

```
src/
  main.rs          elige el modo
  config.rs        config.json (grabación, audio, atajos globales, idioma)
  i18n.rs          idioma de la interfaz (español, inglés, portugués; "auto" sigue a Windows) y `tr!`
  log.rs           %APPDATA%\Clippy, clippy.log y marcas de tiempo
  mf.rs            Media Foundation y el formato de audio de toda la app (PCM 48 kHz estéreo, AAC)
  clipcache.rs     caché de forma de onda y fotogramas clave por clip (la escribe la grabadora, la lee el editor)
  app/             el proceso como app de Windows
    mode.rs        modos, argumentos, candados, títulos de ventana, mensajes a la bandeja
    tray.rs        bandeja, atajos globales, controla la grabadora
    update.rs      actualizaciones (una consulta por día a Releases, como mucho)
    status.rs      recorder.json: con qué graba y por qué no puede (lo lee la ventana principal)
    autostart.rs   inicio con Windows (HKCU\...\Run)
    keys.rs        nombres de teclas Win32
  record/          grabación
    recorder.rs    hilo de grabación: captura -> encoder -> buffer -> guardar
    game.rs        qué es "el juego" (ventana en pantalla completa), para video y audio
    capture.rs, convert.rs, encoder.rs, audio.rs, mux.rs, clock.rs
  settings/        ventana principal (mod.rs: estado; pages.rs: páginas)
  editor/          editor de clips (proceso aparte)
    mod.rs         ventana y bucle principal
    timeline.rs    pistas y piezas (video y audio unidos), imán, fundidos, deshacer
    playback.rs    reproducción: un reproductor por pieza que se ve o se oye, capas y fundidos
    export.rs      diálogo y trabajo de exportación
    panels.rs      barra de arriba y línea de tiempo (pistas, marcadores, clic derecho en un clip)
    browser.rs     panel de la biblioteca: elegir, carpetas, mover y mandar a la Papelera
    inspector.rs   panel del clip: imagen, velocidad y sonido
    preview.rs     vista previa: lienzo con su proporción y recorte de la imagen
    prefs.rs       editor.json (atajos, últimas opciones de exportación, paneles, imán)
    media.rs       exportar: copia sin recodificar o capas ubicadas y fundidas por GPU, velocidad, mezcla de audio
    shortcuts.rs, library.rs, player.rs, devtools.rs
  ui_kit/          estilo y widgets, ventanas nativas, íconos, sonidos
    watch.rs       despierta una ventana cuando algo cambia afuera (sin repintar en reposo)
```

## Archivos del usuario

Todo en `%APPDATA%\Clippy`: `config.json`, `editor.json`, `library.json`, `update.json`,
`clippy.log` y `cache\`.

## Publicar una versión

1. Subir la versión en `Cargo.toml`.
2. En GitHub → Actions → **Build** → *Run workflow*, con el tag (por ejemplo `v0.3.0`) y
   "Publicar el release" tildado. El workflow compila y sube `Clippy.exe` al release de este repo
   (si no existe, lo crea como borrador). Sin tildarlo, igual compila y deja el exe como artefacto descargable.
3. Escribir el texto del release para quien usa la app (Clippy lo muestra como novedades: títulos `#`,
   viñetas `-` y párrafos, sin `**` ni comillas invertidas) y recién ahí publicarlo. Un release publicado
   sin texto no muestra novedades.

## Licencia

[GPL-3.0-or-later](LICENSE). Podés usar, estudiar, cambiar y redistribuir Clippy; si distribuís una
versión modificada, tiene que ser con su código y bajo la misma licencia. El nombre "Clippy" y su
ícono no se pueden usar para identificar versiones modificadas.
