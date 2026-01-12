# Development Guide

This document covers setup, building, and troubleshooting for the cosign testbed repository.

## Setup Instructions

### Prerequisites

- Docker
- cosign (for local testing)
- GitHub CLI (`gh`) - optional, for cleanup

### 1. Generate Cosign Key Pair

Use the provided helper script:

```bash
./setup-cosign-keys.sh
```

Or manually:

```bash
cosign generate-key-pair
```

This creates:
- `cosign.key` (private key) - **NEVER commit this!**
- `cosign.pub` (public key) - safe to commit

You'll be prompted to set a password for the private key.

### 2. Add GitHub Secrets

Add the following secrets to your GitHub repository:

**Settings → Secrets and variables → Actions → New repository secret**

#### `COSIGN_PRIVATE_KEY`
Copy the entire contents of `cosign.key`:

```bash
cat cosign.key
```

Include the full PEM block:
```
-----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----
...
-----END ENCRYPTED COSIGN PRIVATE KEY-----
```

#### `COSIGN_PASSWORD`
The password you set when generating the key pair.

### 3. Commit Public Key (Optional)

```bash
git add cosign.pub
git commit -m "Add cosign public key for verification"
```

### 4. Push and Run Workflow

```bash
git push
```

The CI workflow will:
1. 🗑️ **Clean up** all existing image versions from GHCR
2. 🔨 **Build and push** 7 fresh images in parallel
3. ✍️ **Sign** the images with appropriate cosign versions and methods

## Kyverno Policy Examples

### Key-Based Verification

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-cosign-key-based
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-v2-traditional
      match:
        any:
        - resources:
            kinds:
            - Pod
      verifyImages:
      - imageReferences:
        - "ghcr.io/lucchmielowski/cosign-testbed:v2-traditional"
        attestors:
        - entries:
          - keys:
              publicKeys: |-
                -----BEGIN PUBLIC KEY-----
                <YOUR_COSIGN_PUB_CONTENT>
                -----END PUBLIC KEY-----
    
    - name: verify-v3-bundle
      match:
        any:
        - resources:
            kinds:
            - Pod
      verifyImages:
      - imageReferences:
        - "ghcr.io/lucchmielowski/cosign-testbed:v3-bundle"
        attestors:
        - entries:
          - keys:
              publicKeys: |-
                -----BEGIN PUBLIC KEY-----
                <YOUR_COSIGN_PUB_CONTENT>
                -----END PUBLIC KEY-----
```

### Keyless Verification

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-cosign-keyless
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-v2-keyless
      match:
        any:
        - resources:
            kinds:
            - Pod
      verifyImages:
      - imageReferences:
        - "ghcr.io/lucchmielowski/cosign-testbed:v2-keyless"
        attestors:
        - entries:
          - keyless:
              subject: "https://github.com/lucchmielowski/cosign-testbed/.github/workflows/ci.yml@refs/heads/main"
              issuer: "https://token.actions.githubusercontent.com"
              rekor:
                url: https://rekor.sigstore.dev
    
    - name: verify-v3-keyless
      match:
        any:
        - resources:
            kinds:
            - Pod
      verifyImages:
      - imageReferences:
        - "ghcr.io/lucchmielowski/cosign-testbed:v3-keyless"
        attestors:
        - entries:
          - keyless:
              subject: "https://github.com/lucchmielowski/cosign-testbed/.github/workflows/ci.yml@refs/heads/main"
              issuer: "https://token.actions.githubusercontent.com"
              rekor:
                url: https://rekor.sigstore.dev
```

## Local Testing

### Build Image Locally

```bash
docker build -t test-image:local .
```

### Generate Test Key Pair

```bash
cosign generate-key-pair
```

### Sign with Different Formats

```bash
# Traditional key-based signing
cosign sign --key cosign.key test-image:local

# Keyless signing (requires OIDC provider)
cosign sign test-image:local

# Bundle format (cosign v3+)
cosign sign --key cosign.key --bundle test-image:local
```

### Verify Signatures

```bash
# Key-based verification
cosign verify --key cosign.pub test-image:local

# Keyless verification (requires identity info)
cosign verify \
  --certificate-identity=<your-identity> \
  --certificate-oidc-issuer=<issuer-url> \
  test-image:local
```

## Cleanup

> **Note:** The CI workflow automatically cleans up all images before building new ones. Manual cleanup is only needed if you want to delete images without rebuilding.

### Option 1: GitHub Actions Workflow

Go to **Actions → Cleanup GHCR Images → Run workflow**

1. Click "Run workflow"
2. Type `DELETE` in the confirmation field
3. Click "Run workflow"

This will delete all versions of the image from GHCR.

### Option 2: Local Script

```bash
./cleanup-ghcr.sh
```

**Requirements:**
- GitHub CLI (`gh`) installed and authenticated
- Type `DELETE` when prompted to confirm

**The script will:**
- List all image versions
- Delete each version from GHCR
- Show a summary of deleted/failed versions

## Troubleshooting

### "Error: signing [image]: getting signer: reading key: no PEM block found"

