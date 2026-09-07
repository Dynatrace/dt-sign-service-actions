# dt-sign-service-actions

Reusable GitHub Actions workflows for signing binaries and installers with **Google Cloud KMS**:

- **PKCS#11** for Linux (via OpenSSL CMS)
- **CNG (Cryptography Next Generation)** for Windows (via `signtool.exe`)

Private keys never leave Google Cloud KMS. Authentication to GCP uses Workload Identity Federation, so no long-lived service-account keys live in GitHub.

## Prerequisites

### Caller repository secrets

| Secret | Description |
|---|---|
| `SIGNING_CODE_SIGNING_CERT` | Code-signing certificate content in PEM format |
| `SIGNING_GCP_KMS_KEY` | Name of the GCP KMS signing key to use |
| `SIGNING_GCP_KMS_KEYRING` | GCP KMS Keyring name |
| `SIGNING_GCP_KMS_LOCATION` | GCP KMS location |
| `SIGNING_GCP_PROJECT_ID` | GCP Project ID where KMS resources are located |
| `SIGNING_GCP_SERVICE_ACCOUNT` | GCP Service Account email for signing |
| `SIGNING_GCP_WORKLOAD_IDENTITY_PROVIDER` | GCP Workload Identity Provider resource name |

These secrets are typically defined at the GitHub organization level and scoped to the allow-listed caller repositories.

It is strongly recommended to scope these to a GitHub Environment (e.g. `release`) gated with required reviewers.

The caller job that invokes the workflow must grant `id-token: write` so the runner can mint the OIDC token used by WIF.

## Usage

> Replace `@<ref>` with a tag (e.g. `@v1`) or commit SHA. Avoid `@main` for production.

### Sign a Linux binary or installer

```yaml
sign:
  name: Sign installer
  needs: build
  uses: dynatrace/dt-sign-service-actions/.github/workflows/sign-linux.yml@v1
  permissions:
    contents: read
    id-token: write
  with:
    file-to-sign: 'my-installer.sh'
    # unsigned-artifact-name: 'file-to-sign'   # default; override if signing multiple artifacts in one workflow
    # signed-artifact-name: 'file-signed'      # default; override if signing multiple artifacts in one workflow
    # detached-signature: true                 # default true (detached PEM sidecar); false produces an enveloped CMS blob
    # runs-on: 'ubuntu-latest'                 # default 'ubuntu-latest'; pass a self-hosted label for private runners
    # The signed file is written next to the input as `<file-to-sign>.sig` (detached) or `<file-to-sign>.signed` (enveloped).
  secrets: inherit
```

### Sign a Windows binary

```yaml
sign:
  name: Sign Windows executable
  needs: build
  uses: dynatrace/dt-sign-service-actions/.github/workflows/sign-windows.yml@v1
  permissions:
    contents: read
    id-token: write
  with:
    file-to-sign: 'my-application.exe'
    # unsigned-artifact-name: 'file-to-sign'   # default; override if signing multiple artifacts in one workflow
    # signed-artifact-name: 'file-signed'      # default; override if signing multiple artifacts in one workflow
    # digest-alg: 'sha384'                     # default
    # append-signature: false                  # default false; true for dual signing
    # runs-on: 'windows-latest'                # default 'windows-latest'; pass a self-hosted label for private runners
  secrets: inherit
```

## Inputs

### `sign-linux.yml`

| Input | Required | Default | Description |
|---|---|---|---|
| `unsigned-artifact-name` | no | `file-to-sign` | Name of the artifact containing the file to sign. Override when signing multiple artifacts in one workflow run to avoid collisions. |
| `file-to-sign` | yes | – | Path to the file within the artifact |
| `signed-artifact-name` | no | `file-signed` | Name for the output artifact. Override when signing multiple artifacts in one workflow run to avoid collisions on `upload-artifact`. |
| `detached-signature` | no | `true` | `true` produces a detached PEM signature sidecar (`<file-to-sign>.sig`); `false` produces an enveloped CMS blob (`<file-to-sign>.signed`) |
| `runs-on` | no | `ubuntu-latest` | Runner label(s) |

### `sign-windows.yml`

| Input | Required | Default | Description |
|---|---|---|---|
| `unsigned-artifact-name` | no | `file-to-sign` | Name of the artifact containing the file to sign. Override when signing multiple artifacts in one workflow run to avoid collisions. |
| `file-to-sign` | yes | – | Path to the file within the artifact |
| `signed-artifact-name` | no | `file-signed` | Name for the output artifact. Override when signing multiple artifacts in one workflow run to avoid collisions on `upload-artifact`. |
| `digest-alg` | no | `sha384` | Digest algorithm for signing and timestamping |
| `page-hashes` | no | `false` | Include page hashes (`/ph` flag) |
| `append-signature` | no | `false` | Append signature (dual signing) |
| `skip-verify` | no | `false` | Skip post-sign verification |
| `runs-on` | no | `windows-latest` | Runner label(s) |

### Baked-in defaults

The following values are baked into the reusable workflows and not exposed as inputs. Edit the workflow file in this repo to change them.

| Setting | `sign-linux.yml` | `sign-windows.yml` |
|---|---|---|
| KMS key version | – | `1` |
| Timestamp server | – | `http://timestamp.digicert.com` |
| Signed-file output name | `<file-to-sign>.sig` (detached) / `<file-to-sign>.signed` (enveloped) | overwrites the input file in place |
| Caller-repository allowlist | `ALLOWED_CALLER_REPOSITORIES` env | `ALLOWED_CALLER_REPOSITORIES` env |

## How it works

### Linux (PKCS#11)

1. Authenticates to GCP via Workload Identity Federation
2. Installs the Google Cloud KMS PKCS#11 library
3. Configures PKCS#11 to point at the configured key ring
4. Signs via `openssl cms -sign` with the PKCS#11 engine, embedding the CA chain
5. Verifies the resulting signature and uploads the signed file as an artifact

### Windows (CNG)

1. Authenticates to GCP via Workload Identity Federation
2. Installs the Google Cloud KMS CNG provider (MSI)
3. Locates `signtool.exe` from the highest-version Windows SDK installed on the runner
4. Signs via `signtool.exe sign /csp "Google Cloud KMS Provider"` with an RFC 3161 timestamp
5. Verifies the resulting signature and uploads the signed file as an artifact

## Verification

### Linux

Embedded (default):

```bash
openssl cms -verify -CAfile dt-root.cert.pem -in signed-file.sh
```

Detached (`detached-signature: true`):

```bash
openssl cms -verify \
  -CAfile dt-root.cert.pem \
  -in signature.pem \
  -binary -inform PEM \
  -content original-file.bin
```

### Windows

```powershell
signtool.exe verify /pa signed-file.exe
```

## Versioning

Releases are tagged `vX.Y.Z` (immutable) with a moving major tag `vX`. Consumers should pin to the major tag (`@v1`) for routine use, or to a specific release / commit SHA for the strictest reproducibility.

## Security

- Private keys never leave Google Cloud KMS (HSM-backed)
- Workload Identity Federation eliminates long-lived service-account keys in GitHub
- All third-party Actions are SHA-pinned
