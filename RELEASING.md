# Releasing cargo-proofs

The CLI lives at the root of this repository. The
[existing crate](https://crates.io/crates/cargo-proofs) keeps its name, owners,
published versions, and install command. The initial split has CI only.
The proposed release workflow lives in `.github/templates/release-cli.yml`
and must not be activated until the cutover gates below pass. It does not
deploy the service. Versions already on crates.io cannot be overwritten.

## One-time cutover

1. Review the CLI-only snapshot and confirm both license texts and the
   preview's third-party notices ship in the package. Do not push tags that
   already represent published releases.
2. Create the GitHub Actions environment `crates-io` in
   `proofs-rs/cargo-proofs`. Configure required reviewers, disable
   self-review, and restrict deployment refs to approved release tags and the
   main branch (for manual retry). Protect release tags against unreviewed
   creation, deletion, and movement. Never enable publication without a
   working environment gate. Then set the environment variable
   `CARGO_PROOFS_TRUSTED_PUBLISHING_ENABLED=true` in GitHub Actions settings for
   that environment, not at repository scope.
3. In the crate's Trusted Publishing settings add owner `proofs-rs`,
   repository `cargo-proofs`, workflow filename `release-cli.yml`, environment
   `crates-io`. Confirm the organization/repository and workflow are the
   intended ones; remove the old publisher when retiring the old workflow.
   This existing crate does not need another API token or bootstrap release.
   See [crates.io Trusted Publishing](https://crates.io/docs/trusted-publishing).
4. Change public documentation links and service checkout/CI to use the new
   repository. Once the exact-commit ingestion gate below passes and the
   environment and publisher are configured, review and install the template
   as `.github/workflows/release-cli.yml` in a separate PR. Verify the first
   new publication with OIDC; dry runs do not exercise Trusted Publishing.
   Public package metadata changes with the next actual version.

## Generated artifacts and compatibility

The proofs.rs service maintains the canonical ingestion contract and preview
renderer. The CLI checks in only these generated outputs:

- `protocol/recording-v2.schema.json`
- `preview/server.cjs`
- `preview/THIRD_PARTY_NOTICES.txt`

When those outputs or the protocol change, service maintainers regenerate
them with pinned dependencies, review and commit the generated outputs here,
and advance the exact CLI commit used in service CI. The service CI compares
the generated content byte for byte. Native validation and its integration
suite still check behavior the JSON Schema cannot express.

Before a release reviewer approves the public `crates-io` environment, require
a successful private service CI run against the **same CLI commit SHA**. The
run must check fresh generation, build the crates.io archive, build the CLI
from that archive, exercise its local preview, and run production CLI
exports against production ingestion handlers in the local service fixture.
Record the verification run and both commit SHAs internally, without exposing
private repository details or test logs in the public workflow. Public CI does
not hold private-repo credentials. A coordinated server change must be
deployed before a CLI release that needs it; public CI alone cannot establish
production compatibility.

## Routine release

1. Update `Cargo.toml` and the package version in `Cargo.lock` together in a PR.
2. Merge after checks. Run the service compatibility gate for this exact
   commit. Ensure the version is not already published.
3. With release approval, tag the validated main commit and push the tag:
   `git tag -a cargo-proofs-vX.Y.Z -m 'cargo-proofs X.Y.Z'`.
4. The workflow revalidates tag/version, main ancestry, license, CLI/Lean
   tests, preview, and package dry-run. The protected environment reviewer
   matches the private compatibility evidence and allows the OIDC publication.
5. Verify the new version and its metadata on crates.io. If publication failed
   and the version does not exist, use workflow dispatch on main with the
   existing tag. Do not move the tag or replay an already-published version.
