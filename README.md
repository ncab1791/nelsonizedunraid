# nelsonizedunraid

Unraid Docker templates (XML) for apps that don't have a good Community Applications template.

## Using these templates on Unraid

Unraid 6.10 and later no longer has a "Template Repositories" box, so you copy the XMLs onto the flash drive instead.

1. Open a terminal (the `>_` icon at the top right of the Unraid web UI) and paste:

   ```bash
   cd /boot/config/plugins/dockerMan/templates-user && \
   for f in project-nomad:nomad_admin nomad-mysql:nomad-mysql nomad-redis:nomad-redis \
            nomad-dozzle:nomad_dozzle nomad-disk-collector:nomad_disk_collector; do
     wget -q -O "my-${f#*:}.xml" "https://raw.githubusercontent.com/ncab1791/nelsonizedunraid/main/templates/${f%%:*}.xml" \
       && echo "got ${f#*:}"
   done
   ```

2. Go to **Docker → Add Container**, open the **Template** dropdown, and pick the template under **User templates**.

Re-run the command to pull updated templates. It overwrites the saved template, not your running container, so re-check your settings before you apply.

## Templates

| Template | Container name | What it is |
|---|---|---|
| [`project-nomad.xml`](templates/project-nomad.xml) | `nomad_admin` | Project N.O.M.A.D. Command Center (required) |
| [`nomad-mysql.xml`](templates/nomad-mysql.xml) | `nomad-mysql` | MySQL 8.0 for NOMAD (required) |
| [`nomad-redis.xml`](templates/nomad-redis.xml) | `nomad-redis` | Redis 7 for NOMAD (required) |
| [`nomad-dozzle.xml`](templates/nomad-dozzle.xml) | `nomad_dozzle` | Log viewer for NOMAD (optional) |
| [`nomad-disk-collector.xml`](templates/nomad-disk-collector.xml) | `nomad_disk_collector` | Disk-usage info for NOMAD's UI (optional) |

---

## Project N.O.M.A.D. on Unraid

[Project N.O.M.A.D.](https://github.com/crosstalk-solutions/project-nomad) is an offline knowledge and education server. Upstream ships it as a Docker Compose stack. These templates split that stack into separate Unraid containers.

### Install order

1. **Create the network.** Open the Unraid terminal and run:

   ```bash
   docker network create project-nomad_default
   ```

   Then go to **Settings → Docker**, stop Docker, set **Preserve user defined networks** to **Yes**, and start Docker again. Without this setting, the network is gone after a reboot.

   The name must be exactly `project-nomad_default`, because NOMAD attaches every app it installs (Kiwix, Ollama, Kolibri and others) to that network.

2. **Install `nomad-mysql`.** Set `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD`.
3. **Install `nomad-redis`.** No settings are needed.
4. **Install `nomad_admin`** (`project-nomad.xml`). Fill in:
   - `URL`: the address you'll open it at, e.g. `http://192.168.1.10:8080`
   - `APP_KEY`: at least 16 characters. Generate one with `openssl rand -hex 32`
   - `DB_PASSWORD`: the same value as `MYSQL_PASSWORD` from step 2
5. *(Optional)* Install `nomad_dozzle` and `nomad_disk_collector`.
6. Open `http://<unraid-ip>:8080`.

Under **Docker**, unlock the container list and drag `nomad-mysql` and `nomad-redis` above `nomad_admin` so they autostart first. If `nomad_admin` starts before the database is ready, the `--restart=unless-stopped` setting makes it retry.

### Rules that matter

- **Keep the container name `nomad_admin`.** The app finds its own container by that name to work out where its storage is.
- **MySQL and Redis names use hyphens (`nomad-mysql`, `nomad-redis`), not underscores.** The app rejects `DB_HOST` and `REDIS_HOST` values that contain underscores, so the upstream names `nomad_mysql` and `nomad_redis` won't work as hostnames.
- **Use one storage path everywhere.** The default is `/mnt/user/appdata/project-nomad/storage`. If you change it, change it to the same value (case-sensitive) in:
  - the `Storage` path in `nomad_admin`
  - `NOMAD_STORAGE_PATH` in `nomad_admin`
  - the `Storage` path in `nomad_disk_collector`

  Apps installed from the Command Center store their data under this folder too. ZIM libraries and AI models can be tens of GB, so point it at a share with room.
- **There is no login.** Keep it on your LAN. Don't port-forward it.

### Differences from the upstream Compose file

- **The `updater` sidecar is left out.** It rewrites upstream's `compose.yml`, which doesn't exist on Unraid. Update `nomad_admin` from the Docker tab (**Check for Updates**) instead. The in-app update button won't work.
- The `nomad-update-shared` named volume is replaced with a bind mount under appdata.
- Restart policy and health checks are set with `ExtraParams`.

### Apps NOMAD installs

When you install something from the Supply Depot, NOMAD creates that container itself through the Docker socket. Those containers show up in the Unraid Docker tab, but they have no Unraid template, so manage and update them from the NOMAD UI.
