# Enterprise Cryptographic Asset Discovery

## Architecture and Implementation for AWS Multi-Landing Zones, GitLab CI/CD, Prisma Cloud CSPM, and Colocation

**Document purpose:** Define a practical architecture and implementation
approach for discovering cryptographic assets across hundreds of AWS
accounts and selected colocation (colo) resources, using GitLab for
deployments and Prisma Cloud CSPM for cloud posture and asset context.

**Scope:** AWS landing zones (including GovCloud where authorized), EC2,
ECS, EKS, Lambda, colo containers, colo VMs/servers, GitLab
infrastructure/application pipelines, Prisma Cloud CSPM, and a dedicated
cryptographic discovery platform such as Keyfactor AgileSec.

> **Important:** Validate current product capabilities, licensing,
> supported deployment models, API availability, and authorization
> boundaries with the relevant vendors and internal security teams
> before implementation. Prisma Cloud CSPM should not be assumed to
> provide complete cryptographic asset discovery or filesystem-level
> inspection by itself.

------------------------------------------------------------------------

## 1. Executive Summary

For an enterprise with hundreds of AWS accounts, multiple landing zones,
and a few colo environments, cryptographic discovery should be a
centralized and continuous inventory capability---not a one-time scan.

The recommended design combines three collection layers:

1.  **Build and deployment discovery through GitLab:** Identify
    cryptographic assets in source code, dependencies, packages,
    container images, and deployment configuration before or during
    deployment.
2.  **Cloud and workload discovery:** Enumerate AWS resources and
    collect runtime workload context through AWS APIs, Prisma Cloud
    CSPM, and dedicated sensors or scanners.
3.  **Central cryptographic inventory:** Normalize and correlate
    findings using stable identifiers, deployment metadata, asset
    fingerprints, and observation timestamps.

The systems have complementary roles:

  -----------------------------------------------------------------------
  System                              Primary responsibility
  ----------------------------------- -----------------------------------
  GitLab CI/CD                        Build, scan, deploy, and record
                                      artifact/deployment lineage

  AWS Organizations and AWS APIs      Enumerate accounts, Regions, and
                                      AWS-managed cryptographic resources

  Prisma Cloud CSPM                   Provide cloud asset, configuration,
                                      and posture context

  AgileSec or another cryptographic   Discover cryptographic assets
  discovery tool                      through supported sensors and
                                      integrations

  Central inventory                   Correlate records, track changes,
                                      ownership, and coverage gaps
  -----------------------------------------------------------------------

**Core principle:** GitLab provides evidence of what was built and
deployed; AWS and Prisma provide cloud resource context; cryptographic
scanners identify cryptographic assets; the central inventory correlates
the evidence.

------------------------------------------------------------------------

## 2. Goals and Questions to Answer

The solution should help answer:

-   Where are certificates, private keys, public keys, keystores,
    cryptographic libraries, and related artifacts present?
-   Which AWS account, landing zone, Region, application, workload, or
    colo system contains each asset?
-   What algorithms and key parameters are in use?
-   Which certificates are expiring, and which assets may require
    remediation or migration?
-   What cryptography was introduced by a given deployment?
-   Which discovered assets are currently present in a running workload
    or exposed through a network endpoint?
-   Which assets are managed by AWS services versus embedded in
    applications or operating systems?
-   Which accounts, Regions, hosts, repositories, images, and endpoints
    have not been scanned?
-   Can results support security assessment, compliance reporting, and
    post-quantum cryptography (PQC) planning?

The inventory must distinguish between **configured**, **detected**, and
**observed in use**. These are different evidence levels.

------------------------------------------------------------------------

## 3. Discovery Methods

