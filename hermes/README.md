# Hermes Agent

This Compose project extends the official Hermes image with a supervised
virtual X11 desktop, headed Chromium, Linux accessibility, password-protected
VNC, and the upstream Hermes Agent Dashboard. Hermes keeps its upstream
entrypoint, `gateway run` command, data volume, LM Studio connection, and
shared Docker network.

## Test Notes (2026-08-25)

These notes summarize hands-on testing of Hermes in this Docker setup as of
2026-08-25. They are a personal usage evaluation, not a benchmark.

### Observations

- Because Hermes runs in an isolated environment rather than the actual host
  workspace, it felt closer to a customized chatbot than a personal assistant
  agent.
- The set of practical automations was limited by the container boundary.
- The virtual monitor and Chromium could be configured and launched, but the
  desktop interaction was too slow for normal use.
- The GUI rendered in the virtual monitor was extremely limited. In the
  current container setup, Chromium was effectively the only usable
  application. Closing Chromium showed an empty black screen for about two
  seconds before Chromium reopened automatically, leaving no practical
  desktop environment beyond the browser.
- VNC interaction had severe lag, making remote desktop management unreliable
  and difficult to use as a working environment.
- Overall, the Docker deployment did not provide a sufficiently responsive or
  capable GUI environment for this use case.

### Current decision

Docker-based Hermes use is temporarily paused. The next test will run Hermes
directly on the macOS host to determine whether host-level access, automation,
and GUI responsiveness better match the intended assistant workflow.

### Revisit criteria

When comparing the host setup with Docker, check:

- responsiveness of GUI interactions;
- reliability of browser control;
- availability of applications beyond the browser and behavior when it closes;
- useful automation coverage;
- access to the real host workspace and applications; and
- the security and permission boundary required for everyday use.

## Requirements

- Docker Desktop
- LM Studio with its API server running on port `1234`
- The external `local-ai-network` Docker network
- A native VNC viewer on macOS
- Tailscale for optional remote access from the same tailnet

Create the shared network once if it does not already exist:

```bash
docker network create local-ai-network
```

## Initial Setup

```bash
docker compose run --rm hermes setup
```

For LM Studio, use this Base URL:

```text
http://host.docker.internal:1234/v1
```

If LM Studio authentication is enabled, enter its API token during setup.

## Start

Build the desktop extension and start Hermes:

```bash
docker compose up -d --build
```

The localhost-only services are:

| Service | Local address |
| --- | --- |
| Hermes gateway | `http://localhost:8642` |
| Hermes Agent Dashboard | `http://localhost:9119` |
| Virtual desktop over VNC | `vnc://localhost:5901` |

The host VNC port is deliberately `5901`, while x11vnc continues to use its
standard port `5900` inside the container. This avoids the host port `5900`
reserved by macOS Screen Sharing and Remote Management.

## VNC and Dashboard Credentials

On the first container boot, Hermes generates separate VNC and Dashboard
credentials under the existing `hermes-data` volume. They persist across
container recreation and are not stored in Compose or exposed by
`docker inspect`.

### Check Current Credentials

The following command displays the current Dashboard username, Dashboard
password, and VNC password:

```bash
docker exec hermes bash -c '
credentials_dir=/opt/data/computer-use/credentials

printf "Dashboard username: "
sed -n "1p" "$credentials_dir/dashboard-username"

printf "Dashboard password: "
sed -n "1p" "$credentials_dir/dashboard-password"

printf "VNC password: "
sed -n "1p" "$credentials_dir/vnc-password"
'
```

Run this only in a private terminal because it prints both passwords in plain
text. VNC does not have a username; only the Dashboard uses the displayed
username.

The generated VNC password is eight characters because classic VNC
authentication uses only the first eight password characters. Do not reuse a
macOS login, admin, API, or Dashboard password for VNC.

## Change VNC and Dashboard Credentials

Run the following commands in a macOS terminal. Paste each command block as a
whole, then answer its prompts. Password input is hidden and does not appear in
the terminal.

Change the Dashboard username and password:

```bash
docker exec -it hermes bash -c '
credentials_dir=/opt/data/computer-use/credentials

read -r -p "New Dashboard username: " username
read -r -s -p "New Dashboard password (16+ characters): " password
printf "\n"
read -r -s -p "Confirm Dashboard password: " confirmation
printf "\n"

if [[ ! "$username" =~ ^[A-Za-z0-9._-]+$ ]]; then
  echo "The username may contain only letters, numbers, dots, underscores, and hyphens."
  exit 1
fi

if [[ ${#password} -lt 16 || "$password" != "$confirmation" ]]; then
  echo "The passwords must match and contain at least 16 characters."
  exit 1
fi

printf "%s" "$username" > "$credentials_dir/dashboard-username"
printf "%s" "$password" > "$credentials_dir/dashboard-password"
od -An -N32 -tx1 /dev/urandom | tr -d " \n" > \
  "$credentials_dir/dashboard-secret"

chown hermes:hermes \
  "$credentials_dir/dashboard-username" \
  "$credentials_dir/dashboard-password" \
  "$credentials_dir/dashboard-secret"
chmod 600 \
  "$credentials_dir/dashboard-username" \
  "$credentials_dir/dashboard-password" \
  "$credentials_dir/dashboard-secret"

echo "Dashboard credentials updated."
'
```

Rotating `dashboard-secret` invalidates existing Dashboard login sessions.

Change the VNC password:

