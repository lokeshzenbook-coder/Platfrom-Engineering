> ## Documentation Index
> Fetch the complete documentation index at: https://notes.kodekloud.com/llms.txt
> Use this file to discover all available pages before exploring further.

# Security in Delivery Build Pipelines That Ship Safely

> Hardening CI/CD delivery pipelines by building, scanning, signing, and enforcing admission of container images to prevent supply chain attacks

Earlier guidance often focused on securing what runs inside the cluster—RBAC, admission control, PSS, mTLS. This document shifts the emphasis to what gets into the cluster in the first place by hardening the delivery pipeline, container images, dependencies, and build artifacts. If attackers cannot get through your runtime protections, they will try your supply chain. This lesson covers practical controls—image scanning, signing, admission enforcement—and how SLSA and SBOMs fit into a robust pipeline.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/software-security-learning-objectives-diagram.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=00e5a4ba2cbe2f0f1d17b80c44e65fb7" alt="The image outlines four learning objectives related to software security and integrity, including image scanning, signing, enforcement policies, and SLSA/SBOM concepts." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/software-security-learning-objectives-diagram.jpg" />
</Frame>

## Why supply-chain attacks are high-impact

Supply chain attacks are particularly damaging because they break trust at the source. Common supply-chain risks include:

* Vulnerable base images — pulling a public image that contains many known CVEs.
* Compromised dependencies — malicious packages injected into npm, PyPI, or Go modules during the build.
* Tampered artifacts — images or binaries modified between build and deploy.
* Unscanned images — artifacts reaching production without any vulnerability checks.

These threats target the build process rather than runtime components. To mitigate them, security must shift left into CI/CD and artifact management.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/supply-chain-attack-surface-vulnerabilities.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=f1f7db1d45a5ebd393dddefa0b16a737" alt="The image outlines the supply chain attack surface, highlighting vulnerabilities such as vulnerable base images, compromised dependencies, tampered artifacts, and unscanned images in production. Each point explains potential risks like CVEs, malicious packages, and unchecked vulnerabilities." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/supply-chain-attack-surface-vulnerabilities.jpg" />
</Frame>

## Four pipeline gates to enforce

A practical, enforceable delivery pipeline includes four gates. Each gate addresses a different threat class and should be automated in CI:

