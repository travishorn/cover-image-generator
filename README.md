# Cover

Small single-page SvelteKit app for making 1200x630 cover images.

[travishorn.github.io/cover-image-generator](https://travishorn.github.io/cover-image-generator)

## Local development

```sh
npm install
npm run dev
```

## Production checks

```sh
npm run check
npm run lint
npm run build
```

## GitHub Pages deployment

This repo is configured for GitHub Pages static deployment via GitHub Actions.

### What is already configured

- `@sveltejs/adapter-static` in `svelte.config.js`
- `BASE_PATH` support in `svelte.config.js` so asset paths work on Pages
- Workflow at `.github/workflows/deploy.yml` that builds and deploys to Pages
