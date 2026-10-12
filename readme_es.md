# Guía de Configuración y Compilación de Android Open Source Project (AOSP)

Una guía completa paso a paso para preparar un entorno de compilación en Debian, descargar el código fuente con la herramienta `repo`, configurar variables de entorno y compilar imágenes del sistema de AOSP.

---

## 1. Requisitos del Sistema e Instalación de Paquetes en Debian

### 1.1 Requisitos Mínimos de Hardware
* **Sistema Operativo:** Probado en Debian 13 (Trixie) de 64 bits.
* **Almacenamiento:** Mínimo 400 GB de espacio libre en disco (se recomienda encarecidamente SSD NVMe; clonar una rama requiere aprox. 250 GB y la compilación genera más de 150 GB de artefactos).
* **Memoria RAM:** Mínimo 32 GB (se recomiendan 64 GB o más para evitar fallos por falta de memoria [OOM] durante el enlazado binario).
* **Swap (Espacio de Intercambio):** Si dispones de 32 GB de RAM o menos, configura al menos 16–32 GB adicionales de swap.

---

### 1.2 Instalación de Dependencias en Debian

AOSP requiere herramientas nativas de desarrollo, librerías de enlace y utilidades de compilación. Ejecuta:

```bash
sudo apt update && sudo apt upgrade -y
```

```bash
sudo apt install -y \
    git-core gnupg flex bison build-essential zip curl zlib1g-dev \
    libc6-dev-i386 libncurses5-dev x11proto-core-dev \
    libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc \
    unzip fontconfig python3 python3-pip python-is-python3 \
    rsync ccache libssl-dev bc
```

#### Descripción de paquetes clave:
* `git-core`: Sistema de control de versiones base para la herramienta `repo` y el árbol de código.
* `gnupg`: Herramienta criptográfica para verificar firmas GPG de los repositorios y commits.
* `flex` y `bison`: Generadores de analizadores sintácticos y léxicos requeridos al compilar herramientas nativas y el kernel.
* `build-essential`: Metapaquete con `gcc`, `g++`, `make` y cabeceras estándar de C/C++.
* `curl`: Utilidad para transferir datos vía HTTP/HTTPS (usada para descargar el lanzador de `repo`).
* `zlib1g-dev` y `libssl-dev`: Cabeceras de compresión y criptografía requeridas por herramientas del host.
* `libxml2-utils` y `xsltproc`: Herramientas para parsear y validar archivos XML (manifiestos y árboles de dispositivos).
* `python3` y `python-is-python3`: Asegura que cualquier script o invocación de `python` ejecute Python 3 de manera predeterminada.
* `ccache`: Gestor de caché de compilación para acelerar reconstrucciones posteriores.

---

## 2. Instalación y Configuración de la Herramienta `repo`

Google desarrolló `repo` para gestionar múltiples repositorios Git simultáneamente mediante un manifiesto XML centralizado.

### 2.1 Descarga e Instalación de `repo`

```bash
# 1. Crear el directorio de binarios de usuario si no existe
mkdir -p ~/.bin

# 2. Descargar el script de repo
curl -o ~/.bin/repo https://storage.googleapis.com/git-repo-downloads/repo

# 3. Otorgar permisos de ejecución
chmod a+x ~/.bin/repo
```

### 2.2 Configuración del Entorno de Shell

Agrega `~/.bin` a tu variable `PATH` y define tu identidad en Git:

```bash
# Agregar ~/.bin al PATH en tu archivo ~/.bashrc (o ~/.zshrc)
echo 'export PATH="${HOME}/.bin:${PATH}"' >> ~/.bashrc
source ~/.bashrc

# Verificar instalación
repo --version
```

Configura tu identidad en Git (requerido para confirmar ramas y sincronizar):

```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tu.correo@ejemplo.com"
```

---

## 3. Inicialización y Sincronización del Código Fuente

Crea un directorio dedicado para albergar el árbol de AOSP:

```bash
mkdir -p ~/aosp
cd ~/aosp
```

### 3.1 Inicialización del Repositorio (`repo init`)

El comando `repo init` crea un directorio `.repo` que contiene los metadatos y el manifiesto de la versión deseada:

```bash
repo init --partial-clone --no-use-superproject -b android-latest-release -u https://android.googlesource.com/platform/manifest
```

