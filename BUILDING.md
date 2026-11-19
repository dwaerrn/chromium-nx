# Building Chromium NX

A step-by-step guide, from a computer with nothing installed to a
`chromium-nx.nro` on the SD card. It follows the build scripts in `gx/`; the
places where the repository says nothing (where the kernel source comes from,
packages its Dockerfile is missing) are marked **Note**.

> This guide was written from the scripts, not from a run of them. If a step
> fails, the error is usually a missing package: see *Troubleshooting*.

## What you need

- A 64-bit PC (x86-64) with Windows 10/11, macOS or Linux. Apple Silicon Macs
  work through emulation, slowly (see step 1).
- About **25 GB** of free disk space (the Linux kernel source twice, Debian's
  Chromium, the images) and 8 GB of RAM or more.
- An Internet connection. The first full build takes roughly 30 to 90 minutes,
  most of it the two kernel builds; later builds take a few minutes.

## How the build is laid out

Two Docker containers share two folders:

| Container   | Image                       | What it builds                                          |
|-------------|-----------------------------|---------------------------------------------------------|
| `lkl-gx`    | Debian 12 (`gx/docker/Dockerfile`) | the guest libraries and Chromium's root image (`chromium-full.ext4`) |
| `lkl-gx-nx` | `devkitpro/devkita64`       | the Switch kernel and the NRO                           |

- `/project` in both is this repository on your computer.
- `/work` in both is a Docker volume, `lklwork`: the kernel source and every
  intermediate file. It survives the containers being removed.

The four scripts run in this order:

```
build-lkl.sh  (lkl-gx)     kernel for the PC test bench
build-pc.sh   (lkl-gx)     guest libraries: libEGL, libGLESv2, libgxvideo, GL lists
build-chromium.sh (lkl-gx) chromium-full.ext4 + the empty profile image
build-nx.sh   (lkl-gx-nx)  Switch kernel + chromium-nx.nro -> gx/dist-nx/
```

## 1. Install Docker

**Windows**

1. Install WSL 2: open PowerShell as administrator, run `wsl --install`,
   reboot, and finish the Ubuntu setup it opens (a user name and password).
2. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/).
   In its Settings: *General* → "Use the WSL 2 based engine" on, and
   *Resources → WSL integration* → turn on your Ubuntu distribution.
3. Do **everything below in the Ubuntu terminal**, not in PowerShell, and keep
   the repository inside Ubuntu's own files (`~/chromium-nx`), not under
   `/mnt/c/...`: builds on the Windows drive are many times slower, and Windows
   tools can turn the scripts' line endings into CRLF, which breaks them.

**macOS**

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
   and start it once.
2. In Settings → *Resources*, give it at least 8 GB of memory and 60 GB of disk.
3. **Apple Silicon (M1 and later):** both images are made for x86-64. Add
   `--platform linux/amd64` to every `docker build` and `docker run` below, and
   in Settings → *General* turn on "Use Rosetta for x86/amd64 emulation".
   Expect the build to be several times slower.

**Linux**