No single service discovers every kind of cryptographic asset. Use
complementary methods.

  --------------------------------------------------------------------------
  Method               Typical scope     What it can       Key limitation
                                         reveal            
  -------------------- ----------------- ----------------- -----------------
  AWS control-plane    KMS, ACM, cloud   AWS-managed       Does not inspect
  inventory            resource          key/certificate   application
                       configurations    metadata and      filesystems or
                                         resource settings prove application
                                                           usage

  Host scanning        EC2 and colo      Certificates,     Requires host
                       servers/VMs       keys, keystores,  access, supported
                                         crypto libraries  OS, and
                                         in accessible     permissions
                                         filesystems       

  Container/artifact   ECR and other     Cryptographic     Does not prove
  scanning             registries,       assets embedded   the asset is
                       packages, build   in image layers   loaded or used at
                       artifacts         or packages       runtime

  Network/TLS          Authorized        Certificates and  Sees exposed
  discovery            network endpoints TLS configuration endpoints, not
                                         presented by      all local assets
                                         reachable         or internal key
                                         services          stores

  Source/dependency    GitLab            Crypto APIs,      Source may differ
  scanning             repositories and  libraries,        from the final
                       dependencies      embedded          artifact or
                                         certificates,     runtime
                                         potential key     configuration
                                         material          

  Central              Multiple          Aggregated asset  Coverage depends
  cryptographic        supported sources inventory,        on deployed
  platform                               correlation,      sensors,
                                         analysis,         integrations,
                                         reporting         permissions, and
                                                           product support
  --------------------------------------------------------------------------

### Evidence categories

-   **Configured:** A resource or application configuration references
    an asset or algorithm.
-   **Detected:** A scanner found an asset in source, an image, a file,
    or a store.
-   **Observed:** A runtime or network observation provides evidence the
    asset or cryptographic configuration was active.
-   **Inferred:** An association is derived from metadata or indirect
    evidence and should be labeled as such.

------------------------------------------------------------------------

## 4. Target Architecture

### 4.1 Logical flow

``` text
                          Enterprise Security / Cloud Foundation
                                           |
                             Central orchestration and inventory
                                           |
            +------------------------------+------------------------------+
            |                              |                              |
       GitLab CI/CD                  AWS Organizations               Prisma CSPM
            |                              |                              |
    Source/build scans              Cross-account APIs             Cloud asset context
    Container scans                 KMS / ACM inventory             Config/posture data
    Deployment ledger               Workload metadata               Resource identifiers
            |                              |                              |
            +------------------------------+------------------------------+
                                           |
                                 Cryptographic sensors
                                           |
                   +-----------------------+-----------------------+
                   |                       |                       |
                AWS workloads          Registries/code          Colo environment
              EC2/ECS/EKS/Lambda       and artifacts             hosts/containers
                   |                       |                       |
                   +-----------------------+-----------------------+
                                           |
                              Central cryptographic inventory
                                           |
                         Correlation · Coverage · Risk · Reporting
```

### 4.2 Architecture principles

1.  **Centralize orchestration, not necessarily execution.** Keep
    account enumeration, scheduling, normalization, and reporting
    centralized. Run sensors close to restricted networks when required.
2.  **Use least privilege.** Prefer read-only cross-account roles for
    inventory and narrowly scoped access for scanning.
3.  **Use immutable artifact identity.** Correlate container deployments
    by image digest rather than mutable tags alone.
4.  **Separate discovery from enforcement.** Discovery produces
    evidence; policy engines and operational processes decide what to
    block or remediate.
5.  **Track coverage explicitly.** A missing finding is not proof of
    absence unless the relevant source was successfully scanned.
6.  **Avoid centralizing secret material.** Store metadata and
    fingerprints where possible, not private-key bytes.
7.  **Respect authorization boundaries.** Confirm that platform, sensor,
    telemetry, and result storage locations are approved for each
    environment, especially GovCloud and regulated systems.

------------------------------------------------------------------------

## 5. Roles of Each System

### 5.1 GitLab CI/CD

Use GitLab for:

-   Scanning source code and dependencies.
-   Scanning final container images and build artifacts.
-   Recording pipeline, commit, artifact, and deployment metadata.
-   Triggering supported scanner APIs or jobs.
-   Publishing scan results and deployment evidence to the central
    inventory.

GitLab deployment history is useful evidence, but does not prove a
workload remains running. Runtime inventory must establish current
presence.

