# Du clic au back-end — Piloter l'observabilité depuis le front-end avec OpenTelemetry

Slides de la présentation donnée au **Séminaire du développement 2026**.

## Voir les slides

👉 https://ddecrulle.github.io/seminaire-dev-otel-2025/

## Modifier les slides

Les slides sont écrites en Markdown avec [Marp](https://marp.app/) dans `slides.md`.

```bash
# Prévisualiser en live
npx @marp-team/marp-cli --theme wave.css --html --allow-local-files --watch slides.md

# Générer le HTML
npx @marp-team/marp-cli --theme wave.css --html --allow-local-files slides.md -o slides.html
```

## CI/CD

À chaque push sur `main`, GitHub Actions génère les slides et les publie automatiquement sur GitHub Pages.
