# Djinbar — website

Independent website for the Djinbar Chrome extension.

> This first pass deliberately keeps the site small. The visual identity, full marketing landing
> page, Chrome Web Store URL, legal pages, and final social assets still need dedicated work.

## Product

**The bookmarks bar, rethought.**

Turn your bookmarks bar into contextual workspaces, always within reach while you browse.

## Stack

- Astro 7, statically generated
- npm and Node.js 22.12 or newer
- Docker multi-stage build
- Nginx runtime on port 80
- No runtime JavaScript dependency or environment variable

## Local development

```sh
npm ci
npm run check
npm run build
npm run dev
```

The development server runs on `http://localhost:4321`. The production output is generated in
`dist/`.

## Deployment

Dokploy should build the repository from its root with the checked-in `Dockerfile`. The resulting
container listens on port `80`; no custom install, build, start, publish-directory, or environment
settings are required when Dockerfile mode is selected.

Canonical production URL: `https://djinbar.com`.

See [`docs/DOKPLOY.md`](docs/DOKPLOY.md) for the exact GitHub, Dokploy, domain, and DNS setup.

## Before the public launch

- Set `chromeStoreUrl` in `src/lib/config.ts` when the extension listing is public.
- Replace the provisional text mark and generated SVG favicon with the final Djinbar identity.
- Add dedicated Open Graph artwork.
- Draft and review Djinbar-specific legal, privacy, and support pages; do not reuse Djinlist copy.
