# custom-catalog-builder

Builds and maintains a custom OLM (Operator Lifecycle Manager) [File-Based Catalog](https://docs.openshift.com/container-platform/4.17/operators/admin/olm-managing-custom-catalogs.html) (FBC) for QE testing. The catalog contains curated operator versions used by the certsuite and related projects.

The catalog image is published to `quay.io/redhat-best-practices-for-k8s/qe-custom-catalog:latest` and rebuilt daily via GitHub Actions.

## Included Operators

| Operator | Versions | Default Channel |
|----------|----------|-----------------|
| cloud-native-postgresql (EnterpriseDB) | v1.23.6, v1.24.4, v1.26.0 | new |
| nginx-ingress-operator (NGINX Inc.) | v2.4.2, v3.0.0, v3.0.1 | new |

Operator definitions live in `custom-catalog/index.yaml`.

## Prerequisites

- **Docker** or **Podman** (with buildx support for multi-arch builds)
- **opm** (Operator Package Manager) -- downloaded automatically in CI, or install manually from [operator-framework/operator-registry](https://github.com/operator-framework/operator-registry/releases)

## Building Locally

### 1. Download the opm binary

```bash
# Linux (amd64)
curl -sL https://api.github.com/repos/operator-framework/operator-registry/releases/latest \
  | grep "browser_download_url.*linux-amd64-opm" \
  | cut -d : -f 2,3 \
  | tr -d \" \
  | wget -qi -
chmod +x linux-amd64-opm
```

### 2. Validate the catalog

```bash
./linux-amd64-opm validate custom-catalog
```

### 3. Build the container image

```bash
# Single-arch build
docker build -f custom-catalog.Dockerfile -t qe-custom-catalog:local .

# Multi-arch build (requires buildx + QEMU)
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x \
  -f custom-catalog.Dockerfile \
  -t qe-custom-catalog:local \
  .
```

The Dockerfile uses a multi-stage build with `quay.io/operator-framework/opm:latest`:

1. **Builder stage** -- copies `custom-catalog/` into the image and pre-populates the opm serve cache.
2. **Final stage** -- minimal image containing the opm binary, catalog configs, and pre-built cache. The entrypoint runs `opm serve /configs`.

## Using the Catalog

Apply a `CatalogSource` to your OpenShift cluster:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: qe-custom-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/redhat-best-practices-for-k8s/qe-custom-catalog:latest
  displayName: QE Custom Catalog
  publisher: Red Hat Best Practices for K8s
```

## Modifying the Catalog

### Adding a new operator

1. Add `olm.package`, `olm.channel`, and `olm.bundle` entries to `custom-catalog/index.yaml`.
2. Use SHA digests for image references.
3. Define upgrade paths with the `replaces` field in channel entries.
4. Set a `defaultChannel`.
5. Validate: `opm validate custom-catalog`
6. Open a PR -- the `pre-main.yml` workflow will validate the changes automatically.

### Updating an existing operator version

1. Add a new `olm.bundle` entry with the updated version.
2. Update the channel entries to include the new version in the `replaces` chain.
3. Validate and open a PR.

## CI Workflows

| Workflow | Trigger | What it does |
|----------|---------|-------------|
| `build-catalog.yaml` | Push to main, daily at midnight UTC, manual | Validates catalog, builds multi-arch image, pushes to quay.io |
| `pre-main.yml` | Pull requests to main, manual | Validates catalog and test-builds the image without pushing |

## Scripts

### `rebase_all.sh`

Helper script that rebases all local branches against `upstream/main` and force-pushes them to `origin`. Useful for keeping feature branches up to date with the upstream repository.

```bash
./rebase_all.sh
```