### 5.2 AWS Organizations and AWS APIs

Use Organizations to enumerate accounts and centrally orchestrate
account-level collection. In each account, query required Regions using
approved roles.

Potential inventory sources:

-   AWS KMS APIs for key metadata.
-   AWS Certificate Manager APIs for ACM certificate metadata.
-   EC2, ECS, EKS, and Lambda APIs for workload/resource metadata.
-   AWS Config for supported resource configuration and change history.
-   CloudTrail for relevant API activity and KMS usage evidence.

KMS activity can help associate keys with roles or services, but a lack
of observed activity does not prove a key is unused. Infrequent
workloads, backups, and disaster recovery may not appear in a limited
observation window.

### 5.3 Prisma Cloud CSPM

Use Prisma CSPM as a cloud asset and posture context source. Depending
on the enabled product capabilities and integrations, collect:

-   Cloud account and Region.
-   Resource identifiers and types.
-   Configuration/posture findings.
-   Relevant workload and asset metadata exposed by the product/API.

Do **not** assume CSPM alone can inspect every file inside EC2,
container layers, Lambda packages, or colo systems. Validate the exact
Prisma product modules and data available in your deployment. Use
dedicated cryptographic sensors or scanners for deep asset discovery.

### 5.4 AgileSec or another cryptographic discovery platform

Use the platform and its supported sensors for deeper discovery across
repositories, container images, hosts, network endpoints, and cloud
services, subject to the purchased product, supported integration, and
deployment model.

Validate specifically:

-   Which sensor types are licensed and supported.
-   Which AWS partitions and Regions are supported.
-   Whether the relevant GovCloud environment is authorized and
    technically supported.
-   How private registries and ECR authentication are configured.
-   Whether scanning is agent-based, remote, API-triggered, scheduled,
    or a combination.
-   What asset types and file formats each sensor actually detects.
-   How scan results are exported and correlated.

------------------------------------------------------------------------

## 6. AWS-Native Discovery

### 6.1 KMS inventory

Collect, where available and permitted:

-   AWS account ID and Region.
-   Key ARN and key ID.
-   Key type, spec, and usage.
-   Key state and enabled status.
-   Rotation configuration where applicable.
-   Aliases and tags.
-   Key origin and relevant metadata.
-   Discovery timestamp and source.

Limitations:

-   AWS KMS metadata is not a complete inventory of application-managed
    keys.
-   A key's existence does not prove a particular application uses it.
-   Do not assume all key material is retrievable or should be
    retrieved.
-   Correlate with resource configuration, CloudTrail, and application
    evidence where appropriate.

### 6.2 ACM inventory

Collect, where exposed:

-   Certificate ARN.
-   Account and Region.
-   Domain names and subject alternative names.
-   Issuer and certificate status.
-   Algorithm and public-key parameters.
-   Expiration and renewal status.
-   In-use status and associated resources where available.

ACM inventory does not cover every certificate installed directly on
hosts, appliances, containers, or colo systems.

### 6.3 CloudTrail and AWS Config

Use CloudTrail for relevant activity evidence and AWS Config for
supported resource configuration/history. These sources provide context,
not complete filesystem-level cryptographic discovery.

Use caution when interpreting "unused" assets. A short period with no
observed API activity is insufficient evidence for deletion.

------------------------------------------------------------------------

## 7. Deployment-Time Discovery in GitLab

### 7.1 Pipeline model A: Terraform provisions infrastructure

Recommended flow:

``` text
Terraform plan
    |
IaC compliance + cryptographic configuration checks
    |
Terraform apply
    |
Capture outputs, resource IDs, account, Region, and deployment metadata
    |
Trigger targeted AWS inventory and applicable asset scans
    |
Publish results to central inventory
```

Pre-apply checks can inspect:

-   KMS key and ACM certificate configuration.
-   TLS and encryption settings represented in Terraform.
-   Accidental private-key material in Terraform files or artifacts.
-   Required ownership, application, and environment tags.
-   Relevant cryptographic policy requirements.