Install Docker Engine from your distribution or
[docs.docker.com/engine/install](https://docs.docker.com/engine/install/), then
let your user run it without `sudo` and log out and back in:

```bash
sudo usermod -aG docker $USER
```

**Check** (any system) that Docker works:

```bash
docker run --rm hello-world
```

## 2. Get the repository

```bash
git clone https://github.com/dwaerrn/chromium-nx.git ~/chromium-nx
```

```bash
cd ~/chromium-nx
```

To build a branch other than `main`, for example the one with your changes:

```bash
git checkout claude/magical-cannon-s9etyv
```

## 3. Create the Debian container (`lkl-gx`)

**Note:** the Dockerfile in the repository lacks `chromium:arm64`, the full
browser. Without it `build-chromium.sh` silently skips the console's image and
the empty profile, and `build-nx.sh` then stops with "run build-chromium.sh
first". The README says the container has it, so add it:

```bash
sed -i 's/chromium-headless-shell:arm64 chromium-shell:arm64/chromium:arm64 chromium-headless-shell:arm64 chromium-shell:arm64/' gx/docker/Dockerfile
```

Build the image and start the container (it just sleeps; the scripts are run
inside it with `docker exec`):

```bash
docker build -t lkl-gx gx/docker
```

```bash
docker run -d --name lkl-gx -v "$PWD":/project -v lklwork:/work lkl-gx sleep infinity
```

The Chromium you get is whatever Debian 12 ships at that moment (its security
updates follow Chrome's releases), so the version may be newer than 156.

## 4. Get the kernel source (LKL)

**Note:** the scripts expect the LKL kernel source in `/work/linux` and never
fetch it. It is [github.com/lkl/linux](https://github.com/lkl/linux); the
repository does not say which commit was used, so the latest one is taken
here. If a later step fails to compile inside the kernel, an older commit of
LKL is the first thing to try.

```bash
docker exec lkl-gx git clone --depth 1 https://github.com/lkl/linux.git /work/linux
```

## 5. Build in the Debian container

`NO_TESTS=1` skips the test programs, which run under emulation and need more
packages than the image has; only what the console needs is made.

```bash
docker exec lkl-gx bash /project/gx/build-lkl.sh
```

This one prints **nothing** while it works (10 to 40 minutes; hours on Apple
Silicon): the compiler's output goes to a log inside the container, shown only
if it fails. To watch it, in a second terminal:

```bash
docker exec lkl-gx tail -f /tmp/lkl-build.log
```

**Do not press Ctrl+C and start it again:** Ctrl+C stops only your terminal,
the build goes on inside the container, and two builds in the same kernel tree
break each other (errors such as `fixdep: error opening file`, or
`redefinition of` / `has no member named` in files that are fine). If that
happened, stop everything and start clean:

```bash
docker restart lkl-gx
```

```bash
docker exec lkl-gx make -C /work/linux ARCH=lkl mrproper
```

It is done when it prints a line listing `tools/lkl/liblkl.a`. `build-nx.sh`
works the same way for its kernel (step 7); its log is `/work/gx-nx/kernel.log`.

```bash
docker exec -e NO_TESTS=1 lkl-gx bash /project/gx/build-pc.sh
```

```bash
docker exec -e NO_TESTS=1 lkl-gx bash /project/gx/build-chromium.sh
```

**Check:** the last one should print a line ending in
`chromium-full-switch.ext4: <size> MB`. If it does not, `chromium:arm64` is
missing from the image (step 3).

## 6. Create the devkitPro container (`lkl-gx-nx`)

```bash
docker run -d --name lkl-gx-nx -v "$PWD":/project -v lklwork:/work devkitpro/devkita64:latest sleep infinity
```

The tools the kernel build needs (from the header of `gx/build-nx.sh`), plus
Python, which the script also runs:

```bash
docker exec lkl-gx-nx bash -c "apt-get update && apt-get install -y flex bison bc libelf-dev cpio rsync build-essential python3 binutils"
```

**Note:** the NRO is linked against these devkitPro libraries (the `-l` list
in `build-nx.sh`). The image may already have some; this installs whatever is
missing:

```bash
docker exec lkl-gx-nx dkp-pacman -Sy --needed --noconfirm switch-mesa switch-libdrm_nouveau switch-ffmpeg switch-dav1d switch-curl switch-freetype switch-harfbuzz switch-bzip2 switch-libpng switch-libjpeg-turbo switch-zlib
```

## 7. Build the NRO

```bash
docker exec lkl-gx-nx bash /project/gx/build-nx.sh
```

The first run builds the Switch kernel (a copy of `/work/linux` in
`/work/linux-nx`) and takes a while; later runs only recompile the program.

**Check:** it ends with a listing of `gx/dist-nx/switch/chromium-nx` and
"copy the contents of gx/dist-nx to the root of the SD card". A line starting
with `warning: DEFAULT_COMMAND (host_nx.c) is not the image's command line`
means the browser's flags in `gx/host_nx.c` and in the build scripts differ
(see *Changing the browser's flags*).

## 8. Put it on the console

`gx/dist-nx/switch/chromium-nx/` (in your repository folder) now holds:

- `chromium-nx.nro`: the program (launcher and browser).
- `chromium-full.ext4`: Chromium's root image.

Copy both to `sdmc:/switch/chromium-nx/`, replacing the old ones. Your
`profile.ext4`, settings and downloads there are kept.

Keep a copy of the old two files: if the new build misbehaves, put them back.
If you only changed C code (not the Chromium image), copying the `.nro` alone
is enough.

The forwarder you made with Sphaira points at the `.nro`, so it keeps working.

On Windows the folder is reachable from Explorer at
`\\wsl$\Ubuntu\home\<user>\chromium-nx\gx\dist-nx`.

## Rebuilding after a change

The containers and the `lklwork` volume stay, so after editing code:

```bash
docker start lkl-gx lkl-gx-nx
```

```bash
docker exec lkl-gx-nx bash /project/gx/build-nx.sh
```

Re-run `build-pc.sh` and `build-chromium.sh` (step 5) first only if you
changed something under `gx/guest/`, the Chromium image, or its flags in
`build-chromium.sh`.

## Changing the browser's flags (e.g. loading an extension)

The browser's command line is written in three places that must agree:

1. `DEFAULT_COMMAND` in `gx/host_nx.c`: what actually runs; the launcher
   builds every tile's command line from it.
2. `FULL_FLAGS` in `gx/build-chromium.sh`.
3. `NX_FLAGS` in `gx/build-nx.sh`.

`build-nx.sh` checks that (1) equals (2) + (3) + the GPU flags + the start
page, and only warns if not: the console runs (1) regardless.

To load uBlock Origin Lite from the SD card on every start, add
`--load-extension=/root/SD/switch/chromium-nx/extensions/ubol` to
`DEFAULT_COMMAND` just before `"https://www.google.com/"`, and the same flag at
the end of `NX_FLAGS`. Keep the trailing space inside each quoted piece of
`DEFAULT_COMMAND`. Then rebuild (step 7) and copy only the `.nro`. If Chromium
ignores the flag, also append `,DisableLoadExtensionCommandLineSwitch` to the
`--disable-features=` list in both places; Chromium uses only the last
`--disable-features` it is given, so do not add a second one.

## Troubleshooting

- **`$'\r': command not found`** or **`bad interpreter`**: the scripts have
  Windows line endings. Clone inside WSL (step 1), or run
  `git config --global core.autocrlf false` and clone again.
- **`cannot find -l<name>`** in `build-nx.sh`: a devkitPro library is missing.
  Find its package with `docker exec lkl-gx-nx dkp-pacman -Ss <name>` and
  install it as in step 6.
- **`Unable to locate package ...:arm64`** in `docker build`: Debian's mirrors
  were briefly out of sync; run the build again. If `chromium:arm64` itself is
  the one missing, Debian may be between versions for arm64 that day.
- **The kernel build fails** (`build-lkl.sh` prints the end of its log, or
  `build-nx.sh` shows `kernel.log`): LKL's latest code may have moved on from
  what the project was written against. Try an older commit in `/work/linux`
  (and delete `/work/linux-nx` so the Switch copy is made again).
- **`no space left on device`**: give Docker more disk (Docker Desktop:
  Settings → Resources), or remove old images with `docker system prune`
  (this keeps volumes, so `lklwork` stays).
- **Start over** completely (deletes the kernel and every intermediate file):

  ```bash
  docker rm -f lkl-gx lkl-gx-nx
  ```

  ```bash
  docker volume rm lklwork
  ```
