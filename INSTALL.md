# Host GRWeb on your own site

GRWeb is a static browser game. Download a release, extract it and serve the complete folder over HTTP/HTTPS. No Unity, .NET, Node.js, database or game backend is required on your server.

This guide installs **v0.2.1** at **`https://example.com/grweb/`** on an existing Nginx server. Replace `example.com` with your domain. The shell commands assume root access on Linux, GNU tools and an existing HTTPS virtual host. AlmaLinux 9 uses the same steps, with the SELinux section below when enforcing.

## 1. Download and verify

In an empty working directory on your server:

```bash
curl -fLO https://github.com/mannetroll/grwebdeploy/releases/download/v0.2.1/GRWeb-v0.2.1.tar.gz
curl -fLO https://github.com/mannetroll/grwebdeploy/releases/download/v0.2.1/SHA256SUMS
sha256sum --ignore-missing -c SHA256SUMS
```

Continue only if the archive reports **OK**. `--ignore-missing` skips the alternative ZIP archive listed in the same checksum file. On macOS, compare `shasum -a 256 GRWeb-v0.2.1.tar.gz` with its line in `SHA256SUMS`; on Windows, use `Get-FileHash .\GRWeb-v0.2.1.zip -Algorithm SHA256`.

The [ZIP archive](https://github.com/mannetroll/grwebdeploy/releases/download/v0.2.1/GRWeb-v0.2.1.zip) contains the same folder if you prefer to extract locally and upload with SFTP. Download the named game archive, not GitHub's automatic **Source code** archive.

## 2. Install the complete folder

For a first installation:

```bash
mkdir -p /var/lib/grweb/releases /var/www
tar --no-same-owner -xzf GRWeb-v0.2.1.tar.gz -C /var/lib/grweb/releases
find /var/lib/grweb/releases/GRWeb-v0.2.1 -type d -exec chmod 755 {} +
find /var/lib/grweb/releases/GRWeb-v0.2.1 -type f -exec chmod 644 {} +
ln -s /var/lib/grweb/releases/GRWeb-v0.2.1 /var/www/grweb
```

The final command assumes `/var/www/grweb` is unused. If you already have a GRWeb installation, use [Update or roll back](#6-update-or-roll-back) instead. Nginx needs read and directory traversal access, not write access.

The extracted folder includes:

```text
GRWeb-v0.2.1/
├── index.html, favicon.ico, grweb.js, grweb-input.js, grweb.css
├── Build/
│   ├── Web.loader.js
│   ├── Web.framework.js.unityweb
│   ├── Web.wasm.unityweb
│   ├── Web.data.unityweb
│   └── WebASTC.data.unityweb
├── README.md, INSTALL.md, LICENSE
├── THIRD_PARTY_NOTICES.md, DATA_LICENSES.md, TRACK.md
├── deploy/nginx-grweb.conf
└── docs/, track/, windmills/
```

Keep every file, including both data files and all credits. The browser chooses the texture data suitable for its GPU. Keep `Build/` next to `index.html` and preserve filenames and directory case.

### AlmaLinux with SELinux enforcing

Check `getenforce`. If it reports **Enforcing**, install the management utility if needed and add a persistent label for the release tree once:

```bash
dnf install policycoreutils-python-utils
semanage fcontext -a -t httpd_sys_content_t '/var/lib/grweb(/.*)?'
restorecon -RFv /var/lib/grweb /var/www/grweb
```

Run `restorecon` on each newly extracted release before switching to it. If a custom rule already exists for that exact path, inspect it and use `semanage fcontext -m` to update it instead of adding a duplicate. Keep SELinux enabled.

## 3. Configure Nginx

Install the supplied location snippet:

```bash
mkdir -p /etc/nginx/snippets
install -m 644 /var/www/grweb/deploy/nginx-grweb.conf /etc/nginx/snippets/grweb.conf
```

Inside your domain's **existing HTTPS `server { ... }` block**, add this line once:

```nginx
include /etc/nginx/snippets/grweb.conf;
```

Depending on your server, that block may be in `/etc/nginx/nginx.conf`, `/etc/nginx/conf.d/` or `/etc/nginx/sites-available/`. Keep the site's existing TLS certificates, HTTP-to-HTTPS redirect and other applications. The snippet contains `location` blocks; do not put it at the top level or replace your entire configuration with it.

The [snippet](deploy/nginx-grweb.conf) maps `/grweb/` to `/var/www/grweb/`, redirects `/grweb` to `/grweb/` and returns 404 for missing files. It sets the file types and caching needed by this build. Its `^~` prefix prevents unrelated regex asset locations from intercepting game requests.

```bash
nginx -t && systemctl reload nginx
```

Only reload if validation succeeds. If you choose another filesystem path or URL prefix, update the snippet and the commands consistently. This example needs no new port or reverse proxy; it uses your existing HTTPS listener.

## 4. Required headers and other static hosts

These settings apply to Nginx and any other static host or CDN:

| File | Content-Type |
|---|---|
| `.html` | `text/html` |
| `.js` | `application/javascript` |
| `.css` | `text/css` |
| `.unityweb` | `application/octet-stream` |

The `.unityweb` files are compressed, but **Unity's loader decompresses them itself**. Serve their exact bytes with **no `Content-Encoding` header**. Do not add gzip/Brotli headers, rename them or apply another compression pass. This release uses decompression fallback, not server-side gzip decoding.

Use `Cache-Control: no-cache` with ETags or Last-Modified so reused filenames are revalidated after updates. Avoid immutable or long-lived caching for these files. A proxy/CDN must preserve the headers, revalidate updates and return an actual 404 for missing assets instead of your site's HTML fallback. Keep all game files on the same origin.

The build is single-threaded and needs no COOP/COEP setup. If your site already has a Content Security Policy, it must permit the game's scripts, WebAssembly and blob-based loading/audio; the browser console identifies blocked resources. A Permissions-Policy that disables fullscreen blocks the desktop fullscreen button.

For a temporary local preview, with Python 3 installed, run this **inside the extracted game folder**:

```bash
python3 -m http.server 8000 --bind 127.0.0.1
```

Open `http://127.0.0.1:8000/`. Double-clicking `index.html` with a `file://` URL will not work reliably. Use your HTTPS site for public hosting.

## 5. Check the installation

```bash
curl -I https://example.com/grweb
curl -I https://example.com/grweb/
curl -I https://example.com/grweb/Build/Web.loader.js
curl -I https://example.com/grweb/Build/Web.wasm.unityweb
curl -I https://example.com/grweb/Build/not-present.unityweb
```

Expect a redirect for `/grweb`, 200 for the page and existing assets, and 404 for the missing asset. The `.unityweb` response must be binary with no `Content-Encoding` and must require cache revalidation.

Open your game URL, wait for the download and scene preparation, then click **Start driving**. Test steering, gas/brake, sound and **O / Menu → Record Lap**. On iPhone, try both sliders together and open the menu. [Controls](README.md#play).

If it fails:

- **403:** check directory traversal/read permissions, symlink restrictions and SELinux labels.
- **Loading failure or black canvas:** inspect the browser Console/Network panels and Nginx's error log. Confirm that each requested file exists and has the expected type.
- **Decompression error:** verify the checksum and confirm no proxy changed the bytes or added `Content-Encoding`.
- **An asset request returns HTML:** remove an interfering rewrite or SPA fallback for the game's path.
- **No sound:** tap Start driving, enable sound in the game's menu and check media volume.
- **Old game after an update:** reload the page and check CDN/browser revalidation. A running session keeps its already-loaded version.

## 6. Update or roll back

For each version, download and verify its archive, extract its uniquely named folder under `/var/lib/grweb/releases/`, and set permissions as above. On SELinux systems, run `restorecon -RFv` on the extracted folder. Do not overwrite an existing release directory in place.

Once the desired release is ready, switch the existing symlink atomically. This example activates v0.2.1; substitute the new version when updating later:

```bash
ln -s /var/lib/grweb/releases/GRWeb-v0.2.1 /var/www/grweb-next
mv -Tf /var/www/grweb-next /var/www/grweb
```

These commands assume `/var/www/grweb` is the symlink created above and `grweb-next` is unused. Reload Nginx only when its configuration changes. The snippet disables open-file caching so subsequent requests follow the new link. Verify the deployment and ask players to reload. Keep the previous release directory until the new one is confirmed working.

To roll back, use the same two commands with the previous release's directory. All versions retain their own game files and documentation.