#### Parámetros del comando:
* `-u <url>`: URL del repositorio del manifiesto (`https://android.googlesource.com/platform/manifest`).
* `-b <rama/etiqueta>`: Especifica la rama o etiqueta de lanzamiento a rastrear (por ejemplo, `android-14.0.0_r50` o `android-latest-release`).
* `--partial-clone`: Reduce drásticamente el tiempo de descarga y el espacio ocupado en disco al omitir archivos binarios pesados que no necesitas de inmedia
* `--no-use-superproject`: Obliga a repo a sincronizar los proyectos analizando el archivo XML tradicional de manifiestos, en vez de usar el superproyecto centralizado de Google.

---

### 3.2 Sincronización del Código Fuente (`repo sync`)

Descarga todos los repositorios declarados en el manifiesto:

```bash
repo sync -c -j$(nproc) 
```

#### Parámetros del comando:
* `-c` (`--current-branch`): Descarga únicamente la rama especificada en el manifiesto, omitiendo el resto.
* `-j$(nproc)`: Define el número de descargas en paralelo igual a la cantidad de núcleos de CPU detectados.

---

## 4. Configuración del Entorno de Compilación

AOSP provee scripts auxiliares ubicados en `build/envsetup.sh` que cargan funciones de shell para inspección, selección de objetivos y ejecución.

### 4.1 Carga del Entorno (`envsetup.sh`)

Antes de inspeccionar o compilar cualquier objetivo, es indispensable cargar este script en la sesión actual:

```bash
source build/envsetup.sh
```

#### Funciones y comandos cargados:
* `lunch`: Selecciona el objetivo de compilación, arquitectura y variante.
* `list_products`, `list_releases`, `list_variants`: Comandos de descubrimiento para explorar opciones disponibles.
* `m`: Invoca el sistema de compilación (`Soong` / `Ninja`) desde cualquier subdirectorio del proyecto.
* `croot`: Cambia el directorio de trabajo actual directamente a la raíz de AOSP.
* `godir <archivo>`: Navega automáticamente a la carpeta donde reside un archivo específico.

---

### 4.2 Configuración de CCache (Recomendado)

Habilitar CCache permite reutilizar binarios ya procesados en reconstrucciones posteriores:

```bash
# Habilitar el uso de caché en AOSP
export USE_CCACHE=1

# Especificar la ruta de almacenamiento de la caché
export CCACHE_DIR="${HOME}/.ccache"

# Asignar límite de tamaño (recomendado entre 50 GB y 100 GB)
ccache -M 50G
```

* `USE_CCACHE=1`: Informa a Soong y Kati que deben anteponer `ccache` en las llamadas al compilador.
* `CCACHE_DIR`: Define la ruta donde se guardarán los objetos compilados.
* `-M <tamaño>`: Límite máximo de espacio asignado.

---

## 5. Inspección de Objetivos: `list_products`, `list_releases`, `list_variants`

En versiones modernas de AOSP (a partir de Android 14 con desarrollo basado en tronco o *trunk-based development*), un objetivo se define mediante la combinación:

$$\text{TARGET} = \langle\text{product}\rangle-\langle\text{release\\_config}\rangle-\langle\text{build\\_variant}\rangle$$


Para evitar conjeturas o listas interactivas lentas, `envsetup.sh` incluye comandos especializados de consulta.

### 5.1 `list_products`
* **Para qué sirve:** Escanea el árbol de compilación e imprime todos los productos y dispositivos registrados en los archivos `AndroidProducts.mk`.
* **Cuándo se usa:** Cuando necesitas conocer el nombre exacto de la plataforma de destino (por ejemplo, el emulador `sdk_phone64_x86_64`, dispositivos Pixel, o instancias de Cuttlefish como `aosp_cf_x86_64_phone`).
* **Uso:**
  ```bash
  list_products
  ```
  *Ejemplo con filtro:*
  ```bash
  list_products | grep -E "sdk|cf"
  ```

### 5.2 `list_releases`
* **Para qué sirve:** Lista todas las configuraciones de lanzamiento (*release configs*) disponibles. Estas configuraciones determinan qué banderas de características (*feature flags*) y ajustes de software se activan o desactivan.
* **Cuándo se usa:** Al definir bajo qué perfil de estabilidad o fase de desarrollo vas a compilar (por ejemplo, `trunk_staging`, `trunk_food`, `next`).
* **Uso:**
  ```bash
  list_releases
  ```

### 5.3 `list_variants`
* **Para qué sirve:** Muestra las variantes de compilación soportadas por el sistema.
* **Cuándo se usa:** Para validar qué nivel de depuración y privilegios de seguridad tendrá la imagen final.
* **Uso:**
  ```bash
  list_variants
  ```
