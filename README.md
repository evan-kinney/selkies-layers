# selkies-layers

Prebuilt, patched components for Selkies base images, forked from [linuxserver/selkies-layers](https://github.com/linuxserver/selkies-layers) with an added `multiuser` layer for multi-user Active Directory sign-on.

Images live at `ghcr.io/evan-kinney/selkies-layers`. This fork is not supported by LinuxServer.io.

## Layers

| Layer | Contents | Distros |
| --- | --- | --- |
| `labwc` | patched labwc 0.9.7 (`/usr/bin/labwc`), plus pinned wlroots 0.19.3 (`/usr/lib/<libdir>/libwlroots-0.19.so*`) on distros that lack a packaged wlroots0.19 (arch, debian, ubuntu, kali) | alpine, arch, debian, ubuntu, fedora, kali |
| `wtype` | linuxserver waylandtyper build (`/usr/bin/wtype`) | alpine, arch, debian, ubuntu, fedora, kali |
| `kwin` | patched `libkwin.so.6.*` and the kwin `screencast.so` plugin, built from the distro's own kwin source package so it matches the packaged version to support dual monitors | alpine, arch, fedora, ubuntu, kali |
| `xvfb` | XLibre Xvfb (`/usr/bin/Xvfb`) from the `xlibre-xserver-25.2.2` tag with glamor + DRI3 (`-glamor -dri /dev/dri/renderD###`), patched so the screen pixmap lives on the GPU and the framebuffer follows RandR resizes (no `-screen` needed); to start with 4 CRTCs whose spare outputs (`screen_1`…) stay disconnected until a client sets their `Connected` output property, so Selkies adds a display as a real monitor with a CRTC of its own; to synchronize DRI3 presents explicitly through DRI3 1.4 timeline syncobjs where the kernel and the GPU's driver support them, so frames from the NVIDIA driver reach the screen in order; and to run the fake vblank at the rate a client sets on the root window's `_FAKE_SCREEN_MILLIHZ` or `_FAKE_SCREEN_FPS` (pixelflux publishes its capture rate on both), never below the rate the server started at unless at most 0.1% below it, as 59.94 Hz is under 60; and to offer the `SELKIES-SEMAPHORE` extension where glamor has `GL_EXT_semaphore_fd`, which signals a GPU semaphore a client shares once the server's rendering before it completes, so an encoder on the same GPU waits on the GPU rather than the CPU for the capture's blit; and to answer `XFree86-VidModeExtension` with each screen's size at its vblank rate, which Mesa's `glXGetMscRateOML` reads, so Chromium on ANGLE's GL backend stops logging a failed rate query twice a second | alpine, arch, debian, ubuntu, fedora, kali |
| `multiuser` | `MULTI_USER=true` mode: every Active Directory user who signs in through the browser gets their own concurrent desktop as their own Linux account. The session broker (`/usr/local/bin/selkies-broker`), the per-user session launcher (`/usr/local/bin/selkies-user-session`), the `init-multiuser`, `svc-sssd` and `svc-selkies-broker` s6 services, the nginx template (`/defaults/default-multiuser.conf`) and the PAM service (`/etc/pam.d/nginx`). Distro neutral, see [Multi-user layer](#multi-user-layer) for what the base image must provide | arch, debian, ubuntu, fedora, kali |

Every file inside a layer image sits at its final path, so downstream usage is
a single copy:

```dockerfile
# Dockerfile (amd64)
COPY --from=ghcr.io/evan-kinney/selkies-layers:debiantrixie-labwc / /
COPY --from=ghcr.io/evan-kinney/selkies-layers:debiantrixie-wtype / /

# Dockerfile.aarch64
COPY --from=ghcr.io/evan-kinney/selkies-layers:arm64v8-debiantrixie-labwc / /
COPY --from=ghcr.io/evan-kinney/selkies-layers:arm64v8-debiantrixie-wtype / /
```

## Tags

Per layer three tags are pushed:

```
ghcr.io/evan-kinney/selkies-layers:amd64-alpine324-labwc
ghcr.io/evan-kinney/selkies-layers:arm64v8-alpine324-labwc
ghcr.io/evan-kinney/selkies-layers:alpine324-labwc        # multi-arch manifest
```

Current bases:

| distro | version | tag base |
| --- | --- | --- |
| alpine | 3.24 | `alpine324` |
| arch | arch | `arch` |
| debian | trixie | `debiantrixie` |
| ubuntu | resolute | `ubunturesolute` |
| fedora | 44 | `fedora44` |
| kali | kali | `kali` |

## Building locally

```
./build.sh alpine labwc                # native arch, loads into local docker
./build.sh fedora kwin arm64           # cross build via qemu
./build.sh debian wtype amd64 --push   # push the arch tag
python3 scripts/matrix.py              # print the full matrix as JSON
```

## Multi-user layer

The `multiuser` layer carries everything distro neutral about `MULTI_USER=true` mode. Users,
configuration and the isolation model are documented in
[docker-baseimage-selkies](https://github.com/evan-kinney/docker-baseimage-selkies#multi-user-with-active-directory).
A base image that copies the layer must still provide:

- **Packages** for SSSD with the LDAP provider, its NSS and PAM modules, and nginx's PAM auth module
  (loaded, so `auth_pam` works in a server block):

  | distro | packages |
  | --- | --- |
  | debian, ubuntu, kali | `sssd-ldap sssd-tools libnss-sss libpam-sss libnginx-mod-http-auth-pam` |
  | fedora | `sssd-ldap sssd-tools sssd-client`, the nginx PAM module is not packaged and has to be built against the distro nginx |
  | arch | `sssd`, the nginx PAM module is only in the AUR (`nginx-mod-auth-pam`) and has to be built |

- **Hooks** in its own services: `svc-xorg`, `svc-de`, `svc-selkies`, `svc-pulseaudio`, `svc-watchdog`
  and `svc-xsettingsd` idle (`exec sleep infinity`) when `MULTI_USER=true`, and `init-nginx` installs
  `/defaults/default-multiuser.conf` instead of the single user config and skips its htpasswd setup.
- glibc `getent`, util-linux `setpriv`, `gpasswd`, `sudo` and Selkies 2.1.0 or newer (`python3 -m selkies.session`),
  all present in the Selkies base images.

The nginx worker group, which alone may reach the session sockets, is read from `nginx.conf`
(`www-data`, `nginx` or `http` depending on the distro) or set with `MULTI_USER_NGINX_GROUP`.

Alpine is not built: musl has no NSS, so directory users can never exist as Linux accounts there.
Only the Debian base has been tested end to end.

```dockerfile
# Dockerfile (amd64)
COPY --from=ghcr.io/evan-kinney/selkies-layers:amd64-debiantrixie-multiuser / /

# Dockerfile.aarch64
COPY --from=ghcr.io/evan-kinney/selkies-layers:arm64v8-debiantrixie-multiuser / /
```

GHCR packages start out private. After the first push, make `selkies-layers` public in the package
settings so other repositories and anonymous `docker build`s can pull it.
