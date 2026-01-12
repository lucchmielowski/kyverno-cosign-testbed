# GitHub Attestation & Cosign Test Repository

This repository builds container images for testing GitHub attestations and Kyverno's cosign verification.

## Images Produced

The CI workflow builds and pushes the following images to `ghcr.io/lucchmielowski/gh-attestation-test`:

### GitHub Attestation Images
- **`:latest`** - Signed with GitHub build provenance attestation
- **`:unsigned`** - No attestation or signature

### Cosign Test Images (for Kyverno verification testing)
- **`:v2-traditional`** - Signed with cosign v2.4.1 using traditional OCI signature manifests
- **`:v3-traditional`** - Signed with cosign v3.0.4 using traditional format (default, backward compatible)
- **`:v3-bundle`** - Signed with cosign v3.0.4 using NEW unified bundle format (stored as OCI referrer)

## Setup Instructions

### 1. Generate Cosign Key Pair

If you don't have a cosign key pair yet, generate one locally:

```bash
cosign generate-key-pair
```

This creates:
- `cosign.key` (private key)
- `cosign.pub` (public key)

You'll be prompted to set a password for the private key.

### 2. Add GitHub Secrets

Add the following secrets to your GitHub repository (Settings → Secrets and variables → Actions):

**`COSIGN_PRIVATE_KEY`**
```bash
cat cosign.key
```
Copy the entire contents including `-----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----` and `-----END ENCRYPTED COSIGN PRIVATE KEY-----`

**`COSIGN_PASSWORD`**
The password you set when generating the key pair.

### 3. Commit Public Key (Optional)

Optionally commit the public key to the repository for verification testing:

```bash
cp cosign.pub .
git add cosign.pub
git commit -m "Add cosign public key for verification"
```

### 4. Push and Run Workflow

```bash
git push
```

The GitHub Actions workflow will automatically build and sign all 5 images.

## Verifying Signatures

### Verify GitHub Attestation

```bash
gh attestation verify oci://ghcr.io/lucchmielowski/gh-attestation-test:latest \
  --owner lucchmielowski
```

### Verify Cosign Signatures

With cosign v2.4.1 or v3.x:

```bash
# v2-traditional (verify with any cosign v2 or v3)
cosign verify --key cosign.pub ghcr.io/lucchmielowski/gh-attestation-test:v2-traditional

# v3-traditional (verify with cosign v3)
cosign verify --key cosign.pub ghcr.io/lucchmielowski/gh-attestation-test:v3-traditional

# v3-bundle (verify with cosign v3, looks for .sigstore.json bundle)
cosign verify --key cosign.pub ghcr.io/lucchmielowski/gh-attestation-test:v3-bundle
```

## Using with Kyverno

Create a Kyverno policy to test signature verification:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-cosign-signatures
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
        - "ghcr.io/lucchmielowski/gh-attestation-test:v2-traditional"
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
        - "ghcr.io/lucchmielowski/gh-attestation-test:v3-bundle"
        attestors:
        - entries:
          - keys:
              publicKeys: |-
                -----BEGIN PUBLIC KEY-----
                <YOUR_COSIGN_PUB_CONTENT>
                -----END PUBLIC KEY-----
```

## Testing Locally

Build and test locally:

```bash
# Build image
docker build -t test-image:local .

# Generate key pair
cosign generate-key-pair

# Sign with different formats
cosign sign --key cosign.key test-image:local
cosign sign --key cosign.key --bundle test-image:local

# Verify
cosign verify --key cosign.pub test-image:local
```

## Troubleshooting

### "Error: signing [image]: getting signer: reading key: no PEM block found"

Make sure the `COSIGN_PRIVATE_KEY` secret contains the full key including headers:
```
-----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----
...
-----END ENCRYPTED COSIGN PRIVATE KEY-----
```

### "Error: password required"

Ensure `COSIGN_PASSWORD` secret is set correctly.

### Bundle signature not found

The `--bundle` flag in cosign v3 creates a `.sigstore.json` referrer. Make sure you're using cosign v3.0+ to verify bundle signatures.

## Purpose

- **`:latest` / `:unsigned`**: Test GitHub native attestations vs unsigned images
- **`:v2-traditional`**: Test Kyverno compatibility with cosign v2 signatures
- **`:v3-traditional`**: Test Kyverno with cosign v3 default behavior (backward compatible)
- **`:v3-bundle`**: Test Kyverno with NEW cosign v3 bundle format verification

This allows comprehensive testing of signature verification across different cosign versions and formats.
