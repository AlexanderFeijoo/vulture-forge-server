# Uncle Al's Fat Stash — Server Maintenance Guide

Minecraft 1.20.1 Forge modpack managed with [packwiz](https://packwiz.infra.link/).
Players use [Prism Launcher](https://prismlauncher.org/) to install the pack.

---

## Quick Reference

| Item | Value |
|------|-------|
| Server IP | `<SERVER_IP>` |
| SSH | `ssh mcadmin@<SERVER_IP>` |
| RCON | `mcrcon -H 127.0.0.1 -P 25575 -p <RCON_PASSWORD>` |
| Player modpack URL | `http://<SERVER_IP>/modpack/fat-stash.mrpack` |
| Minecraft version | 1.20.1 |
| Forge version | 47.4.15 |

---

## Permissions

`mcadmin` does not have direct access to `/opt/minecraft/`. Use:

- **`sudo -u minecraft bash -c '...'`** for git and packwiz commands (runs as the file-owning user)
- **`sudo systemctl ...`** for service management
- **`sudo -i`** when you need a full root shell

---

## Server Layout

```
/opt/minecraft/
  pack/              Git clone of this repo (packwiz .toml manifests)
  server/            Forge server runtime
    world/           World data
    mods/            Mod jars (installed by packwiz-installer)
    config/          Mod configs
    server.properties
    packwiz-installer-bootstrap.jar
  backups/
    daily/           Automated daily backups (7-day retention)
    preupdate/       Manual snapshots taken before version updates

/var/www/html/modpack/
  fat-stash.mrpack   Served by nginx — players add this URL in Prism Launcher
```

Minecraft runs as systemd service `minecraft` under user `minecraft`.

---

## Common Operations

### Start / stop / restart
```bash
sudo systemctl stop minecraft
sudo systemctl start minecraft
sudo systemctl restart minecraft
```

### View logs
```bash
# Live tail (follow startup, wait for "Done" message)
sudo journalctl -u minecraft -f

# Recent logs
sudo journalctl -u minecraft --since "1 hour ago"

# Logs from a specific day
sudo journalctl -u minecraft --since "2026-02-01" --until "2026-02-02"
```

### RCON console
```bash
mcrcon -H 127.0.0.1 -P 25575 -p <RCON_PASSWORD>
```
Useful commands: `list` (online players), `save-all` (force save), `say <msg>` (broadcast).

### Check backup status
```bash
# Is the daily timer active?
sudo systemctl list-timers mc-backup.timer

# View last backup log
sudo journalctl -u mc-backup.service --since today

# List backups
ls -lht /opt/minecraft/backups/daily/
```

---

## Release Workflow

Full step-by-step process for releasing a new modpack version.

### Local (Windows PC)

**1. Make mod changes**
```bash
cd <path-to-your-local-clone>

# Add a mod from Modrinth
packwiz mr install <mod-name>

# Add a mod from CurseForge
packwiz cf install <mod-name>

# Remove a mod
packwiz remove <mod-name>

# Bump version in pack.toml (edit manually)
```

**2. Refresh and export**
```bash
packwiz refresh
packwiz modrinth export
```
This produces a file like `Uncle Al's Fat Stash-0.16.0.mrpack`.

**3. Test locally in Prism Launcher**
- Open Prism Launcher → Add Instance → Import from zip
- Select the `.mrpack` file you just exported
- Launch the instance and load into a singleplayer world
- Check for crashes, missing textures, or broken recipes
- Delete the test instance when done

**4. Rename the export to the standard filename**
```
ren "Uncle Al's Fat Stash-0.X.0.mrpack" fat-stash.mrpack
```

**5. Commit and push**
```bash
git add -A
git commit -m "v0.X.0 description of changes"
git push
```

**6. Copy the .mrpack to the server**
```bash
scp fat-stash.mrpack mcadmin@<SERVER_IP>:/home/mcadmin/
```

### Server (SSH as mcadmin)

**7. Take a pre-update backup**
```bash
sudo /opt/minecraft/scripts/mc-backup-preupdate.sh "pre-v0.X.0"
```

**8. Move the .mrpack for player downloads**
```bash
sudo mv /home/mcadmin/fat-stash.mrpack /var/www/html/modpack/fat-stash.mrpack
```

**9. Stop the server**
```bash
sudo systemctl stop minecraft
```

**10. Pull the latest pack manifests**
```bash
sudo -u minecraft bash -c 'cd /opt/minecraft/pack && git pull'
```

**11. Install/update mod jars**
```bash
sudo -u minecraft bash -c 'cd /opt/minecraft/server && java -jar packwiz-installer-bootstrap.jar -g -s server file:/opt/minecraft/pack/pack.toml'
```

**12. Start the server and watch startup**
```bash
sudo systemctl start minecraft
sudo journalctl -u minecraft -f
```
Wait for: `Done (Xs)! For help, type "help"`

**13. Verify**
- Download the modpack URL in a browser to confirm nginx is serving the new file
- Connect in-game and confirm mods are working

---

## Server-Only Mods

Some mods only run on the server (not in the player pack). Set `side = "server"` in the mod's `.pw.toml` file:

```toml
name = "No Chat Reports"
filename = "NoChatReports-FORGE-1.20.1-v2.2.2.jar"
side = "server"

[download]
...
```

- `packwiz-installer-bootstrap` installs it on the server
- `packwiz modrinth export` excludes it from the player .mrpack

Current server-only mods: **No Chat Reports**

---

## Restore from Backup

### Restore world (daily backup)

```bash
sudo systemctl stop minecraft
sudo mv /opt/minecraft/server/world "/opt/minecraft/server/world.broken.$(date +%s)"
cd /opt/minecraft/server
sudo tar -xzf /opt/minecraft/backups/daily/world-backup-YYYY-MM-DD_HH-MM-SS.tar.gz
sudo chown -R minecraft:minecraft /opt/minecraft/server/world
sudo systemctl start minecraft
```

### Full rollback (pre-update backup)

Use when a mod update broke things — reverts mods, configs, and world.

```bash
sudo systemctl stop minecraft
TS=$(date +%s)
sudo mv /opt/minecraft/server/world "/opt/minecraft/server/world.pre-rollback.$TS"
sudo mv /opt/minecraft/server/mods "/opt/minecraft/server/mods.pre-rollback.$TS"
sudo mv /opt/minecraft/server/config "/opt/minecraft/server/config.pre-rollback.$TS"
cd /opt/minecraft/server
sudo tar -xzf /opt/minecraft/backups/preupdate/full-backup-LABEL-TIMESTAMP.tar.gz
sudo chown -R minecraft:minecraft /opt/minecraft/server/{world,mods,config}
sudo systemctl start minecraft
```

Always `mv` broken state aside — never `rm -rf`.
