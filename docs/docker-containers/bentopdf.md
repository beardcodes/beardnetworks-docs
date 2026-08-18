# BentoPDF

Merging two PDFs should not involve uploading a document to a stranger's server. The free online PDF tools are convenient and they are also an unknown party receiving your bank statements, contracts and passport scans.

BentoPDF does the same jobs entirely in your browser. The container serves static files; the actual PDF work happens client-side in WebAssembly, and nothing is uploaded anywhere.

[BentoPDF](https://github.com/alam00000/bentopdf) is a privacy-first PDF toolkit.

1. **Everything runs in the browser** — merge, split, rotate, compress, convert, OCR, sign
2. **No uploads, no server processing**, so no file ever leaves the machine you are sitting at
3. **Works offline** once loaded, and supports air-gapped deployment
4. **No accounts, no database, no persistent state** to back up
5. **Tools can be disabled individually** if you want a narrower deployment
6. **Custom branding** for an internal deployment

## Which image to use

BentoPDF ships two builds, and picking the wrong one is the main setup mistake:

| Build | Image | What it is |
|---|---|---|
| **Self-Hosted** | `ghcr.io/alam00000/bentopdf-simple:latest` | Every PDF tool, without the marketing site — no hero, FAQ, testimonials or footer |
| **Commercial** | `ghcr.io/alam00000/bentopdf:latest` | The full marketing site, as used by bentopdf.com and by commercial licence holders running public deployments |

**Use the Self-Hosted build.** Upstream is explicit that it is not a feature-reduced version — it has the same tools, just without the landing-page furniture you do not want on an internal service.

!!! note "Check the licence before publishing it"

    The two builds exist because public-facing commercial deployments are treated differently from internal ones. For a homelab on your own LAN this is a non-issue, but read the repository's licence terms before putting a public instance on the internet for other people to use.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - A browser with WebAssembly, which is any browser from the last several years

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/bentopdf && cd /opt/docker/bentopdf
sudo nano docker-compose.yml
```

```yaml
services:
  bentopdf:
    image: ghcr.io/alam00000/bentopdf-simple:latest
    container_name: bentopdf
    restart: unless-stopped
    ports:
      - 3005:8080
```

That is the whole thing. **No volumes** — there is no state to persist, because the application never holds your files. It is a static web server for an application that runs in your browser.

The container listens on 8080; 3005 on the host is arbitrary, chosen to dodge the crowd already sitting on 8080 and 3000.

```bash
docker compose up -d
```

Browse to `http://<host-ip>:3005`.

## 2. Confirm the privacy claim yourself

Do not take this on trust, and do not take my word for it either. Open your browser's developer tools, go to the **Network** tab, and merge two PDFs.

You should see the WASM modules and assets load once, and then **no requests carrying your file**. That is the entire security model, and it is directly observable in about thirty seconds.

The stronger test: disconnect the machine from the network after the page has loaded, then use the tools. They keep working.

<!-- screenshot: BentoPDF tool grid with the merge and split tools -->

## 3. Disable tools you do not want

The image supports switching individual tools off, which is useful for a shared or family-facing deployment where a wall of forty options is noise.

Check the repository README for the current environment variable syntax — the tool list changes as features are added, so a copied example goes stale quickly.

## 4. Put it behind Traefik

```yaml
services:
  bentopdf:
    image: ghcr.io/alam00000/bentopdf-simple:latest
    container_name: bentopdf
    restart: unless-stopped
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.bentopdf.rule=Host(`pdf.home.lan`)"
      - "traefik.http.routers.bentopdf.entrypoints=https"
      - "traefik.http.routers.bentopdf.tls.certresolver=letsencrypt"
      - "traefik.http.services.bentopdf.loadbalancer.server.port=8080"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

Serve it over HTTPS. Some browser APIs are restricted to secure contexts, and a tool whose selling point is privacy should not arrive over plain HTTP.

There is nothing to protect with [Tinyauth](tinyauth.md) — no accounts, no stored documents, no state. Adding a login gains you nothing except friction. Keeping it on the LAN is sufficient.

## 5. Digital signatures need a CORS proxy

If you want the digital signature feature, upstream notes it requires a CORS proxy, because signature verification reaches out to certificate authority endpoints that do not send permissive CORS headers.

Everything else works without it. Set it up only if you actually sign documents — the README covers the configuration.

## Updating

```bash
cd /opt/docker/bentopdf
docker compose pull
docker compose up -d
```

The safest app in this documentation to update: there is no database to migrate and no config to break. Worst case you pull a bad image and roll back the tag.

Hard-refresh the browser afterwards (Ctrl+Shift+R) — the old WASM modules cache aggressively, which is exactly what you want for offline use and mildly annoying after an upgrade.

## Backup

Nothing to back up. No volumes, no database, no state.

Keep the four-line Compose file in your Git repo and the entire service is reproducible.

## Troubleshooting

**Blank page or "WebAssembly not supported".** A very old browser, or a corporate policy blocking WASM. Also check you are on HTTPS.

**A large PDF fails or the tab crashes.** Everything happens in browser memory, so a 500 MB scanned document is bounded by the RAM of the machine you are *viewing* from, not the server. Split it first, or use a machine with more memory.

**OCR is slow.** It is running locally on your CPU. That is the trade for the file never leaving the machine.

**Digital signature verification fails.** The CORS proxy, step 5.

**Nothing changed after updating.** Cached WASM. Hard refresh.

**Tools still show marketing sections.** You pulled the commercial build. Use `bentopdf-simple`.

## Where this sits in my lab

BentoPDF runs on the Docker VM on my first Proxmox node, behind Traefik at an internal hostname, LAN-only.

It exists to make a specific habit stop: reaching for whatever the search engine returns when something needs merging or a page removing. Those sites are free because the documents are the product, and the documents in question are usually the ones you would least like to hand over.

The whole service is four lines of YAML and no state — it is the cheapest thing in this documentation to run and one of the more useful.
