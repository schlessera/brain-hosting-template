# brain-hosting-template

> **Placeholder.** This repository is not usable yet. It is here so that the
> work of building it has somewhere to live.

This will be the starting point for self-hosting the optional chat UI: a GitHub
template you generate from, ending with a private repository that builds and
runs your own instance.

```sh
gh repo create my-brain-host --template schlessera/brain-hosting-template --private --clone
```

## Why this is separate from `brain-template`

Two repositories, because they answer to different clocks.

- **[brain-template](https://github.com/schlessera/brain-template)** generates
  your *brain*: markdown you own, that only you write to, backed up and synced
  as your content.
- **This** generates your *hosting*: a container, a compose file, a reverse
  proxy. Infrastructure, which arrives by version bump.

Putting a Dockerfile in the repository that holds your notes would tie the two
together and make an infrastructure upgrade a commit in your content history.
`brain-kit`'s roadmap states the rule this follows: **your content repo stays
yours** — the toolchain is delivered by version bump and never merges into it.

Your brain is mounted into the container this repository builds. They meet at
runtime, not in git.

## What it will contain

A thin deployment shell over the published `@schlessera/brain-ui-server` and
`@schlessera/brain-ui-react` packages — **no application behaviour of its own**.
Every line of what the chat UI does lives in
[schlessera/brain-kit](https://github.com/schlessera/brain-kit) and arrives from
the registry.

Expected: a Dockerfile, a compose file, a reverse-proxy config, branding, an
entrypoint, and the environment contract. Extracted from a real working
deployment rather than written from scratch, so that what ships is something
that has actually run.

## What it will not contain

- **Application behaviour.** If it is a feature, it belongs in brain-kit.
- **Anybody's deployment specifics.** No hosts, no domains, no identifiers.
- **Your content.** That is `brain-template`, mounted in.

## Where the work is tracked

In [schlessera/brain-kit](https://github.com/schlessera/brain-kit/issues) until
this repository has enough shape to carry its own.

## License

MIT
