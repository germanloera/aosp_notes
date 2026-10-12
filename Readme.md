# Android Open Source Project (AOSP) Setup & Build Guide

A comprehensive, step-by-step guide to setting up a Debian build environment, downloading the source code via the `repo` tool, configuring environment variables, and building Android Open Source Project (AOSP) images.

Para consultar la versión en espanol, revisa la [Guía de AOSP](readme_es.md).

## 1. System Requirements & Debian Package Setup

### 1.1 Minimum Hardware Requirements

* **OS:** Tested on Debian 13 (Trixie), 64-bit.
* **Storage:** At least 400 GB of free disk space (NVMe SSD strongly recommended; checking out a single branch requires ~250 GB, while building requires another 150 GB+).
* **RAM:** Minimum 32 GB (64 GB or more recommended to prevent Out-Of-Memory errors during compilation and linking).
* **Swap:** If you have 32 GB of RAM or less, configure at least 16–32 GB of swap space.

### 1.2 Installing Build Dependencies on Debian

AOSP requires several development tools, native build utilities, and libraries. Run the following commands to update your package lists and install all required dependencies:

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

#### Package Breakdown:

* `git-core`: Version control system used by AOSP and the `repo` tool.
* `gnupg`: Verification tool used for verifying GPG signatures.
* `flex` & `bison`: Lexer and parser generators used during native code builds (e.g., Linux kernel, host tools).
* `build-essential`: Meta-package providing `gcc`, `g++`, `make`, and standard C/C++ libraries.
* `curl`: Utility to download files over HTTP/HTTPS (used to fetch the `repo` launcher).
* `zlib1g-dev`, `libssl-dev`: Compression and cryptographic headers used by host utilities.
* `libxml2-utils` & `xsltproc`: XML validation and parsing utilities (for manifests and device trees).
* `python3` & `python-is-python3`: Ensures scripts calling `python` execute via Python 3 (required by modern AOSP toolchains and `repo`).
* `ccache`: Compiler cache to speed up subsequent rebuilds.

---

## 2. Installing and Configuring the `repo` Tool

Google created `repo` to manage multiple Git repositories simultaneously using an XML manifest file.

### 2.1 Download and Setup `repo`

```bash
# 1. Create a personal binary directory if it does not exist
mkdir -p ~/.bin

# 2. Download the repo script
curl -o ~/.bin/repo https://storage.googleapis.com/git-repo-downloads/repo

# 3. Grant execute permissions
chmod a+x ~/.bin/repo
```

### 2.2 Configure Shell Environment Variables

Add `~/.bin` to your `PATH` and configure Git with your identity:

```bash
# Append ~/.bin to PATH in your ~/.bashrc or ~/.zshrc
echo 'export PATH="${HOME}/.bin:${PATH}"' >> ~/.bashrc
source ~/.bashrc

# Verify installation
repo --version
```

Configure your Git user identity (mandatory for commits and repo management):

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

---

## 3. Initializing and Syncing the Source Tree

Create a dedicated directory to house the AOSP tree:

```bash
mkdir -p ~/aosp
cd ~/aosp
```

### 3.1 Initializing Repo (`repo init`)

The `repo init` command initializes a `.repo` directory containing the manifest files and repository configurations.

```bash
repo init --partial-clone --no-use-superproject -b android-latest-release -u https://android.googlesource.com/platform/manifest
```

#### Command & Parameters:

* `-u <url>`: Target manifest repository URL (`https://android.googlesource.com/platform/manifest`).
* `-b <branch/tag>`: Specifies the exact release branch or tag to track (e.g., `android-14.0.0_r50` or `android-latest-release`).
* `--partial-clone`: Drastically reduces download time and disk space usage by skipping heavy binary files you do not need immediately.
* `--no-use-superproject`: Forces repo to sync projects by parsing the traditional manifest XML file instead of using Google's centralized superproject.

### 3.2 Syncing the Source Code (`repo sync`)

Fetch all repositories specified in the manifest:

```bash
repo sync -c -j$(nproc) 
```

#### Command & Parameters:

* `-c` (`--current-branch`): Fetch only the specified branch (saves bandwidth and disk space).
* `-j$(nproc)`: Sets parallel download jobs matching the number of CPU cores available on your system.

---

## 4. Build Environment Configuration

AOSP uses custom shell functions defined in `build/envsetup.sh` to configure paths, lunch targets, discovery utilities, and target device flags.

### 4.1 Sourcing Build Scripts

Before running any target selection or build commands, you must source the build environment script:

```bash
source build/envsetup.sh
```

#### Key Shell Functions Loaded:

* `lunch`: Selects the build target, architecture, and variant.
* `list_products`, `list_releases`, `list_variants`: Modern inspection utilities to discover available target combinations.
* `m`: Invokes the build system (`Soong` / `Ninja`) from any sub-directory inside the source tree.
* `croot`: Changes working directory to the root of the AOSP tree.
* `godir <file>`: Quickly jumps to any directory containing a specific file.

### 4.2 Configuring CCache (Optional but recommended)

To accelerate repeated builds, enable CCache:

```bash
# Enable compiler caching in AOSP
export USE_CCACHE=1

# Specify the cache storage location
export CCACHE_DIR="${HOME}/.ccache"

# Configure cache size limit (50 GB - 100 GB is recommended)
ccache -M 50G
```