Post-apply collection should record resource identifiers from outputs or
other approved deployment evidence and reconcile them with AWS
inventory.

Terraform checks do not inspect the deployed application filesystem and
do not prove actual runtime use.

### 7.2 Pipeline model B: infrastructure and application pipelines are separate

Recommended flow:

``` text
Infrastructure pipeline:
Terraform apply -> environment/resource reference

Application pipeline:
Build -> source/dependency scan -> image/artifact scan
       -> publish immutable digest -> deploy

Deployment ledger:
Join application release to target environment and resource identities
```

The infrastructure pipeline should publish a stable environment
reference. The application pipeline should record its own release
identity and deployment event. Avoid assuming an infrastructure apply
represents every later application release.

### 7.3 What to scan in GitLab

  -----------------------------------------------------------------------
  Target                              Potential findings
  ----------------------------------- -----------------------------------
  Application source                  Crypto APIs, libraries, embedded
                                      certificates/public keys, potential
                                      secret exposure

  Dependencies                        Cryptographic packages and versions

  Terraform                           KMS/ACM references, TLS settings,
                                      encryption configuration

  Container image                     Certificates, supported
                                      key/keystore formats, crypto
                                      libraries

  Build artifacts                     Crypto assets packaged in
                                      JAR/WAR/ZIP or other formats

  Lambda package/layers               Bundled certificates, libraries,
                                      dependencies, and supported
                                      cryptographic artifacts
  -----------------------------------------------------------------------

Scan both source and final artifacts because source content can differ
from what is packaged and deployed.

### 7.4 Illustrative GitLab CI structure

This is a pipeline structure, not a vendor-specific ready-to-run sensor
configuration. Replace the placeholder commands with the approved
scanner CLI/API and authentication method.

``` yaml
stages:
  - validate
  - build
  - crypto_scan
  - publish
  - deploy
  - record

crypto_source_scan:
  stage: crypto_scan
  script:
    - echo "Run approved source and dependency cryptographic scan"
    - echo "Publish scan ID and result reference"

crypto_container_scan:
  stage: crypto_scan
  script:
    - echo "Scan the final image using its immutable digest"
    - echo "Publish scan ID and result reference"

deploy:
  stage: deploy
  script:
    - echo "Deploy approved immutable artifact"
    - echo "Capture target account, Region, and resource identifiers"
  environment:
    name: production

record_deployment:
  stage: record
  script:
    - echo "Publish deployment ledger record"
    - echo "Include pipeline, commit, digest, target, and timestamps"
```

Ensure pipeline dependencies enforce the intended order. Whether a scan
blocks deployment should be determined by policy, severity, exceptions,
and your existing compliance process.

------------------------------------------------------------------------

## 8. Runtime Discovery by Workload Type

  -----------------------------------------------------------------------
  Workload                Recommended combination Important gaps
  ----------------------- ----------------------- -----------------------
  EC2                     AWS inventory +         CSPM resource
                          supported host sensor   visibility does not
                          or authorized remote    imply filesystem
                          scan                    visibility

  ECS                     ECR image scan + ECS    Fargate does not
                          service/task metadata + provide ordinary host
                          optional host/runtime   access; image scan
                          scan                    cannot see
                                                  runtime-injected assets

  EKS                     Registry scan +         Mounted secrets and
                          Kubernetes workload     externally injected
                          metadata + supported    assets require separate
                          node/pod discovery      coverage

  Lambda                  Deployment              No persistent host to
                          ZIP/container image and scan; runtime evidence
                          layer scans + Lambda    is supplementary
                          configuration/version   
                          inventory               

  Colo containers         Registry scan +         Private networking and
                          container/cluster       registry authentication
                          metadata + remote       must be handled
                          sensor                  

  Colo VMs/servers        Host sensor or approved Coverage depends on OS
                          scheduled remote        support, credentials,
                          scanning                access, and file
                                                  permissions
  -----------------------------------------------------------------------

### 8.1 EC2

Enumerate instances and correlate them with host discovery. Capture
instance ID, account, Region, OS, application owner, and scan
timestamps. Use a supported sensor or authorized remote scanner with
approved filesystem access.

