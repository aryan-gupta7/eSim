# eSim 2.5 --- Ubuntu 25.04 Installation Issues Report

**Submitted by:** Aryan Gupta  
**Repository:** https://github.com/aryan-gupta7/eSim (branch: origin/installers)  
**Original upstream:** https://github.com/FOSSEE/eSim (branch: installers)  
**eSim version:** 2.5 (downloaded from esim.fossee.in/downloads)  
**Test environment:** Ubuntu 25.04 (Plucky Puffin) on VMware Workstation (Windows host)

---

## Background

eSim 2.5 was downloaded as a ZIP from the official FOSSEE website. The installation is driven by a top-level `install-eSim.sh` script which detects the Ubuntu version and routes to a version-specific script inside `install-eSim-scripts/`. The upstream repository only ships scripts up to Ubuntu 24.04, so running this on Ubuntu 25.04 immediately hits a wall.

To work through the issues, relevant files from the downloaded ZIP (`library/`, `nghdl.zip`, etc.) were copied into the forked GitHub repository's installer branch since they were required during installation but were absent from the upstream repo.

---

## Issue 1 -- No install script for Ubuntu 25.04

### What happened

Running `./install-eSim.sh --install` on Ubuntu 25.04 exited immediately:

```
Unsupported Ubuntu version: 25.04
```

### Root cause

The version dispatch logic in `install-eSim.sh` had a `case` block with explicit entries for 22.04, 23.04, and 24.04. Ubuntu 25.04 fell through to the default `*` case and exited.

### Fix

Added a `25.04` case to the dispatch function and created a new `install-eSim-25.04.sh` script (derived from `install-eSim-24.04.sh`) that incorporates all the other fixes described in this report. The updated dispatch function in `install-eSim.sh`:

```bash
run_version_script() {
    SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/install-eSim-scripts"

    case $VERSION_ID in
        "22.04")
            if [[ "$FULL_VERSION" == "22.04.4" ]]; then
                SCRIPT="$SCRIPT_DIR/install-eSim-22.04.sh"
            else
                SCRIPT="$SCRIPT_DIR/install-eSim-23.04.sh"
            fi
            ;;
        "23.04")
            SCRIPT="$SCRIPT_DIR/install-eSim-23.04.sh"
            ;;
        "24.04")
            SCRIPT="$SCRIPT_DIR/install-eSim-24.04.sh"
            ;;
        "25.04")
            SCRIPT="$SCRIPT_DIR/install-eSim-25.04.sh"
            ;;
        *)
            echo "Unsupported Ubuntu version: $VERSION_ID ($FULL_VERSION)"
            exit 1
            ;;
    esac

    if [[ -f "$SCRIPT" ]]; then
        echo "Running script: $SCRIPT $ARGUMENT"
        bash "$SCRIPT" "$ARGUMENT"
    else
        echo "Installation script not found: $SCRIPT"
        exit 1
    fi
}
```

This was a straightforward fix once the dispatch pattern was understood -- adding the case entry and creating the new script file.

---

## Issue 2 -- KiCad fails to install due to missing libgit2-1.8

### What happened

After getting past the version check, KiCad installation failed with:

```
The following packages have unmet dependencies:
 kicad : Depends: libgit2-1.8 (>= 1.8.0) but it is not installable
E: Unable to correct problems, you have held broken packages.
E: Unable to satisfy dependencies.
   kicad:amd64=8.0.9-0~ubuntu25.04.1 Depends libgit2-1.8 (>= 1.8.0)
   but none of the choices are installable: [no choices]
```

### Root cause

Ubuntu 25.04 ships libgit2 version 1.9 as the default. The KiCad 8.0 PPA package built for Ubuntu 25.04 was compiled against libgit2-1.8 and declares a hard dependency on that specific soname package. The `libgit2-1.8` package was built for plucky (confirmed on Launchpad at version `1.8.4+ds-3ubuntu2`) but it lives in the proposed pocket, which is not enabled by default, so `apt` cannot find it.

This took the most time to diagnose. The apt error message is clear enough about the missing dependency, but understanding *why* it was missing required checking the Ubuntu package search (packages.ubuntu.com), which confirmed `libgit2-1.9` is the default in plucky, and then cross-referencing Launchpad to confirm the `libgit2-1.8` package existed but was sitting in proposed. A plain `apt-get install libgit2-1.8` does not work because the package is not in any enabled source.

**References checked:**
- packages.ubuntu.com confirmed libgit2-1.9 is the plucky default, libgit2-1.8 absent from release pocket
- launchpad.net/ubuntu/+source/libgit2/1.8.4+ds-3ubuntu2 confirmed the 1.8 build exists for plucky in proposed

### Fix

Since the package is not reachable through normal apt, the `.deb` is fetched directly from the Ubuntu primary archive and installed with `dpkg` before the KiCad `apt-get install` runs. `mktemp` is used for the temp file to avoid path collisions:

```bash
if [[ "$ubuntu_version" == "25.04" ]]; then
    echo "Installing libgit2-1.8 compatibility shim for Ubuntu 25.04..."
    tmp_deb=$(mktemp /tmp/libgit2-1.8.XXXXXX.deb)
    wget -q "https://launchpad.net/ubuntu/+archive/primary/+files/libgit2-1.8_1.8.4+ds-3ubuntu2_amd64.deb" \
        -O "$tmp_deb"
    sudo dpkg -i "$tmp_deb"
    rm -f "$tmp_deb"
fi

sudo apt-get install -y --no-install-recommends kicad kicad-footprints \
    kicad-libraries kicad-symbols kicad-templates
```

---

## Issue 3 -- KiCad symbol library copied to wrong config directory