* `USE_CCACHE=1`: Instructs the Soong/Kati build engine to prefix compiler calls with `ccache`.
* `CCACHE_DIR`: Specifies the directory where object cache files are stored.
* `-M <size>`: Sets the maximum disk space CCache can consume.

---

## 5. Discovering Targets: `list_products`, `list_releases`, `list_variants`

Modern AOSP (Android 14+ trunk-based development) uses a three-part combo format:

$$\text{TARGET} = \langle\text{product}\rangle-\langle\text{release\\_config}\rangle-\langle\text{build\\_variant}\rangle$$


Instead of guessing target names or scrolling through hundreds of entries in an interactive menu, `envsetup.sh` provides dedicated discovery commands.

### 5.1 `list_products`

* **What it does:** Scans the build configuration tree and prints all registered device and product configurations available in the current source tree.
* **When to use it:** When you are looking for the exact product code name (e.g., Pixel device codename, emulator product like `sdk_phone64_x86_64`, or Cuttlefish `aosp_cf_x86_64_phone`).
* **Usage:**
  ```bash
  list_products
  ```
  *Filter example:*
  ```bash
  list_products | grep -E "sdk|cf"
  ```

### 5.2 `list_releases`

* **What it does:** Lists all defined release configurations (trunk configurations). Release configs define feature flag values and build settings under modern trunk-based development.
* **When to use it:** When determining which release branch profile or feature set to compile against (e.g., `trunk_staging`, `trunk_food`, `next`).
* **Usage:**
  ```bash
  list_releases
  ```

### 5.3 `list_variants`

* **What it does:** Lists the available compilation variants supported by the build system.
* **When to use it:** When verifying available permission and debugging presets.
* **Usage:**
  ```bash
  list_variants
  ```
* **Output values:**
  * `user`: Strict production build. Root access is disabled by default, debug flags off, ADB requires explicit authorization.
  * `userdebug`: Production behavior with `adb root` access, logging enabled, and debugging symbols available.
  * `eng`: Engineering build. All assertions enabled, minimal optimization for faster compile times, all developer debug features enabled.

---

## 6. Target Selection and Environment Variables

You can configure your build target either using the interactive/direct `lunch` command or by exporting environment variables directly before invoking the build.

### 6.1 Using Environment Variables (`TARGET_PRODUCT`, etc.)

In automated build scripts, CI/CD pipelines (like Jenkins or GitHub Actions), or non-interactive shell sessions, you can define target parameters explicitly via environment variables rather than passing arguments to `lunch`:

```bash
export TARGET_PRODUCT='sdk_phone64_x86_64'
export TARGET_RELEASE='trunk_staging'
export TARGET_BUILD_VARIANT='userdebug'
```

#### What `export TARGET_PRODUCT='sdk_phone64_x86_64'` does:

* **Definition:** Specifies the exact device/platform definition file (`AndroidProducts.mk`) to compile.
* **What `sdk_phone64_x86_64` represents:** A 64-bit x86_64 Android Virtual Device (AVD) image designed for Android Emulator testing and SDK development.
* **When to use it:**
  * In automated CI/CD build scripts where interactive `lunch` menus cannot be prompted.
  * When pinning the target platform persistently across terminal sessions or build wrapper scripts.
* **Executing after exporting:**
  Once exported, simply run `lunch` with no arguments (or run `m` directly if your environment has sourced `envsetup.sh`):
  ```bash
  lunch
  ```
  `lunch` will detect `TARGET_PRODUCT`, `TARGET_RELEASE`, and `TARGET_BUILD_VARIANT` automatically and skip the interactive prompt.

### 6.2 Using the Direct `lunch` Command

Alternatively, supply the combo directly:

```bash
lunch sdk_phone64_x86_64-trunk_staging-userdebug
```

*(For older Android versions prior to trunk-based development: `lunch aosp_x86_64-userdebug`)*

---

## 7. Building the Source Tree

Execute the compilation using the `m` command:

```bash
m -j$(nproc)
```

### Command & Parameters:

* `m`: Top-level build wrapper for Soong/Kati/Ninja. Replaces older direct `make` calls and can be run from any sub-directory inside the source tree.
* `-j$(nproc)`: Number of concurrent build tasks matching CPU cores.
  * *Note:* If your system runs out of memory (OOM) during linking, lower the job count (e.g., `-j8` or `-j4`).

### Common Build Targets:

* `m`: Builds full platform images (`boot.img`, `system.img`, `vendor.img`, `super.img`, `userdata.img`).
* `m systemimage`: Compiles only the system partition image.
* `m framework`: Builds only the Android Java Framework (`framework.jar`).
* `m droid`: Explicit target equivalent to full default platform build.

---

## 8. Output Directory & Running the Build

Once the build completes successfully, all generated images and binaries reside in:

```bash
cd out/target/product/${TARGET_PRODUCT:-sdk_phone64_x86_64}/
```

### Launching the Emulator:

If you built an emulator target like `sdk_phone64_x86_64` or `aosp_cf_x86_64_phone`, run:

```bash
emulator
```

Useful runtime flags:

* `-show-kernel`: Streams Linux kernel boot logs directly to stdout.
* `-gpu host`: Uses the host GPU for hardware-accelerated OpenGL/Vulkan rendering.
* `-wipe-data`: Reverts user data partitions to clean states.
* `-selinux permissive`: Temporarily boots SELinux in permissive mode for debugging driver or policy failures.
