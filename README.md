# Nosdesk GitHub Integration Plugin

Links Nosdesk tickets to GitHub issues for context alongside support work.

This is an **official** Nosdesk plugin, signed by the Nosdesk root key
and distributed via the Nosdesk registry at
<https://nosdesk.com/registry/>.

## What it does

- **Link issues**: search and link GitHub issues to Nosdesk tickets.
- **View status**: see issue state (open/closed), labels, and recent
  activity without leaving the ticket.
- **Quick access**: click linked issues to open them in GitHub.

## Installing

End users don't clone this repo. Install via the Nosdesk admin UI:

1. Administration, Plugins, Registry
2. Find "GitHub Integration" and click Install

The backend downloads the signed zip from this repository's latest
release, verifies the Nosdesk root signature, and installs.

## Configuration

Create a GitHub Personal Access Token at
<https://github.com/settings/tokens>. The fine-grained token flow
needs only **Issues: Read-only** on the repositories you want
visibility into.

In the Nosdesk admin UI (Administration, Plugins, settings):

- `github_token` (secret): the PAT from above
- `default_owner` (optional): e.g. `nosdesk`, used for text search
- `default_repo` (optional): e.g. `helpdesk`

Tokens are encrypted at rest via the Nosdesk instance's configured
AES-256-GCM key path; the backend proxy injects
`Authorization: Bearer <token>` for requests to `api.github.com`
automatically.

## Using

1. Open any ticket.
2. Find the "GitHub Issues" panel in the sidebar.
3. Click "Link Issue" to search. Enter either:
   - A direct reference: `nosdesk/helpdesk#42`
   - Free-text search (requires `default_owner` + `default_repo`).
4. Click a result to link it. Hover a linked issue for the unlink
   button.

## Release process (maintainers)

Releases are tagged `vX.Y.Z` on this repo. Each release has a
signed zip attached as an asset; that signed zip is what the
Nosdesk registry points at.

Per-release workflow:

1. Bump `manifest.json` version, commit, tag (`git tag v0.2.0` then
   `git push --tags`).
2. Build `dist/` if applicable (this plugin currently ships
   `bundle.js` as a committed artefact; a future source+build
   layout will put `dist/` under a CI step).
3. Sign from a machine with the Nosdesk root key:

   ```
   nosdesk-cli plugin sign ./dist \
       --key ~/.nosdesk-root/root-1e0fad90.sk \
       --out github-integration-0.2.0.zip \
       --source nosdesk-root
   ```

4. Create the GitHub release and attach the zip:

   ```
   gh release create v0.2.0 github-integration-0.2.0.zip \
       --title 'GitHub Integration 0.2.0' \
       --notes-file RELEASE_NOTES.md
   ```

5. Compute the zip's SHA-256:

   ```
   shasum -a 256 github-integration-0.2.0.zip
   ```

6. Open a PR on `Nosdesk/nosdesk-com` adding the version entry
   under
   `apps/marketing/registry-source/plugins/github-integration/versions.toml`
   with the release asset URL + sha256. CI re-signs and publishes.

## Source layout

Current layout (single committed `bundle.js` for simplicity):

```
manifest.json       # plugin identity, component + settings definitions
bundle.js           # ES module with default export { GitHubPanel: ... }
```

When the plugin grows past the "ship the artefact directly" phase,
move to:

```
src/                # TS/JS source
package.json        # build deps
dist/               # build output (gitignored)
.github/workflows/
    build.yml       # produces dist/ on tag push
```

## Plugin API

This plugin uses:

- `api.storage.get` / `api.storage.set`: persist linked issues per
  ticket.
- `api.fetch`: proxied requests to GitHub API with automatic auth
  header injection.
- `context.ticket`: access the current ticket.
- Component rendering in the `ticket-sidebar` slot.

Components receive `(api, context)` as props.

## Trust model

This plugin is signed by the Nosdesk root key. Nosdesk 1.1 and
later, which compile the root key in, verify the signature
locally before executing any bundle bytes. The embedded signature
is also cross-checked against the registry-published
`publisher_pubkey`, `tier`, and plugin name, so a mix-up between
registry claim and shipped bytes is refused at install time.

## License

See `LICENSE`.