```bash
docker exec -it hermes bash -c '
password_file=/opt/data/computer-use/credentials/vnc-password

read -r -s -p "New VNC password (exactly 8 letters or numbers): " password
printf "\n"

if [[ ! "$password" =~ ^[A-Za-z0-9]{8}$ ]]; then
  echo "The VNC password must contain exactly 8 letters or numbers."
  exit 1
fi

printf "%s" "$password" > "$password_file"
chown hermes:hermes "$password_file"
chmod 600 "$password_file"

echo "VNC password updated."
'
```

Restart Hermes once after making either or both changes:

```bash
docker restart hermes
```

The Dashboard uses the new username and password. VNC does not use a username;
enter only the new eight-character VNC password in the viewer.

## Remote Access with Tailscale

The Compose ports remain bound to `127.0.0.1`. Keep those bindings and use
Tailscale Serve to publish only to authenticated devices in the same tailnet.

VNC needs a raw TCP forwarder:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale \
  serve --bg --tcp=5901 tcp://localhost:5901
```

The Dashboard needs an HTTPS reverse proxy on its own port:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale \
  serve --bg --https=9119 localhost:9119
```

Check the node name and active endpoints:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve status
```

From another tailnet device, use the MagicDNS name shown by the status command:

```text
vnc://<mac-magicdns-name>:5901
https://<mac-magicdns-name>:9119
```

The VNC endpoint is not a website; open it with a native VNC viewer. Tailscale
encrypts traffic between tailnet devices, while x11vnc still requires the VNC
password above. The Dashboard additionally requires its own username and
password.

Disable only these Hermes Serve listeners with:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --tcp=5901 off
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --https=9119 off
```

Serve remains private to the tailnet and follows Tailscale access controls. Do
not use Tailscale Funnel for either VNC or the Dashboard. See the official
[Tailscale Serve CLI reference](https://tailscale.com/docs/reference/tailscale-cli/serve)
and [Serve examples](https://tailscale.com/docs/reference/examples/serve).

## Computer Use Architecture

```text
Hermes Computer Use → AT-SPI/X11 → Xvfb :99 → Chromium
                                      ↑
                                   x11vnc
                                      ↑
                            localhost/Tailscale Serve
                                      ↑
                                native VNC viewer
```

Xvfb, the D-Bus/AT-SPI session, Openbox, headed Chromium, and x11vnc are
independent s6 services. They start automatically and share `DISPLAY=:99`.
The upstream Dashboard is also enabled as its own supervised s6 service.
Chromium uses the persistent profile at:

```text
/opt/data/computer-use/chromium-profile
```

The image runs Chromium as the unprivileged `hermes` user and never uses
`--headless`. It does use `--no-sandbox`: Debian's setuid Chromium sandbox
cannot create its required namespace under Docker Desktop's default container
security profile (`Operation not permitted`). Granting the container broad
`SYS_ADMIN` capability or disabling its seccomp profile would weaken the whole
container, so the exception is limited to the already-container-isolated
Chromium process.

## Diagnostics

Check Compose and the container logs:

```bash
docker compose ps
docker compose logs hermes
```

Check the supervised desktop and Dashboard services:

```bash
docker exec hermes sh -c '
  for service in xvfb desktop-session at-spi openbox chromium x11vnc dashboard; do
    printf "%s: " "$service"
    /command/s6-svstat "/run/service/$service"
  done
'
```

Check the X server and window tree:

```bash
docker exec hermes xdpyinfo -display :99
docker exec hermes sh -lc 'DISPLAY=:99 xwininfo -root -tree'
```

Check Hermes Computer Use and its bundled `cua-driver`:

```bash
docker exec hermes hermes computer-use status
docker exec hermes hermes computer-use doctor
```

Check the Dashboard authentication gate:

```bash
curl -s http://localhost:9119/api/status
```

The response should report that authentication is required and that the
`basic` provider is active. The generated username/password provider is meant
for localhost or a trusted VPN such as Tailscale, not direct public exposure.
For public deployments, follow the upstream
[Hermes Dashboard authentication guide](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/web-dashboard.md)
and use OAuth or OIDC.

## Web Search with SearXNG

Configure Hermes Web Search with:

```text
http://searxng:8080
```

Test connectivity:

```bash
docker exec hermes curl -s \
  "http://searxng:8080/search?q=test&format=json"
```

## Terminal Backend

When Hermes itself runs inside Docker, select the `Local` terminal backend.
That executes terminal commands in this Hermes container. Select the Docker
backend only when nested Docker sandbox execution is intentionally configured.

## Operations

```bash
# Follow logs
docker compose logs -f

# Restart while retaining all data and generated credentials
docker compose restart

# Stop while retaining the hermes-data volume
docker compose down

# Refresh the upstream base image, rebuild, and start
docker compose build --pull
docker compose up -d
```

Hermes configuration, sessions, generated credentials, and the Chromium
profile are stored in the existing named volume:

```text
hermes-data
```

Inspect it without modifying it:

```bash
docker volume inspect hermes-data
```

## Reset Hermes

To stop Hermes and remove its container, the `hermes-data` volume, and any
orphan containers from this Compose project, run this from the `hermes`
directory:

```bash
docker compose down --volumes --remove-orphans
```

> Warning: This permanently deletes Hermes configuration, sessions, generated
> VNC and Dashboard credentials, and the Chromium profile stored in
> `hermes-data`. Back up anything you need before running it.

The command leaves the external `local-ai-network` and the local Hermes Docker
image intact. No separate `docker volume rm hermes-data` command is required.
If you only want to stop Hermes while retaining its data, use
`docker compose down` under [Operations](#operations) instead.

Verify that the container and volume are gone:

```bash
docker volume ls --filter name=hermes-data
docker ps -a --filter name=hermes
```
