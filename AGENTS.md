# AGENTS.md - mulle-clang-cpack context for AI agents

## What this repo is

Packaging infrastructure for the **mulle-clang** compiler (a mulle-objc Objective-C compiler based on clang). It wraps a pre-built compiler from `/opt/mulle-clang-project/<VERSION>` into distributable packages using CMake's CPack.

Output: `.deb` (and `.rpm`) packages named like `mulle-clang-21.1.8.1-trixie-amd64.deb`.

The packages install:
- The compiler tree into `/opt/mulle-clang-project/<VERSION>/`
- Symlinks in `/usr/bin/`: `mulle-clang`, `mulle-scan-build`, `mulle-nm`


## Key environment variables

| Variable | Example | Purpose |
|---|---|---|
| `VERSION` | `21.1.8.1` | Compiler version to build/package |
| `RC` | `` or `-RC1` | Release candidate suffix |
| `MULLE_CLANG_PROJECT_TAG` | `21.1.8.1` | Same as VERSION, used in release scripts |
| `MAKETOOL` | `ninja` | Build tool (default: ninja) |
| `CC` / `CXX` | `clang` / `clang++` | Compiler to use for building (default: clang) |


## Repo layout

```
CMakeLists.txt      # CPack config: installs /opt tree + /usr/bin symlinks
create-deb          # Host-side script: manages VMs, orchestrates full build
package-build       # Guest-side script: clean/download/build/verpack/upload steps
generate-package    # Thin wrapper: runs cmake + cpack for DEB/RPM
```


## Build pipeline

### Full automated (host side)
```bash
VERSION=21.1.8.1 RC= ./create-deb "trixie"
```
This SSHes into the named VM and runs the steps below.

### Manual (inside VM / guest)
```bash
VERSION=21.1.8.1 RC= ./package-build clean download build verpack
```

Steps in `package-build`:
1. `clean` - wipe `mono/` dir, recreate it
2. `download` - fetch source tarball from GitHub (`mulle-cc/mulle-clang-project`)
3. `build` - compile via `mulle-clang-project/clang/bin/cmake-ninja.linux`
4. `verpack` - run `generate-package`, move `.deb` to parent dir
5. `upload` - scp `.deb` to `oswald.codeon.de` debian repo (optional)


## Build environments

### Local VMs (virsh/libvirt)
- Used for **new distros** (e.g. trixie, bookworm)
- VMs are pre-configured with SSH access
- Network: `192.168.122.x` range, hostnames in `/etc/hosts`
- SSH key: `~/.ssh/id_rsa_vm`
- Current VMs: buster, bullseye, bookworm, trixie, focal, etc.
- New VMs: create with virsh, add MAC→IP mapping in `virsh net-edit default`

### AWS EC2 (for new architectures)
- Used for **new architectures** (e.g. aarch64/ARM) — native builds, not cross-compilation
- Use ARM instances like `c7g.8xlarge` for native ARM builds
- See README for setup steps

### Resource requirements
- 4GB free disk (non-debug), 20GB+ for debug
- 16GB RAM minimum
- As many CPUs as possible (build scales well)


## Compiler source location (this machine)

The actual compiler source being packaged lives at:
```
/home/src/srcL/mulle-clang-21.1.8/mulle-clang-project/
```
Build output goes into:
```
/home/src/srcL/mulle-clang-21.1.8/build/
```
The `package-build` script normally downloads source from GitHub, but the local
source can be used directly for development/testing.


## Runtime compatibility check (manual)

Before releasing, verify `COMPATIBLE_MULLE_OBJC_RUNTIME_LOAD_VERSION` in:
```
mulle-clang-project/clang/lib/CodeGen/CGObjCMulleRuntime.cpp
```
Cross-reference against the runtime source at:
```
/home/src/srcO/mulle-objc/mulle-objc-runtime
```
This is a manual inspection step.


## Release process (mulle-release-commander)