* **Significado de las variantes:**
  * `user`: Versión final para producción. Acceso root deshabilitado, banderas de depuración apagadas y ADB requiere autorización explícita.
  * `userdebug`: Comportamiento de producción pero con privilegios de `adb root`, registro de logs habilitado y símbolos de depuración activos.
  * `eng`: Versión de ingeniería. Todas las aserciones activas, optimizaciones reducidas para acelerar compilación y todas las herramientas de desarrollo habilitadas.

---

## 6. Selección de Objetivo y Variables de Entorno

Puedes configurar el objetivo de forma directa con `lunch` o predefinirlo mediante variables de entorno en scripts automatizados.

### 6.1 Uso de Variables de Entorno (`TARGET_PRODUCT`, etc.)

En entornos de integración continua (CI/CD como Jenkins, GitLab CI o GitHub Actions) o scripts desatendidos, puedes fijar las opciones exportando las variables en lugar de interactuar con el menú:

```bash
export TARGET_PRODUCT='sdk_phone64_x86_64'
export TARGET_RELEASE='trunk_staging'
export TARGET_BUILD_VARIANT='userdebug'
```

#### Detalle de `export TARGET_PRODUCT='sdk_phone64_x86_64'`:
* **Definición:** Especifica la definición concreta de hardware/producto a compilar (`AndroidProducts.mk`).
* **Qué representa `sdk_phone64_x86_64`:** Una imagen de emulador Android Virtual Device (AVD) de 64 bits optimizada para arquitecturas x86_64 de PC host, ideal para pruebas de framework y desarrollo de SDK.
* **Cuándo se usa:**
  * En pipelines automáticos donde no es viable esperar interacción en la terminal.
  * Para mantener fija la configuración entre múltiples sesiones de shell o llamadas a scripts internos.
* **Ejecución posterior:**
  Habiendo exportado estas variables y cargado `envsetup.sh`, basta con invocar `lunch` sin argumentos:
  ```bash
  lunch
  ```
  El comando detectará automáticamente `TARGET_PRODUCT`, `TARGET_RELEASE` y `TARGET_BUILD_VARIANT` y configurará el entorno sin realizar preguntas adicionales.

---

### 6.2 Uso Directo del Comando `lunch`

También puedes pasar la combinación completa en un solo argumento:

```bash
lunch sdk_phone64_x86_64-trunk_staging-userdebug
```

*(En versiones anteriores a Android 14 sin trunk-based development, el formato era: `lunch aosp_x86_64-userdebug`)*

---

## 7. Compilación del Código Fuente

Ejecuta la compilación utilizando el comando `m`:

```bash
m -j$(nproc)
```

### Parámetros y Funcionamiento:
* `m`: Interfaz principal para invocar Soong, Kati y Ninja. Reemplaza la ejecución directa de `make` y puede ejecutarse desde cualquier subcarpeta del árbol.
* `-j$(nproc)`: Cantidad de tareas concurrentes según los núcleos del procesador.
  * *Nota:* Si el proceso falla por falta de memoria RAM durante la fase de enlazado (*linking*), reduce este valor (por ejemplo, `-j8` o `-j4`).

### Objetivos Comunes de Compilación:
* `m`: Compila la plataforma completa (`boot.img`, `system.img`, `vendor.img`, `super.img`, `userdata.img`).
* `m systemimage`: Compila exclusivamente la partición del sistema (`system.img`).
* `m framework`: Compila únicamente el núcleo de librerías Java del framework (`framework.jar`).
* `m droid`: Objetivo explícito equivalente a la compilación completa estándar.

---

## 8. Directorio de Salida y Ejecución

Al concluir la compilación exitosamente, todas las imágenes generadas se ubican en:

```bash
cd out/target/product/${TARGET_PRODUCT:-sdk_phone64_x86_64}/
```

### Lanzamiento en el Emulador:

Si compilaste un objetivo compatible con emulador (`sdk_phone64_x86_64` o `aosp_cf_x86_64_phone`), puedes iniciarlo ejecutando:

```bash
emulator
```

#### Parámetros útiles de ejecución:
* `-show-kernel`: Muestra en consola la salida en vivo del kernel de Linux durante el arranque.
* `-gpu host`: Utiliza la tarjeta gráfica física de tu equipo para aceleración OpenGL/Vulkan por hardware.
* `-wipe-data`: Restaura las particiones de usuario a su estado inicial limpio.
* `-selinux permissive`: Inicia SELinux en modo permisivo para diagnosticar problemas de permisos y políticas.
