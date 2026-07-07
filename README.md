# dial9-rs.github.io

Source for the [dial9-rs](https://dial9-rs.github.io) blog. Built with [Zola](https://www.getzola.org/) using the [cela](https://github.com/edwardzcn-decade/cela) theme (vendored as a git submodule).

## Serving locally

The theme lives in a git submodule, so make sure it's checked out first:

```sh
git submodule update --init --recursive
```

Install Zola (macOS):

```sh
brew install zola
```

Then start the dev server with live reload:

```sh
zola serve
```

The site is served at http://127.0.0.1:1111 and rebuilds automatically as you edit.

To produce a static build in `public/` (what CI deploys):

```sh
zola build
```

## Deployment

Pushing to `main` triggers the GitHub Actions workflow in `.github/workflows/`, which builds the site and publishes it to GitHub Pages.
