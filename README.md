# VulnCon-SBOM-Workshop

This repo creates a [Gitpod Environment](https://gitpod.io/?autostart=true&useLatest=true#https://github.com/SBOM-Community/VulnCon-SBOM-Workshop) for the follow along workshop for [Practical Software Bill of Materials: From Generation to Distribution Workshop](https://docs.google.com/presentation/d/1v0QT48iHWxNUJ4j3b0hyVp2zG25h60CK7PhVOUeexMc/edit?usp=sharing) at VulnCon 2025

## Getting Started

To help you get started, we have created a Gitpod environment that has all the tools you need to complete the workshop. To get started, click the link below to start the Gitpod environment.

- [Start Gitpod Environment](https://gitpod.io/?autostart=true&useLatest=true#https://github.com/SBOM-Community/VulnCon-SBOM-Workshop)

Alternatively you can install all of these tools and follow along with this README.md file.

``` bash
# Install Syft for SBOM generation
brew install trivy

# Install sbomasm and sbomqs for SBOM augmentation and quality scoring
brew tap interlynk-io/interlynk
brew install sbomasm sbomqs

# Install parlay for SBOM enrichment
brew install parlay
```

## Better SBOM Generation

### `Generation` Step

Generate an SBOM for `kubectl` using Syft.

``` bash
curl -L -o /tmp/kubectl.tgz \
    "https://github.com/kubernetes/kubectl/archive/refs/tags/v0.31.1.tar.gz"
tar xvf /tmp/kubectl.tgz -C .

trivy fs \
   --format cyclonedx \
   --skip-db-update \
   --offline-scan \
   --output generated.cdx.json \
   kubectl-0.31.1
```

#### Expected `Generation` Output

This step should generate the following file:

- `generated.cdx.json`

### `Augmentation` Step

Add missing metadata to the generated SBOM using `sbomasm`. This step adds
metadata to the document and primary component.

``` bash
# Augment the Generated CycloneDX with updated document information
sbomasm edit --subject Document \
    --author 'VulnCon SBOM Generation Workshop' \
    --supplier 'kubernetes (https://kubernetes.io/kubectl)' \
    --lifecycle 'pre-build' \
    --repository 'https://github.com/kubernetes/kubectl' \
    --license 'Apache-2.0 (https://raw.githubusercontent.com/kubernetes/kubectl/refs/heads/master/LICENSE)' \
    generated.cdx.json > augmented-docs-sbom.cdx.json

# Augment the Generated CycloneDX with updated primary component information
sbomasm edit --subject primary-component \
    --author 'VulnCon SBOM Generation Workshop' \
    --supplier 'kubernetes (https://kubernetes.io/kubectl)' \
    --repository 'https://github.com/kubernetes/kubectl' \
    --license 'Apache-2.0 (https://raw.githubusercontent.com/kubernetes/kubectl/refs/heads/master/LICENSE)' \
    augmented-docs-sbom.cdx.json > augmented-sbom.cdx.json
```

#### Expected `Augmentation` Output

The following files were created in this step:

- `augmented-docs-sbom.cdx.json` - SBOM with updated document metadata
- `augmented-sbom.cdx.json` - SBOM with updated document and primary component metadata

You can view a diff of the changes made to the SBOM by running the following command:

``` bash
code --diff generated.cdx.json augmented-sbom.cdx.json
```

### `Enrichment` Step

Enrich the SBOM with additional metadata using `parlay`. This step adds
metadata to components.

``` bash
parlay ecosystems enrich \
    augmented-sbom.cdx.json > enriched-sbom.cdx.json

# Since parlay doesn't pretty its generated json, use jq to format it.
jq . enriched-sbom.cdx.json > enriched-sbom.pretty.cdx.json
```

#### Expected `Enrichment` Output

The following files were created in this step:

- `enriched-sbom.cdx.json` - the component metadata enriched SBOM
- `enriched-sbom.pretty.cdx.json` - the component metadata enriched SBOM, pretty printed

You can view a diff of the changes made to the SBOM by running the following command:

``` bash
code --diff augmented-sbom.cdx.json enriched-sbom.pretty.cdx.json
```

### `Validation` Step

Use `sbomqs` to validate the SBOM. This step checks the SBOM for common issues.

``` bash
sbomqs score generated.cdx.json
sbomqs score enriched-sbom.cdx.json
```

## SBOM Sharing

TODO

## SBOM Operations

TODO

## VEX

TODO
