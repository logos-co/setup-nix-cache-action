# setup-nix-cache-action

Install Nix with the self-hosted **Logos Attic binary cache**
([`cache.nix.logos.co`](https://cache.nix.logos.co), see
[infra-ci#263](https://github.com/status-im/infra-ci/issues/263))
preconfigured, and publish what the job builds:

- every ref **pulls** from both the `public` and `ci` caches — they are public
  to read, so this needs no credentials and works on fork PRs;
- `master`/`main` **pushes** to the `public` cache, every other ref to the
  `ci` cache (publishing happens in a post step at the end of the job).

## Usage

```yaml
jobs:
  build:
    # ATTIC_TOKEN_PUBLIC only exists in the public-cache environment; master
    # jobs must opt into it to publish to the public cache.
    environment: ${{ github.ref == 'refs/heads/master' && 'public-cache' || '' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: logos-co/setup-nix-cache-action@v1
        with:
          attic-token-ci: ${{ secrets.ATTIC_TOKEN_CI }}
          attic-token-public: ${{ secrets.ATTIC_TOKEN_PUBLIC }}

      - run: nix build -L
```

That's the whole integration. The action picks the cache and token from the
ref; when the matching token is empty (fork PRs get no secrets at all, and
`ATTIC_TOKEN_PUBLIC` is empty outside the `public-cache` environment) it
skips publishing instead of failing — pulling always works. The endpoint is
baked in: it is public knowledge, not a secret.

The repo must be listed in
[infra-ci's `attic.yml`](https://github.com/status-im/infra-ci/blob/master/ansible/group_vars/attic.yml)
for the secrets to exist; ask infra to add it and run the ansible role.

## Inputs

| input | default | notes |
|---|---|---|
| `attic-endpoint` | `https://cache.nix.logos.co` | override only for testing another Attic deployment |
| `attic-token-ci` | `''` | pass `secrets.ATTIC_TOKEN_CI` (repo-wide) |
| `attic-token-public` | `''` | pass `secrets.ATTIC_TOKEN_PUBLIC` (public-cache environment) |
| `cache` | auto | override the publish cache (`public` on master/main, `ci` otherwise) |
| `install-nix` | `true` | set `false` if the workflow installs Nix itself |
| `extra-nix-config` | `''` | extra lines for the Nix config at install time |
| `include-paths` | `''` | newline-separated regexes; push only matching store paths |
| `exclude-paths` | `''` | newline-separated regexes; never push matching store paths |
| `skip-push` | `''` | `true` keeps the substituters but pushes nothing |

### Not pushing what a PR can't reuse

A PR job's leaf artifacts (bundles, AppImages, `.app`s) are rebuilt on every
new commit and after merge, so uploading them to `ci` only costs time; large
NARs can take several minutes even when Attic deduplicates them. Exclude them
on pull requests and keep publishing them from the default branch:

```yaml
      - uses: logos-co/setup-nix-cache-action@v1
        with:
          exclude-paths: ${{ github.event_name == 'pull_request' && '-my-app-bundle-' || '' }}
```

## Bring your own Nix installer

With `install-nix: false` the action only sets up publishing; add the
substituters to your installer's config yourself:

```
extra-substituters = https://cache.nix.logos.co/public https://cache.nix.logos.co/ci
extra-trusted-public-keys = public:l4HrXgL4nw246+LBh2SOJyhz64BoGegOYLheT/iIAPU= ci:aVJqjS4NWX5WfqHO0AEhIScjGu/JyK5FoydPrnEbMvc=
fallback = true
```

## Local builds

This action is CI-only. For local builds, add the **public** cache (never
`ci` — it holds unreviewed PR builds) to the repo's `flake.nix`:

```nix
nixConfig = {
  extra-substituters = [ "https://cache.nix.logos.co/public" ];
  extra-trusted-public-keys = [ "public:l4HrXgL4nw246+LBh2SOJyhz64BoGegOYLheT/iIAPU=" ];
};
```

## Key rotation

The cache signing keys are baked into this action (and served live at
`https://cache.nix.logos.co/_api/v1/cache-config/<cache>`). When infra
rotates them, update them here and every repo referencing `@v1` picks the
change up on its next run; only `flake.nix` copies need separate PRs.

## License

Licensed and distributed under either of

- MIT license: [LICENSE-MIT](LICENSE-MIT) or https://opensource.org/licenses/MIT
- Apache License, Version 2.0: [LICENSE-APACHE](LICENSE-APACHE) or https://www.apache.org/licenses/LICENSE-2.0

at your option. These files may not be copied, modified, or distributed except according to those terms.
