# Keyfactor AgileSec – Cryptographic Discovery and Scanning Strategy

## 1. Overview

The organization operates a hybrid technology environment consisting primarily of AWS GovCloud Landing Zones, Microsoft Azure, and a limited on-premises/colocation infrastructure footprint.

Infrastructure and application deployments are managed through Terraform Infrastructure as Code (IaC) and GitLab CI/CD pipelines. Prisma Cloud provides cloud security posture management and runtime visibility.

Keyfactor AgileSec will provide centralized cryptographic discovery, inventory, and compliance visibility across this environment. The objective is to identify cryptographic assets, technologies, configurations, and dependencies; establish an enterprise-wide cryptographic inventory; and support cryptographic compliance and Post-Quantum Cryptography (PQC) readiness initiatives.

The proposed scanning strategy is organized into five major scanning areas.

## 2. Scanning Areas

### 2.1 Source Code and Infrastructure as Code (IaC)

**Scope:** Source repositories and infrastructure/application deployment code.

Target areas:
- GitLab repositories containing application source code.
- Terraform IaC Patterns and Examples.
- Application configuration files and deployment definitions.
- CI/CD pipeline templates and supporting scripts.
- Cryptographic libraries, algorithms, certificates, keys, and other supported cryptographic artifacts embedded in source code and compiled artifacts.

**Scanning approach:**
- Use the AgileSec GitLab Sensor to discover cryptographic assets in GitLab repositories.
- Configure repository inclusion and exclusion patterns to establish the scanning scope.
- Integrate AgileSec scans with GitLab CI/CD pipelines for event-driven scanning where appropriate.
- Configure scheduled scans for ongoing repository discovery.

### 2.2 Cloud Services and Key Management

**Scope:** Cryptographic assets and key-management services provided by cloud platforms.

Target areas:

**AWS GovCloud**
- AWS Key Management Service (KMS).
- AWS Certificate Manager (ACM).
- Other supported AWS cryptographic and key-management services.

**Microsoft Azure**
- Azure Key Vault.
- Other supported Azure cryptographic and key-management services.

**Scanning approach:**
- Use AgileSec Connectors to collect cryptographic information through supported cloud service APIs.
- Configure the required cloud permissions and authentication mechanisms.
- Evaluate AWS and Azure coverage based on the available AgileSec connectors and supported service capabilities.

### 2.3 Runtime and Workload Discovery

**Scope:** Cryptographic assets embedded in deployed workloads and running infrastructure.

Target areas:
- Amazon EC2 instances.
- Amazon ECS and EKS workloads.
- Container images and associated application artifacts.
- Application servers and supported operating system environments.
- Cryptographic libraries, certificates, keys, and supported keystore formats.
- Azure virtual machines and supported container workloads.

**Scanning approach:**
- Use appropriate AgileSec Host and Container Sensors.
- Scan supported hosts, container images, and application artifacts.
- Deploy remote scanning components where customer-managed execution is required.
- Coordinate scanning scope with existing infrastructure and workload inventories.

### 2.4 Network and Endpoint Discovery

**Scope:** Cryptographic properties exposed through network-accessible endpoints.

Target areas:
- Application endpoints.
- Internal and external-facing APIs.
- Load balancers and web application endpoints.
- TLS-enabled services.
- SSH-enabled endpoints and other supported network protocols.

**Scanning approach:**
- Use AgileSec Network Scan capabilities to discover supported cryptographic properties exposed by network endpoints.
- Identify supported certificates, protocols, cipher suites, and cryptographic configurations.
- Define network targets, access requirements, and scanning schedules.
- Coordinate with network and security teams to ensure appropriate connectivity and scanning authorization.

### 2.5 Hybrid and On-Premises Infrastructure

**Scope:** Cryptographic assets within the organization's colocation and on-premises infrastructure.

Target areas:
- Physical and virtual servers.
- Application infrastructure.
- Internal services and systems.
- Supported databases and key-management infrastructure.
- Other infrastructure components containing cryptographic assets.

**Scanning approach:**
- Use AgileSec Remote Scan execution and appropriate Sensors.
- Run scanning components within customer-managed infrastructure when required.
- Support environments that cannot be directly reached by centrally executed scans.
- Configure network access, execution permissions, and scheduling based on infrastructure requirements.

## 3. AgileSec Scanning and Integration Architecture

AgileSec supports different execution models depending on the Sensor or Connector and the target environment.

### 3.1 Platform-Based Scanning

Sensors and Connectors can be configured and executed through the AgileSec platform.

- Configure scanning targets and credentials.
- Execute scans on demand.
- Establish recurring scanning schedules.
- Process and consolidate findings within the AgileSec platform.

### 3.2 API-Based Scanning

AgileSec provides REST APIs for programmatic scan execution.

This approach supports integration with:
- GitLab CI/CD pipelines.
- Enterprise automation workflows.
- Other internal platforms and agentic systems.

For GitLab integration, pipelines can invoke the AgileSec Scan API to initiate scans when required.

### 3.3 Remote Scanning

Remote scanning uses the AgileSec Unified Sensor framework deployed in customer-managed environments.

This approach is applicable when:
- Target systems are hosted in restricted internal networks.
- The AgileSec platform cannot directly reach the target environment.
- Scanning execution must remain within a customer-managed network boundary.
- Local scheduling and execution are required.