**Cause:** The `COSIGN_PRIVATE_KEY` secret is incorrectly formatted.

**Solution:** Ensure the secret contains the full PEM block including headers:
```
-----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----
...
-----END ENCRYPTED COSIGN PRIVATE KEY-----
```

### "Error: password required"

**Cause:** The `COSIGN_PASSWORD` secret is missing or incorrect.

**Solution:** Verify the password matches the one you used when generating the key pair.

### "Bundle signature not found"

**Cause:** Trying to verify a bundle signature with an older cosign version.

**Solution:** The `--bundle` flag in cosign v3 creates a `.sigstore.json` referrer. Use cosign v3.0+ to verify bundle signatures.

### "Failed to verify signature: no matching signatures"

**Cause:** The image may not be signed, or you're using the wrong verification method/key.

**Solution:**
- Verify you're using the correct public key or identity information
- Check the image was actually signed by inspecting the CI logs
- For keyless signatures, ensure the identity and issuer match exactly

### Cleanup fails with "404 Not Found"

**Cause:** The package doesn't exist in GHCR yet, or you don't have permissions.

**Solution:**
- First push should create the package automatically
- Ensure you have `packages: write` permission
- Check the package exists at `https://github.com/users/lucchmielowski/packages/container/package/cosign-testbed`

## CI Workflow Structure

The workflow consists of 8 jobs:

```
cleanup (runs first)
  ↓
├─ build-push-and-attest (latest)
├─ build-push-unsigned (unsigned)
├─ build-sign-v2-traditional (v2-traditional)
├─ build-sign-v2-keyless (v2-keyless)
├─ build-sign-v3-traditional (v3-traditional)
├─ build-sign-v3-keyless (v3-keyless)
└─ build-sign-v3-bundle (v3-bundle)

(All build jobs run in parallel after cleanup completes)
```

### Job Details

| Job | Image Tag | Cosign Version | Signing Method |
|-----|-----------|----------------|----------------|
| `build-push-and-attest` | `:latest` | N/A | GitHub attestation |
| `build-push-unsigned` | `:unsigned` | N/A | None |
| `build-sign-v2-traditional` | `:v2-traditional` | v2.4.1 | Key-based (traditional) |
| `build-sign-v2-keyless` | `:v2-keyless` | v2.4.1 | Keyless (Fulcio/Rekor) |
| `build-sign-v3-traditional` | `:v3-traditional` | v3.0.4 | Key-based (traditional) |
| `build-sign-v3-keyless` | `:v3-keyless` | v3.0.4 | Keyless (Fulcio/Rekor) |
| `build-sign-v3-bundle` | `:v3-bundle` | v3.0.4 | Key-based (bundle format) |

## Helper Scripts

### `setup-cosign-keys.sh`

Generates a cosign key pair and provides instructions for adding secrets to GitHub.

```bash
./setup-cosign-keys.sh
```

### `cleanup-ghcr.sh`

Deletes all versions of the cosign-testbed image from GHCR.

```bash
./cleanup-ghcr.sh
```

## File Structure

```
.
├── .github/
│   └── workflows/
│       ├── ci.yml           # Main build and sign workflow
│       └── cleanup.yml      # Manual cleanup workflow
├── Dockerfile               # Minimal Alpine-based test image
├── README.md                # User-facing documentation
├── DEVELOPMENT.md           # This file
├── setup-cosign-keys.sh     # Helper script for key generation
├── cleanup-ghcr.sh          # Helper script for cleanup
├── cosign.pub               # Public key (safe to commit)
└── .gitignore               # Protects cosign.key from commits
```

## Environment Variables

The CI workflow uses the following environment variables:

```yaml
env:
  REGISTRY: "ghcr.io/lucchmielowski"
  IMAGE_NAME: "cosign-testbed"
```

To use this setup for your own repository, update these values in `.github/workflows/ci.yml`.

## Permissions

The workflow requires the following permissions:

```yaml
permissions:
  contents: read          # Read repository contents
  packages: write         # Push to GHCR and delete versions
  id-token: write         # Keyless signing with OIDC
  attestations: write     # Create GitHub attestations
```

## Security Notes

- ⚠️ **NEVER commit `cosign.key`** - it's in `.gitignore` for protection
- ✅ `cosign.pub` is safe to commit and useful for verification examples
- 🔒 `COSIGN_PRIVATE_KEY` and `COSIGN_PASSWORD` must be stored as GitHub secrets
- 🔑 Keyless signing doesn't require storing any keys - it uses OIDC tokens

## Contributing

When adding new image variants:

1. Add a new job to `.github/workflows/ci.yml`
2. Ensure the job has `needs: cleanup`
3. Update README.md with the new image and verification method
4. Update this document if new setup steps are required

## Resources

- [Cosign Documentation](https://docs.sigstore.dev/cosign/overview/)
- [Kyverno Image Verification](https://kyverno.io/docs/writing-policies/verify-images/)
- [GitHub Attestations](https://docs.github.com/en/actions/security-guides/using-artifact-attestations)
- [Sigstore Public Good Instance](https://www.sigstore.dev/)
