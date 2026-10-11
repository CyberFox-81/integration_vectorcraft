<!--
  - SPDX-FileCopyrightText: 2026 Britton Couchman <brittondcouchman@gmail.com>
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
# VectorCraft integration for Nextcloud

Open and edit the files in your Nextcloud with [VectorCraft](https://github.com/storytold/vectorcraft), the open-source, Rust-native vector illustration app, without leaving the browser.

- **Edit in VectorCraft** appears in the Files action menu of every SVG, Illustrator (.ai), EPS and VectorCraft file you can read.
- **Save** (Ctrl/⌘+S) writes the drawing back to Nextcloud in its own format, and Nextcloud keeps the previous state as a file version.
- **Save As**, **Save a Copy** and **Export** (PDF, PNG, JPEG, WebP, SVG…) ask for a name and folder in your Nextcloud the first time; VectorCraft's own Save As dialog picks the name and format first.
- If someone else changes the file while you have it open, saving asks whether to replace their version or save a copy.
- Read-only files (for example a share without edit rights) open normally; saving offers a copy.
- Optionally, clicking a drawing in Files opens it in VectorCraft: tick **Open drawings here from Files** in the editor's top bar.

VectorCraft runs entirely in your browser as WebAssembly. Files go only between your browser and your Nextcloud; there is no extra server to run.

## Requirements

- Nextcloud 33, 34 or 35
- A browser with WebGPU or WebGL2 (any current Chrome, Edge, Firefox or Safari)
- The web server must serve `.wasm` files as `application/wasm`. Nextcloud's own `.htaccess` does this on Apache; the [recommended nginx configuration](https://docs.nextcloud.com/server/latest/admin_manual/installation/nginx.html) does too.
- Recommended: gzip or Brotli for `application/wasm`. VectorCraft is fetched once and then cached.

## Install

Until the app is in the Nextcloud App Store, the latest version is in [CyberFox-81/nextcloud_plugins](https://github.com/CyberFox-81/nextcloud_plugins), whose README has step-by-step commands for Docker, Nextcloud AIO and regular servers. In short:

1. Download [`integration_vectorcraft.tar.gz`](https://github.com/CyberFox-81/nextcloud_plugins/raw/main/integration_vectorcraft/integration_vectorcraft.tar.gz) and extract it into your Nextcloud's `custom_apps/` (or `apps/`) folder, so you have `custom_apps/integration_vectorcraft/appinfo/info.xml`.
2. Make it readable by the web server user, e.g. `chown -R www-data:www-data custom_apps/integration_vectorcraft`.
3. Enable it: `sudo -u www-data php occ app:enable integration_vectorcraft`. In Docker: `docker exec -u www-data nextcloud php occ app:enable integration_vectorcraft`.

To make VectorCraft the click-to-open default for every user who hasn't chosen otherwise:

```sh
sudo -u www-data php occ config:app:set integration_vectorcraft open_by_default --value=true --type=boolean
```

## How it works

```
Files ── "Edit in VectorCraft" ──▶ /apps/integration_vectorcraft/edit/{fileId}   (src/editor.ts)
                                  │  WebDAV GET/PUT with the user's session
                                  ▼
                                <iframe> /apps/integration_vectorcraft/frame?host=parent   (templates/frame.php)
                                  │  VectorCraft (web/*.wasm) — postMessage bridge
```

The editor page downloads the file over WebDAV and passes it to VectorCraft with `postMessage`. When you save, VectorCraft passes the bytes back and the page uploads them with `If-Match`, so a newer version on the server is never overwritten without asking. A save in a format the opened file isn't in, and an export, ask for a name and folder with Nextcloud's own folder picker the first time; later saves under the same name go to the same file. The frame page is the only one whose Content Security Policy allows compiling WebAssembly.

VectorCraft's browser build saves by downloading a file. `patches/vectorcraft-nextcloud.patch` adds an opt-in embedding bridge (`?host=parent`) to its web shell; the protocol is in [web/README.md](web/README.md). It is meant to be proposed upstream, so a future official VectorCraft build can be used as is.

## Build from source

```sh
make js      # npm ci && vite build → js/
make wasm    # clone VectorCraft, apply the patch, build → web/   (needs Rust, see below)
make dist    # → build/dist/integration_vectorcraft-<version>.tar.gz
```

`make wasm` runs `scripts/build-vectorcraft-web.sh`, which needs the `wasm32-unknown-unknown` Rust target, `wasm-bindgen-cli` at the version in VectorCraft's `Cargo.lock` (0.2.129 at the pinned commit), and optionally `wasm-opt` from binaryen. Set `VECTORCRAFT_REF` to build another VectorCraft commit, `FAST=1` for quicker development builds, and `CRAFT_FONTS_DIR` to a clone of [storytold/craft-fonts](https://github.com/storytold/craft-fonts) to embed the shared Japanese fonts.

For development, `npm run watch` rebuilds the scripts as you edit; symlink or copy the repository into `custom_apps/integration_vectorcraft` of a test server.

## Releasing to the App Store

One-time setup (see the [App Store developer guide](https://nextcloudappstore.readthedocs.io/en/latest/developer.html)):

1. Create the app's key and certificate request, and keep the key private and backed up:
   ```sh
   mkdir -p ~/.nextcloud/certificates && cd ~/.nextcloud/certificates
   openssl req -nodes -newkey rsa:4096 -keyout integration_vectorcraft.key -out integration_vectorcraft.csr -subj "/CN=integration_vectorcraft"
   ```
2. Submit `integration_vectorcraft.csr` as `integration_vectorcraft/integration_vectorcraft.csr` in a pull request to [nextcloud/app-certificate-requests](https://github.com/nextcloud/app-certificate-requests). Save the signed certificate they reply with as `~/.nextcloud/certificates/integration_vectorcraft.crt`.
3. Register the app at [apps.nextcloud.com/developer/apps/new](https://apps.nextcloud.com/developer/apps/new) with the certificate and this signature:
   ```sh
   echo -n "integration_vectorcraft" | openssl dgst -sha512 -sign ~/.nextcloud/certificates/integration_vectorcraft.key | openssl base64
   ```

Each release:

```sh
make js && make wasm
scripts/sign-release.sh --key ~/.nextcloud/certificates/integration_vectorcraft.key \
  --cert ~/.nextcloud/certificates/integration_vectorcraft.crt --docker nextcloud
```

The script code-signs the app with `occ integrity:sign-app` (borrowed from a running Nextcloud container, or `--occ` for a local install; `--archive` signs a prebuilt archive), packs `integration_vectorcraft-<version>.tar.gz` and prints the App Store signature. Attach the archive to a GitHub release, then submit its download URL and the signature at [apps.nextcloud.com/developer/apps/releases/new](https://apps.nextcloud.com/developer/apps/releases/new).

Every new version also goes to [nextcloud_plugins](https://github.com/CyberFox-81/nextcloud_plugins), replacing the previous one, with that repository's script (from a clone next to this one):

```sh
../nextcloud_plugins/scripts/update-plugin.py build/dist/integration_vectorcraft-<version>.tar.gz
```

## Limitations

- VectorCraft is young; see its [roadmap](https://github.com/storytold/vectorcraft/blob/main/ROADMAP.md) for what's still missing.
- One person edits at a time. There is no live co-editing; concurrent saves are caught and you choose what happens.
- PDFs open in VectorCraft from its own File › Open; in Files, PDFs are left to PDF apps.

## License

This app is licensed under the AGPL-3.0-or-later. VectorCraft, bundled in `web/`, is licensed under MIT OR Apache-2.0; its bundled fonts and assets carry their own open licenses (see `web/VECTORCRAFT-ATTRIBUTION.md` or `web/VECTORCRAFT-NOTICE`). This is a community integration based on VectorCraft by the ArtCraft team, not made or endorsed by them.