### 8.2 ECS

Scan ECR images before deployment. After deployment, collect
service/task identity and the image digest actually running. For ECS on
EC2, host discovery may add coverage. For Fargate, use image scanning
and supported runtime/integration methods rather than assuming host
access.

### 8.3 EKS

Collect cluster, namespace, workload, pod, image digest, and node
identity where available. Scan images and separately account for mounted
Kubernetes Secrets, external secret stores, and runtime-injected
certificates. Do not copy secret values into the inventory.

### 8.4 Lambda

Scan deployment packages, container images, and layers. Inventory
function configuration, runtime, versions, aliases, and deployment
timestamps. Match findings to the specific deployed version/package
digest where possible.

### 8.5 Colo

Place remote sensors or scanners inside approved colo network segments
when central services cannot directly reach the systems. Cover internal
registries, servers, appliances, internal PKI, and HSM-related metadata
where supported and authorized.

------------------------------------------------------------------------

## 9. Correlation and Central Data Model

Maintain at least three linked datasets rather than a single flat table.

### 9.1 Deployment record

One record per application or infrastructure deployment.

  Field                  Description
  ---------------------- --------------------------------------------
  deployment_id          Unique deployment event ID
  gitlab_project         Repository/project identity
  pipeline_id / job_id   Traceable pipeline execution
  commit_sha             Source revision
  artifact_digest        Immutable image/package identity
  artifact_type          Container, ZIP, package, etc.
  environment            Dev/test/prod or approved environment name
  landing_zone           Landing zone identifier
  account_id / region    Deployment target
  resource_ids           EC2, ECS, EKS, Lambda, or other target IDs
  deployed_at            Deployment timestamp in UTC
  scan_ids               Relevant scan references
  result_reference       Link or identifier for stored results

### 9.2 Cryptographic asset record

One record per discovered asset.

  -----------------------------------------------------------------------
  Field                               Description
  ----------------------------------- -----------------------------------
  asset_id                            Stable asset identity

  asset_type                          Certificate, key, keystore,
                                      library, token, etc.

  fingerprint                         Certificate/key fingerprint where
                                      supported

  algorithm                           RSA, ECC, AES, SHA family, etc.

  key_size / parameters               Key size, curve, or parameters
                                      where available

  issuer / expiration                 Certificate metadata where
                                      applicable

  asset_location                      Repository path, image path,
                                      host/store, endpoint, or AWS
                                      resource

  discovery_source                    GitLab, AgileSec, AWS API, host,
                                      network, etc.

  first_seen / last_seen              Observation timestamps

  scan_status                         Success, partial, failed, not
                                      scanned

  evidence_level                      Configured, detected, observed,
                                      inferred

  owner / application                 Responsible team or service, if
                                      known
  -----------------------------------------------------------------------

### 9.3 Runtime observation record

One record per asset/workload observation.

  Field                 Description
  --------------------- -----------------------------------------------
  asset_id              Reference to cryptographic asset
  workload_id           Instance, task, pod, function, or server
  account_id / Region   Runtime location
  cluster / namespace   Where applicable
  image_digest          Image observed running
  deployment_id         Matched deployment, if known
  observed_at           Observation timestamp
  evidence_type         Image scan, host scan, TLS scan, API metadata
  confidence            Directly observed, inferred, unmatched
  sensor_id / scan_id   Source and traceability

### 9.4 Correlation keys

Prefer the following, in order where applicable:

-   Exact certificate or key fingerprint.
-   Immutable container image digest.
-   AWS resource ARN or stable resource ID.
-   GitLab project + commit SHA + artifact digest.
-   Kubernetes cluster/namespace/workload identity.
-   ECS cluster/service/task identity.
-   Lambda function/version/package digest.
-   Host identity plus asset path/fingerprint.
-   Network endpoint plus certificate fingerprint and observation time.

Names and mutable tags are useful attributes but are weak primary join
keys.

------------------------------------------------------------------------

