# Instalar el Editor de Video en tu VPS — Playbook para Claude Code

Este archivo instala en **tu propio VPS** el mismo servicio que edita videos largos
de YouTube: recibe el video desde n8n, le quita los silencios, lo transcribe con
Whisper, genera título/resumen/capítulos con Claude, sube el video final y avisa a
un webhook con todo listo para publicar.

Además deja el proyecto **documentado con la misma estructura que se usa en
producción** — `CLAUDE.md` maestro + sub-`CLAUDE.md` por carpeta, `troubleshooting.md`
e `ideas.md` — y esa documentación se va escribiendo **durante** la instalación, no
al final.

**Recorrido:**

| Paso | Qué pasa |
|---|---|
| 0 | ¿Tiene accesos al VPS? Si no, se buscan según el proveedor |
| 1 | Revisión del servidor (CPU, RAM, disco, ffmpeg) |
| 2 | Datos que faltan: OpenRouter, almacenamiento, modelo de Whisper |
| **2.5** | **Estructura del proyecto y documentación viva** |
| 3 | Instalación (dependencias, código, modelos) |
| 4 | Servicio systemd + firewall |
| 5 | Verificación, incluida la de sync audio/video |
| 6 | Conexión con n8n |
| **7** | **Cierre: revisar `ideas.md` y dejar la documentación al día** |

---

## ⚠️ INSTRUCCIONES PARA CLAUDE (leer antes de ejecutar nada)

Este documento es un **playbook conversacional**, no un script. Reglas:

1. **NO ejecutes ningún comando del Paso 3 en adelante** hasta haber completado el
   Paso 0, el Paso 1 y el Paso 2.
2. **Una pregunta a la vez.** Haz la pregunta, espera la respuesta del usuario, y
   solo entonces avanza. Si tienes la herramienta `AskUserQuestion`, úsala.
3. **Nunca inventes credenciales.** Si falta la IP, el usuario o la contraseña,
   pídelas. No adivines ni uses ejemplos de este archivo como valores reales.
4. **Nunca escribas la contraseña en un archivo del repositorio** ni la repitas en
   resúmenes. Va únicamente en el comando de conexión y en el `.env` del servidor.
5. Los valores entre `<...>` son placeholders que debes reemplazar con datos reales
   del usuario.
6. Si un comando falla, **detente y muestra el error**. No sigas al paso siguiente
   asumiendo que funcionó.
7. **Documenta mientras instalas, no al final.** El Paso 2.5 crea la estructura de
   documentación del proyecto (`CLAUDE.md`, `troubleshooting.md`, `ideas.md`) y a
   partir de ahí cada paso tiene un bloque **📝 Documenta ahora**. Cúmplelo en el
   momento: un proyecto documentado al final es un proyecto documentado a medias.
8. **Ideas al margen → `ideas.md`, no al código.** Si mientras instalas el usuario
   dice "también estaría bueno que...", "más adelante podríamos...", NO lo
   implementes: anótalo en `ideas.md`, confírmalo en una línea y sigue con lo que
   estabas haciendo. Se revisan todas juntas en el Paso 7.

---

## Paso 0 — ¿Ya tienes acceso al VPS?

**Pregunta al usuario, textualmente:**

> ¿Ya tienes los accesos SSH de tu VPS (IP, usuario y contraseña), o todavía no?

- Respondió **sí** → ve a **Paso 0.A**
- Respondió **no** / "no sé" / "creo que sí pero no los encuentro" → ve a **Paso 0.B**

---

### Paso 0.A — Ya tiene accesos: recolectar y probar

Pide estos 4 datos (los primeros 3 son obligatorios):

| Dato | Ejemplo | De dónde sale |
|---|---|---|
| IP del servidor | `203.0.113.45` | Panel del proveedor |
| Usuario | `root` | Casi siempre `root` en un VPS nuevo |
| Contraseña | `••••••••` | La que definió al crear el VPS |
| Puerto SSH | `22` | Por defecto 22; solo cambia si el usuario lo movió |

#### Sobre el "hostkey" (esto confunde a todo el mundo)

El **hostkey NO lo entrega el proveedor**. No está en Hostinger, ni en Hetzner, ni
en ningún panel. Es la **huella digital de la llave SSH que el propio servidor
genera** cuando se instala, y sirve para que tu cliente confirme que se está
conectando al servidor correcto y no a un impostor.

Se obtiene **desde tu computadora, conectándote una vez**:

```powershell
# 1) Obtener la huella del servidor (Windows / Mac / Linux tienen ssh-keyscan)
ssh-keyscan -t ed25519 <IP> | ssh-keygen -lf -
# Devuelve algo como:  256 SHA256:Hz8rB49panVza5YsrWY+/RLeIwBIYXuQtPE/iLzi9KY ...
```

Ese `SHA256:...` es el hostkey. Se usa así con plink: `-hostkey "SHA256:..."`.

#### Cómo se conecta Claude (importante)

Claude no puede escribir una contraseña en un prompt interactivo — se quedaría
colgado. Hay que pasarla en el comando:

**Windows** → instalar PuTTY (trae `plink` y `pscp`):

```powershell
winget install -e --id PuTTY.PuTTY
```

Primera conexión (acepta y cachea el hostkey):

```powershell
echo y | plink -pw "<PASSWORD>" <USER>@<IP> "echo conectado; uname -a"
```

Conexiones siguientes (con el hostkey pinneado — más seguro):

```powershell
plink -pw "<PASSWORD>" -hostkey "SHA256:<HUELLA>" <USER>@<IP> "<COMANDO>"
```

Subir archivos:

```powershell
pscp -pw "<PASSWORD>" -hostkey "SHA256:<HUELLA>" "<ARCHIVO_LOCAL>" <USER>@<IP>:/root/video-editor/<DESTINO>
```

**Mac / Linux** → usar `sshpass`:

```bash
sshpass -p '<PASSWORD>' ssh -o StrictHostKeyChecking=accept-new <USER>@<IP> "<COMANDO>"
sshpass -p '<PASSWORD>' scp <ARCHIVO_LOCAL> <USER>@<IP>:/root/video-editor/<DESTINO>
```

> **Mejora recomendada al terminar la instalación:** configurar una llave SSH
> (`ssh-keygen` + `ssh-copy-id`, o el panel de SSH Keys del proveedor) y desactivar
> el login por contraseña. Deja de necesitarse `-pw` y el servidor queda mucho más
> protegido contra ataques de fuerza bruta, que en un VPS con puerto 22 abierto
> empiezan a las pocas horas de encenderlo.

**Verificación obligatoria antes de seguir:** ejecuta el comando de "primera
conexión". Si no imprime `conectado`, no avances: reporta el error al usuario.

Cuando la conexión funcione → **Paso 1**.

---

### Paso 0.B — No tiene accesos: identificar el proveedor

**Pregunta al usuario:**

> ¿En qué proveedor tienes (o vas a contratar) el VPS? Hostinger, Hetzner,
> DigitalOcean, Contabo, otro, o todavía no tienes ninguno.

Luego abre **solo la sección de su proveedor**.

---

#### 🟣 Hostinger

Hostinger da la IP y el usuario en el panel, pero **la contraseña no se puede ver:
solo se puede reemplazar**. Y el hostkey no lo da (ver Paso 0.A).

**1. IP y usuario SSH**

