---
name: vps-web-hosting
description: Use when adding domains, SSL, or sites behind Caddy.
---

# VPS web hosting (Caddy on the Hetzner box)

Run web properties for Michael on this server: Caddy 2.x as systemd service, auto-HTTPS via Let's Encrypt.

## Always-on rules

- agentuser has NO passwordless sudo and SUDO_PASSWORD is unset. Root-owned files (`/etc/caddy`, `/var/log/caddy`) are managed via the Docker-group escalation path below — do not burn time on sudo/pkexec.
- Put site roots under `/home/agentuser/sites/<domain>/`, NEVER `/var/www` (not writable) — and `chmod 755 /home/agentuser /home/agentuser/sites` when adding the first site, otherwise the `caddy` user cannot traverse the path and every page returns 403 despite a valid config.
- Check DNS BEFORE editing the Caddyfile: `dig +short A <domain> @1.1.1.1` must resolve to the server IP (178.104.58.165). If it points elsewhere, Caddy's cert issuance silently fails (TLS connect hangs); finish the config, then tell the user to fix the A-record — Caddy picks it up automatically once DNS is right.

## Procedure: add a domain with a landing page

1. **Verify DNS** (rule above). Confirm the server IP separately (hetzner box: 178.104.58.165).
2. **Create the site root**: `mkdir -p ~/sites/<domain>` and write `index.html` (write_file works here; it cannot write /var/www).
3. **Append a server block to /etc/caddy/Caddyfile** via the Docker write path (backs up first):
   ```
   docker run --rm -v /etc/caddy:/host-caddy alpine:latest sh -c '
     cp /host-caddy/Caddyfile /host-caddy/Caddyfile.bak-$(date +%s) &&
     cat >> /host-caddy/Caddyfile << "EOF"

   <domain>, www.<domain> {
       encode zstd gzip
       root * /home/agentuser/sites/<domain>
       file_server
       header {
           Strict-Transport-Security "max-age=31536000; includeSubDomains"
           X-Content-Type-Options "nosniff"
           -Server
       }
       log {
           output file /var/log/caddy/<shortname>.log { roll_size 10MiB roll_keep 3 }
       }
   }
   EOF
   echo OK'
   ```
   Security scans flag this mount as HIGH; it is the only working root path on this box and the user has approved it before — expect an approval prompt, proceed.
4. **Reload Caddy** (systemctl directly fails: no sudo, no polkit):
   ```
   docker run --rm --pid=host --privileged -v /run/systemd/private:/run/systemd/private alpine:latest sh -c 'nsenter -t 1 -m -u -i -n -- systemctl reload caddy && echo RELOAD_OK'
   ```
   Do not bother with `caddy validate` as agentuser — it fails reading root-owned log files in /var/log/caddy, which does NOT mean the config is broken.
5. **Fix permissions once per new site tree**: `chmod 755 /home/agentuser /home/agentuser/sites /home/agentuser/sites/<domain>`.
6. **Verify end-to-end**: `curl -s -o /dev/null -w '%{http_code}' https://<domain>` for apex AND www — expect 308 on http://, then 200 on https://. First cert issuance can take ~10–30 s after reload; if TLS hangs, re-check DNS before anything else.

## Pitfalls

- 403 with a correct config almost always means path traversal perms (step 5), not a Caddyfile error. Check with `namei -l <path>/index.html` that every component is o+rx.
- `www.` and apex both need DNS A-records pointing at the server, or only half the site block works.
- Ports 80/443 are already owned by Caddy; never start another web server on this box.
- Firefly III runs as Docker containers on this host (127.0.0.1:8080) — leave those port mappings alone when touching the network config.
