# TCL G13 (G13_2K_GB), Latinoamérica: ficha y limpieza

Sesión del 2026-10-20 en un televisor real, conectado por Wi-Fi.
Lista base: [`examples/tcl-43s5k/debloat.txt`](../../examples/tcl-43s5k/debloat.txt), mismo hardware
(G13, MT9221) pero firmware de India y más nuevo (AU10, parche 2025-12-01).

## Identificación

| Campo | Valor observado |
|---|---|
| Modelo comercial | **TCL 50S5K QLED**, Colombia (según la etiqueta; el firmware solo dice `Smart TV`, y `persist.tv.client_sku` = `co`) |
| `ro.product.manufacturer` / `brand` | `TCL` / `TCL` |
| `ro.product.device` | `G13` |
| `ro.product.name` | `G13_2K_GB` |
| Android / SDK | Google TV 14 / `34` |
| `ro.build.display.id` | `UKN2.241110.001 release-keys` |
| `ro.build.version.incremental` | `AU07` |
| `ro.software.version_id` | `V8-T221T09-LF1V069` |
| `vendor.mtk.factory.customer.clienttype` | `TCL-LA-MT9221-03` (Latinoamérica) |
| Parche de seguridad | `2025-06-01` |
| SoC | MediaTek MT9221-Lite (`ro.hardware` = `mt5867`, `ro.soc.model` = `t56`) |
| RAM | 954 MB, `ro.config.low_ram=true` |
| ABI | solo 32 bits (`armeabi-v7a`) |
| Pantalla | panel 1920x1080, interfaz a 1280x720 |
| `ro.adb.secure` | `1` (el puerto 5555 pide autorización) |

## Conexión

- Depuración inalámbrica (TLS): `adb pair <ip>:<puerto de vinculación> <código>` y luego
  `adb connect <ip>:<puerto>`.
- Los scripts (`tools/tv.py`, `disable.ps1`) usan adb-shell en TCP plano por el 5555, sin TLS. Por eso se
  abrió de forma temporal con `adb tcpip 5555`. `tv.py` reutiliza `~/.android/adbkey`, que ya estaba
  autorizada por el emparejamiento, así que la TV no volvió a pedir permiso.
- **Al terminar:** apagar la depuración (USB e inalámbrica) en Opciones de desarrollador, reiniciar la TV
  y comprobar con `tools\venv\Scripts\python.exe tools\adb-closed.py <ip>` que el 5555 quedó cerrado.

## Estado antes de la limpieza

- Ningún paquete desactivado (`pm list packages -d` vacío).
- Home: `com.google.android.apps.tv.launcherx` (prioridad 2). También responden a HOME
  `setupwraith/.RecoveryActivity` (1), `com.tcl.factory.view/.tv.FactoryTvActivity` (0) y
  `com.android.tv.settings/.system.FallbackHome` (-1000).
- **La red Wi-Fi guardada pertenece a `com.google.android.tungsten.setupwraith`**
  (`cname=` en `dumpsys wifi`). Si se desactiva setupwraith, Android borra esa red y la TV se queda sin
  Wi-Fi. `disable.ps1` lo protege, pero aquí además se deja fuera de la lista.
- Protector de pantalla sin configurar (`screensaver_components` = `null`, se usa dreamx).

## Decisiones

| Paquete | Decisión | Motivo |
|---|---|---|
| `com.tcl.ttvs` | Activo | Es analítica de TCL **y** el actualizador de firmware |
| `com.google.android.katniss` | Desactivado (lote 3) | Asistente de Google; el dueño no lo usa. Sin él, el botón del micrófono no hace nada |
| `com.google.android.tungsten.setupwraith` | Activo por ahora | Es dueño del Wi-Fi; solo hace falta si se cambia el launcher |
| `com.google.android.apps.tv.dreamx` | Activo por ahora | Depende del launcher nuevo (salvapantallas del reloj) |
| `com.google.android.apps.tv.launcherx` | Activo | El launcher queda para después |

Lotes aplicados. Las listas son locales, en `backup/tcl-g13/`, y no se suben al repositorio. Los registros
de recuperación de `tools/recovery.py` quedan en `%USERPROFILE%\.cleanyourtv\recovery-<ip>-<hash>.json`, uno
por lista. Son la única copia del estado anterior, así que no hay que borrarlos.

