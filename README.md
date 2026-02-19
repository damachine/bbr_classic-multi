# bbr_classic-multi

BBRv1 (TCP BBR Classic) as out-of-tree kernel module for kernels patched with the BBR3 patch (e.g. linux-tkg, linux-cachyos). Supports manual build, DKMS, and PKGBUILD installation.

On BBR3-patched kernels, BBRv1 is replaced by BBRv3. This module brings BBRv1 back as `bbr_classic`, so you can use both side by side.

See also: [CachyOS/linux-cachyos#706 (comment)][perf-ref] for benchmark results.

## Build (manual)

Requires kernel headers.

```sh
git clone https://github.com/damachine/bbr_classic-multi.git
cd bbr_classic-multi
make
sudo make load      # test
sudo make install   # permanent (no DKMS)
```

## Install via DKMS

Builds and installs.

```sh
git clone https://github.com/damachine/bbr_classic-multi.git
cd bbr_classic-multi
sudo make dkms-install
```

Uninstall:

```sh
sudo make dkms-uninstall
```

## Install via PKGBUILD (Arch / CachyOS)

Builds and installs a DKMS package locally.

```sh
git clone https://github.com/damachine/bbr_classic-multi.git
cd bbr_classic-multi
makepkg -si
```

## Load & activate

```sh
sudo modprobe tcp_bbr_classic
sudo sysctl -w net.core.default_qdisc=fq
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr_classic
```

Verify:

```sh
sysctl net.ipv4.tcp_congestion_control
```

## Persistent

```sh
sudo tee /etc/sysctl.d/99-bbr-classic.conf << EOF2
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr_classic
EOF2
sudo sysctl --system
```

## All make targets

```sh
make                       # download tcp_bbr.c and build the module
make clean                 # remove build directory and downloaded tcp_bbr.c
sudo make load             # load module for testing (insmod)
sudo make install          # install module permanently (no DKMS)
sudo make uninstall        # remove permanently installed module
sudo make dkms-install     # install via DKMS (auto-rebuild on kernel update)
sudo make dkms-uninstall   # remove DKMS installation
```

## How it works

`tcp_bbr.c` is the unmodified BBRv1 source from upstream Linux 6.19 — the version *before* the BBR3 patch is applied. During build, `Makefile` generates `tcp_bbr_classic.c` with the following patches:
- `"bbr"` → `"bbr_classic"` — module name visible to `sysctl` and `modprobe`
- `struct bbr` → `struct bbr_classic` — avoids symbol collision with the kernel's built-in BBRv3 module
- `register_btf_kfunc_id_set` call replaced with `ret = 0` — out-of-tree modules cannot resolve BTF kfunc IDs against vmlinux, so `insmod` fails with `-EINVAL` on kernels built with `CONFIG_DEBUG_INFO_BTF_MODULES=y` (CachyOS, TKG, Zen, Liquorix)
- `.min_tso_segs` line commented out — the BBR3 patch renames this field to `.tso_segs` with a different signature, so the original BBRv1 reference no longer compiles against BBR3-patched kernel headers

The BBRv1 algorithm itself is untouched.

## Testing performance

Requires `iperf3` and a server that allows selecting the congestion control algorithm.

```sh
# compare against BBRv3
iperf3 -c <server> -C bbr

# test BBR Classic
iperf3 -c <server> -C bbr_classic
```

## Credits

Original idea and approach by [cmspam/bbr_classic](https://github.com/cmspam/bbr_classic).  

## License

GPL-2.0

[perf-ref]: https://github.com/CachyOS/linux-cachyos/issues/706#issuecomment-3897122950
