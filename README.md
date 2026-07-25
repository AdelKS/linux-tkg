# Linux-tkg

This repository provides scripts to automatically download, patch and compile any version ≥ 5.4 of the Linux Kernel.

`linux-tkg` also ships with a selection of patches that can improve the desktop/gaming experience: most can be toggled by editing the [customization.cfg](./customization.cfg) file, or by simply following the interactive install script.

- [Linux-tkg](#linux-tkg)
  - [Important information](#important-information)
  - [Customize linux-tkg](#customize-linux-tkg)
    - [CPU task schedulers](#cpu-task-schedulers)
      - [Runtime scheduler swap: sched-ext](#runtime-scheduler-swap-sched-ext)
      - [Build-time scheduler swap](#build-time-scheduler-swap)
    - [Bring your own patches](#bring-your-own-patches)
    - [Customize kernel config](#customize-kernel-config)
  - [Install procedure](#install-procedure)
    - [Arch \& derivatives](#arch--derivatives)
    - [DEB and RPM based distributions](#deb-and-rpm-based-distributions)
    - [Generic install](#generic-install)
    - [Gentoo](#gentoo)

## Important information

- **Non-pacman distros support can be considered experimental. You're invited to report issues you might encounter with it.**
- **If your distro isn't using systemd, please set _configfile="running-kernel" in customization.cfg or you might end up with a non-bootable kernel**
- Building recent linux kernels with GCC will require ~20-25GB of disk space. Using llvm/clang, LTO, ccache and/or enabling more drivers in the defconfig will push that requirement higher, so make sure you have enough free space on the volume you're using to build.
- Nvidia drivers might need to be patched to cleanly build/work on latest kernels.
  [Frogging-Family nvidia-all](https://github.com/Frogging-Family/nvidia-all) can help you with that.

## Customization options

linux-tkg works interactively out of the box, but can also be fully configured for repeatable builds.

Configuration priority, from highest to lowest:

1. Environment variables
2. An external config file set through `_EXT_CONFIG_PATH` (defaults to `~/.config/frogminer/linux-tkg.cfg`)
3. The repository's [`customization.cfg`](./customization.cfg)

The external config only needs to contain values you want to override. Options with an interactive selector are prompted when left empty.

> [!TIP]
> **🐸 Major tweaks — Build your kernel, your way.**
>
> These options have the biggest impact on how your kernel is built and behaves.

- `_cpusched` selects EEVDF/CFS, BORE, PDS/BMQ, or legacy MuQSS/Undead PDS as the [build-time CPU scheduler](#build-time-scheduler-swap) (prompted by default).
- `_compiler` selects GCC (default) or [LLVM](https://docs.kernel.org/kbuild/llvm.html).
- `_lto_mode` enables Thin or Full LTO for LLVM builds (disabled by default).
- `_compileroptlevel` selects `-O2` (default), `-O3` for potentially higher performance, or `-Os` for a smaller kernel.
- `_processor_opt` tunes the build for a specific CPU microarchitecture (prompted by default).
- `_glitched_base`, `_zenify`, `_clear_patches`, and `_misc_adds` control the main optional patchsets (enabled by default), mixing selected [Zen](https://github.com/zen-kernel/zen-kernel) and [Clear Linux](https://github.com/clearlinux-pkgs/linux) tweaks with linux-tkg's signature Glitched Base flavor.
- `_default_cpu_gov` selects the default [CPU frequency governor](https://docs.kernel.org/admin-guide/pm/cpufreq.html) (`ondemand` by default, or `schedutil` above 32 threads).
- `_aggressive_ondemand` enables linux-tkg's tuned `ondemand` governor for supported schedulers (enabled by default).
- `_timer_freq` sets the kernel timer frequency (prompted with a scheduler-specific default).
- `_custom_commandline` embeds `intel_pstate=passive` by default.
- `_preempt_rt` enables [real-time preemption](https://docs.kernel.org/core-api/real-time/) and limits the available CPU schedulers (disabled by default).
- `_kernel_on_diet` uses linux-tkg's curated module list, while `_modprobeddb` uses [Modprobed-db](https://github.com/graysky2/modprobed-db) to keep only detected modules. Both reduce build time and kernel size, but can omit required modules.
- `_vanilla` provides a cleaner upstream-style base by skipping extra linux-tkg patchsets and using the upstream CPU scheduler (disabled by default).
- [NTsync](https://docs.kernel.org/6.14/userspace-api/ntsync.html) can improve gaming performance through faster Windows synchronization and pairs especially well with [wine-tkg](https://github.com/Frogging-Family/wine-tkg-git). It is built into Linux 6.14+; `_ntsync` only backports it to 6.8–6.13. Older Fsync variants remain for legacy setups, while Futex2 is deprecated.

> [!NOTE]
> **Built-in defaults**
>
> Some tweaks are grouped instead of having individual toggles, including selected memory-management, scheduling, CFS/EEVDF, CAKE, and a raised `vm.max_map_count=2147483642` default. Many of their tunables can still be adjusted at runtime through `sysctl` or subsystem-specific configuration; see [AdelKS's Linux Gaming Guide](https://github.com/AdelKS/LinuxGamingGuide) for more gaming tips and tuning.
>
> On Arch Linux, `/usr/lib/sysctl.d/10-arch.conf` overrides the kernel's `vm.max_map_count` default with `1048576`; see the [ArchWiki gaming guide](https://wiki.archlinux.org/title/Gaming#Increase_vm.max_map_count) to raise it persistently.
>
> Use `_vanilla=true` to stay close to upstream, which is useful for isolating regressions. The build can still be tailored with `_configfile`, `*.myfrag` config fragments, and user patches.

See [`customization.cfg`](./customization.cfg) for all options, defaults, and caveats.

### CPU task schedulers

A CPU task scheduler is the algorithm that decides which task (app, game, background service... etc) one runs when, for how long, on which core, and make it take turns with which other tasks. Those decisions can have an impact on throughput (how many things get done globally per unit of time), and latency (how long tasks have to wait before running again). Gaming is generally more sensitive to latency and some schedulers can be more more adapted for that.

#### Runtime scheduler swap: sched-ext

Starting kernel ≥ 6.12, it's possible to switch CPU schedulers at runtime while keeping the kernel's built-in default scheduler one as fallback/backup,
using [sched-ext](https://github.com/sched-ext/scx). `sched-ext` offers various schedulers, `LAVD` is geared towards gaming workloads and used by Steam's
Deck handheld console.

Notes:

- Arch users get scx schedulers from the `scx-scheds` package or on the [AUR](https://aur.archlinux.org/packages/scx-scheds-git) thanks to @sirlucjan.
- For persistence of the chosen runtime scheduler upon reboots:
  - set scheduler in `/etc/default/scx`
  - enable the `scx` service: `systemctl enable scx`.

#### Build-time scheduler swap

[EEVDF](https://lwn.net/Articles/925371/) is the only CPU scheduler available in the upstream "vanilla" kernel sources ≥ 6.6, and [CFS](https://en.wikipedia.org/wiki/Completely_Fair_Scheduler) for earlier kernel versions.

Alternative build-time default schedulers are optionally available in linux-tkg:

- BORE (Burst-Oriented Response Enhancer) by Masahito Suzuki - CFS/EEVDF based : [code repository](https://github.com/firelzrd/bore-scheduler)
- Project C / PDS & BMQ by Alfred Chen: [blog](http://cchalpha.blogspot.com/ ), [code repository](https://gitlab.com/alfredchen/projectc)
- MuQSS by Con Kolivas : [blog](http://ck-hack.blogspot.com/), [code repository](https://github.com/ckolivas/linux)
- Undead PDS : TkG's port of the pre-Project C "PDS-mq" scheduler by Alfred Chen. While PDS-mq got dropped with kernel 5.1 in favor of its BMQ evolution/rework, it wasn't on par with PDS-mq in gaming. "U" PDS still performed better in some cases than other schedulers, so it's been kept undead for a while.

These alternative schedulers may offer a better performance/latency ratio in some scenarios. The availability of each scheduler depends on the chosen Kernel version: the script will display what's available on a per-version basis.

### Bring your own patches

To apply your own patches with `linux-tkg`:

- Create a `linuxXY-tkg-userpatches` folder at the root of the clone
  - where `X` and `Y` specify the kernel version the patches applies to.
    - Examples: `linux65-tkg-userpatches`, `linux618-tkg-userpatches`
- Set `_user_patches=true` in your customization file.
- Drop your patches within that folder the `.mypatch` extension.

The install script will then find them, and ask if you want to apply each patch.

### Customize kernel config

`linux-tkg` starts by default on [Arch's full defconfig](https://gitlab.archlinux.org/archlinux/packaging/packages/linux/-/blob/main/config.x86_64),
tweaks some CONFIG options in it (either always or based on input from `customization.cfg`), then proceeds to build the kernel.

To modify kernel config options:
- The baseline kernel defconfic can be overriden with the `_configfile` [customization](#customization-options)
  - The interactive script will then proceed to modify some entries within, see bellow to eventually revert those changes.
- Select config options can be modified right before the kernel starts building
  - Through GUI/TUI to modify kernel CONFIG options.
    - The interactive script prompts the user for `make xconfig/menuconfig`, then prompts to save the eventual changes into "frag" files
      - Note: make sure to append the `.myfrag` extension to the name so the script can automatically reuse them.
  - Through "frag" files: defconfig "fragments" to override the defconfig file
    - These files must reside at the root of the `linux-tkg` clone and have a `.myfrag` extension.
    - The intractive script will prompt to apply them, one frag at a time.

## Install procedure

For all the supported linux distributions, `linux-tkg` has to be cloned with `git`. It is recommended to clone it only once and update it with `git pull`.
as it keeps a relatively big clone of the kernel's sources within (in the `linux-src-git` folder, created during the first build after a fresh clone).

### Arch & derivatives

```shell
git clone https://github.com/Frogging-Family/linux-tkg.git
cd linux-tkg
# Optional: edit the "customization.cfg" file
makepkg -si
```

The options selected at build-time are installed to `/usr/share/doc/$pkgbase/customization.cfg`, where `$pkgbase` is the package name.

**Note:** the `base-devel` package group is expected to be installed, see [here](https://wiki.archlinux.org/title/Makepkg) for more information.

### DEB and RPM based distributions

The interactive `install.sh` script will create, depending on the selected distro, `.deb` or `.rpm` packages, move them in the the subfolder `DEBS` or `RPMS` then prompts to install them with the distro's package manager.

- `.deb` packages: for Debian, Ubuntu...
- `.rpm` packages: Fedora, RHEL, SUSE...

```shell
git clone https://github.com/Frogging-Family/linux-tkg.git
cd linux-tkg
# Optional: edit the "customization.cfg" file
./install.sh install
```

Uninstalling custom kernels installed through the script has to be done
manually. `install.sh` can can help out with some useful information:

```shell
cd path/to/linux-tkg
./install.sh uninstall-help
```

### Generic install

The interactive `install.sh` script can be used to perform a "Generic" install by choosing `Generic` when prompted or in the `_distro` [customization option](#customization-options).

```shell
git clone https://github.com/Frogging-Family/linux-tkg.git
cd linux-tkg
# Optional: edit the "customization.cfg" file
./install.sh install
```

The script will compile the kernel then prompt before doing the following:

```shell
sudo cp -R . /usr/src/linux-tkg-${kernel_flavor}
cd /usr/src/linux-tkg-${kernel_flavor}
sudo make modules_install
sudo make install
```

**Notes:**

- All the needed dependencies to patch, configure, compile or install the kernel are expected to be installed by the user beforehand.
- If you only want the script to patch the sources in `linux-src-git`, you can use `./install.sh config`
- `${kernel_flavor}` is a default naming scheme but can be customized with the `_kernel_localversion` [customization option](#customization-options).
- `_libunwind_replace` [customization option](#customization-options) can replace `libunwind` with `llvm-libunwind`.
- The script uses Arch's `.config` file as a base. A custom one can be provided in the `_configfile` [customization option](#customization-options).
- The installed files will not be tracked by your package manager and uninstalling requires manual intervention.
  `./install.sh uninstall-help` can help with useful information if your install procedure follows the `Generic` approach.
- Installing the kernel with `make install` calls `/sbin/installkernel` (see [here](https://docs.kernel.org/kbuild/kbuild.html#installkernel))
  to put the kernel at the right place and trigger an initramfs (and UKI) generation, check your distro's documentation on how to configure it to your needs
  - [arch](https://wiki.archlinux.org/title/Kernel-install)
  - [gentoo](https://wiki.gentoo.org/wiki/Installkernel)

### Gentoo

The interactive `install.sh` script supports Gentoo by following the same procedure as `Generic`, with minor additions

1. Applies few Gentoo patches
   - `https://dev.gentoo.org/~mpagano/genpatches/trunk/$kver/4567_distro-Gentoo-Kconfig.patch`
   - `https://dev.gentoo.org/~mpagano/genpatches/trunk/$kver/3000_Support-printing-firmware-info.patch`
2. If using an init other than `systemd`, set `_gentoo_init='script'` in `customization.cfg`
   - Note: for a minimal defconfig, it is then advised to provide your own defconfig through `_configfile=` in `customization.cfg`: the default Arch defconfig is otherwise used with `systemd` related config options enabled regardless.
3. Symlinks the newly installed `/usr/src/linux-tkg-${kernel_flavor}` src folder to `/usr/src/linux`
4. Offers to do a `emerge @module-rebuild` for convenience

```shell
git clone https://github.com/Frogging-Family/linux-tkg.git
cd linux-tkg
# Optional: edit the "customization.cfg" file
./install.sh install
```