## 10. Prisma CSPM Integration

1.  Identify the Prisma CSPM API/export mechanisms enabled and approved
    in the environment.
2.  Extract cloud resource identifiers, account/Region, resource type,
    and available configuration/posture findings.
3.  Normalize identifiers to the central inventory schema.
4.  Match Prisma resources to AWS API inventory and deployment ledger
    entries.
5.  Use Prisma context to enrich cryptographic findings with
    workload/resource ownership and posture information where available.
6.  Track unmatched records and stale observations rather than silently
    dropping them.

Validate whether the deployed Prisma product exposes workload-level
metadata, image digests, or runtime evidence. Do not assume those fields
are available from CSPM alone.

------------------------------------------------------------------------

## 11. Central Collection and Storage

A practical logical design:

``` text
Collectors:
  - AWS account/Region inventory collector
  - GitLab deployment event publisher
  - Prisma CSPM API/export collector
  - AgileSec/sensor result collector
  - Colo sensor/result collector

Normalization:
  - Map vendor fields to common schema
  - Normalize account, Region, timestamps, identifiers
  - Deduplicate by fingerprint and source identity
  - Preserve provenance and raw-result references

Storage:
  - Deployment ledger
  - Cryptographic asset inventory
  - Runtime observations
  - Scan coverage/status history

Reporting:
  - Coverage by account/Region/workload/source
  - Cryptographic asset trends
  - Certificate expiration and algorithm inventory
  - Unmatched and stale records
  - Remediation and ownership views
```

Choose storage and reporting services that are approved for the data
classification and cloud boundary. Avoid storing secret bytes or
unnecessary sensitive scan content.

------------------------------------------------------------------------

## 12. Event-Driven and Scheduled Operations

Suggested initial cadence (adjust after measuring scan duration, risk,
and operational impact):

  Discovery activity                 Starting cadence
  ---------------------------------- ------------------------------------------
  GitLab source and artifact scans   Every relevant build
  Deployment ledger updates          Every deployment
  AWS KMS/ACM/resource inventory     Daily
  Prisma CSPM synchronization        Daily or supported update cadence
  Host and colo scans                Daily or weekly based on criticality
  Network TLS discovery              Daily or weekly for authorized endpoints
  Full reconciliation                Weekly or monthly, depending on scale

Potential triggers:

-   GitLab deployment completed.
-   New image digest published.
-   New AWS account or Region onboarded.
-   New EC2/ECS/EKS/Lambda resource detected.
-   Certificate nearing expiration.
-   Periodic scheduled reconciliation.

A missing finding must not be interpreted as confirmed removal unless
the relevant scan succeeded and the source was covered.

------------------------------------------------------------------------

## 13. Security and GovCloud Considerations

Before deploying collectors or sensors:

-   Confirm product and sensor support for the relevant AWS partition
    and Regions.
-   Confirm authorization for GovCloud and regulated environments.
-   Validate platform ingestion and result-storage boundaries.
-   Use least-privilege, temporary credentials where supported.
-   Restrict access to repositories, registries, hosts, and secrets.
-   Do not collect or centralize private-key bytes unless explicitly
    required and formally approved.
-   Keep scanner credentials in approved secret-management systems.
-   Ensure remote sensors can communicate only with approved endpoints.
-   Record scan failures, permission gaps, and unsupported targets.
-   Validate that Prisma APIs and integrations are available and
    authorized in each target environment.

Private networking and colo environments may require customer-managed
remote sensors located inside the relevant network boundary.

------------------------------------------------------------------------

## 14. Rollout Plan

### Phase 1 --- Establish cloud inventory baseline

-   Enumerate AWS accounts from Organizations.
-   Establish a cross-account read-only inventory role.
-   Collect KMS, ACM, EC2, ECS, EKS, and Lambda metadata.
-   Capture account/Region scan coverage and collection timestamps.
-   Validate the central data schema.

**Deliverable:** Account and cloud-resource baseline with explicit
coverage.

### Phase 2 --- Integrate GitLab deployment evidence

