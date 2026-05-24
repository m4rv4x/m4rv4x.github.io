# m4rv4x.github.io

Personal blog and notes hub for experiments, write-ups, and technical posts.

## Live site

- https://m4rv4x.github.io

## Stack

- Jekyll
- Chirpy theme
- GitHub Pages

## Local development

Install the Ruby dependencies first:

```bash
bundle install
```

Then use the repo helpers:

```bash
./tools/run.sh
```

This wraps `bundle exec jekyll s -l` with the repo defaults.

For remote/dev-container testing you can bind a different host:

```bash
./tools/run.sh --host 0.0.0.0
```

If you prefer the raw Jekyll command, this still works:

```bash
bundle exec jekyll serve
```

Then open:

```text
http://localhost:4000
```

## Validation

Before pushing content changes, run the repo test helper:

```bash
./tools/test.sh
```

That production-builds the site and runs `htmlproofer` with the repo's local URL ignores.

## Repository structure

```text
_posts/      Blog posts
_tabs/       Custom pages
_data/       Site data
assets/      Static assets
_config.yml  Site configuration
```

## Purpose

This repo is where I publish notes, posts, and small technical write-ups around tooling, infra, self-hosting, and experiments.

## License

MIT