The execution model should be selected according to the target environment, network connectivity, and applicable security requirements.

## 4. GitLab Integration Strategy

GitLab is the primary source-code and IaC repository platform and will be an important integration point for cryptographic discovery.

### 4.1 GitLab Sensor Configuration

The GitLab Sensor configuration includes:

| Configuration | Purpose |
|---|---|
| Base GitLab URL | Identifies the GitLab instance. |
| Accepted Paths | Defines the repositories included in the scan. |
| Excluded Paths | Defines repositories excluded from the accepted scanning scope. |
| Branch | Specifies the branch or branches to scan. |
| Authentication Token | Provides access to the GitLab repositories. |
| Incremental Scan | Enables incremental scanning where supported and configured. |

Accepted Paths supports glob patterns and can be used to define scanning scope at an organizational group or subgroup level.

For example:

- `Cloud-Foundation-Services/**` – Include projects under the specified group and its subgroups.
- `Cloud-Foundation-Services/AWS/**` – Include projects under the AWS subgroup.
- `Cloud-Foundation-Services/AWS/Terraform-Patterns` – Include a specific project.

Excluded Paths can be used to remove selected repositories from the included scope.

The GitLab authentication token must have the required repository access and permissions.

### 4.2 GitLab Pipeline Integration

A reusable AgileSec integration template should be maintained centrally and consumed by application and infrastructure repositories.

Recommended implementation:

- Maintain a centrally managed GitLab pipeline template for AgileSec scanning.
- Define AgileSec API credentials as protected and masked group-level CI/CD variables, or retrieve them from an approved enterprise secrets-management service.
- Use a dedicated service identity and appropriately scoped credentials.
- Allow application teams to include the centrally maintained template in their pipelines.
- Support on-demand scans triggered by selected pipeline events, such as merge requests or code changes, where appropriate.
- Maintain scheduled scans for broader repository coverage.

The AgileSec API token and GitLab repository authentication token are separate credentials and must be managed independently.

Not every GitLab pipeline needs to initiate a scan. A combination of scheduled enterprise-level scanning and targeted pipeline-triggered scanning can provide ongoing discovery without unnecessary duplicate executions.

## 5. Credential and Security Management

The following security practices should be applied:

- Use dedicated automation identities where supported.
- Apply least-privilege permissions to cloud and GitLab access.
- Store API tokens and authentication credentials in approved secrets-management services or protected GitLab CI/CD variables.
- Avoid embedding credentials in Terraform code, repository files, pipeline YAML, or application configuration.
- Establish token expiration, rotation, and revocation procedures.
- Separate credentials across environments or trust boundaries where appropriate.
- Ensure scanning execution complies with Landing Zone network and security requirements.

## 6. Relationship with Existing Security Platforms

### Prisma Cloud

Prisma Cloud will continue to provide cloud security posture management and runtime security visibility.

AgileSec will complement this capability by focusing on cryptographic asset discovery, cryptographic inventory, and cryptographic compliance.

The two platforms serve complementary purposes. Any direct integration, automated correlation, or data exchange between Prisma Cloud and AgileSec should be validated against supported product capabilities rather than assumed.

### GitLab CI/CD and Terraform IaC

GitLab provides the source repository and automation framework for infrastructure and application deployments.

AgileSec adds cryptographic discovery to this development and deployment lifecycle through repository scanning and optional pipeline-triggered execution.

## 7. Recommended Implementation Approach

The implementation should proceed in the following phases.

| Phase | Focus | Key Activities |
|---|---|---|
| Phase 1 | GitLab Discovery | Configure GitLab Sensor, establish repository scope, and validate cryptographic discovery. |
| Phase 2 | Cloud Services | Configure supported AWS GovCloud and Azure Connectors. |
| Phase 3 | Runtime Discovery | Enable Host and Container Sensors for selected workloads. |
| Phase 4 | Network Discovery | Establish Network Scan targets and validate endpoint coverage. |
| Phase 5 | Hybrid Infrastructure | Deploy Remote Sensors for selected on-premises and colocation environments. |
| Phase 6 | Automation and Reporting | Integrate selected GitLab pipelines, establish recurring scans, and operationalize reporting. |

## 8. Expected Outcomes

The proposed AgileSec implementation is intended to provide:

- Centralized cryptographic asset inventory across AWS GovCloud, Azure, and on-premises environments.
- Visibility into cryptographic assets in source code, IaC, cloud services, workloads, and network endpoints.
- Identification of cryptographic technologies, algorithms, certificates, and supported key-related artifacts.
- Improved visibility into cryptographic dependencies across applications and infrastructure.
- Support for enterprise cryptographic compliance and policy assessments.
- Improved readiness for Post-Quantum Cryptography (PQC) migration planning.
- Integration with existing GitLab CI/CD and enterprise security processes.

## 9. Important Implementation Considerations

The scanning scope and level of discovery will depend on the specific AgileSec Sensor or Connector, supported target technologies, product licensing, permissions, and network connectivity.

Cloud service Connectors primarily collect information through supported APIs, while Sensors perform direct discovery against supported repositories, hosts, networks, and containers.

Network scanning identifies supported cryptographic properties exposed by endpoints; it should not be treated as a substitute for inspecting internal application source code, host files, or container contents.

Final coverage, execution placement, and integration requirements should be validated with Keyfactor against the organization's AgileSec deployment model and licensed capabilities.