-   Add source/dependency and artifact scan jobs.
-   Record immutable image digests and package checksums.
-   Capture pipeline, commit, environment, target account, Region, and
    resource IDs.
-   Publish structured deployment metadata.

**Deliverable:** Traceability from deployment to source and artifact.

### Phase 3 --- Integrate Prisma CSPM

-   Ingest approved cloud asset and posture data.
-   Normalize resource identifiers.
-   Match Prisma resources to AWS inventory and deployment records.
-   Track unmatched and stale observations.

**Deliverable:** Cloud context joined to deployment evidence.

### Phase 4 --- Add cryptographic sensors and workload discovery

-   Scan ECR and other approved registries.
-   Scan GitLab repositories and artifacts.
-   Add host discovery for EC2 and colo.
-   Add workload-specific discovery for ECS, EKS, and Lambda.
-   Add authorized TLS endpoint discovery.

**Deliverable:** Deeper cryptographic asset visibility.

### Phase 5 --- Continuous monitoring and reporting

-   Schedule recurring scans and reconciliation.
-   Trigger targeted scans after deployments and image changes.
-   Report coverage, ownership, risk, and asset changes.
-   Integrate actionable findings into existing compliance and
    remediation processes.

**Deliverable:** Operational cryptographic inventory with ongoing
coverage.

------------------------------------------------------------------------

## 15. Readiness Checklist

-   [ ] All AWS accounts and required Regions are enumerated.
-   [ ] Cross-account roles and permissions are approved.
-   [ ] KMS and ACM inventory is collected centrally.
-   [ ] GitLab deployments record immutable artifact identities.
-   [ ] Source, dependency, and artifact scanning is integrated.
-   [ ] Prisma CSPM data is ingested and normalized.
-   [ ] EC2 host discovery is implemented where required.
-   [ ] ECS/EKS image and runtime metadata are correlated.
-   [ ] Lambda packages, layers, and versions are covered.
-   [ ] Colo containers and hosts have an approved discovery path.
-   [ ] Network TLS scanning scope is authorized and documented.
-   [ ] Findings include provenance, timestamps, and evidence level.
-   [ ] Scan failures and coverage gaps are visible.
-   [ ] Private-key material is not unnecessarily centralized.
-   [ ] GovCloud and regulated-boundary requirements are satisfied.
-   [ ] Owners and remediation workflows are defined.

------------------------------------------------------------------------

## 16. Pilot Recommendation

Start with one representative application and one AWS account that
exercises your common deployment path.

1.  Build and scan the source and final container image in GitLab.
2.  Publish and deploy the image using an immutable digest.
3.  Record GitLab deployment metadata and AWS resource identifiers.
4.  Retrieve the corresponding cloud asset context from Prisma CSPM and
    AWS APIs.
5.  Run cryptographic discovery against the image and, where applicable,
    the running workload.
6.  Correlate the results in the central inventory.
7.  Validate the joins, coverage status, and evidence level manually.
8.  Expand to other workload types and accounts after the pilot is
    reliable.

**Target outcome:** A repeatable pattern that scales across landing
zones and distinguishes what was built, what was deployed, what is
currently observed, and what remains unscanned.

------------------------------------------------------------------------

## 17. Vendor Documentation and Validation

Use current official documentation and your organization's licensed
product documentation to verify implementation details:

-   Keyfactor AgileSec documentation:
    https://docs.keyfactor.com/agilesec/latest/
-   AWS Organizations: https://docs.aws.amazon.com/organizations/
-   AWS KMS: https://docs.aws.amazon.com/kms/
-   AWS Certificate Manager: https://docs.aws.amazon.com/acm/
-   AWS Config: https://docs.aws.amazon.com/config/
-   AWS CloudTrail: https://docs.aws.amazon.com/awscloudtrail/
-   GitLab CI/CD documentation: https://docs.gitlab.com/ci/
-   Prisma Cloud documentation: https://docs.prismacloud.io/

Confirm product editions, current API fields, sensor capabilities,
supported platforms, licensing, and GovCloud availability before
treating any integration as supported.