| Gate  | Goal                                               | Common tools & practices                                                                                                                                                                                 |
| ----- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Build | Produce minimal, reproducible images               | Multi-stage Dockerfiles, pinned dependencies, minimal base images (e.g., [distroless](https://github.com/GoogleContainerTools/distroless), [Alpine](https://alpinelinux.org/)), reproducible build flags |
| Scan  | Detect known vulnerabilities and misconfigurations | `Trivy`, `Grype`, `Anchore`; fail pipelines on disallowed severities                                                                                                                                     |
| Sign  | Prove provenance and integrity of artifacts        | `Cosign` (key-based or keyless with Sigstore Fulcio/Rekor), recorded attestations                                                                                                                        |
| Admit | Enforce that only scanned & signed artifacts run   | Admission controllers like `Kyverno`, `OPA/Gatekeeper` validating signatures and SBOM attestation                                                                                                        |

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/secure-pipeline-four-gates-tools.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=a193a32249abc458575e5fcde05653c8" alt="The image outlines a secure pipeline with four gates: Build, Scan, Sign, and Admit, each accompanied by relevant tools and practices." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/secure-pipeline-four-gates-tools.jpg" />
</Frame>

Make sure code and artifacts flow through these gates before deployment. The sections below provide concrete tools, commands, and an end-to-end flow you can adopt.

## Image scanning with Trivy

Use image scanning as a hard gate in CI—not just informational output. Trivy is a widely used open-source scanner that finds vulnerabilities in OS packages, application dependencies, and common misconfigurations.

Key CI behavior:

* Fail the pipeline when unacceptable severities are found.
* Treat scanner errors as failures (don’t allow unknown scanner state to be an implicit pass).

Example usage:

```bash theme={null}
# Scan an image and print detected vulnerabilities
trivy image registry.company.com/app:v1.2

# Fail the pipeline on any CRITICAL or HIGH vulnerabilities
trivy image --exit-code 1 \
  --severity CRITICAL,HIGH \
  registry.company.com/app:v1.2
```

Trivy exit codes:

* 0: no vulnerabilities found at the specified severities (pipeline can continue)
* 1: matching vulnerabilities found (pipeline should fail)
* 2: scanner error or unexpected problem (treat as a failure and investigate)

<Callout icon="warning" color="#FF6B6B">
  Treat a scanner error (exit code 2) as a pipeline failure. Silent failures or degraded scanners can let vulnerable images slip through.
</Callout>

## Image signing with Cosign (Sigstore)

Signing artifacts proves provenance and integrity. Sigstore's Cosign is a broadly adopted tool for signing container images and storing signatures in a registry.

Why sign?

* Provenance: shows the artifact originated from your pipeline.
* Integrity: proves the artifact hasn't been altered since signing.
* Trust: restricts which identities/keys can create valid signatures.

Key-based signing example:

```bash theme={null}
# Generate a key pair (one-time)
cosign generate-key-pair

# Sign an image after it passes scanning
cosign sign --key cosign.key registry.company.com/app:v1.2

# Verify a signature using the public key
cosign verify --key cosign.pub registry.company.com/app:v1.2
```

Keyless signing

* Cosign also supports keyless signing using Sigstore Fulcio and Rekor with CI OIDC tokens, removing the need to manage long-lived private keys. Keyless is typically recommended for CI automation where secret key management is harder to secure.

## Complete automated flow

A typical automated flow in CI/CD:

1. Build the image with minimal base layers and reproducibility in mind.
2. Scan the image with Trivy; if disallowed severities are found, fail the pipeline.
3. Upon a clean scan, sign the image with Cosign (keyed or keyless).
4. Push the signed image and stored signatures/attestations to your registry.
5. Deploy—admission controls in the cluster verify signatures/attestations before allowing workloads.

Any failure at a gate prevents the artifact from reaching production.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/image-signing-pipeline-cosign-trivy.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=5541906ae3caacc67cebaf16a5dbd927" alt="The image illustrates the complete pipeline for image signing with Cosign, involving stages such as coding, building, scanning with Trivy, signing with Cosign, pushing to a registry, and deploying." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/image-signing-pipeline-cosign-trivy.jpg" />
</Frame>

## Admission control: verifying signatures with Kyverno

Enforce policy at runtime by validating image signatures and attestations. The Kyverno ClusterPolicy below rejects Pod creation for images from your registry unless they verify against the embedded public key.

```yaml theme={null}
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: check-image-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "registry.company.com/*"
          attestors:
            - entries:
                - keys:
                    - |
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

If an image is unsigned or the signature does not verify against the provided public key(s), Pod creation is rejected—closing the loop: Trivy ensures images are clean, Cosign proves provenance, and Kyverno enforces acceptance at runtime.

<Callout icon="lightbulb" color="#1CB2FE">
  Use a secure key management strategy (or keyless signing) and automate signing only after a successful scan to prevent accidental acceptance of unverified images.
</Callout>

## SLSA and SBOM — what they are and why they matter

* SLSA (Supply-chain Levels for Software Artifacts) provides a maturity model for build integrity:
  * Level 1: documented builds and basic evidence
  * Level 4: hermetic, reproducible builds with tamper-proof review and enforced provenance
    SLSA helps you reason about how much assurance your pipeline provides.

* SBOM (Software Bill of Materials) lists components used in your software. When a CVE is disclosed, SBOMs help you quickly identify impacted services. Common SBOM formats include `SPDX` and `CycloneDX`. Tools such as `Syft` and `Trivy` can generate SBOMs for images and artifacts.

<Frame>
  <img src="https://mintcdn.com/kodekloud-c4ac6d9a/uBQs-hUjzRb0XBPP/images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/slsa-supply-chain-levels-sbom-comparison.jpg?fit=max&auto=format&n=uBQs-hUjzRb0XBPP&q=85&s=e2e6879455095580ba7715073480d559" alt="The image compares SLSA's supply-chain levels for software artifacts with SBOM's software bill of materials, highlighting key aspects and tools for each." width="1920" height="1080" data-path="images/Prep-Course-Certified-Cloud-Native-Platform-Engineer-CNPE/Security-and-Policy-Enforcement/Security-in-Delivery-Build-Pipelines-That-Ship-Safely/slsa-supply-chain-levels-sbom-comparison.jpg" />
</Frame>

## Quick reference and recommended commands

* Block images with unacceptable vulnerabilities:

```bash theme={null}
trivy image --exit-code 1 --severity CRITICAL,HIGH registry.company.com/app:v1.2
```

* Sign images with Cosign (key-based example):

```bash theme={null}
cosign generate-key-pair
cosign sign --key cosign.key registry.company.com/app:v1.2
cosign verify --key cosign.pub registry.company.com/app:v1.2
```

* Enforce signed images with Kyverno by verifying signatures at admission.

## Further reading and references

* Trivy: [https://github.com/aquasecurity/trivy](https://github.com/aquasecurity/trivy)
* Cosign / Sigstore: [https://sigstore.dev/](https://sigstore.dev/)
* Kyverno: [https://kyverno.io/](https://kyverno.io/)
* SLSA: [https://slsa.dev/](https://slsa.dev/)
* SBOM formats: [https://spdx.dev/](https://spdx.dev/), [https://cyclonedx.org/](https://cyclonedx.org/)
* Syft (SBOM generation): [https://github.com/anchore/syft](https://github.com/anchore/syft)

## Wrap up

A secure delivery pipeline enforces the four gates—build, scan, sign, and admit—automatically. Implement these controls to move security left into CI/CD, reduce the risk of supply-chain compromises, and ensure only verified artifacts reach your cluster.

<CardGroup>
  <Card title="Watch Video" icon="video" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/4fa48a7f-b1e6-4e09-bded-51f38be9355d" />

  <Card title="Practice Lab" icon="flask-conical" cta="Learn more" href="https://learn.kodekloud.com/user/courses/prep-course-certified-cloud-native-platform-engineer-cnpe/module/35a7fadb-02d8-4557-a819-2e4dcfa970cc/lesson/b53de961-c16f-40c3-bae8-75ab707cb814" />
</CardGroup>
