# FreshRSS

[FreshRSS](https://freshrss.org/) is an RSS and Atom feed reader. This stack uses SQLite, so there is no separate database server.

This stack runs FreshRSS with a Tailscale sidecar, as described in [the standard setup](../../documentation/standard-setup.md).

## At a glance

| Item          | Value                                                   |
| ------------- | ------------------------------------------------------- |
| Web interface | `https://freshrss.<tailnet>.ts.net`                     |
| Service port  | `80`                                                    |
| Image         | `freshrss/freshrss`                                     |
| Data          | `./freshrss-data/app/data` (configuration and database) |
|               | `./freshrss-data/app/extensions` (extensions)           |

## Before you start

Set these values in `.env` before the first start. FreshRSS uses them only while its data folder is empty.

- **`TAILNET_NAME`.** Your Tailnet name with `.ts.net`. `compose.yaml` builds the base address of FreshRSS as `https://<SERVICE>.<TAILNET_NAME>`.
- **`ADMIN_USERNAME`, `ADMIN_PASSWORD`, and `ADMIN_EMAIL`.** The administrator account.
- **`ADMIN_API_PASSWORD`.** The password for clients that use the API. Compose stops with an error if `ADMIN_PASSWORD` or `ADMIN_API_PASSWORD` is empty.

Do not use `$`, backticks, or backslashes in these values.

## Deviations from the standard setup

- **Unattended installation.** `FRESHRSS_INSTALL` and `FRESHRSS_USER` in `compose.yaml` install FreshRSS with SQLite and create the administrator at the first start. On later starts, FreshRSS reports `FreshRSS already installed; no change performed.` and ignores `.env`.
- **Trusted proxy.** `TRUSTED_PROXY=127.0.0.1` makes FreshRSS accept the client address that Tailscale Serve forwards.
- **Feed updates.** `CRON_MIN` in `.env` sets the minutes of each hour at which FreshRSS refreshes the feeds.
- **Health check.** The health check runs `./cli/health.php`, which requests `/api/`. If you disable the API in the web interface, the container reports unhealthy although the web interface works.
- **Optional host Tailscale mode.** `compose.host.yaml` is a standalone alternative
  for a host already connected to Tailscale. It omits the sidecar, shared network
  namespace, and sidecar dependency, and binds HTTP only to host loopback.
  Tailscale Serve on the host supplies private HTTPS. The normal stack is unchanged.

## First run

Open the web interface and log in with the administrator account from `.env`. Change the passwords in the web interface from now on, not in `.env`.

## Configuration

- **Change the base address.** FreshRSS shows the base address read-only under **Configuration** > **System**. To change it after the first start, run `docker compose exec application ./cli/reconfigure.php --base-url https://freshrss.<tailnet>.ts.net`.
- **Local network access.** The `ports` block stays commented out. If you enable it, the stack publishes plain HTTP on port `80` of the Docker host, which is often in use.
- **MagicDNS.** `TS_ACCEPT_DNS=true` is only needed if FreshRSS itself must resolve MagicDNS names, such as an external database. The default SQLite setup does not need it.

### Use an existing host Tailscale connection

Copy `compose.host.yaml` outside this repository into a new runtime directory.
Create a private `.env` there, with permissions `600`. Set `SERVICE`, `IMAGE_URL`,
`TZ`, `CRON_MIN`, `ADMIN_USERNAME`, `ADMIN_PASSWORD`, and `ADMIN_API_PASSWORD`
as in the normal stack. Also set `HOST_PORT=8088` and
`BASE_URL=https://<host>.<tailnet>.ts.net:8444`. Both passwords are required;
use distinct random hexadecimal values. Do not put working credentials here.

From that runtime directory, run:

```sh
docker compose -p freshrss -f compose.host.yaml config --quiet
docker compose -p freshrss -f compose.host.yaml up -d
tailscale serve --bg --https=8444 http://127.0.0.1:8088
```

Use an unused HTTPS port. This adds a listener without replacing other Serve
paths. Keep Funnel disabled. Open `BASE_URL` while connected to the tailnet.
The host must stay awake, and Docker and Tailscale must stay running.
`HOST_TRUSTED_PROXY` defaults to the image's automatic proxy detection;
set it to the Docker gateway address if client-address detection requires it.

To stop only this alternative, run `tailscale serve --https=8444 off` and
`docker compose -p freshrss -f compose.host.yaml down` from the runtime directory.
The data directory is retained. Back it up while the container is stopped before
updating `IMAGE_URL`; first-start variables do not change existing credentials.

## Links

- [FreshRSS documentation](https://freshrss.github.io/FreshRSS/en/)
- [FreshRSS Docker documentation](https://github.com/FreshRSS/FreshRSS/blob/edge/Docker/README.md)
- [FreshRSS extensions](https://github.com/FreshRSS/Extensions)
- [FreshRSS source code](https://github.com/FreshRSS/FreshRSS)
