# buildroot-docker-v86

Build a **Docker-enabled Buildroot Linux** kernel (`bzImage`) for the
**v86 x86-in-browser emulator**, using free GitHub Actions cloud CI.

Docker 28.3.3 (`docker`, `docker-init`), containerd, runc, libseccomp,
iptables, and CA certificates — all embedded in a single bootable kernel
(initramfs), running as 32-bit i686 in v86.

---

## What you need to do (once, ~5 minutes)

1. **Create a GitHub repo** and push this folder to it
   (e.g. `github.com/<you>/buildroot-docker-v86`).
2. Open the repo → **Actions** tab → **"Build Docker v86"** workflow
   (it's `on: workflow_dispatch`, so it only runs when you click **Run workflow**).
3. Click **Run workflow** → the build starts. It takes **~1.5–2.5 hours**
   (buildroot compiles a full cross toolchain + glibc + Go + kernel from scratch).
4. When green, the **artifact** `docker-v86-images` contains the built `bzImage`.

---

## Using the kernel in this v86 generator

Upload the built `bzImage` (e.g. via the upload tool or directly to
`https://user.uploads.dev`), then set it as the `bzimage` URL in the generator's
`index.html`:

```js
var FILES = {
  // ... other files unchanged ...
  bzimage: "https://user.uploads.dev/file/<your-uploaded-bzImage-url>",
};
```

No other changes needed — v86 loads `bzimage` directly.

---

## What the workflow does

| Step | Detail |
|------|--------|
| Download buildroot | `buildroot-2024.02.5.tar.gz` (version browser-buildroot pins) |
| Merge config | `standard/.config` from browser-buildroot (glibc, kernel 6.6.37, i686 pentium3) |
| Add Docker | `BR2_PACKAGE_DOCKER_ENGINE=y` + containerd/runc/docker-cli/libseccomp/CA-certs |
| Bundle rootfs | `BR2_TARGET_ROOTFS_INITRAMFS=y` — embeds the cpio rootfs **into** the bzImage (v86 can't mount a separate rootfs) |
| Kernel options | docker-kernel.config merged into the kernel build |
| Boot script | `S99docker` in the rootfs overlay auto-starts containerd + dockerd (vfs storage driver, works on ramfs) |
| Artifact | uploads `bzImage`, `rootfs.cpio.lz4`, kernel `vmlinux` |

---

## Notes / troubleshooting

- **Storage driver:** dockerd is started with `--storage-driver=vfs`. vfs is the
  only overlay that works on an in-RAM rootfs; it's slow but functional for
  testing `docker run`.
- **cgroups:** buildroot 2024.02.5's `docker-engine` auto-selects
  `cgroupfs-mount` (the cgroup-v1 package). It's kept; the `S99docker` script
  tries cgroup2 first, falls back to cgroup1.
- **Size:** the kernel+initramfs will be **~60–90 MB** (Go binaries are fat).
  Make sure your upload target accepts that size.
- **Build time:** first run builds the toolchain; GitHub Actions caches nothing
  here, so re-runs are just as slow. For a faster local build, mirror the
  steps and use `make -j$(nproc)`.

## Files

```
.github/workflows/build.yml   # the CI pipeline
standard/.config              # browser-buildroot "standard" config (glibc)
standard/board/browser_linux/ # board files: kernel config, overlay, S99docker
buildroot-docker.config       # the exact BR2_* Docker fragment (reference)
```

---

Credit: base config from [Darin755/browser-buildroot](https://github.com/Darin755/browser-buildroot)
(Buildroot 2024.02.5, kernel 6.6.37).
