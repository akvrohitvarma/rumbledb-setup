# Self-Hosting RumbleDB in Docker

A guide to running [RumbleDB](https://rumbledb.org/) (a JSONiq engine) inside a Docker container and exposing it over the network. This covers local setup, LAN access, and optionally proxying it behind a domain with Nginx Proxy Manager.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [1. Setting Up the Container](#1-setting-up-the-container)
- [2. Installing RumbleDB](#2-installing-rumbledb)
- [3. Running RumbleDB](#3-running-rumbledb)
- [4. Fixing the Mixed Content Issue](#4-fixing-the-mixed-content-issue)
- [5. Accessing RumbleDB Over the LAN](#5-accessing-rumbledb-over-the-lan)
- [6. Exposing RumbleDB to the Internet](#6-exposing-rumbledb-to-the-internet)
- [Optional: Proxying with Nginx Proxy Manager](#optional-proxying-with-nginx-proxy-manager)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Docker installed on your machine (or a Portainer instance managing Docker)
- A terminal or Portainer console to interact with your container
- Port 8001 available on your host

---

## 1. Setting Up the Container

RumbleDB requires **Java 11 specifically**. It will not run on newer Java versions. The `eclipse-temurin:11-jdk` image provides exactly this.

**Using Docker CLI:**

```bash
docker run -dit \
  --name rumbledb \
  -p 8001:8001 \
  eclipse-temurin:11-jdk \
  /bin/bash
```

**Using Portainer:**

1. Go to **Containers > Add Container**.
2. Set the image to `eclipse-temurin:11-jdk`.
3. Enable **Interactive & TTY** (equivalent to `-it`).
4. Under **Network ports configuration**, publish a new port: host `8001` to container `8001` (TCP).
5. Deploy the container.

> **Why not `openjdk:11`?** — The `openjdk:11` tag on Docker Hub may resolve to a much newer JDK (e.g., Java 27). The `eclipse-temurin:11-jdk` image from Adoptium guarantees Java 11.

---

## 2. Installing RumbleDB

Open a shell in your container. In Portainer, click on the container, then **Console > /bin/bash > Connect**. In Docker CLI, use `docker exec -it rumbledb bash`.

Install `curl` if it's not available, then download the RumbleDB JAR:

```bash
apt-get update && apt-get install -y curl
curl -L -o rumbledb.jar https://github.com/RumbleDB/rumble/releases/download/v1.21.0/rumbledb-1.21.0-standalone.jar
```

This is a ~436 MB download. Adjust the version number if a newer release is available on the [RumbleDB releases page](https://github.com/RumbleDB/rumble/releases).

---

## 3. Running RumbleDB

### Server Mode (Web UI)

```bash
java -jar rumbledb.jar --server yes --port 8001 --host 0.0.0.0
```

The `--host 0.0.0.0` flag is critical. Without it, RumbleDB binds to `127.0.0.1` and is only accessible from inside the container.

Once it starts, you should see a message pointing you to `http://localhost:8001/public.html`. Open your browser and navigate to:

```
http://<your-host-ip>:8001/public.html
```

For example, if your Docker host's LAN IP is `192.168.0.59`:

```
http://192.168.0.59:8001/public.html
```

### Shell Mode (Interactive REPL)

If you just want to type queries directly without a browser:

```bash
java -jar rumbledb.jar --shell yes
```

This gives you a `rumble$` prompt. Useful for quick testing.

---

## 4. Fixing the Mixed Content Issue

RumbleDB's built-in `public.html` loads jQuery from an `http://` URL:

```html
<script src="http://ajax.googleapis.com/ajax/libs/jquery/1.8.3/jquery.min.js"></script>
```

If you ever access the page over HTTPS (directly or through a reverse proxy), the browser will block this script due to mixed content. The Evaluate button will stop working because jQuery never loads.

Even if you only plan to use HTTP, it's worth fixing this to avoid issues later.

### Patching the JAR

The HTML file is embedded inside the RumbleDB JAR at `assets/public.html`. To fix it:

```bash
# Install zip utilities (needed for jar commands)
apt-get update && apt-get install -y zip unzip

# Extract the HTML file from the JAR
jar xf rumbledb.jar assets/public.html

# Replace http:// with https:// for the jQuery CDN
sed -i 's|http://ajax.googleapis.com|https://ajax.googleapis.com|g' assets/public.html

# (Optional) Check for any other http:// references
grep -i 'http://' assets/public.html

# If there are localhost references and you plan to use a domain, fix those too:
# sed -i 's|http://localhost:8001|https://yourdomain.com|g' assets/public.html

# Pack the modified file back into the JAR
jar uf rumbledb.jar assets/public.html
```

After patching, restart RumbleDB:

```bash
# Stop the running instance (Ctrl+C if in foreground, or:)
pkill -f rumbledb

# Start again
java -jar rumbledb.jar --server yes --port 8001 --host 0.0.0.0
```

> **Note:** If you recreate the container, you will need to repeat this patch. To make it permanent, commit the container as a custom Docker image: `docker commit rumbledb rumbledb-patched`.

---

## 5. Accessing RumbleDB Over the LAN

If Docker is running on a machine at `192.168.0.59`, any device on the same network can access:

```
http://192.168.0.59:8001/public.html
```

No additional configuration is needed as long as the port mapping (`-p 8001:8001`) was set when creating the container.

If it's not reachable, check your host machine's firewall:

```bash
# Ubuntu/Debian
sudo ufw allow 8001

# Or check iptables
sudo iptables -L -n | grep 8001
```

---

## 6. Exposing RumbleDB to the Internet

To let people outside your network access RumbleDB, you need to forward the port on your router.

### Router Port Forwarding

Log into your router (usually at `192.168.0.1` or `192.168.1.1`) and create a port forwarding rule:

| Setting | Value |
|---|---|
| External port | 8001 |
| Internal IP | Your Docker host's LAN IP (e.g., `192.168.0.59`) |
| Internal port | 8001 |
| Protocol | TCP |

### Accessing from Outside

Find your public IP by visiting [whatismyip.com](https://whatismyip.com) or running `curl ifconfig.me`. Others can then access:

```
http://<your-public-ip>:8001/public.html
```

### Optional: Using a Domain Name

If you own a domain, create an **A record** pointing a subdomain to your public IP:

| Type | Name | Value |
|---|---|---|
| A | json | your.public.ip.address |

Then access via `http://json.yourdomain.com:8001/public.html`.

### Static IP Recommendation

Home networks assign IPs dynamically via DHCP. If your Docker host's LAN IP changes, the port forwarding rule breaks. Set a **static IP** or a **DHCP reservation** for your Docker host in your router settings to avoid this.

---

## Optional: Proxying with Nginx Proxy Manager

If you want to access RumbleDB on a clean URL like `json.yourdomain.com` (without the `:8001` port), you can put it behind [Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM).

This section assumes you already have NPM running. If not, see the [NPM installation guide](https://nginxproxymanager.com/guide/).

### Setting Up the Proxy Host

1. In NPM, go to **Proxy Hosts > Add Proxy Host**.
2. Fill in the details:
   - **Domain Names:** `json.yourdomain.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** your Docker host's LAN IP (e.g., `192.168.0.59`)
   - **Forward Port:** `8001`
3. Save.

### SSL Setup

Under the **SSL** tab, you can request a Let's Encrypt certificate. However, be aware that enabling SSL introduces the mixed content problem described in [Section 4](#4-fixing-the-mixed-content-issue). Make sure you have patched the JAR before enabling SSL.

If you patched the jQuery URL but the page still has issues with the API endpoint, you may also need to patch the `localhost` references in `assets/public.html`:

```bash
sed -i 's|http://localhost:8001|https://json.yourdomain.com|g' assets/public.html
jar uf rumbledb.jar assets/public.html
```

### Redirecting Root to public.html

By default, visiting `json.yourdomain.com` will not load the query page. Add this in the proxy host's **Advanced** tab:

```nginx
location = / {
    return 301 /public.html;
}
```

### Hairpin NAT

If you can access `json.yourdomain.com` from outside your network but not from inside it, your router likely doesn't support hairpin NAT (routing traffic back to itself when the destination is your own public IP).

Workaround: add a local DNS override. Edit your hosts file:

- **macOS/Linux:** `/etc/hosts`
- **Windows:** `C:\Windows\System32\drivers\etc\hosts`

Add:

```
192.168.0.59  json.yourdomain.com
```

This makes local requests resolve directly to the LAN IP instead of going through the router.

### Why `sub_filter` Might Not Work

A common approach to fix mixed content in Nginx is to use `sub_filter` directives to rewrite `http://` URLs in the response body. In practice, this often fails with RumbleDB because:

1. The backend may send gzip-compressed responses, and `sub_filter` only works on uncompressed content.
2. Adding `proxy_set_header Accept-Encoding "";` doesn't always force the backend to stop compressing.
3. The `sub_filter_types` directive may not cover all content types served by RumbleDB.

Patching the JAR directly (as described in Section 4) is the most reliable solution. It fixes the problem at the source rather than trying to rewrite responses on the fly.

---

## Troubleshooting

### RumbleDB refuses to start: "requires Java 8 or Java 11"

You're running a Java version other than 8 or 11. Use the `eclipse-temurin:11-jdk` Docker image.

### Page loads but the Evaluate button does nothing

Open your browser's Developer Tools (F12) and check the **Network** tab. If `jquery.min.js` shows as blocked or failed, you have a mixed content issue. Follow [Section 4](#4-fixing-the-mixed-content-issue) to patch the JAR.

### Container accessible locally but not from other devices

- Verify the port mapping exists: `docker port rumbledb`
- Check that RumbleDB was started with `--host 0.0.0.0`
- Check the host firewall

### Parser errors when running queries

- Make sure every query starts with `jsoniq version "1.0";`
- Every `let` chain must end with a `return`
- Watch out for invisible Unicode characters (zero-width spaces) when copy-pasting from web pages or chat interfaces. If you get a parser error on a line that looks correct, delete the line and retype it manually.

### Container IP changed after restart

Set a static IP or DHCP reservation in your router for the Docker host machine. Update any port forwarding rules, hosts file entries, and NPM proxy hosts with the new IP.

---

## Quick Reference

| Task | Command |
|---|---|
| Start server mode | `java -jar rumbledb.jar --server yes --port 8001 --host 0.0.0.0` |
| Start shell mode | `java -jar rumbledb.jar --shell yes` |
| Run a query file | `java -jar rumbledb.jar --query-path myquery.jq` |
| Patch jQuery URL | `jar xf rumbledb.jar assets/public.html && sed -i 's\|http://ajax.googleapis.com\|https://ajax.googleapis.com\|g' assets/public.html && jar uf rumbledb.jar assets/public.html` |
| Commit container as image | `docker commit rumbledb rumbledb-patched` |

---

## License

This guide is provided as-is. RumbleDB is licensed under the [Apache License 2.0](https://github.com/RumbleDB/rumble/blob/master/LICENSE).
