# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

The custom-catalog-builder creates and maintains a custom OLM (Operator Lifecycle Manager) catalog for QE (Quality Engineering) testing purposes. It builds a File-Based Catalog (FBC) containing curated operator versions that are used for testing the certsuite and related projects.

The custom catalog is published to `quay.io/redhat-best-practices-for-k8s/qe-custom-catalog` and is rebuilt daily via GitHub Actions.

## What This Repository Does

1. Maintains a curated list of operators with specific versions in `custom-catalog/index.yaml`
2. Validates the catalog using the `opm` (Operator Package Manager) tool
3. Builds and pushes multi-architecture container images containing the catalog
4. Provides a stable, reproducible operator catalog for QE testing scenarios

## Project Structure

```
custom-catalog-builder/
├── custom-catalog/
│   └── index.yaml          # FBC index file containing operator definitions
├── custom-catalog.Dockerfile  # Multi-stage Dockerfile for building the catalog image
├── .github/
│   ├── workflows/
│   │   ├── build-catalog.yaml  # Main workflow: validates and pushes catalog image
│   │   └── pre-main.yml        # PR workflow: test builds without pushing
│   └── dependabot.yml          # Dependabot configuration for Docker and GitHub Actions
├── rebase_all.sh               # Helper script for rebasing local branches
├── LICENSE                     # Apache 2.0 License
└── README.md                   # Basic project overview
```

## Included Operators

The catalog currently includes the following operators:

- **cloud-native-postgresql**: PostgreSQL operator from EnterpriseDB
  - Versions: v1.23.6, v1.24.4, v1.26.0
  - Channels: old, new (default)

- **nginx-ingress-operator**: NGINX Ingress Controller operator
  - Versions: v2.4.2, v3.0.0, v3.0.1
  - Channels: old, new

## Build Process

This project does not have a Makefile. All build operations are performed via GitHub Actions workflows.

### GitHub Actions Workflows

**build-catalog.yaml** (Production Build):
- Triggers: Push to main, daily schedule (midnight UTC), manual dispatch
- Steps:
  1. Download latest `opm` binary from operator-framework/operator-registry
  2. Validate catalog with `opm validate custom-catalog`
  3. Build multi-arch image (linux/amd64, linux/arm64, linux/ppc64le, linux/s390x)
  4. Push to `quay.io/redhat-best-practices-for-k8s/qe-custom-catalog:latest`

**pre-main.yml** (PR Validation):
- Triggers: Pull requests to main, manual dispatch
- Steps: Same as production but without pushing to registry
- Used to validate changes before merging

### Local Build Commands

To build and validate locally:

```bash
# Download the opm binary (Linux)
curl -s https://api.github.com/repos/operator-framework/operator-registry/releases/latest \
  | grep "browser_download_url.*linux-amd64-opm" \
  | cut -d : -f 2,3 \
  | tr -d \" \
  | wget -qi -
chmod +x linux-amd64-opm

# Validate the catalog
./linux-amd64-opm validate custom-catalog

# Build the container image locally
docker buildx build \
  --platform linux/amd64 \
  --file custom-catalog.Dockerfile \
  --tag qe-custom-catalog:local \
  .
```

## Catalog Format

The catalog uses the OLM File-Based Catalog (FBC) format. The `custom-catalog/index.yaml` file contains:

- **olm.package**: Package metadata with default channel
- **olm.channel**: Channel definitions with upgrade paths (replaces field)
- **olm.bundle**: Bundle definitions pointing to operator images with:
  - Image references (SHA digests)
  - GVK (Group/Version/Kind) properties
  - Package version information

### Example Entry Structure

```yaml
---
schema: olm.channel
package: operator-name
name: channel-name
entries:
- name: operator-name.v1.0.0
  replaces: operator-name.v0.9.0
---
schema: olm.package
name: operator-name
defaultChannel: channel-name
---
schema: olm.bundle
name: operator-name.v1.0.0
package: operator-name
image: registry.example.com/operator@sha256:...
properties:
- type: olm.gvk
  value:
    group: example.com
    kind: CustomResource
    version: v1
- type: olm.package
  value:
    packageName: operator-name
    version: 1.0.0
```

## Development Guidelines

### Adding a New Operator

1. Add the operator's package, channel, and bundle entries to `custom-catalog/index.yaml`
2. Ensure image references use SHA digests for reproducibility
3. Define upgrade paths using the `replaces` field in channel entries
4. Set an appropriate `defaultChannel`
5. Run `opm validate custom-catalog` to verify the catalog is valid
6. Submit a PR - the pre-main workflow will validate the changes

### Updating Operator Versions

1. Add new bundle entries with the updated version
2. Update channel entries to include the new version with appropriate `replaces` chain
3. Validate with `opm validate custom-catalog`
4. Submit a PR

### Key Dependencies

- **opm**: Operator Package Manager from operator-framework/operator-registry
- **Docker buildx**: For multi-architecture image builds
- **QEMU**: For cross-platform emulation during builds

### Container Image

The Dockerfile uses a multi-stage build:
1. **Builder stage**: Uses `quay.io/operator-framework/opm:latest` to pre-populate the serve cache
2. **Final stage**: Minimal image with opm binary, configs, and pre-built cache

The image is labeled with `operators.operatorframework.io.index.configs.v1=/configs` to indicate the FBC root location.

## Using the Catalog

To use this catalog in an OpenShift/Kubernetes cluster:

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

## Related Projects

- **certsuite**: The certification test suite that uses operators from this catalog for testing
- **oct**: Offline Catalog Tool that maintains a database of Red Hat certified operators
- **operator-framework/operator-registry**: Provides the opm tool and FBC specification

## Maintenance

- The catalog is rebuilt daily at midnight UTC
- Dependabot monitors Docker base images and GitHub Actions for updates
- The `rebase_all.sh` script helps maintain local branches synchronized with upstream
