# trusted-builder

Reusable GitHub Actions workflow used by the "Signed Isn't Enough" demo. It builds a container image from the calling repository, then:

1. pushes it to GHCR and signs it keyless with Cosign v3,
2. records an SPDX SBOM and, if the caller has `vex/openvex.json`, an OpenVEX document in GitHub's attestation store (not the registry, to keep admission checks fast),
3. attaches SLSA v1 provenance with `actions/attest`.

Because the build and every signature happen inside this workflow, a caller cannot tamper with how its image is built or signed (SLSA Build L3). All signatures carry this workflow's identity:

```
https://github.com/vuthongkms/trusted-builder/.github/workflows/build-sign-attest.yml@refs/tags/v1
```

The provenance (`buildDefinition.externalParameters.workflow`) records which repository and ref called the builder, so a verifier must check those fields too. The identity alone only says "built by the trusted builder", not "built from my repository's reviewed branch".

## Usage

```yaml
jobs:
  build:
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    uses: vuthongkms/trusted-builder/.github/workflows/build-sign-attest.yml@v1
    with:
      image: ghcr.io/<owner>/<repo>
      tag: ${{ github.ref_name }}
```

## Verify

```bash
cosign verify ghcr.io/<owner>/<repo>@sha256:... \
  --certificate-identity https://github.com/vuthongkms/trusted-builder/.github/workflows/build-sign-attest.yml@refs/tags/v1 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com

gh attestation verify oci://ghcr.io/<owner>/<repo>@sha256:... -R <owner>/<repo> \
  --signer-workflow vuthongkms/trusted-builder/.github/workflows/build-sign-attest.yml \
  --source-ref refs/heads/main
```