### What happened

KiCad installed successfully, but eSim's custom symbols were not visible in the schematic editor. The `copyKicadLibrary` function in the install script was silently copying to the wrong directory.

### Root cause

The script had `~/.config/kicad/6.0` hardcoded throughout the `copyKicadLibrary` function. The Ubuntu 25.04 setup installs KiCad 8.0, which stores its configuration in `~/.config/kicad/8.0`. No error was thrown, the copy just went to a directory KiCad 8.0 never reads.

This was simple to spot once KiCad launched without any eSim symbols and the directory structure was checked manually.

### Fix

Updated all references from `6.0` to `8.0` in `copyKicadLibrary`:

```bash
# Before:
if [ -d ~/.config/kicad/6.0 ]; then
    ...
    mkdir -p ~/.config/kicad/6.0
fi
cp kicadLibrary/template/sym-lib-table ~/.config/kicad/6.0/

# After:
if [ -d ~/.config/kicad/8.0 ]; then
    ...
    mkdir -p ~/.config/kicad/8.0
fi
cp kicadLibrary/template/sym-lib-table ~/.config/kicad/8.0/
```

---

## Issue 4 -- libcanberra-gtk-module removed from Ubuntu 25.04

### What happened

The NGHDL dependency installer failed with:

```
Package libcanberra-gtk-module is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source
Error: Package 'libcanberra-gtk-module' has no installation candidate
Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

### Root cause

`libcanberra-gtk-module` is a GTK2 sound event module. It was dropped entirely from Ubuntu 25.04, confirmed by searching packages.ubuntu.com, which shows the package only up to noble (24.04). Ubuntu 25.04 uses GNOME 48 on Wayland and has deprecated GTK2 support. The GTK3 variant `libcanberra-gtk3-module` is still present and available.

The error message and package search made this quick to resolve.

### Fix

Made the GTK2 canberra install conditional on the Ubuntu version:

```bash
ubuntu_version=$(lsb_release -rs)
if [[ "$ubuntu_version" != "25.04" ]]; then
    sudo apt install -y libcanberra-gtk-module libcanberra-gtk3-module
else
    echo "Skipping libcanberra-gtk-module (dropped in Ubuntu 25.04)"
    sudo apt install -y libcanberra-gtk3-module
fi
```

---

## Issue 5 -- GHDL 4.1.0 configure script does not recognise LLVM 20

### What happened

The GHDL build (inside `nghdl.zip`) failed at the configure step:

```
gcc (Ubuntu 14.2.0-19ubuntu2) 14.2.0
Use full IEEE library
Build machine is: x86_64-linux-gnu
Unhandled version llvm 20.1.2
Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

### Root cause

GHDL's `configure` script has a hardcoded `case` statement that enumerates each supported LLVM major version. When it encounters an unlisted version it falls through to the default and aborts. Ubuntu 25.04 ships LLVM 20 as the default, which GHDL 4.1.0 does not list.

This is a recurring issue with GHDL, it has happened with every new LLVM major release. The GHDL GitHub issue tracker documents the same error with LLVM 11.1 (NixOS/nixpkgs issue #177748), LLVM 10 (PR #1192), LLVM 9 (Debian bug #952324), and so on. The fix each time is either to patch the configure script's case statement or to use an older LLVM version that is already listed.

Patching the configure script in the upstream GHDL tarball inside the zip would work, but it is fragile. The cleaner approach is to install LLVM 18 (the most recent version GHDL 4.1.0 supports) alongside the system LLVM 20, Ubuntu supports multiple LLVM versions coexisting via versioned packages, and point the configure call at it explicitly.

This took the most reasoning to arrive at a stable fix because the options were: patch the configure script, pin LLVM system-wide, or install a parallel LLVM version. The parallel install approach was chosen since it does not affect system state.

**References checked:**
- NixOS/nixpkgs issue #177748, identical error with LLVM 11.1, confirmed the root cause pattern
- ghdl/ghdl issue #858, PR #1192, LLVM version whitelist mechanism and how past versions were added
- ghdl/ghdl issue #1906, upstream discussion of this recurring maintenance problem

### Fix

In `installDependency`, install LLVM 18 for Ubuntu 25.04 instead of the default LLVM:

```bash
if [[ "$ubuntu_version" == "25.04" ]]; then
    sudo apt install -y llvm-18 llvm-18-dev clang-18
else
    sudo apt remove -y llvm llvm-dev
    sudo apt install -y llvm llvm-dev
    sudo apt install -y clang
fi
```

In `installGHDL`, point the configure script at `llvm-config-18`:

```bash
if [[ "$ubuntu_version" == "25.04" ]]; then
    ./configure --with-llvm-config=/usr/bin/llvm-config-18
else
    ./configure --with-llvm-config=/usr/bin/llvm-config
fi
```

---

## Files Changed

| File | Change |
|------|--------|
| `install-eSim.sh` | Added 25.04 case to version dispatch function |
| `install-eSim-scripts/install-eSim-25.04.sh` | New file, copy of 24.04 script with Issues 2 and 3 fixed |
| `nghdl/install-nghdl-scripts/install-nghdl-25.04.sh` | New file, copy of 24.04 script with Issues 4 and 5 fixed |
| `src`, `library/`, `nghdl.zip` | Copied from release ZIP into repo (required for install, absent upstream) |

---

## Notes

The nghdl.zip also had its own version dispatch that only covered up to 24.04 the same Issue 1 pattern appeared there, and was fixed the same way by adding a 25.04 case pointing to the new `install-nghdl-25.04.sh`.

All five issues were resolved and eSim 2.5 installed successfully on Ubuntu 25.04.