# Locally hosted Ultrachromic

This directory contains the security-reviewed Ultrachromic snapshot from
commit [`136760012e62b2f5712808febc377b04386d581b`](https://github.com/CTalvio/Ultrachromic/tree/136760012e62b2f5712808febc377b04386d581b),
merged on 2026-09-09. It is mounted read-only at
`/usr/share/jellyfin/web/custom-themes` in the Jellyfin container. The unique
destination avoids masking any directory shipped by Jellyfin Web itself.

The deployment deliberately has no jsDelivr, GitHub Pages, or
`raw.githubusercontent.com` dependency. The background image is stored under
`assets/` and served from the same Jellyfin origin. `saved.css` was excluded
because it imports unpinned styles from older external projects.

Google Fonts is the sole external dependency. `jf_font.css` requests Plus
Jakarta Sans through `https://fonts.googleapis.com`; Google's returned CSS
normally fetches the font files from `https://fonts.gstatic.com`. Remove the
first `@import` from a preset to disable Google Fonts.

## Enable a preset

Start or recreate Jellyfin after adding the Compose mount:

```sh
cd media-server
docker compose up -d jellyfin
```

In Jellyfin, open **Dashboard -> Branding -> Custom CSS** and enter exactly one
of these same-origin imports.

Monochromic:

```css
@import url('/web/custom-themes/ultrachromic/presets/monochromic_preset.css');
```

Kaleidochromic:

```css
@import url('/web/custom-themes/ultrachromic/presets/kaleidochromic_preset.css');
```

Novachromic:

```css
@import url('/web/custom-themes/ultrachromic/presets/novachromic_preset.css');
```

If Jellyfin is published below a base path, prefix the URL with that base path.
For example, a `/jellyfin` deployment uses
`/jellyfin/web/custom-themes/ultrachromic/...`.

Before saving the setting, the selected stylesheet can be checked directly at:

```text
http://SERVER_IP:8096/web/custom-themes/ultrachromic/presets/monochromic_preset.css
```

After changing presets, reload the client without cache. Custom CSS affects
Jellyfin Web-based clients; native clients that do not embed Jellyfin Web will
ignore it.

If a reverse proxy sets Content Security Policy headers, retain its existing
policy and add only `https://fonts.googleapis.com` to `style-src` and
`https://fonts.gstatic.com` to `font-src` when those origins are not already
permitted. Do not copy the broad CSP example from the upstream project.

## Local changes to the reviewed snapshot

- All preset component imports point to files in this directory.
- All Ultrachromic background URLs point to the bundled PNG.
- The obsolete Jellyfin `banner-dark.png` override was removed from the light
  style so Jellyfin 10.11.11 can retain its built-in logo styling.
- `security-overrides.css` restores Jellyfin's Forgot Password button, which
  the upstream login styles hide.
- `saved.css` is not deployed.
- Inherited trailing whitespace was normalized without changing CSS behavior.

The remaining CSS and PNG came from the pinned commit above. No automated
update mechanism is used. Review and vendor a new commit explicitly before
upgrading this snapshot.

Verify the deployed files from this directory with:

```sh
shasum -a 256 -c SHA256SUMS
```
