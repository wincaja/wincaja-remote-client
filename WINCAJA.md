# WinCaja Remote - App de Escritorio

Fork personalizado de [RustDesk](https://github.com/rustdesk/rustdesk) con branding de WinCaja para soporte remoto.

## Que es este repo

Este repo contiene el codigo fuente de la app de escritorio **WinCaja Remote** que instalan los clientes y tecnicos de soporte en Windows. Es un fork de RustDesk con las siguientes personalizaciones:

- Nombre: "WinCaja Remote" (en vez de "RustDesk")
- Logo e iconos de WinCaja
- Servidor pre-configurado: `52.224.5.18`
- Key publica pre-configurada
- Texto: "Creada por los desarrolladores de WinCaja"

## Repositorios relacionados

| Repo | Funcion |
|------|---------|
| **Este repo** (`wincaja/wincaja-remote-client`) | App de escritorio (.exe) con branding WinCaja |
| `wincaja/hbb_common` (branch `wincaja-custom`) | Submodulo con configuracion del cliente (APP_NAME, IP, key). Requerido para compilar |
| `wincaja-remote` (VM local `~/wincaja-remote`) | Servidor Docker (hbbs + hbbr) que gestiona las conexiones |

### Por que 3 repos?

- **Este repo**: Contiene la interfaz, iconos, textos y toda la logica de la app
- **hbb_common**: RustDesk separa su configuracion base (nombre de app, servidor, key) en un submodulo aparte. No se puede evitar, es parte de la arquitectura de RustDesk
- **wincaja-remote**: El servidor que corre en la VM de Azure. Sin el, las conexiones no funcionan

## Archivos de branding modificados

| Archivo | Que contiene |
|---------|-------------|
| `flutter/windows/runner/resources/app_icon.ico` | Icono de la app en Windows |
| `flutter/windows/runner/Runner.rc` | Metadata: nombre, copyright, descripcion |
| `flutter/windows/CMakeLists.txt` | Nombre del ejecutable (`wincaja-remote`) |
| `res/icon.ico` | Icono general |
| `res/icon.png` | Icono en PNG |
| `res/tray-icon.ico` | Icono de bandeja del sistema |
| `src/lang/es.rs` | Textos en espanol (incluye "Creada por los desarrolladores de WinCaja") |
| `src/lang/en.rs` | Textos en ingles |
| `Cargo.toml` | Descripcion del paquete |

## Configuracion del servidor (submodulo hbb_common)

En `libs/hbb_common/src/config.rs`:

| Variable | Valor |
|----------|-------|
| `APP_NAME` | `"WinCaja Remote"` |
| `RENDEZVOUS_SERVERS` | `["52.224.5.18"]` |
| `RS_PUB_KEY` | `"IhYO7aSmD9wSyvMpddfKjKNXPcgjTTpRdOxaja2zmHs="` |

## Como compilar el .exe

### Opcion 1: GitHub Actions (recomendado)

1. Haz los cambios de branding en este repo y/o en `wincaja/hbb_common` (branch `wincaja-custom`)
2. Push a GitHub
3. Ve a **Actions** → **"Build WinCaja Remote (Windows)"** → **"Run workflow"**
4. Espera ~20 minutos
5. El `.exe` aparece en **Releases**

**Recomendacion**: Haz TODOS los cambios primero y verifica los archivos. Solo entonces dispara UN build. Cada build toma ~20 minutos.

### Opcion 2: Build local (Windows)

Requiere instalar: Visual Studio 2022, Rust 1.75, Flutter 3.24.5, LLVM 15, vcpkg, Python 3. No recomendado salvo que necesites iterar rapido.

## Pagina de descarga

El `.exe` se descarga desde `soporte.wincaja.mx` (ruta `/`). El link apunta al ultimo Release de este repo. La pagina esta en el repo `wincaja/WinCaja-Assist` → `app/routes/index.tsx`.

## Sistema de actualizaciones

La app verifica al abrirse si hay una version nueva consultando el servidor de versiones:

```
App se abre → GET http://52.224.5.18:21114/version/latest
           → Si hay version nueva → Muestra aviso "Actualizar"
           → Si es la misma version → Funciona normal
```

### Archivos involucrados

| Archivo | Repo | Que hace |
|---------|------|----------|
| `src/common.rs` (funcion `check_software_update`) | Este repo | Ejecuta la verificacion al abrir la app |
| `libs/hbb_common/src/lib.rs` (URL en `version_check_request`) | hbb_common | Define la URL del servidor de versiones (`52.224.5.18:21114`) |
| `version-api/version.json` | wincaja-remote (servidor) | Archivo que controla cual es la version vigente |

### Como sacar una nueva version (paso a paso completo)

**Paso 1**: Haz los cambios en este repo (branding, iconos, textos) y/o en `hbb_common`

**Paso 2**: Actualiza el numero de version en DOS archivos de este repo:

```
# En .github/workflows/build-wincaja.yml
VERSION: "1.1.0"   # <-- cambiar aqui

# En Cargo.toml
version = "1.1.0"  # <-- y aqui
```

**Paso 3**: Commit y push

```bash
git add .
git commit -m "feat: version 1.1.0 - descripcion"
git push origin master
```

**Paso 4**: Compilar en GitHub Actions

Ve a [Actions](https://github.com/wincaja/wincaja-remote-client/actions) → "Build WinCaja Remote (Windows)" → "Run workflow" → espera ~20 min

**Paso 5**: Actualizar el servidor de versiones

Conectate al servidor (VM Azure) y edita:

```bash
cd ~/wincaja-remote
nano version-api/version.json
```

Cambia a:

```json
{
  "url": "https://github.com/wincaja/wincaja-remote-client/releases/download/v1.1.0/wincaja-remote-1.1.0-x86_64.exe"
}
```

Reinicia:

```bash
docker restart rustdesk-version-api
```

**Paso 6**: Verificar

```bash
curl http://localhost:21114/version/latest
```

Los usuarios veran el aviso de actualizacion la proxima vez que abran la app.

## Workflow de GitHub Actions

El archivo `.github/workflows/build-wincaja.yml` define el proceso de compilacion:

1. Genera el bridge Rust-Flutter (Ubuntu)
2. Compila la app para Windows x86_64 (Windows Server 2022)
3. Empaqueta el instalador auto-extraible
4. Elimina el Release anterior si existe
5. Sube el `.exe` como artifact y crea un nuevo Release
