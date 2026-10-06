# Driving the chromium container over CDP

Source: [docs/images/docker-chromium.md](../docs/images/docker-chromium.md)

`compose.yaml` turns on the Chrome DevTools Protocol (CDP) for the browser in the `chromium` container and publishes it on `127.0.0.1:9222`. That's the same endpoint `~/Desktop/chromium-debug/cdp.mjs` uses, so the script drives the container browser without any changes:

```bash
docker compose up -d
curl -s http://127.0.0.1:9222/json/version
node ~/Desktop/chromium-debug/cdp.mjs tabs
```

## What the compose file does

```yaml
services:
  chromium:
    environment:
      - CHROME_CLI=--remote-debugging-port=9222 --user-data-dir=/config/chromium-cdp
    ports:
      - 3001:3001
      - 127.0.0.1:9222:9223

  cdp-proxy:
    image: alpine/socat
    network_mode: service:chromium
    command: tcp-listen:9223,fork,reuseaddr tcp-connect:127.0.0.1:9222
```

1. **`CHROME_CLI`:** the image's autostart runs `wrapped-chromium ${CHROME_CLI}`, so any extra flags go here.
2. **A separate profile:** Chromium refuses remote debugging on its default profile folder, so the browser uses `/config/chromium-cdp` (`./config/chromium-cdp` on the host). Logins from the default profile and from `~/Desktop/chromium-debug/profile` don't carry over.
3. **The `cdp-proxy` sidecar:** a browser with a window only listens for CDP on `127.0.0.1` inside the container. It ignores `--remote-debugging-address`, so Docker's port mapping can't reach it. The sidecar shares the container's network and forwards port 9223 to the browser's 9222. Docker then publishes 9223 as `127.0.0.1:9222` on the host.

I didn't use `network_mode: host`, because that would also open the container's plain-HTTP port 3000 to the LAN.

## Things to expect

- **No authentication.** Anyone who can reach the CDP port fully controls the browser, including its cookies and logins. Keep the `127.0.0.1:` prefix on the port.
- **Chromium isn't restarted.** The container launches the browser once at startup. If it exits (for example, because its window was closed in the web view), CDP goes away until you run `docker compose restart`.
- **Restart both containers together.** The proxy joins the `chromium` container's network when it starts. After `docker restart chromium` on its own, the proxy is left on the old network and CDP can't be reached. Use `docker compose restart` (or `docker compose up -d`) so both come back together.
- **Port clash with the host browser.** The host setup in `~/Desktop/chromium-debug/README.md` also uses port 9222. Run only one of them, or change the host port here and run the script with `CDP_PORT=<port> node cdp.mjs ...`.
- **The page size follows the web viewer.** The browser window fills the desktop, which is sized to the browser tab you have open on port 3001. Screenshots come out at that size, not at 1280×900.

## Netflix and other DRM sites (error M7701-1003)

This image installs Debian's Chromium, which never downloads the Widevine DRM module. `chrome://components` has no Widevine entry, and restarting doesn't help. Netflix shows error M7701-1003.

Copy the module from the host profile into the container profile instead, and add the file that tells Chromium where to find it. The file must hold the path as the container sees it:

```bash
H=~/Desktop/chromium-debug/profile/WidevineCdm
C=./config/chromium-cdp/WidevineCdm
V=4.10.3050.0   # the version folder in $H
mkdir -p $C && cp -a $H/$V $C/
printf '{"Path":"/config/chromium-cdp/WidevineCdm/%s"}' $V > $C/latest-component-updated-widevine-cdm
docker compose restart
```

Then check it from an https page:

```bash
node ~/Desktop/chromium-debug/cdp.mjs goto https://www.netflix.com
node ~/Desktop/chromium-debug/cdp.mjs eval "navigator.requestMediaKeySystemAccess('com.widevine.alpha',[{initDataTypes:['cenc'],videoCapabilities:[{contentType:'video/mp4;codecs=\"avc1.42E01E\"'}]}]).then(()=>'widevine OK').catch(e=>'widevine FAIL: '+e.message)"
```

- **The module never updates itself.** If Netflix starts rejecting it, copy a newer version from the host profile the same way.
- **It lives in `config/`, which isn't in git.** A fresh `config/` needs the copy again.
