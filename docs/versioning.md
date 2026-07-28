# Versioning

This action follows the [standard GitHub Actions release convention](https://github.com/actions/toolkit/blob/main/docs/action-versioning.md):

- Every release gets an immutable semver tag: `v1.0.0`, `v1.1.0`, `v1.1.1`, …
- The major-version alias `v1` is a **floating tag**: it always points at the
  latest `v1.x.y` release. Consumers referencing `@v1` pick up fixes (key
  rotations, bug fixes) on their next run without changing their workflows.
- Breaking changes (removed/renamed inputs, behavior changes) go to a new
  major: tag `v2.0.0`, start a `v2` alias, leave `v1` untouched.

## Cutting a release

```sh
git tag -m vX.Y.Z vX.Y.Z <commit>
git tag -fm vX vX <commit>
git push origin vX.Y.Z
git push -f origin vX
```

## Pinning

`@v1` is the default and what the Logos repos use. Pin an exact `@vX.Y.Z`
(or a commit SHA) if a workflow must not change behavior without a PR.
