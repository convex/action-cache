# Developing guide

## Running locally

```sh
npm ci
npm run dev
```

## Testing

```sh
npm run build
npm run check
npm run test:e2e
```

When component APIs change, regenerate and commit the generated files:

```sh
npm run build:codegen
```

## Releases

Include a changeset with changes that need a release:

```sh
npx changeset
```

Choose the version bump, describe the user-visible change, and commit the
generated file with your implementation. Docs-only and development-tooling
changes do not need a changeset.

Review and merge the release PR when ready. Click **Approve workflows to run**
on the PR if its checks are waiting for approval. To refresh it manually, run
the [Update release PR workflow](./.github/workflows/version.yml) on main.

To release, tag the merged release PR's commit with `v<version>`. This starts
the [Release workflow](./.github/workflows/release.yml):

```sh
sha=$(gh pr view <release-pr> -R convex/action-cache --json mergeCommit -q .mergeCommit.oid)
gh api repos/convex/action-cache/git/refs -f ref=refs/tags/v1.2.3 -f sha="$sha"
```

Or, from a local clone: `git tag v1.2.3 <commit> && git push origin v1.2.3`.

The tag must match `package.json`'s version. Stable releases must be on `main`;
the commit may be older than the current tip. The workflow builds and tests that
commit, stages the tested package on npm, then creates a GitHub Release with the
version's changelog notes.

Review the package on npm's **Staged Packages** page and approve with 2FA, or
run:

```sh
npm stage approve <stage-id>
```

Use the stage ID from the workflow log. Approve versions in order to avoid
moving `latest` backward.

### Local preparation and prereleases

To prepare a release locally instead of using the bot:

```sh
npm ci
npm run build
npm run check
npx changeset version
```

Review and commit the version, changelog, and consumed changesets, then merge
through a checked PR.

For a branch prerelease, prepare a prerelease version with Changesets on that
branch and tag its commit, such as `v1.2.3-alpha.0`. The first prerelease
identifier selects the npm channel, such as `alpha`.

### Recovery

The tag records intent to release; npm is the source of truth for what was
staged. If the workflow fails, check npm's staged packages first:

- Not staged: rerun the workflow, or fix the source and move the tag to the
  corrected commit. Rerunning a run uses its original commit.

  ```sh
  gh api -X PATCH repos/convex/action-cache/git/refs/tags/v1.2.3 -f sha=<commit> -F force=true
  ```

- Staged, but the GitHub Release step failed: rerun the failed job.
- Staged, but rejected in review: `npm stage reject <stage-id>` frees the
  version, so you can move the tag to a corrected commit. A published version
  cannot be reused; prepare a new version instead.