The release steps live in `../mulle-cc-<VERSION>.mrc/`. Each step is a pair of
`i-<name>.md` (instructions) + `s-<name>.sh` (script), tracked in `db.json`.

### db.json structure

```json
{
  "files": [
    {
      "title": "...",
      "filename": "i-<name>.md",
      "scriptFilename": "s-<name>.sh",
      "uuid": "...",
      "status": "todo|done|failed|in-progress",
      ...
    }
  ],
  "environmentVariables": {
    "MULLE_CLANG_PROJECT_TAG": "21.1.8.1",
    "PWD": "/home/src/srcO/mulle-cc",
    "MULLE_OBJC_RUNTIME_LOAD_VERSION": "18"
  }
}
```

`environmentVariables` are injected as env vars when running each step's script.
`PWD` is the working directory scripts run from.

### How the agent runs a release

For a new release, the agent:

1. **Copies** the previous `.mrc` dir to `mulle-cc-<NEW_VERSION>.mrc`
2. **Cleans** it: deletes all `l-*.log` files, resets all `status` fields in `db.json` to `"todo"`
3. **Updates** `environmentVariables` in `db.json` with the new version/settings
4. **Edits** scripts or docs as needed (new distro, new arch, changed process, etc.)
5. **For each step** in `db.json` order:
   - **Read `i-*.md` first** - this is the source of truth for what needs to happen
   - The `s-*.sh` script is a helper, not the definition - use it, adapt it, or go beyond it as the instructions require
   - Run the work (script or manual actions) with env vars from `environmentVariables` injected, from `PWD`
   - Capture output to `l-<name>-<uuid-prefix>-<timestamp>.log`
   - Update `db.json` status to `"done"` or `"failed"`
   - **Stop on first failure**

### Linux release steps (in order):
1. **Release mulle-clang compiler** - verify version + runtime compat
2. **Create debian VM if needed** - virsh setup
3. **Prepare mulle-clang-cpack** - update version strings in scripts, commit, push, create branch `mulle/<MAJOR.MINOR.PATCH>`, push to GitHub
4. **Create mulle-clang-project x86_64 debian package** - runs `create-deb "trixie"` (and optionally bookworm)
5. **Upload debian package to release** - `gh release upload` to `mulle-cc/mulle-clang-project`
6. **Edit github-ci projects** - update hardcoded compiler URLs in `mulle-cc/github-ci`, `mulle-objc/github-ci`, `MulleFoundation/github-ci`
7. **Edit developer pages** - update README + Dockerfiles in developer repos (mulle-objc-developer, mulle-foundation-developer, etc.)

macOS/Homebrew steps exist in the `.mrc` but are **out of scope for now** - Linux only.


## Prerequisites on build VM

Install via `mulle-clang-project/clang/bin/install-prerequisites --no-lldb`:
- cmake, ninja-build, git, wget
- clang (preferred over gcc for building LLVM)
- rpm (for RPM package generation)

```bash
sudo apt-get install rpm clang
```


## Package naming convention

```
mulle-clang-<VERSION><RC>-<DIST>-<ARCH>.deb
# e.g. mulle-clang-21.1.8.1-trixie-amd64.deb
```


## GitHub repo

`https://github.com/mulle-cc/mulle-clang-cpack`

Branch per release: `mulle/<MAJOR.MINOR.PATCH>` (e.g. `mulle/21.1.8`)


## Notes on scope and process

- **No automated testing** - releases are done manually via chat
- **No cross-compilation** - each arch requires a native build machine
- **New distro** → create a new local virsh VM
- **New arch** → spin up an AWS EC2 instance (e.g. `c7g.8xlarge` for aarch64)
- The `install-prerequisites` script in `mulle-clang-project/clang/bin/` handles
  build dependency setup on a fresh VM
- The release process is driven by `mulle-release-commander` steps in
  `../mulle-cc-0.27.0.mrc/` — each step has an `i-*.md` (instructions) and
  `s-*.sh` (script) pair; status is tracked in `db.json`