1. Entra a [hpanel.hostinger.com](https://hpanel.hostinger.com) e inicia sesión.
2. Menú superior → **VPS** → botón **Manage** (Administrar) al lado de tu servidor.
3. Estás en la página **Overview** (Resumen). Baja hasta la tarjeta de
   **detalles del VPS**: ahí están la **dirección IPv4** y el **usuario SSH**
   (normalmente `root`).
4. El puerto es el **22** salvo que lo hayas cambiado.

**2. Contraseña root**

No se muestra en ninguna parte del panel (por seguridad). Es la que definiste
cuando configuraste el servidor por primera vez. Si no la tienes, la reemplazas:

1. En el VPS → menú lateral → **Settings** (Configuración).
2. Busca la opción **Root password** (Contraseña de root).
3. Escribe una contraseña nueva y fuerte → **Update**.
4. Verifica que se aplicó: menú lateral → **Backup & Monitoring** → **Latest
   actions**. Debe aparecer `ct_set_rootpasswd` con estado **Success**.
5. Tarda unos segundos/minutos en aplicarse. Recién entonces funciona por SSH.

> Atajo útil: el botón **Terminal** arriba a la derecha del Overview abre una
> consola en el navegador ya logueada como root, sin pedir contraseña. Sirve para
> mirar cosas a mano, pero **Claude necesita los accesos SSH reales** para poder
> instalar y subir archivos.

**3. Firewall**

Hostinger tiene firewall propio en el panel, además del del sistema operativo. Al
llegar al Paso 4 hay que abrir el puerto **5051** también ahí:
VPS → menú lateral → **Firewall** → crear regla que acepte TCP en el puerto 5051.

**4. Plan y sistema operativo**

- Sistema operativo: **Ubuntu 22.04 o 24.04** (limpio, sin panel).
- Plan mínimo real: **2 vCPU / 8 GB RAM** (tipo KVM 2). Con 4 GB funciona, pero
  hay que usar un modelo de Whisper más chico (se explica en el Paso 2).
- Si el VPS viene con Ubuntu + panel preinstalado (CyberPanel, etc.), igual sirve;
  solo consume RAM de más.

**Fuentes:**
[Conectarse al VPS por SSH](https://www.hostinger.com/support/5723772-how-to-connect-to-your-vps-via-ssh-at-hostinger/) ·
[Cambiar la contraseña SSH del VPS](https://www.hostinger.com/support/8942826-how-to-change-your-vps-ssh-password-at-hostinger/)

---

#### 🔴 Hetzner (Cloud)

1. [console.hetzner.cloud](https://console.hetzner.cloud) → tu proyecto → el servidor.
2. La **IP pública (IPv4)** está en la ficha del servidor, arriba.
3. Usuario: `root`.
4. Contraseña: Hetzner la envía **por email al crear el servidor** (si no elegiste
   llave SSH). Si se perdió: servidor → **Rescue** → **Reset root password**.
5. Firewall: menú **Firewalls** del proyecto (o dentro del servidor) → abrir TCP 5051.
6. Tipo recomendado: **CPX21 / CX32** (2-3 vCPU, 8 GB) con Ubuntu 22.04/24.04.

---

#### 🔵 DigitalOcean

1. [cloud.digitalocean.com](https://cloud.digitalocean.com) → **Droplets** → tu droplet.
2. La **IP pública** aparece en la lista y en la página del droplet.
3. Usuario: `root`.
4. Contraseña: llega **por email** solo si creaste el droplet con autenticación por
   contraseña. Si usaste llave SSH, no hay contraseña: se entra con la llave. Para
   forzar una nueva: droplet → **Access** → **Reset root password**.
5. Firewall: **Networking → Firewalls** → abrir TCP 5051.
6. Tamaño recomendado: 2 vCPU / 8 GB, Ubuntu 22.04/24.04.

---

#### ⚪ Contabo u otro proveedor

El patrón es siempre el mismo, busca estas tres cosas en el panel:

1. **IP / IPv4 pública** del servidor (a veces "Dirección IP", "Host").
2. **Usuario** de administración (`root` en casi todos los VPS Linux; en algunos
   AWS/GCP es `ubuntu` o `admin` y se entra con llave, no con contraseña).
3. **Contraseña root** — sección tipo "Reset root password" / "Change password" /
   la que llegó por email al crear el servidor.
4. **Firewall** del panel (si tiene) → abrir TCP 5051.

Si el proveedor solo permite llaves SSH, dile al usuario que genere una con
`ssh-keygen -t ed25519` y la cargue en el panel; después Claude se conecta con
`ssh -i <ruta_llave>` en lugar de `-pw`.

---

#### 🆕 Todavía no tiene VPS

Requisitos mínimos para este servicio:

| Recurso | Mínimo | Recomendado |
|---|---|---|
| CPU | 2 vCPU | 4 vCPU |
| RAM | 4 GB (con Whisper `small`/`medium`) | 8 GB o más |
| Disco | 40 GB | 80 GB+ |
| SO | Ubuntu 22.04 | Ubuntu 24.04 limpio |

Con menos de 2 vCPU la transcripción de un video de 30 min se vuelve muy lenta
(más de 30 minutos). Cualquier proveedor sirve; lo único que importa es que sea un
**VPS Linux con acceso root**, no un hosting compartido.

Cuando ya tenga el VPS creado → vuelve al **Paso 0.A**.

---

## Paso 1 — Revisar el servidor

Con la conexión funcionando, ejecuta y muéstrale el resultado al usuario:

```bash
# SO, CPU, RAM, disco y si hay ffmpeg
cat /etc/os-release | head -2
nproc
free -h
df -h /
ffmpeg -version 2>/dev/null | head -1 || echo "ffmpeg NO instalado"
python3 --version
```

Evalúa y avisa:

- **RAM < 6 GB** → hay que crear swap (Paso 3) y usar Whisper `medium` o `small`.
- **Disco libre < 15 GB** → no alcanza: PyTorch + modelos + videos temporales.
- **SO no Ubuntu/Debian** → los comandos `apt` de abajo no aplican; adapta o
  detente y consúltalo con el usuario.

---

## Paso 2 — Datos que faltan antes de instalar

Pregunta **una por una**:

### 2.1 API key de OpenRouter (obligatoria)

Es la que genera el título, el resumen y los capítulos con Claude.

> ¿Tienes una API key de OpenRouter? Se crea en
> https://openrouter.ai/keys y empieza con `sk-or-...`. Necesita saldo cargado
> (con 5 USD alcanza para cientos de videos).

### 2.2 Dónde guardar el video final

El servicio tiene que devolverle a n8n una **URL descargable** del video editado.
Dos opciones:

> ¿Dónde quieres guardar los videos terminados?
> **(A) En el propio VPS** — cero configuración, el servicio los sirve por HTTP.
> Ideal para empezar. Ocupa disco del VPS.
> **(B) Cloudflare R2** — almacenamiento de objetos, 10 GB gratis, no consume
> disco del VPS y las URLs son más rápidas. Requiere cuenta de Cloudflare.

Si elige **A**: solo hace falta la IP (ya la tienes).
Si elige **B**: pídele estos 5 valores de Cloudflare → R2 → *Manage API tokens* y
la configuración del bucket:

- `R2_ACCESS_KEY_ID`
- `R2_SECRET_ACCESS_KEY`
- `R2_BUCKET` (nombre del bucket)
- `R2_ENDPOINT` → `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`
- `R2_PUBLIC_URL` → el dominio público del bucket (`https://pub-xxxx.r2.dev`),
  que se habilita en el bucket → **Settings** → **Public Development URL**

### 2.3 Modelo de Whisper

Decídelo tú según la RAM que viste en el Paso 1, y explícale la elección:

| RAM del VPS | `WHISPER_MODEL` | Calidad |
|---|---|---|
| 8 GB o más | `large-v3-turbo` | La mejor (es el que corre en producción) |
| 4-6 GB | `medium` | Muy buena |
| Menos de 4 GB | `small` | Aceptable, comete más errores en nombres técnicos |

### 2.4 Idioma del video

Este pipeline viene configurado para **español** (`WHISPER_LANGUAGE=es`) y genera
la metadata en español. Si el usuario graba en otro idioma, cambia esa variable
(`en`, `pt`, ...) y ajusta el prompt de `generate_metadata`.

---

## Paso 2.5 — Estructura del proyecto y documentación viva

**Esto se hace ANTES de instalar, no después.** Son 5 archivos y toman 3 minutos;
son la diferencia entre un servidor que se entiende en 6 meses y uno que nadie se
atreve a tocar.

Pregunta al usuario dónde quiere la **carpeta local del proyecto** (ej.
`C:\Users\<usuario>\video-editor` o `~/video-editor`) y créala.

### 2.5.1 Las dos estructuras: local y VPS

Hay **dos** estructuras y no son intercambiables:

- **La carpeta local** es la **copia de trabajo**: aquí se escribe y se edita el
  código, y aquí vive TODA la documentación. Es la fuente desde la que se deploya.
- **El VPS** es la **fuente de verdad de lo que corre**. Solo tiene lo que el
  servicio necesita para ejecutarse. **La documentación no se sube al VPS.**

Regla de oro: **nunca se edita código directo en el VPS.** Se edita local, se sube
con `pscp`/`scp`, se reinicia el servicio. Si alguien edita en el VPS, el local
queda desactualizado y el próximo deploy pisa el cambio sin avisar.

**Carpeta local (copia de trabajo + documentación):**

```
video-editor/
├── CLAUDE.md              ← índice maestro: panorama, accesos, deploy, reglas
├── troubleshooting.md      ← registro GLOBAL de errores (revisar ANTES, registrar DESPUÉS)
├── ideas.md                ← ideas que surgen al margen, para revisar al cerrar objetivos
├── .env                    ← claves reales. NUNCA a git.
├── .gitignore
├── api.py                  ← Flask API: endpoints + orquestación del pipeline
├── execution/              ← módulos del pipeline
│   ├── CLAUDE.md           ← sub-CLAUDE.md: qué hace cada módulo y sus trampas
│   ├── jump_cut_vad.py
│   └── simple_video_edit.py
└── docs/                   ← documentación de referencia
    ├── CLAUDE.md           ← sub-CLAUDE.md: índice de qué hay en docs/
    └── (blueprints, notas, capturas del panel del proveedor...)
```

**VPS (`/root/video-editor/` — solo lo que se ejecuta):**

```
/root/video-editor/
├── api.py                  ← idéntico al local (destino del deploy)
├── .env                    ← variables de entorno del servidor (chmod 600)
├── venv/                   ← entorno virtual de Python (~2.5 GB con torch CPU)
├── logs/api.log            ← log persistente del servicio
├── output/                 ← videos terminados, solo si STORAGE=local
│   └── YYYY-MM-DD/*.mp4
└── execution/              ← coincide EXACTAMENTE con el local
    ├── jump_cut_vad.py
    └── simple_video_edit.py
```

> `api.py` hace `sys.path.insert(0, "execution/")` e importa los módulos con
> **nombre plano** (`from jump_cut_vad import ...`). Por eso `execution/` **no se
> puede reorganizar en subcarpetas** sin tocar los imports: rompería el servicio en
> producción.

**Fuera de la carpeta del proyecto (lo que se descarga solo):**

| Qué | Dónde queda | Tamaño | Cuándo se descarga |
|---|---|---|---|
| PyTorch + torchaudio (CPU) | `/root/video-editor/venv/` | ~2 GB | Paso 3.4 (`pip install`) |
| Modelo Whisper (`large-v3-turbo`) | `~/.cache/huggingface/hub/` | ~1.5 GB | Paso 3.9, primera carga |
| Modelo Silero VAD | dentro del paquete pip `silero-vad` | ~2 MB | ya viene en el wheel, no descarga en runtime |
| ffmpeg / ffprobe | `/usr/bin/` | ~80 MB | Paso 3.1 (`apt-get`) |
| Videos temporales de cada request | `/tmp/tmpXXXX/` | 2-3× el peso del video | por request, se borran siempre en el `finally` |

Total en disco después de instalar: **~5 GB** más el espacio de los videos.

### 2.5.2 Qué contiene cada archivo de documentación

| Archivo | Contenido | Cuándo se actualiza |
|---|---|---|
| `CLAUDE.md` (raíz) | El panorama: qué es el proyecto, accesos al VPS, servicios y puertos, el endpoint, el pipeline de alto nivel, variables de entorno, workflow de deploy, y las **reglas obligatorias**. Da el mapa y apunta a los sub-CLAUDE.md. | Cambios de arquitectura, VPS, deploy o del pipeline general |
| `execution/CLAUDE.md` | El detalle de los módulos: qué hace cada uno, qué función importa `api.py`, y las decisiones contraintuitivas de cada archivo. | Cambios dentro de un módulo |
| `docs/CLAUDE.md` | Índice de qué hay en `docs/` y para qué sirve cada documento. | Cada vez que se agrega un doc nuevo a `docs/` |
| `troubleshooting.md` | Registro **global** de errores ya resueltos: síntoma, causa raíz, solución y cómo verificar. | Cada bug corregido, sin excepción |
| `ideas.md` | Ideas que surgieron al margen del objetivo en curso, con contexto de dónde salieron. | En el momento en que la idea se menciona |

Regla de cascada: el `CLAUDE.md` de la raíz da el panorama y apunta a los
sub-`CLAUDE.md`; los sub tienen el detalle de su carpeta. **Quien lea solo el de la
raíz debe entender el proyecto completo y saber dónde profundizar.**

### 2.5.3 Crear los archivos (rellena los `<...>` con los datos reales del Paso 0 y 2)

**`CLAUDE.md`** en la raíz de la carpeta local:

````markdown
# Video Editor — Guía del proyecto

Servicio Flask en un VPS que recibe videos largos desde n8n, les quita los
silencios, los transcribe, genera la metadata de YouTube con Claude y devuelve el
resultado a un webhook.

**Esta carpeta es la copia de trabajo: la fuente desde la que se deploya al VPS.
El VPS es la fuente de verdad de lo que corre.**

---

## Estructura

(pega aquí el árbol de la carpeta local, con una línea por archivo explicando qué es)

Cada subcarpeta relevante tiene su propio `CLAUDE.md` con el detalle. Este archivo
es el índice maestro.

---

## ⚠️ Reglas OBLIGATORIAS en cualquier cambio

1. **Sync A/V en los cortes.** Todo cambio que toque el corte o la concatenación
   (`jump_cut_vad.py`, parámetros del VAD, `concatenate_segments`) DEBE verificar
   con `ffprobe` que la duración del stream de video y la del audio sigan
   coincidiendo (<0.1s), más una escucha del final del video. Ya hubo drift antes.
2. **Actualizar los CLAUDE.md en cascada** (el maestro y/o el de la subcarpeta).
3. **`troubleshooting.md` se revisa ANTES de tocar una sección** y se registra
   DESPUÉS de corregir cualquier bug.
4. **Nada se sube al VPS sin revisar el diff primero.** Bajar el archivo del VPS,
   comparar contra el local, subir solo lo revisado.
5. **Nunca editar código directo en el VPS.** Se edita local y se deploya.

---

## VPS

- **Proveedor:** <Hostinger / Hetzner / ...>
- **IP:** <IP>
- **Usuario:** <root>
- **Puerto SSH:** 22
- **Contraseña:** NO va en este archivo (ver "Accesos" abajo)
- **Hostkey:** SHA256:<huella>

### Accesos

La contraseña no se versiona. Vive en `.ssh-access` (ignorado por git) o, mejor,
se reemplaza por una llave SSH y se desactiva el login por contraseña.

```powershell
# Ejecutar un comando remoto (Windows / plink)
plink -pw "<PASSWORD>" -hostkey "SHA256:<huella>" <USER>@<IP> "COMANDO"

# Subir un archivo
pscp -pw "<PASSWORD>" -hostkey "SHA256:<huella>" "<local>" <USER>@<IP>:/root/video-editor/<destino>
```

### Servicio

| Servicio | Puerto | systemd | Directorio |
|---|---|---|---|
| video-editor | 5051 | `video-editor` | `/root/video-editor/` |

```bash
systemctl restart video-editor
systemctl is-active video-editor
journalctl -u video-editor -n 50 --no-pager
```

---

## Endpoint

**POST** `http://<IP>:5051/process` (multipart/form-data)

| Campo | Descripción |
|---|---|
| `file` | Video mp4/mov/mkv |
| `title` | Título/tema del video |
| `webhook_url` | URL del webhook de n8n |
| `api_secret` | El valor de `API_SECRET` del `.env` |

Responde 202 inmediato. El resultado llega al webhook al terminar.

---

## Pipeline

```
1. Silero VAD    → detecta voz, marca los silencios a eliminar
2. Whisper       → transcribe el video ORIGINAL (word-level)
3. FFmpeg        → corta silencios + audio (highpass 80Hz + loudnorm -16 LUFS)
4. Claude        → título, resumen, capítulos y recursos (OpenRouter)
5. Almacenamiento→ <VPS local / Cloudflare R2>
6. Webhook n8n   → envía el resultado completo
```

---

## Variables de entorno (`.env`)

| Variable | Uso |
|---|---|
| `OPENROUTER_API_KEY` | Claude vía OpenRouter |
| `VIDEO_PORT` | 5051 |
| `API_SECRET` | Protege el endpoint |
| `WHISPER_MODEL` / `WHISPER_LANGUAGE` | Transcripción |
| `STORAGE` | `local` o `r2` |
| `PUBLIC_BASE_URL` | URL pública si `STORAGE=local` |
| `R2_*` | Credenciales de Cloudflare R2 si `STORAGE=r2` |

---

## Restricciones de memoria

- **`Semaphore(1)` en `api.py`:** un solo video pesado a la vez. Nunca dos Whisper
  ni dos PyTorch en RAM al mismo tiempo.
- Whisper `<modelo elegido>` en CPU int8: ~1.5 GB en inferencia.
- Silero VAD: ~500 MB de PyTorch.
- Pico estimado: ~3-4 GB.

---

## Deploy

```
1. Editar el archivo en la carpeta local
2. (Opcional pero recomendado) bajar el del VPS y hacer diff
3. Subir con pscp/scp
4. systemctl restart video-editor && verificar logs
```
````

**`troubleshooting.md`**:

```markdown
# troubleshooting.md — Registro global de errores

Registro de errores YA RESUELTOS de este proyecto.

- **Antes** de tocar una sección: busca aquí si el error ya pasó.
- **Después** de corregir cualquier bug: regístralo aquí. Sin excepción.

Formato de cada entrada:

---

## #1 — <YYYY-MM-DD> Título del síntoma

- **Síntoma:** qué se veía o qué devolvía el sistema (pegar el error textual)
- **Causa raíz:** por qué pasaba de verdad, no el primer diagnóstico
- **Solución:** qué se cambió y en qué archivo (`archivo.py:línea`)
- **Cómo verificarlo:** el comando o prueba exacta que confirma que está resuelto
- **Prevención:** la regla que evita que vuelva a pasar
```

**`ideas.md`**:

```markdown
# ideas.md — Ideas pendientes

Ideas que surgieron **mientras se trabajaba en otra cosa**. No se implementan en el
momento: se anotan aquí con el contexto de dónde salieron y se revisan cuando se
cierra el objetivo en curso.

Por qué existe este archivo: la mitad de las buenas ideas aparecen a mitad de otra
tarea. Implementarlas ahí mismo descarrila lo que estabas haciendo; no anotarlas
las pierde. Esto resuelve las dos cosas.

## Protocolo

- **Cuando surge una idea** (el usuario dice "también estaría bueno", "más adelante
  podríamos", "se me ocurre que..."): anotarla aquí, confirmar en una línea
  ("Anotado en ideas.md, sigo con X") y **seguir con la tarea en curso**.
- **Al cerrar un objetivo:** leer las pendientes, listarlas al usuario y preguntar
  si quiere tomar alguna ahora.
- Una idea implementada se mueve a **Hechas**; una descartada se mueve a
  **Descartadas** con el motivo. Nunca se borra: el motivo del descarte vale tanto
  como la idea.

---

## Pendientes

### <YYYY-MM-DD> — Título corto de la idea
- **Surgió mientras:** <qué se estaba haciendo cuando apareció>
- **Qué es:** <1-3 líneas>
- **Por qué vale:** <el problema que resuelve>
- **Esfuerzo:** bajo / medio / alto
- **Depende de:** <nada / otra idea / algo externo>

---

## Descartadas

### <fecha> — <idea> — Motivo: <por qué no>

---

## Hechas

### <fecha> — <idea> — Resuelta en: <archivo / commit>
```

**`execution/CLAUDE.md`**:

```markdown
# execution/ — Módulos del pipeline

Módulos que importa `../api.py` (que hace `sys.path.insert(0, "execution/")`, por
eso los imports son de **nombre plano**). Todos se deployan a
`/root/video-editor/execution/`.

| Módulo | Qué hace | Funciones que usa api.py |
|---|---|---|
| `jump_cut_vad.py` | Corte de silencios con Silero VAD v6 + concatenación y normalización de audio con FFmpeg. | `extract_audio`, `get_speech_timestamps_silero`, `merge_close_segments`, `add_padding`, `concatenate_segments`, `get_duration` |
| `simple_video_edit.py` | Transcripción con faster-whisper + metadata generada por Claude vía OpenRouter. | `transcribe_video`, `generate_metadata` |

## Notas que no son obvias leyendo el código

- **No reorganizar esta carpeta en subcarpetas:** los imports son planos.
- **`concatenate_segments` corta con un ffmpeg por segmento a propósito.** Un solo
  `filter_complex` sobre un video largo revienta la RAM (OOM con 123 segmentos).
- **El concat final re-encodea el video, no usa `-c:v copy`.** Con `copy` se pierde
  ~medio frame por empalme y el audio termina adelantado.
- **`loudnorm` altera la duración**, así que corre una sola vez sobre el audio ya
  concatenado, nunca por segmento.
- **Whisper transcribe el video ORIGINAL**, con silencios: así los timestamps de
  los capítulos se pueden reajustar restando lo eliminado.
- El vocabulario técnico de Whisper está en `vocab_prompt` (`simple_video_edit.py`).
  Si un nombre sale mal transcrito de forma repetida, se agrega ahí.
```

**`docs/CLAUDE.md`**:

```markdown
# docs/ — Documentación de referencia

No se deploya. Es contexto para entender y extender el servicio.

| Archivo | Contenido |
|---|---|
| (agregar aquí cada doc nuevo con una línea de qué contiene) | |

> El registro de errores (`troubleshooting.md`) y las ideas (`ideas.md`) son
> **globales** y viven en la **raíz**, no aquí.
```

**`.gitignore`**:

```
.env
.ssh-access
venv/
logs/
output/
__pycache__/
*.pyc
*.mp4
```

### 2.5.4 Confirmación antes de seguir

Muéstrale al usuario los 6 archivos creados y explícale en 2 líneas para qué sirve
cada uno. Especialmente `ideas.md`: es el que nadie espera y el que más se usa.

---

## Paso 3 — Instalación en el servidor

### 3.0 Cómo se crea cada archivo (local primero, siempre)

Todos los archivos de código (`api.py`, `execution/*.py`, `.env`) se crean **en la
carpeta local** y se suben al VPS. No se escriben directo en el servidor.

Las carpetas del VPS se crean en el paso 3.3. Después, cada archivo se sube así:

```powershell
pscp -pw "<PASSWORD>" "<carpeta_local>\api.py" <USER>@<IP>:/root/video-editor/api.py
pscp -pw "<PASSWORD>" "<carpeta_local>\execution\jump_cut_vad.py" <USER>@<IP>:/root/video-editor/execution/jump_cut_vad.py
```

En Mac/Linux, lo mismo con `sshpass -p '<PASSWORD>' scp <local> <USER>@<IP>:<destino>`.

### 3.1 Dependencias del sistema

```bash
apt-get update
apt-get install -y ffmpeg python3-venv python3-pip
```

### 3.2 Swap (solo si la RAM es menor a 6 GB)

```bash
fallocate -l 4G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
free -h
```

### 3.3 Carpetas y entorno virtual

```bash
mkdir -p /root/video-editor/execution /root/video-editor/logs /root/video-editor/output
cd /root/video-editor
python3 -m venv venv
./venv/bin/pip install --upgrade pip
```

### 3.4 Dependencias Python

```bash
cd /root/video-editor
# torch versión CPU: evita descargar ~2 GB de wheels con CUDA que el VPS no usa
./venv/bin/pip install torch torchaudio --index-url https://download.pytorch.org/whl/cpu
./venv/bin/pip install flask faster-whisper silero-vad==6.2.1 boto3 requests python-dotenv
```

### 3.5 Archivo `.env`

Créalo en la **carpeta local** con los datos reales del Paso 2, y súbelo a
`/root/video-editor/.env`. Este archivo nunca va a git (ya está en `.gitignore`):

```ini
# --- Claude vía OpenRouter (obligatorio) ---
OPENROUTER_API_KEY=<sk-or-...>

# --- Puerto del servicio ---
VIDEO_PORT=5051

# --- Protección del endpoint (recomendado: inventa una cadena larga al azar) ---
API_SECRET=<cadena_aleatoria_larga>

# --- Transcripción ---
WHISPER_MODEL=large-v3-turbo
WHISPER_LANGUAGE=es

# --- Almacenamiento: "local" o "r2" ---
STORAGE=local

# Si STORAGE=local, la URL pública del propio VPS:
PUBLIC_BASE_URL=http://<IP_DEL_VPS>:5051

# Si STORAGE=r2, completa estos y deja PUBLIC_BASE_URL vacío:
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=
R2_ENDPOINT=
R2_PUBLIC_URL=
```

```bash
chmod 600 /root/video-editor/.env
```

### 3.6 `execution/jump_cut_vad.py` — corte de silencios

Créalo local con este contenido exacto y súbelo a
`/root/video-editor/execution/jump_cut_vad.py`:

```python
#!/usr/bin/env python3
"""Corte de silencios con Silero VAD + concatenación con FFmpeg."""

import os
import subprocess
import tempfile

_silero_model = None

# Encoder de video: CRF 18 = alta calidad, preset fast = balance velocidad/tamaño
ENCODER_ARGS = ["-c:v", "libx264", "-preset", "fast", "-crf", "18"]


def get_duration(input_path: str) -> float:
    """Duración del archivo en segundos (ffprobe)."""
    cmd = [
        "ffprobe", "-v", "error", "-show_entries", "format=duration",
        "-of", "default=noprint_wrappers=1:nokey=1", input_path,
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return float(result.stdout.strip())


def extract_audio(input_path: str, output_path: str, sample_rate: int = 16000):
    """Extrae el audio como WAV mono 16 kHz (lo que espera Silero)."""
    cmd = [
        "ffmpeg", "-y", "-i", input_path,
        "-vn", "-ar", str(sample_rate), "-ac", "1",
        "-acodec", "pcm_s16le",
        "-loglevel", "error", output_path,
    ]
    subprocess.run(cmd, capture_output=True, check=True)


def get_speech_timestamps_silero(
    audio_path: str,
    min_speech_duration: float = 0.25,
    min_silence_duration: float = 0.5,
):
    """
    Detecta los segmentos donde hay voz con Silero VAD v6.
    Devuelve lista de (inicio, fin) en segundos. El modelo se carga una sola vez.
    """
    global _silero_model
    from silero_vad import load_silero_vad, read_audio
    from silero_vad import get_speech_timestamps as _get_ts

    if _silero_model is None:
        _silero_model = load_silero_vad()

    SAMPLE_RATE = 16000
    wav = read_audio(audio_path, sampling_rate=SAMPLE_RATE)

    # threshold 0.5 = sensibilidad balanceada; speech_pad_ms protege el borde
    # de las palabras para no cortar la primera/última sílaba.
    speech_timestamps = _get_ts(
        wav,
        _silero_model,
        sampling_rate=SAMPLE_RATE,
        threshold=0.5,
        min_speech_duration_ms=int(min_speech_duration * 1000),
        min_silence_duration_ms=int(min_silence_duration * 1000),
        speech_pad_ms=100,
    )

    return [(ts["start"] / SAMPLE_RATE, ts["end"] / SAMPLE_RATE) for ts in speech_timestamps]


def merge_close_segments(segments, max_gap: float):
    """Une segmentos separados por menos de max_gap segundos."""
    if not segments:
        return []

    merged = [segments[0]]
    for start, end in segments[1:]:
        prev_start, prev_end = merged[-1]
        if start - prev_end <= max_gap:
            merged[-1] = (prev_start, end)
        else:
            merged.append((start, end))
    return merged


def add_padding(segments, padding_s: float, duration: float):
    """Agrega margen a cada segmento y fusiona los que queden solapados."""
    if not segments:
        return []

    padded = [(max(0, s - padding_s), min(duration, e + padding_s)) for s, e in segments]

    merged = [padded[0]]
    for start, end in padded[1:]:
        prev_start, prev_end = merged[-1]
        if start <= prev_end:
            merged[-1] = (prev_start, max(prev_end, end))
        else:
            merged.append((start, end))
    return merged


def concatenate_segments(input_path, segments, output_path, enhance_audio: bool = False):
    """
    Corta cada segmento con su propio ffmpeg y después los une con el demuxer
    concat, aplicando el filtro de audio UNA vez sobre el resultado.

    Por qué así (no con un filter_complex gigante):
    - Un ffmpeg por segmento mantiene los buffers chicos. Un solo filter_complex
      sobre videos largos revienta la RAM (probado: OOM con 123 segmentos / 22 min).
    - Intermedios .nut con audio PCM evitan el priming delay del encoder AAC, que
      es la causa clásica de que el audio se desincronice del video al concatenar.
    - loudnorm altera la duración, así que corre una sola vez sobre el audio ya
      concatenado, nunca por segmento.
    """
    if not segments:
        raise ValueError("concatenate_segments: lista de segmentos vacía")

    print(f"Concatenando {len(segments)} segmentos...")

    with tempfile.TemporaryDirectory() as tmpdir:
        segment_files = []

        for i, (start, end) in enumerate(segments):
            seg_path = os.path.join(tmpdir, f"seg_{i:04d}.nut")
            duration = end - start

            cmd = [
                "ffmpeg", "-y",
                "-ss", f"{start:.6f}",
                "-i", input_path,
                "-t", f"{duration:.6f}",
                "-avoid_negative_ts", "make_zero",
                "-map", "0:v:0", "-map", "0:a:0",
                *ENCODER_ARGS,
                # Intra-only (todo keyframes, sin B-frames): segmento "concat-safe"
                "-g", "1", "-keyint_min", "1", "-sc_threshold", "0", "-bf", "0",
                "-pix_fmt", "yuv420p",
                "-c:a", "pcm_s16le",
                "-loglevel", "error",
                seg_path,
            ]
            result = subprocess.run(cmd, capture_output=True, text=True)
            if result.returncode != 0:
                raise RuntimeError(f"segmento {i} falló:\n{result.stderr[-1500:]}")
            segment_files.append((seg_path, duration))

        # Lista ffconcat con la duración EXACTA de cada archivo
        concat_path = os.path.join(tmpdir, "concat.ffconcat")
        with open(concat_path, "w") as f:
            f.write("ffconcat version 1.0\n")
            for p, dur in segment_files:
                f.write(f"file '{p}'\n")
                f.write(f"duration {dur:.6f}\n")

        final_cmd = [
            "ffmpeg", "-y",
            "-f", "concat", "-safe", "0",
            "-i", concat_path,
            "-map", "0:v:0", "-map", "0:a:0",
            # Re-encodear el video (NO -c:v copy): con copy se pierde ~medio frame
            # por empalme y el video se atrasa respecto al audio.
            *ENCODER_ARGS,
        ]
        if enhance_audio:
            # Cadena mínima para voz limpia: highpass quita el rumble por debajo
            # de 80 Hz (inaudible en voz) y loudnorm deja -16 LUFS, el estándar
            # de YouTube. Nada de compresores ni EQ: coloreaban la voz sin aportar.
            final_cmd += ["-af", "highpass=f=80,loudnorm=I=-16:TP=-1.5:LRA=11"]
        final_cmd += [
            "-c:a", "aac", "-b:a", "192k",
            "-movflags", "+faststart",
            "-loglevel", "error",
            output_path,
        ]

        result = subprocess.run(final_cmd, capture_output=True, text=True)
        if result.returncode != 0:
            raise RuntimeError(f"concat final falló:\n{result.stderr[-1500:]}")

    print(f"Listo: {output_path}")
```

### 3.7 `execution/simple_video_edit.py` — transcripción + metadata

Local → `/root/video-editor/execution/simple_video_edit.py`:

```python
#!/usr/bin/env python3
"""Transcripción con Whisper + metadata generada por Claude (OpenRouter)."""

import json
import os
import re
import time
from pathlib import Path

import requests as http_requests
from dotenv import load_dotenv

load_dotenv(Path(__file__).parent.parent / ".env")

OPENROUTER_API_KEY = os.getenv("OPENROUTER_API_KEY")
WHISPER_MODEL = os.getenv("WHISPER_MODEL", "large-v3-turbo")
WHISPER_LANGUAGE = os.getenv("WHISPER_LANGUAGE", "es")


def transcribe_video(video_path: str, model_size: str = None) -> list[dict]:
    """
    Transcribe el video con faster-whisper.
    Devuelve una lista de dicts {word, start, end}.
    """
    from faster_whisper import WhisperModel

    model_size = model_size or WHISPER_MODEL

    # Vocabulario técnico: sin esto Whisper escribe mal estos nombres.
    # Ajusta la lista a los términos que repites en tus videos.
    vocab_prompt = (
        "Vocabulario técnico del video: Claude Code, n8n, OpenAI, ChatGPT, "
        "Anthropic, Gemini, Cursor, prompt, tokens, LLM, API, workflow, "
        "Supabase, YouTube, Instagram."
    )

    print(f"Transcribiendo con Whisper ({model_size})...")
    start_time = time.time()

    # int8 en CPU: ~1.5 GB de RAM en inferencia con large-v3-turbo
    model = WhisperModel(model_size, device="cpu", compute_type="int8")
    segments, _ = model.transcribe(
        video_path,
        word_timestamps=True,
        language=WHISPER_LANGUAGE,
        vad_filter=True,
        initial_prompt=vocab_prompt,
    )

    words = []
    for segment in segments:
        if segment.words:
            for w in segment.words:
                words.append({"word": w.word.strip(), "start": w.start, "end": w.end})

    print(f"Transcripción lista: {len(words)} palabras en {time.time() - start_time:.1f}s")
    return words


def generate_metadata(words: list[dict], cuts: list, duration: float, title: str) -> dict:
    """
    Genera título, resumen, capítulos y recursos con Claude vía OpenRouter.
    Los capítulos vienen con timestamps del video ORIGINAL y se ajustan restando
    los silencios eliminados antes de cada marca.
    """
    _empty = {
        "title": title,
        "summary": "Contenido del video.",
        "chapters": "00:00:00 Introducción",
        "resources": "",
    }

    if not words:
        return _empty

    # Agrupa las palabras en bloques de ~30s para darle timestamps al modelo
    chunks = []
    current_chunk = []
    chunk_start = 0.0

    for w in words:
        current_chunk.append(w)
        if w["end"] - chunk_start >= 30:
            chunks.append({
                "start": chunk_start,
                "end": w["end"],
                "text": " ".join(x["word"] for x in current_chunk),
            })
            chunk_start = w["end"]
            current_chunk = []

    if current_chunk:
        chunks.append({
            "start": chunk_start,
            "end": current_chunk[-1]["end"],
            "text": " ".join(x["word"] for x in current_chunk),
        })

    transcript_with_times = "\n".join(
        f"[{c['start']:.0f}s - {c['end']:.0f}s]: {c['text']}" for c in chunks
    )

    prompt = f"""Analiza esta transcripción de video y genera la metadata para publicarlo en YouTube.

CORRECCIONES DE TRANSCRIPCIÓN: la transcripción es automática (Whisper) y suele
equivocarse con nombres técnicos. Al interpretarla, aplica estas correcciones:
- "Cloud Code", "Clod Code", "Claud Code" → "Claude Code"
- "n 8 n", "n8 n", "ene 8 ene" → "n8n"
- "Chat G P T", "chat gpt" → "ChatGPT"
- "open a i", "open ay" → "OpenAI"
- "you tube" → "YouTube"
Aplica tu criterio con otros nombres técnicos que suenen mal transcritos.

TRANSCRIPCIÓN (con timestamps en segundos):
{transcript_with_times}

DURACIÓN DEL VIDEO: {duration:.0f} segundos ({duration/60:.1f} minutos)
TEMA/CONTEXTO: {title}

Responde SOLO con este JSON, sin ningún texto adicional:
{{
    "title": "<Título optimizado para SEO en YouTube. Máximo 70 caracteres. Que genere curiosidad o resuelva un problema concreto. Usa números si aplica. TODO EN ESPAÑOL.>",
    "summary": "<Resumen de 2-4 oraciones sobre lo que cubre el video, en tercera persona ('En este video se explica...'). Específico, no genérico. TODO EN ESPAÑOL.>",
    "chapters": [
        {{"time": "00:00:00", "title": "Introducción"}},
        {{"time": "00:02:30", "title": "Título del tema"}}
    ],
    "resources": "<Herramientas, links o materiales que el presentador prometió dejar en la descripción, uno por línea. Cadena vacía si no menciona ninguno.>"
}}

Pautas para los capítulos:
- Entre 5 y 15 capítulos, marcando las transiciones de tema
- El primer capítulo DEBE ser 00:00:00
- Separados 1-2+ minutos entre sí (salvo intro/outro)
- Títulos concisos (2-6 palabras) EN ESPAÑOL"""

    print("Generando metadata con Claude vía OpenRouter...")
    resp = http_requests.post(
        "https://openrouter.ai/api/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {OPENROUTER_API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "model": "anthropic/claude-sonnet-4-6",
            "max_tokens": 4000,
            "messages": [{"role": "user", "content": prompt}],
        },
        timeout=120,
    )
    resp.raise_for_status()

    response_text = resp.json()["choices"][0]["message"]["content"].strip()

    json_match = re.search(r"\{[\s\S]*\}", response_text)
    if not json_match:
        return _empty

    try:
        data = json.loads(json_match.group())
        title = data.get("title", title)
        summary = data.get("summary", "Contenido del video.")
        chapters_list = data.get("chapters", [{"time": "00:00:00", "title": "Introducción"}])
        resources = data.get("resources", "")
    except json.JSONDecodeError:
        return _empty

    # Ajuste de capítulos: Claude los da en tiempos del video original, pero el
    # video entregado ya no tiene los silencios. Hay que restar lo eliminado
    # antes de cada marca o los capítulos quedan corridos.
    adjusted_chapters = []
    sorted_cuts = sorted(cuts, key=lambda x: x[0]) if cuts else []

    for chapter in chapters_list:
        time_str = chapter.get("time", "00:00:00")
        chapter_title = chapter.get("title", "Capítulo")

        match = re.match(r"^(\d{1,2}):(\d{2}):(\d{2})$", time_str)
        if match:
            hours, minutes, seconds = match.groups()
            original_time = int(hours) * 3600 + int(minutes) * 60 + int(seconds)
        else:
            match = re.match(r"^(\d{1,2}):(\d{2})$", time_str)
            if not match:
                adjusted_chapters.append(f"{time_str} {chapter_title}")
                continue
            minutes, seconds = match.groups()
            original_time = int(minutes) * 60 + int(seconds)

        time_removed = 0
        for cut_start, cut_end in sorted_cuts:
            if cut_end <= original_time:
                time_removed += cut_end - cut_start
            elif cut_start < original_time:
                time_removed += original_time - cut_start

        adjusted = max(0, original_time - time_removed)
        h = int(adjusted // 3600)
        m = int((adjusted % 3600) // 60)
        s = int(adjusted % 60)
        adjusted_chapters.append(f"{h:02d}:{m:02d}:{s:02d} {chapter_title}")

    return {
        "title": title,
        "summary": summary,
        "chapters": "\n".join(adjusted_chapters) if adjusted_chapters else "00:00:00 Introducción",
        "resources": resources,
    }
```

### 3.8 `api.py` — el servicio Flask

Local → `/root/video-editor/api.py`:

```python
#!/usr/bin/env python3
"""
Video Editor API — endpoint /process

POST /process (multipart/form-data)
  file        -- archivo de video (mp4/mov/mkv)
  title       -- título/tema del video
  webhook_url -- URL a la que avisar cuando termine
  api_secret  -- si API_SECRET está definido en el .env

Responde 202 al instante y sigue procesando en background. El resultado llega
al webhook (5-30 min según el largo del video).
"""

import logging
import os
import shutil
import sys
import tempfile
import threading
from datetime import date
from pathlib import Path

import requests
from dotenv import load_dotenv
from flask import Flask, jsonify, request, send_from_directory

BASE_DIR = Path(__file__).parent
load_dotenv(BASE_DIR / ".env")
sys.path.insert(0, str(BASE_DIR / "execution"))

from jump_cut_vad import (
    add_padding,
    concatenate_segments,
    extract_audio,
    get_duration,
    get_speech_timestamps_silero,
    merge_close_segments,
)
from simple_video_edit import generate_metadata, transcribe_video

LOG_DIR = BASE_DIR / "logs"
LOG_DIR.mkdir(exist_ok=True)
OUTPUT_DIR = BASE_DIR / "output"
OUTPUT_DIR.mkdir(exist_ok=True)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s  %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler(LOG_DIR / "api.log", encoding="utf-8"),
    ],
)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# Cola global: un solo video pesado a la vez. Sin esto, dos requests simultáneos
# cargan dos Whisper + dos PyTorch en RAM y el kernel mata el proceso (OOM).
_processing_lock = threading.Semaphore(1)

API_SECRET = os.getenv("API_SECRET")
STORAGE = os.getenv("STORAGE", "local").strip().lower()
PUBLIC_BASE_URL = os.getenv("PUBLIC_BASE_URL", "").rstrip("/")

R2_ACCESS_KEY_ID = os.getenv("R2_ACCESS_KEY_ID")
R2_SECRET_ACCESS_KEY = os.getenv("R2_SECRET_ACCESS_KEY")
R2_BUCKET = os.getenv("R2_BUCKET")
R2_ENDPOINT = os.getenv("R2_ENDPOINT")
R2_PUBLIC_URL = (os.getenv("R2_PUBLIC_URL") or "").rstrip("/")


def _check_auth(data: dict) -> bool:
    if not API_SECRET:
        return True
    key = data.get("api_secret", "") or request.headers.get("X-API-Key", "")
    return key == API_SECRET


def _upload(file_path: str, filename: str) -> str:
    """Guarda el video final y devuelve su URL pública."""
    today = date.today().strftime("%Y-%m-%d")
    key = f"{today}/{filename}"

    if STORAGE == "r2":
        import boto3

        s3 = boto3.client(
            "s3",
            endpoint_url=R2_ENDPOINT,
            aws_access_key_id=R2_ACCESS_KEY_ID,
            aws_secret_access_key=R2_SECRET_ACCESS_KEY,
            region_name="auto",
        )
        s3.upload_file(file_path, R2_BUCKET, key, ExtraArgs={"ContentType": "video/mp4"})
        logger.info(f"[UPLOAD] R2: {key}")
        return f"{R2_PUBLIC_URL}/{key}"

    if not PUBLIC_BASE_URL:
        raise RuntimeError("STORAGE=local requiere PUBLIC_BASE_URL en el .env")

    dest = OUTPUT_DIR / today
    dest.mkdir(parents=True, exist_ok=True)
    shutil.copy2(file_path, dest / filename)
    logger.info(f"[UPLOAD] local: {key}")
    return f"{PUBLIC_BASE_URL}/files/{key}"


def _segments_to_cuts(speech_segments: list, duration: float) -> list:
    """Convierte los segmentos con voz en la lista de tramos eliminados."""
    cuts = []
    prev_end = 0.0
    for start, end in speech_segments:
        if start > prev_end + 0.01:
            cuts.append((prev_end, start))
        prev_end = end
    if prev_end < duration - 0.01:
        cuts.append((prev_end, duration))
    return cuts


def _process_video_task(video_path: str, title: str, webhook_url: str, tmp_dir: Path):
    output_path = None
    audio_path = None

    logger.info(f"[PROCESS] '{title}' esperando cola...")
    _processing_lock.acquire()
    logger.info(f"[PROCESS] '{title}' arrancando")

    try:
        duration = get_duration(video_path)
        logger.info(f"[PROCESS] Duración: {duration:.1f}s ({duration/60:.1f} min)")

        # 1. Audio mono 16 kHz para el VAD
        audio_path = str(tmp_dir / "audio.wav")
        extract_audio(video_path, audio_path)

        # 2. Silero VAD: dónde hay voz
        logger.info("[PROCESS] Silero VAD...")
        speech_segments = get_speech_timestamps_silero(
            audio_path,
            min_speech_duration=0.25,
            min_silence_duration=0.6,  # pausas < 0.6s se conservan (ritmo natural)
        )
        speech_segments = merge_close_segments(speech_segments, max_gap=0.3)
        # Silero ya aplica speech_pad_ms=100; este padding es ADICIONAL.
        # 0.05s extra = ~150ms por lado: corte ajustado sin cortar palabras.
        speech_segments = add_padding(speech_segments, padding_s=0.05, duration=duration)
        logger.info(f"[PROCESS] VAD: {len(speech_segments)} segmentos con voz")

        # 3. Whisper sobre el video ORIGINAL (con silencios), para que los
        #    timestamps de los capítulos se puedan ajustar correctamente.
        logger.info("[PROCESS] Transcribiendo...")
        words = transcribe_video(video_path)
        logger.info(f"[PROCESS] Transcripción: {len(words)} palabras")

        # 4. Corte + audio normalizado
        kept = sum(e - s for s, e in speech_segments)
        logger.info(f"[PROCESS] Conserva {kept:.1f}s, elimina {duration - kept:.1f}s")
        output_path = str(tmp_dir / "output.mp4")
        concatenate_segments(video_path, speech_segments, output_path, enhance_audio=True)

        # 5. Metadata con Claude
        cuts = _segments_to_cuts(speech_segments, duration)
        metadata = generate_metadata(words, cuts, kept, title)
        transcript_text = " ".join(w["word"] for w in words) if words else ""

        # 6. Subir y avisar al webhook
        safe_title = (
            "".join(c for c in title if c.isalnum() or c in " -_").strip().replace(" ", "_")[:60]
        )
        video_url = _upload(output_path, f"{safe_title}.mp4")

        payload = {
            "status": "ok",
            "title": metadata["title"],
            "video_url": video_url,
            "summary": metadata["summary"],
            "chapters": metadata["chapters"],
            "resources": metadata.get("resources", ""),
            "transcript": transcript_text,
        }
        resp = requests.post(webhook_url, json=payload, timeout=30)
        logger.info(f"[WEBHOOK] -> {webhook_url} ({resp.status_code})")

    except Exception as e:
        logger.error(f"[PROCESS] ERROR: {e}", exc_info=True)
        try:
            requests.post(webhook_url, json={"status": "error", "error": str(e)}, timeout=30)
        except Exception:
            pass

    finally:
        _processing_lock.release()
        for path in [video_path, output_path, audio_path]:
            if path and Path(path).exists():
                try:
                    Path(path).unlink()
                except Exception:
                    pass
        shutil.rmtree(tmp_dir, ignore_errors=True)


@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok", "service": "video-editor", "storage": STORAGE})


@app.route("/files/<path:key>", methods=["GET"])
def files(key):
    """Sirve los videos terminados cuando STORAGE=local."""
    return send_from_directory(OUTPUT_DIR, key)


@app.route("/process", methods=["POST"])
def process_route():
    form_data = request.form.to_dict()

    if not _check_auth(form_data):
        return jsonify({"error": "Unauthorized"}), 401

    title = form_data.get("title", "Video")
    webhook_url = form_data.get("webhook_url")

    if not webhook_url:
        return jsonify({"error": "Falta 'webhook_url'"}), 400
    if "file" not in request.files:
        return jsonify({"error": "Falta el archivo de video (campo 'file')"}), 400

    file = request.files["file"]
    if not file.filename:
        return jsonify({"error": "Nombre de archivo vacío"}), 400

    tmp_dir = Path(tempfile.mkdtemp())
    video_path = str(tmp_dir / "input.mp4")
    file.save(video_path)

    size_mb = Path(video_path).stat().st_size / 1024 / 1024
    logger.info(f"[PROCESS] title='{title}' | {size_mb:.1f}MB | webhook={webhook_url}")

    threading.Thread(
        target=_process_video_task,
        args=(video_path, title, webhook_url, tmp_dir),
        daemon=True,
    ).start()

    return jsonify({
        "status": "processing",
        "message": "Video recibido. El resultado se enviará al webhook cuando esté listo.",
    }), 202


if __name__ == "__main__":
    port = int(os.getenv("VIDEO_PORT", 5051))
    logger.info(f"[API] Video Editor arrancando en el puerto {port}")
    app.run(host="0.0.0.0", port=port, debug=False)
```

### 3.9 Pre-descargar los modelos

Sin esto, el primer video se queda "colgado" varios minutos descargando modelos.

```bash
cd /root/video-editor
./venv/bin/python -c "from silero_vad import load_silero_vad; load_silero_vad(); print('Silero OK')"
./venv/bin/python -c "
import os
from faster_whisper import WhisperModel
m = os.getenv('WHISPER_MODEL', 'large-v3-turbo')
WhisperModel(m, device='cpu', compute_type='int8'); print('Whisper OK:', m)
"
```

> El modelo `large-v3-turbo` pesa ~1.5 GB de descarga. Queda cacheado en
> `~/.cache/huggingface/`, no se vuelve a bajar.

### 📝 Documenta ahora (Paso 3)

En `CLAUDE.md`: completa la sección **VPS** con la IP, el usuario y el hostkey
reales; la tabla de **variables de entorno** con las que quedaron en el `.env`
(nombres, nunca valores); y el modelo de Whisper elegido en **Restricciones de
memoria**.

Si algo falló y lo resolviste en este paso (falta una dependencia, `pip` roto,
disco lleno), es la entrada **#1 de `troubleshooting.md`**. Regístrala ahora, con el
error textual.

---

## Paso 4 — Servicio systemd y firewall

### 4.1 Unidad systemd

Crea `/etc/systemd/system/video-editor.service`:

```ini
[Unit]
Description=Video Editor API
After=network.target

[Service]
Type=simple
WorkingDirectory=/root/video-editor
ExecStart=/root/video-editor/venv/bin/python /root/video-editor/api.py
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now video-editor
systemctl is-active video-editor
journalctl -u video-editor -n 30 --no-pager
```

### 4.2 Firewall

```bash
# Firewall del sistema operativo (si ufw está activo)
ufw status || true
ufw allow 5051/tcp || true
```

Si el proveedor tiene su **propio** firewall en el panel (Hostinger, Hetzner,
DigitalOcean), hay que abrir el puerto 5051 **también ahí**. Es la causa número uno
de "el servicio corre pero n8n no lo alcanza".

### 📝 Documenta ahora (Paso 4)

En `CLAUDE.md`, sección **Servicio**: confirma el nombre de la unidad systemd y los
comandos de restart/logs. Si tuviste que abrir el puerto en el firewall del panel
del proveedor, escríbelo ahí mismo con la ruta exacta del menú — es lo primero que
se olvida y lo primero que se necesita cuando el servicio deja de responder.

---

## Paso 5 — Verificación

### 5.1 El servicio responde

```bash
# Desde el propio VPS
curl -s http://localhost:5051/health
# Desde tu computadora
curl -s http://<IP>:5051/health
```

Debe devolver `{"status":"ok","service":"video-editor","storage":"local"}`.

### 5.2 Prueba real con un video corto

Usa un video de 1-2 minutos **con voz y con silencios** (no un video mudo: el VAD
no encontraría ningún segmento y fallaría, que es el comportamiento correcto).

```bash
# webhook de prueba: crea uno gratis en https://webhook.site y copia tu URL
curl -X POST http://<IP>:5051/process \
  -F "file=@prueba.mp4" \
  -F "title=Video de prueba" \
  -F "webhook_url=https://webhook.site/<TU_ID>" \
  -F "api_secret=<API_SECRET>"
```

Debe responder `202` con `"status":"processing"`. Después sigue el proceso:

```bash
journalctl -u video-editor -f
```

Vas a ver la secuencia: `Duración` → `Silero VAD` → `Transcribiendo` →
`Concatenando N segmentos` → `Generando metadata` → `UPLOAD` → `WEBHOOK -> ... (200)`.

### 5.3 Verificar sincronía audio/video (OBLIGATORIO)

Es el error más costoso de este pipeline: el video sale "bien" pero el audio va
corrido. Compara la duración de los dos streams del archivo final:

```bash
VIDEO=$(ls -t /root/video-editor/output/*/*.mp4 | head -1)
ffprobe -v error -select_streams v:0 -show_entries stream=duration -of csv=p=0 "$VIDEO"
ffprobe -v error -select_streams a:0 -show_entries stream=duration -of csv=p=0 "$VIDEO"
```

Las dos duraciones deben diferir en **menos de 0.1s**. Además, descarga el video y
escucha 10 segundos del final: si los labios y la voz no coinciden ahí, hay drift.

### 5.4 Limpieza del disco (solo si `STORAGE=local`)

Los videos terminados se acumulan. Opcional, borra los de más de 14 días:

```bash
crontab -l 2>/dev/null | { cat; echo "0 4 * * * find /root/video-editor/output -type f -mtime +14 -delete"; } | crontab -
```

### 📝 Documenta ahora (Paso 5)

Este es el paso que más entradas de `troubleshooting.md` genera: registrá **cada**
error que apareció en la prueba real, con el síntoma textual y la causa raíz. Los
típicos: OOM (`Killed` sin traceback), el VAD sin segmentos porque el audio estaba
mudo, el primer video eterno por la descarga de modelos.

En `CLAUDE.md`: anota el resultado de la verificación de **sync A/V** (las dos
duraciones que devolvió `ffprobe`) como línea base. Si en el futuro alguien toca el
corte, ese número es con lo que va a comparar.

---

## Paso 6 — Conectar n8n

En n8n, el nodo **HTTP Request** que envía el video. Campos:

| Campo | Valor |
|---|---|
| Method | `POST` |
| URL | `http://<IP_DEL_VPS>:5051/process` |
| Send Body | activado |
| Body Content Type | `Form-Data (multipart)` |
| Timeout (en Options) | `1000000` |

Parámetros del body:

| Nombre | Tipo | Valor |
|---|---|---|
| `file` | **n8n Binary File** | campo binario del nodo anterior (normalmente `data`) |
| `title` | Form Data | el título/tema del video |
| `webhook_url` | Form Data | la URL del webhook de n8n que recibe el resultado |
| `api_secret` | Form Data | el mismo valor del `.env` |

Nodo listo para importar (reemplaza los `<...>`):

```json
{
  "nodes": [
    {
      "parameters": {
        "method": "POST",
        "url": "http://<IP_DEL_VPS>:5051/process",
        "sendBody": true,
        "contentType": "multipart-form-data",
        "bodyParameters": {
          "parameters": [
            {
              "parameterType": "formBinaryData",
              "name": "file",
              "inputDataFieldName": "data"
            },
            {
              "name": "title",
              "value": "={{ $json.name.replace(/\\.[^.]+$/, '') }}"
            },
            {
              "name": "webhook_url",
              "value": "<URL_DEL_WEBHOOK_QUE_RECIBE_EL_RESULTADO>"
            },
            {
              "name": "api_secret",
              "value": "<API_SECRET>"
            }
          ]
        },
        "options": { "timeout": 1000000 }
      },
      "name": "VPS — Enviar Video",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [208, 0],
      "retryOnFail": true,
      "waitBetweenTries": 5000
    }
  ],
  "connections": {},
  "pinData": {}
}
```

Del otro lado necesitas un nodo **Webhook** en n8n (método POST) cuya URL pongas en
`webhook_url`. Ahí llega esto:

```json
{
  "status": "ok",
  "title": "Título optimizado para YouTube",
  "video_url": "http://<IP>:5051/files/2026-08-01/Video_de_prueba.mp4",
  "summary": "Resumen del video en 2-4 oraciones",
  "chapters": "00:00:00 Introducción\n00:02:30 Primer tema\n...",
  "resources": "Herramienta mencionada\nLink prometido",
  "transcript": "Transcripción completa del video"
}
```

Si algo falla, llega `{"status": "error", "error": "<detalle>"}` — conviene un nodo
**IF** sobre `status` para separar los dos caminos.

> El nodo HTTP Request recibe el `202` en segundos y sigue. El video **no** viene
> en esa respuesta: llega al webhook entre 5 y 30 minutos después. No encadenes la
> publicación al nodo HTTP Request, encadénala al Webhook.

### 📝 Documenta ahora (Paso 6)

En `CLAUDE.md`, sección **Endpoint**: confirma la URL final y los campos del body.
Guarda el JSON del nodo ya funcionando en `docs/nodo-n8n-process.json` y agrega la
línea correspondiente a `docs/CLAUDE.md`. Cuando el workflow de n8n se rompa dentro
de tres meses, ese JSON es el punto de comparación.

---

## Paso 7 — Cierre: revisar `ideas.md` y dejar la documentación al día

**Este paso no es opcional.** Es el que convierte la instalación en un proyecto
mantenible.

### 7.1 Revisar las ideas que quedaron pendientes

Lee `ideas.md` y preséntaselas al usuario así:

> Estas son las ideas que fueron surgiendo mientras instalábamos, que anoté para no
> descarrilar el objetivo:
>
> 1. **<idea>** — surgió cuando <contexto>. Esfuerzo: <bajo/medio/alto>.
> 2. **<idea>** — ...
>
> ¿Querés que tomemos alguna ahora, o las dejamos anotadas y cerramos aquí?

Si el usuario elige una: se vuelve el objetivo en curso, y las demás siguen en
`ideas.md`. Si no eligió ninguna, quedan ahí para la próxima sesión — que es
exactamente para lo que existe el archivo.

Si `ideas.md` está vacío, dilo igual: "no quedaron ideas pendientes anotadas".

### 7.2 Repaso final de la documentación

Verifica una por una y reportá el estado:

| Archivo | Debe tener |
|---|---|
| `CLAUDE.md` | Árbol de la carpeta, IP/usuario/hostkey reales, variables del `.env` (nombres), pipeline, comandos del servicio, workflow de deploy y las 5 reglas obligatorias |
| `execution/CLAUDE.md` | Los 2 módulos con sus funciones y las notas no obvias |
| `docs/CLAUDE.md` | Una línea por cada archivo que haya en `docs/` |
| `troubleshooting.md` | Una entrada por cada error que apareció durante la instalación |
| `ideas.md` | Las ideas del proceso, cada una con el contexto de dónde salió |
| `.gitignore` | `.env`, `venv/`, `logs/`, `output/`, `*.mp4` |

Si algún archivo quedó con los placeholders `<...>` del template, complétalo ahora.
Un `CLAUDE.md` con placeholders es peor que no tenerlo: se lee como si fuera real.

### 7.3 De aquí en adelante

Cada vez que se retome este proyecto, lo primero es **leer `CLAUDE.md`,
`troubleshooting.md` e `ideas.md`** — en ese orden. Y en cada cambio:

1. ¿Toca el corte o la concatenación? → verificar sync A/V con `ffprobe`.
2. ¿Cambió algo del panorama? → actualizar el `CLAUDE.md` que corresponda, en cascada.
3. ¿Se corrigió un bug? → entrada nueva en `troubleshooting.md`.
4. ¿Surgió una idea al margen? → a `ideas.md`, no al código.
5. ¿Va al VPS? → revisar el diff contra el local antes de subir, y nunca editar
   directo en el servidor.

---

## Tiempos de referencia (VPS 2 vCPU)

| Paso | Video 10 min | Video 30 min |
|---|---|---|
| VAD + corte FFmpeg | ~1 min | ~3 min |
| Transcripción Whisper | ~2 min | ~6 min |
| Metadata con Claude | ~10 s | ~15 s |
| Subida | ~30 s | ~90 s |
| **Total** | **~4 min** | **~11 min** |

---

## Cómo funciona por dentro

```
POST /process
  → responde 202 al instante
  → thread en background (Semaphore(1): un video pesado a la vez)

      1. get_duration(video)

      2. extract_audio(video) → audio.wav mono 16 kHz

      3. get_speech_timestamps_silero(audio.wav)
         min_speech=0.25s, min_silence=0.6s

      4. merge_close_segments(max_gap=0.3s)     une cortes pegados
      5. add_padding(0.05s)                      margen para no cortar palabras

      6. transcribe_video(video ORIGINAL)        Whisper word-level
         ↑ el original, no el editado: así los capítulos se pueden reajustar

      7. concatenate_segments(enhance_audio=True)
         un ffmpeg por segmento → intermedios .nut + PCM → concat →
         highpass 80Hz + loudnorm -16 LUFS

      8. generate_metadata(words, cuts, ...)     Claude vía OpenRouter
         y ajusta los timestamps de capítulos restando los silencios eliminados

      9. _upload(output.mp4)                     local o R2
     10. POST al webhook_url con el JSON completo
```

Decisiones que parecen raras y no lo son:

- **`Semaphore(1)`**: dos videos en paralelo = dos Whisper + dos PyTorch en RAM =
  OOM kill. El segundo request recibe 202 igual, pero su thread espera en la cola.
- **Un ffmpeg por segmento** en lugar de un `filter_complex` único: el
  `filter_complex` gigante llegó a consumir 15 GB de RAM con un video de 22 min.
- **Re-encodear en el concat final** (no `-c:v copy`): con `copy` se pierde ~medio
  frame por empalme y el audio termina adelantado.
- **Whisper sobre el video original**: los capítulos que devuelve Claude están en
  tiempos del original; se restan los silencios eliminados para reubicarlos.
- **VAD también corta el silencio del arranque**, es intencional. Si el video
  empieza con un ruido audible (golpe al micrófono, respiración), Silero puede
  clasificarlo como voz y conservarlo: es comportamiento esperado.

---

## Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| `curl http://<IP>:5051/health` no responde desde afuera, sí desde el VPS | Firewall del panel del proveedor | Abrir TCP 5051 en el firewall del panel, no solo en `ufw` |
| El servicio arranca y muere en loop | Falta una dependencia o el `.env` | `journalctl -u video-editor -n 50 --no-pager` y leer el traceback |
| `Killed` en los logs, sin traceback | OOM: se agotó la RAM | Crear swap (3.2) y bajar `WHISPER_MODEL` a `medium` o `small` |
| El webhook llega con `status: error` y `lista de segmentos vacía` | El VAD no detectó voz | El audio está mudo, muy bajo o en otra pista. Revisar con `ffprobe` |
| Primer video eterno y después normal | Descarga de modelos | Correr el Paso 3.9 |
| Capítulos corridos respecto al video | El ajuste por cortes falló | Verificar que `_segments_to_cuts` recibe la duración del video ORIGINAL |
| Audio desincronizado del video | Drift en la concatenación | Revisar 5.3. No cambiar `-c:v copy` en el concat final |
| `401 Unauthorized` desde n8n | `api_secret` no coincide | Comparar el valor del nodo con el `.env` del servidor |
| Nombres técnicos mal transcritos | Falta vocabulario | Agregar los términos a `vocab_prompt` en `simple_video_edit.py` |

Después de cualquier cambio en el código:

```bash
systemctl restart video-editor && systemctl is-active video-editor
journalctl -u video-editor -n 30 --no-pager
```

---

## Endurecer el servidor (recomendado después de que todo funcione)

1. **Llave SSH** en lugar de contraseña, y `PasswordAuthentication no` en
   `/etc/ssh/sshd_config`.
2. **`API_SECRET` siempre definido**: el puerto 5051 queda abierto a internet; sin
   secreto cualquiera puede mandarte videos a procesar.
3. **HTTPS con dominio** (Caddy o Nginx + Let's Encrypt) si vas a mandar contenido
   sensible: en HTTP plano, el video y la transcripción viajan sin cifrar.
4. **fail2ban** (`apt-get install -y fail2ban`) contra fuerza bruta en SSH.
