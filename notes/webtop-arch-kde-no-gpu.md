# Running webtop `arch-kde` on a machine without a GPU

Source: [docs/images/docker-webtop.md](../docs/images/docker-webtop.md)

You don't need a GPU. Without one, the container renders the desktop and encodes the video stream on the CPU, and the docs say that "is fast enough for smooth sessions on modest hardware".

## Quick test run

Use the `arch-kde` tag instead of `latest`:

```bash
docker run --rm -it \
  --shm-size=1gb \
  -p 3001:3001 \
  lscr.io/linuxserver/webtop:arch-kde bash
```

Open **https://<host-ip>:3001**. It has to be `https`, and you'll need to accept the self-signed certificate warning. `ctrl+d` exits and removes the container. Leave out `--device /dev/dri` and the Nvidia flags, because they only apply when the machine has a GPU.

## Persistent setup (docker-compose)

```yaml
services:
  webtop:
    image: lscr.io/linuxserver/webtop:arch-kde
    container_name: webtop
    environment:
      - PUID=1000        # from `id your_user`
      - PGID=1000
      - TZ=Etc/UTC
      - CUSTOM_USER=admin      # optional basic auth
      - PASSWORD=changeme
    volumes:
      - /path/to/data:/config  # home dir, keeps settings across recreates
    ports:
      - 3001:3001
    shm_size: "1gb"
    restart: unless-stopped
```

Then run `docker compose up -d`.

## How graphics work without a GPU

There's no physical display. Everything happens in software inside the container:

1. **Virtual display:** the container runs its own display server (the `svc-xorg` service in the init graph). KDE draws into it as if it were a real screen.
2. **Software rendering:** with no `/dev/dri` device, OpenGL falls back to CPU rendering. That's normally Mesa's `llvmpipe`, though the docs don't name it.
3. **Encoding:** Selkies (`svc-selkies`) captures the screen, compresses it into a video stream on the CPU, and sends it to your browser.
4. **Browser decoding:** your browser decodes the stream with WebCodecs, which is why HTTPS is required. Your keyboard and mouse input goes back over the same connection.

The GPU detection happens automatically when a container starts. It finds nothing on a GPU-less machine, so the container stays on the CPU path and you don't need to set anything.

## Things to expect

- **CPU is the limit.** KDE with desktop effects plus video encoding can use several cores. If it feels sluggish, turn off KDE's compositor effects or lower the resolution in the web client.
- **Heavy 3D work will be slow.** Games and WebGL-heavy pages run on software rendering. Ordinary desktop use is fine.
- **Keep `--shm-size=1gb`.** Browsers and desktop apps can crash without it.
- **Authentication is off by default.** If this is a cloud VM, don't open port 3001 to the internet. Use `CUSTOM_USER`/`PASSWORD` on a trusted network, or put it behind a reverse proxy with proper auth. Anyone who can reach the GUI gets root inside the container through passwordless `sudo`.

If something doesn't work, go back to the minimal `docker run` command and add options one at a time, as the docs recommend.
