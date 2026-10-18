# go-crdt/docs

Versioned documentation for [go-crdt](https://github.com/go-crdt), built with
[Hugo](https://gohugo.io/) and this fleet's
[Hextra fork](https://github.com/tannevaled/hextra). Published to the
`gh-pages` branch and served at <https://go-crdt.github.io/docs/>.

⛔ What this replaced was MkDocs Material plus mike, which needed a Python
toolchain, a `requirements.txt` and a `pip install` in CI to render a static
site for a pure-Go organisation.

## Local preview

```bash
git submodule update --init --recursive   # themes/hextra, pinned at a release tag
hugo server                               # http://localhost:1313
```

## Versions

Each documented version is its own build under `/docs/<version>/`, and
`/docs/latest/` follows the newest — which is what `mike` did before. The version
comes from `params.docs.version` in `hugo.yaml`: set it to the release the pages
describe (`v0.16.0` → `/docs/0.16/`), or leave it at `main` while no release is
tagged. The `docs` workflow reads it, builds with that `--baseURL`, and replaces
**only** that version's path.

⛔ The build runs with `--panicOnWarning`, so a link to a page that is not there
fails the lane instead of 404ing for a reader.
