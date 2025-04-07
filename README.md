# VulnCon-SBOM-Workshop

This repo creates a [Gitpod Environment](https://gitpod.io/?autostart=true&useLatest=true#https://github.com/SBOM-Community/VulnCon-SBOM-Workshop) for the follow along workshop for [Practical Software Bill of Materials: From Generation to Distribution Workshop](https://docs.google.com/presentation/d/1v0QT48iHWxNUJ4j3b0hyVp2zG25h60CK7PhVOUeexMc/edit?usp=sharing) at VulnCon 2025

## Getting Started

To help you get started, we have created a Gitpod environment that has all the tools you need to complete the workshop. To get started, click the link below to start the Gitpod environment.

- [Start Gitpod Environment](https://gitpod.io/?autostart=true&useLatest=true#https://github.com/SBOM-Community/VulnCon-SBOM-Workshop) __<-- Start Here__

Alternatively you can install all of these tools and follow along with this README.md file.

__No need to execute this in your Gitpod environment.__

``` bash
# Install Trivy for SBOM generation
brew install trivy

# Install sbomasm and sbomqs for SBOM augmentation and quality scoring
brew tap interlynk-io/interlynk
brew install sbomasm sbomqs

# Install parlay for SBOM enrichment
brew install parlay

# Install cosign for SBOM signing
brew install cosign

# To attest and convert the SBOM, fetch binaries for bnd and unpack
# for your local architecture (these are prereleases so they are not on 
# homebrew yet, sorry!)

# https://github.com/carabiner-dev/bnd/releases/latest
# https://github.com/carabiner-dev/unpack/releases/tag/v0.1.0-pre5

# Install bomctl for SBOM sharing and distribution
brew tap bomctl/bomctl
brew install bomctl

# Install duckdb
brew install duckdb

# Install osv-scanner
brew install osv-scanner
```

## Better SBOM Generation

### `Generation` Step

Generate an SBOM for `kubectl` using Trivy.

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

### Rename the SBOM

At this point we want to rename the SBOM to meet the OpenSSF naming recomendation
schema. This is to make it easy to relate the SBOM with its released artifacts:

```bash
mv enriched-sbom.cdx.json kubectl-v0.31.1.cdx.json
```

### Generate SPDX Variant

For completeness and maximum compatibility, create the SPDX variant
of our enriched SBOM:

```bash
unpack sbom kubectl-v0.31.1.cdx.json --format=spdx >  kubectl-v0.31.1.spdx.json
```

### Signing and Attesting

#### Plain Signing

Next up, we'll sign the enriched software bill of materials. This step will
create a detached certificate and signature fiel pair you can distribute to
verify the SBOM.

> [!NOTE]
> The following command will open the cosign signing flow, authenticate with
> your OIDC provider (Google/GitHub/Microsoft) and you are done. If you are
> following in the GitPod tutorial, you will need to copy + paste the URL in
> your browser and then put the resulting code in the terminal.

```bash
# Sign the CycloneDX variant
cosign sign-blob kubectl-v0.31.1.cdx.json \
   --output-certificate=kubectl-v0.31.1.cdx.json.pem \
   --output-signature=kubectl-v0.31.1.cdx.json.sig

# Sign the SPDX variant
cosign sign-blob kubectl-v0.31.1.spdx.json \
   --output-certificate=kubectl-v0.31.1.spdx.json.pem \
   --output-signature=kubectl-v0.31.1.spdx.json.sig
```

Verify the signed SBOM with `cosign verify`:

``` bash
cosign verify-blob kubectl-v0.31.1.cdx.json \
    --certificate=kubectl-v0.31.1.cdx.json.pem \
    --signature=kubectl-v0.31.1.cdx.json.sig 
```

#### Attesting

Creating an attestation cryptographically binds the SBOM to the artifact it
describes. In addition, attestations wrapped in bundles can be distributed with
all their verification material in a single file which is often more convenient
than the artifact+cert+signature trio.

```bash
# Create the attestation bundle, it will use as its subject the hash of the
# downloaded source tarball:
bnd predicate kubectl-v0.31.1.cdx.json \
    --type=https://cyclonedx.org/bom \
    --subject-file=/tmp/kubectl.tgz \
    --out=kubectl-v0.31.1.cdx.bundle.json

# Same with the SPDX variant:
bnd predicate kubectl-v0.31.1.spdx.json \
    --type=https://spdx.dev/Document \
    --subject-file=/tmp/kubectl.tgz \
    --out=kubectl-v0.31.1.spdx.bundle.json
```

Now finaly, we should pack all attestations into a single distributable file:

```bash
# This packs both attestations into a linear json bundle 
bnd pack -o kubectl-v0.31.1.bundle.jsonl \
  kubectl-v0.31.1.cdx.bundle.json \
  kubectl-v0.31.1.spdx.bundle.json
```

After this step you will have a single file `kubectl-v0.31.1.bundle.jsonl` containing
both signed SBOMs wrapped in their attestations together with all the verification
material required to verify the documents. 

### Verification

Recepients of signed SBOMs should verify the authenticity and integrity of the
documents. Throughout this tutorial we've produced two kinds of signed SBOMs.
Let's verify them.

#### Detached Signature Verification

To verify the signature of an SBOM with a detached signature and its certificate
invoke cosign:

```bash
cosign verify-blob kubectl-v0.31.1.cdx.json \
  --certificate kubectl-v0.31.1.cdx.json.pem \
  --signature kubectl-v0.31.1.cdx.json.sig \
  --certificate-identity-regexp='.*' \
  --certificate-oidc-issuer-regexp='.*'
```

#### Verify a a bundled attestation

To verify a bundled attestation, you only need the bundle file:

```bash
# Verify the attestation with bnd:
bnd verify --skip-identity kubectl-v0.31.1.cdx.bundle.json

# Expected:
# ✅ Bundle Verification OK!

# Inspect the bundle's metadata
bnd inspect kubectl-v0.31.1.cdx.bundle.json

# Extract the unsigned attestation wrapping the SBOM:
bnd extract statement kubectl-v0.31.1.cdx.bundle.json

# > Note the subject data, binding the sbom to the file

```

> [!WARNING]
> Note that both examples, in cosign's case allowing any string with the regular
> expression `'.*'` and in the second with `--skip-identity` we are not checking
> the signer identity. The expected identity must be checked when verifying as 
> part of the policy.

## SBOM Sharing

This will recursively fetch the SBOM at this URL, and any internally referenced SBOMs (for this example it will fetch two SBOMs)

``` bash
bomctl fetch https://raw.githubusercontent.com/bomctl/bomctl-playground/main/examples/bomctl-container-image/bomctl_bomctl_v0.3.0.cdx.json
```

After fetching the SBOMs, you can list the SBOMs that have been fetched:

``` bash
bomctl list
```

View the SBOMs that have been fetched:

``` bash
bomctl export https://anchore.com/syft/file/bomctl_0.3.0_linux_amd64.tar.gz-1b838d44-9d3c-47d0-9f7f-846397e701fa#DOCUMENT
```

Convert between SBOM formats:

``` bash
bomctl export -f cyclonedx https://anchore.com/syft/file/bomctl_0.3.0_linux_amd64.tar.gz-1b838d44-9d3c-47d0-9f7f-846397e701fa#DOCUMENT
```

Lets list one of the two SBOMs that we have fetched into a OCI Registry:

``` bash
# Get the URL and port of the OCI registry running in Gitpod
url="$(gp url 5000)"
export BOMCTL_PORT_URL="${url#*://}"

# Push the SBOM to the OCI registry and convert to SPDX format
bomctl push -f spdx urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db oci://${BOMCTL_PORT_URL}/hello-bomctl:latest
```

This will push the SBOM to the OCI registry and convert it to SPDX format.

You can view the layers in Zot, the OCI registry running in Gitpod, by clicking on the "PORTS" tab (next to the "TERMINAL" tab) and then clicking on the "Address" next to the port 5000.

## SBOM Operations

If you want to run some GUAC demos, see: https://docs.guac.sh/setup-install/

Collect SBOMs:

```bash
# Fetch an SBOM
curl -LO https://raw.githubusercontent.com/SBOM-Community/VulnCon-SBOM-Workshop/refs/heads/main/examples/scalibr.spdx.json
# Look at the SBOM
cat scalibr.spdx.json
```

Correlate an SBOM with vulnerability data:

```bash
# OSV Scanner correlates data from the OSV vulnerability database
osv-scanner scan --sbom scalibr.spdx.json --format vertical
```

Use DuckDB to understand all unique packages across multiple SBOMs:

```bash
duckdb
D CREATE TABLE sbom AS
· SELECT * FROM read_json_auto('*.json', auto_detect=true, maximum_object_size="333554428");
D SELECT DISTINCT package.unnest.name AS package_name
·   FROM
·       sbom,
‣       UNNEST(packages) AS package;
```

Use OSV Scanner to fix any vulnerabilities in a project:

```bash
git clone https://github.com/whiskeytastingfoundation/wtf-frontend
cd wtf-frontend
osv-scanner fix \
    --max-depth=3 \
    --min-severity=5 \
    --ignore-dev  \
    --strategy=in-place \
    -L package-lock.json

```

## VEX

TODO