1. **Rastreo y anuncios de TCL:** `com.tcl.messagebox`, `com.tcl.logkit`, `com.tcl.partnercustomizer`,
   `android.autoinstalls.config.google.gtvpai`, `com.google.android.partnersetup`
2. **Apps que sobran:** `com.tcl.browser`, `com.google.android.play.games`,
   `com.google.android.feedback`, `com.android.tv.feedbackconsent`
3. **Asistente:** `com.google.android.katniss`

Deshacer un lote: `tools\restore.ps1 backup\tcl-g13\lote1.txt` (o `lote2.txt`, `lote3.txt`).

La versión en inglés para el proyecto original está en `examples/tcl-50s5k/debloat.txt` (rama
`examples/tcl-50s5k`).

## Resultados

Los tres lotes se aplicaron el 2026-10-20 con `tools\disable.ps1`. Los 10 paquetes quedaron en
`disabled-user`. Ninguno pidió `TV_INPUT_HARDWARE` y ninguno era dueño de Home, así que las protecciones
no omitieron nada.

Comprobado por ADB después de cada lote:

| Comprobación | Lote 1 | Lote 2 | Lote 3 |
|---|---|---|---|
| Home (`resolve-activity`) | launcherx | launcherx | launcherx |
| Wi-Fi (`Supplicant state`) | `COMPLETED` | `COMPLETED` | `COMPLETED` |
| Internet (`ping 8.8.8.8`) | ok | ok | ok |
| Entradas de TV (`dumpsys tv_input`, `TvInputInfo`) | 10 | 10 | 10 |

Pendiente de comprobar en la TV: botón Home del control, cada entrada HDMI con imagen, botón Input y
YouTube/Netflix.

## Memoria y rendimiento

No se midió antes de la limpieza, así que no hay comparación directa. Medidas después de los lotes 1 y 2
(TV encendida hace 1 h):

| Medida | Valor |
|---|---|
| RAM disponible (`MemAvailable`) | 272 MB de 954 MB |
| Swap (zram) en uso | 321 MB de 700 MB |
| CPU | 364 % libre de 400 % (el `load average` de unos 54 es normal en MediaTek) |
| Arranque en frío de YouTube (`am start -W`, 3 veces) | 1527, 900, 1023 ms |
| Arranque en frío de SmartTube (3 veces) | 1289, 1006, 992 ms |

Mayores consumidores de RAM (PSS): launcherx + coreservices unos 125 MB, gms 64 MB, katniss unos 50 MB
en dos procesos, ttvs 20 MB.

Después del lote 3 (katniss), `MemAvailable` subió a 330 MB. Parte de esa subida viene de haber
cerrado SmartTube durante la medición, así que no se puede atribuir toda al Asistente. Lo siguiente que más
memoria liberaría es cambiar launcherx por el launcher de este repositorio.

## Bloqueo de anuncios

### DNS privado (AdGuard)

Antes los dos valores estaban sin definir (`null`, el modo automático por defecto).

```powershell
tools\venv\Scripts\python.exe tools\tv.py 'settings put global private_dns_specifier dns.adguard-dns.com'
tools\venv\Scripts\python.exe tools\tv.py 'settings put global private_dns_mode hostname'
```

Comprobado: `www.youtube.com` resuelve con normalidad y `doubleclick.net` devuelve `127.0.0.1`.
No quita los anuncios de YouTube. Deshacer:
`tools\venv\Scripts\python.exe tools\tv.py 'settings put global private_dns_mode opportunistic'`.

### SmartTube (YouTube sin anuncios)

Cliente de YouTube de terceros y de código abierto ([yuliskov/SmartTube](https://github.com/yuliskov/SmartTube)),
que no está en Play Store. Se instaló la versión estable `32.56s`, archivo
`SmartTube_stable_32.56_armeabi-v7a.apk`, porque esta TV es solo de 32 bits. Su SHA-256
(`d2e9bc90…43457b6`) coincide con el que publica GitHub.

```powershell
adb connect <ip>:5555
adb install SmartTube_stable_32.56_armeabi-v7a.apk
```

Paquete `org.smarttube.stable`, `primaryCpuAbi=armeabi-v7a`. Arrancó en `BrowseActivity`. Se actualiza
desde la propia app, no desde Play Store. Desinstalar: Ajustes → Apps, o `adb uninstall org.smarttube.stable`.
