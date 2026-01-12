# Cosign Testbed

Container images for testing GitHub attestations and Kyverno's cosign verification.

**Registry:** `ghcr.io/lucchmielowski/cosign-testbed`

## Available Images

### GitHub Attestation Images
- **`:latest`** - Signed with GitHub build provenance attestation
- **`:unsigned`** - No attestation or signature

### Cosign Test Images
- **`:v2-traditional`** - Cosign v2.4.1 with key-based signing (traditional OCI signature manifests)
- **`:v2-keyless`** - Cosign v2.4.1 with keyless signing (Fulcio + Rekor)
- **`:v3-traditional`** - Cosign v3.0.4 with key-based signing (backward compatible with v2)
- **`:v3-keyless`** - Cosign v3.0.4 with keyless signing (Fulcio + Rekor)
- **`:v3-bundle`** - Cosign v3.0.4 with bundle format (`.sigstore.json` as OCI referrer)

## Verification Methods

### GitHub Attestation

```bash
gh attestation verify oci://ghcr.io/lucchmielowski/cosign-testbed:latest \
  --owner lucchmielowski
```

### Cosign Key-Based Signatures

```bash
# v2-traditional (verify with any cosign v2 or v3)
cosign verify --key cosign.pub ghcr.io/lucchmielowski/cosign-testbed:v2-traditional

# v3-traditional (verify with cosign v3)
cosign verify --key cosign.pub ghcr.io/lucchmielowski/cosign-testbed:v3-traditional

# v3-bundle (verify with cosign v3)
cosign verify --key cosign.pub ghcr.io/lucchmielowski/cosign-testbed:v3-bundle
```

### Cosign Keyless Signatures

```bash
# v2-keyless (verify with GitHub Actions OIDC identity)
cosign verify \
  --certificate-identity=https://github.com/lucchmielowski/gh-attestation-test/.github/workflows/ci.yml@refs/heads/main \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  ghcr.io/lucchmielowski/cosign-testbed:v2-keyless

# v3-keyless (verify with GitHub Actions OIDC identity)
cosign verify \
  --certificate-identity=https://github.com/lucchmielowski/gh-attestation-test/.github/workflows/ci.yml@refs/heads/main \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  ghcr.io/lucchmielowski/cosign-testbed:v3-keyless
```

## Purpose

This testbed provides comprehensive coverage for testing signature verification:

| Image | Purpose |
|-------|---------|
| `:latest` | GitHub native attestations |
| `:unsigned` | Unsigned baseline for comparison |
| `:v2-traditional` | Cosign v2 key-based (traditional) |
| `:v2-keyless` | Cosign v2 keyless (Fulcio/Rekor) |
| `:v3-traditional` | Cosign v3 key-based (traditional) |
| `:v3-keyless` | Cosign v3 keyless (Fulcio/Rekor) |
| `:v3-bundle` | Cosign v3 key-based (bundle format) |

---

**For setup instructions, see [DEVELOPMENT.md](DEVELOPMENT.md)**
