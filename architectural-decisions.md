# Architectural decisions

This document records the decisions that shape the current SparrowX platform, the reasoning behind them, and the trade-offs they introduce. SparrowX is a fictional company; these decisions are part of a portfolio demonstration of platform, infrastructure, and delivery architecture.

## Platform and infrastructure decisions

### 1. Amazon ECS/Fargate instead of Kubernetes

The project targets a small company that needs reliable container delivery without taking on the operational complexity of running a Kubernetes control plane. ECS/Fargate provides task scheduling, rolling deployments, health checks, Service Connect, IAM integration, and AWS-native operations with less platform overhead.

Kubernetes would be reasonable for a larger platform or an organization requiring portability and a broad Kubernetes ecosystem. It is intentionally outside this project’s scope.

### 2. CloudFormation as infrastructure as code

CloudFormation was selected because the target platform is AWS-centric and the templates can directly compose nested stacks, exports, IAM, ECS, ALB, RDS, Secrets Manager, SSM, and CloudFront resources.

The trade-off is more verbose templates and less portability than Terraform. Reusable modules, environment root stacks, and the reusable service template keep the complexity organized.

### 3. One reusable module set, two environment instances

`dev` and `prod` use the same CloudFormation module shape with different parameter files. This avoids maintaining separate infrastructure implementations while preserving environment isolation through distinct VPCs, clusters, ALBs, ECR namespaces, databases, secrets, exports, and deployment histories.

The environments currently share an AWS account for demonstration cost and simplicity. A future production implementation could separate accounts while retaining the same environment-qualified naming model.

### 4. Feature flags for modular environment deployment

Each root stack accepts enable/disable parameters for major nested stacks, including the VPC/network, ECR, ECS cluster, ALB, PostgreSQL, and CloudFront. CloudFormation conditions use those values to decide which nested stacks are created.

This allows an environment to be assembled according to its needs, for example, a lightweight environment may omit databases or CloudFront, without maintaining separate root templates. It also makes the platform easier to extend with additional optional capabilities.

The feature flags are configuration controls, not a replacement for dependency planning. Services still require the foundation components they depend on.

### 5. Three NAT gateways, one per Availability Zone

The network uses three Availability Zones and supports one NAT Gateway in each AZ. Routing private subnets through their local NAT Gateway improves network-layer availability and avoids forcing private-subnet egress through another AZ, which can introduce cross-AZ data-transfer cost and an additional failure dependency.

This is a production-oriented network decision. NAT Gateways are comparatively expensive, so the feature flags allow a lower-cost demonstration configuration when high availability is not the priority. The current environment parameter files enable all three gateways. This decision improves the availability of the network egress layer; it does not by itself make every workload highly available, especially where low-cost RDS or single-task settings are used.

### 6. Separate root infrastructure stacks from service stacks

The root stack owns shared foundations. Each application service is deployed as its own CloudFormation stack using `cloudformation/service.yaml`.

This boundary allows an application team to deploy one service without updating the whole platform and allows the foundation to export the values that service deployments need. It also makes the dependency order explicit: foundation first, services second.

### 7. Bootstrap prerequisites are separate from environment stacks

The root stacks require a few account-level prerequisites, including an S3 bucket for CloudFormation packaging and IAM roles that GitHub Actions can assume. These can be provisioned through a separate bootstrap/pre-stack before creating the `dev` and `prod` root stacks.

Keeping these prerequisites outside the environment stacks avoids circular dependencies and makes the root stacks reusable across accounts or configurations. The deployment workflows receive the artifact bucket, AWS account, region, and role information through GitHub variables rather than hardcoding account-specific values.

## Networking and application runtime decisions

### 8. Private workloads behind an ALB

ECS tasks do not receive public IP addresses. Public requests enter through the Application Load Balancer, which performs path-based routing and health checks. This reduces the public attack surface and gives every service a common ingress contract.

The design keeps the frontend and APIs reachable through environment-specific routes while keeping task-to-database traffic private.

### 9. Service Connect for internal backend traffic

The reporting service needs to call other APIs without routing those calls through the public ALB. ECS Service Connect provides environment-local service names such as `customer-api:8000`, making the private dependency path explicit and avoiding public-network coupling.

### 10. Frontend deployed on ECS alongside the APIs

The React frontend is packaged as an Nginx container and deployed through the same ECS platform as the backend services. This gives a small team one operational model, one deployment family, one environment structure, and one reusable CI/CD approach.

An S3 static-site deployment could be cheaper and is a valid architecture in other contexts. Here, using ECS keeps the frontend onboarding and ownership model consistent with the APIs and avoids creating a separate hosting model and delivery pipeline for a small team.

### 11. A database per stateful service

Customer, notification, task, and billing services each use a separate PostgreSQL database. This demonstrates service-owned persistence boundaries and prevents one sample service from having unrestricted access to another service’s data.

PostgreSQL is the first database stack because it is a practical fit for the relational workloads and integrates cleanly with RDS, Secrets Manager, private networking, and the deployment workflow. The infrastructure is structured so additional database types, such as a different relational engine, DynamoDB, or a cache, could be added as new modules without redesigning the entire platform.

The trade-off is more infrastructure and cost. A real company would evaluate database consolidation, operational overhead, workload requirements, backups, and data lifecycle needs.

## Configuration and security decisions

### 12. Service-owned environment parameter files

Each application repository owns `ecs-parameters-dev.yaml` and `ecs-parameters-prod.yaml`. These files express service-specific values such as ECR repository, container port, sizing, route, listener priority, health paths, database enablement, and upstream URLs.

The shared deployment workflow owns the mechanics; the service repository owns its deployment intent. This is a practical onboarding contract with low repetition.

### 13. SSM Parameter Store for the ACM certificate ARN

The CloudFront module receives the ACM certificate ARN through an SSM Parameter Store parameter rather than hardcoding the ARN in the CloudFormation templates or committing it to Git. The environment configuration stores the parameter path, and CloudFormation resolves the value dynamically when the CloudFront nested stack is created.

This is especially useful for CloudFront because its ACM certificate must be in `us-east-1`. The parameter path can remain stable while the certificate value changes independently, and the ARN is not exposed as a literal in the repository configuration.

### 14. GitHub repository variables for deployment context

`AWS_ACCOUNT_ID`, `AWS_ROLE_NAME`, and `AWS_REGION` are configured as GitHub repository or environment variables. The infrastructure workflows also receive `CLOUDFORMATION_ARTIFACT_BUCKET` this way.

This keeps account-specific deployment context out of workflow source files and makes the repositories easier to reuse across AWS accounts, regions, or future configurations. GitHub OIDC then allows workflows to assume the configured role without long-lived AWS access keys.

### 15. Private networking and least-purpose access

ECS tasks and RDS databases run in private subnets. Database credentials are generated and stored in Secrets Manager, security groups limit traffic to required paths, and ECS execution and application task roles are separate.

These controls are combined with immutable image tags, Trivy scanning, health checks, CloudWatch logs, SAST/DAST quality gates, and ECS deployment circuit breakers.

## Delivery and reliability decisions

### 16. Build Once, Promote Many

Images are tagged with the Git commit SHA. CI builds and scans the image once, development deploys that artifact, and production promotion copies the exact image by digest between environment-specific ECR repositories.

This preserves artifact provenance and avoids the risk that a production rebuild differs from the image tested in development.

### 17. SSM Parameter Store for release metadata

The workflows publish image tag, digest, repository, and security metadata to environment-specific SSM parameter paths. Later workflows resolve those values for development deployment, production candidate promotion, and rollback.

This separates artifact metadata from GitHub workflow state and makes the selected image explicit.

### 18. Reusable workflow templates with semantic releases

Application repositories compose reusable GitHub Actions workflows rather than duplicating AWS, Docker, testing, scanning, and deployment logic. GitHub OIDC supplies short-lived AWS access through an IAM role instead of long-lived access keys stored in repositories.

Centralizing the workflows means a platform improvement is made once in `workflows-templates` instead of being copied manually into every service repository. Updating two or three repositories manually may be manageable; updating five to ten becomes error-prone, and doing it across a growing microservice estate becomes unsustainable. Establishing this pattern early makes the platform easier to grow and scale.

The templates can also evolve without forcing every service to upgrade immediately. They are released with semantic tags such as `v4.4.0`, `v4.5.0`, or `v5.0.0`. A change to a reusable workflow is published as a new version, with release notes describing behavior changes, required inputs, security changes, and migration steps. Each microservice can then update its workflow reference at its own pace, or according to an agreed upgrade deadline, after reviewing the release notes and making any required repository changes.

This model also makes workflow capabilities replaceable. For example, the centralized image security step could move from Trivy to Grype, Snyk Container, or another approved scanner without redesigning every application pipeline individually. The trade-off is that workflow inputs, version tags, repository variables, environment protection, and release compatibility must be managed carefully.

The reusable workflows also provide a standardized DevSecOps pipeline across every application repository: pull-request tests and source scanning; post-merge authoritative image builds, Trivy scanning, signing, and metadata publication; automated development deployment with smoke tests, API tests, and DAST; and manually approved production promotion with post-deployment validation. Quality gates are applied at each stage, so security and delivery controls are consistent while runtime-specific inputs remain configurable.

### 19. Immutable image promotion by digest

Production promotion does not rebuild an application image and does not trust a mutable tag by itself. The development pipeline records the image tag and digest in SSM Parameter Store after the image is built, scanned, and successfully deployed. The promotion workflow resolves both values, verifies the source image, uses Skopeo to copy the exact image content from the development ECR namespace to the production ECR namespace, verifies that the target digest matches, and deploys the selected tag.

This preserves artifact identity across environments: the image tested in `dev` is the image deployed in `prod`. The tag remains useful for human-readable deployment history, while the digest provides the content-level guarantee required for immutable promotion. The same mechanism also makes manual rollback possible without rebuilding.

### 20. Production promotion is deliberate

Development deployment is automated after a merge to `main`. Production requires a manually confirmed promotion workflow. The operator explicitly promotes the candidate after development smoke tests pass.

This keeps production deployment visible and gives the project a realistic approval boundary without introducing a complex release-management system.

### 21. Four rollback methods

Rollback is not limited to one mechanism:

- ECS circuit-breaker rollback handles an unhealthy rolling deployment.
- The production pipeline automatically redeploys the previous image when post-deployment production tests fail.
- Git revert restores source-controlled desired state.
- Manual selected-image rollback provides the quickest operator recovery without rebuilding.

Each method addresses a different failure type and preserves a clear audit trail.

## Delivery governance and cost decisions

### 22. Protected `main` branches

Direct pushes to `main` are blocked. Changes must be submitted through pull requests and pass the required CI checks before they can be merged.

This creates a controlled path from change to deployment. It ensures that application, workflow, and infrastructure changes are reviewed, tested, and visible in version history before they can trigger development or production delivery. Force pushes and unauthorized branch deletion should also remain disabled.

This is especially important for the infrastructure and reusable workflow repositories, where an unreviewed change can affect multiple environments or many microservice pipelines.

### 23. Cost-aware demonstration settings

The project uses small task sizes, low desired counts, short log retention, and configurable RDS settings. These choices keep the demonstration affordable and reproducible. They should not be interpreted as production availability or capacity recommendations.

The three-NAT-Gateway network layout demonstrates a production-oriented availability and cost decision, while other settings can be reduced through feature flags when the goal is a low-cost development environment.

### 24. Dual deployment switches with a reusable guard

Deployment enablement is controlled at two layers. Each application repository sets `appStack.state` in its `ecs-parameters-dev.yaml` and `ecs-parameters-prod.yaml`, while `ecs-infrastructure` sets `RootStack.State` in the environment parameters. The reusable `deployment-guard` workflow reads both values and returns `deployment-enabled=true` only when both are `enabled`.

This allows a service owner to pause one application and allows the platform team to pause an entire environment or root stack. The guard is evaluated before deployment-related jobs, so the behavior is standardized without duplicating switch logic in every application workflow.

### 25. Prometheus and Grafana as optional ECS services

Observability is implemented as an optional nested CloudFormation stack in each environment root stack. It deploys Prometheus and Grafana as private Fargate services with Service Connect discovery, CloudWatch logs, security groups, health checks, and deployment circuit breakers.

Prometheus discovers ECS workloads and scrapes their `/metrics` endpoints. Grafana is provisioned with Prometheus as its data source and serves dashboards under the environment’s `/grafana/` path. Prometheus is available under `/prometheus/` for testing when `AllowPublicAccess` is enabled.

Public ALB access is explicitly marked as testing-only in the infrastructure parameters. The default design keeps both services private, while the feature flag permits controlled demonstration access without changing the observability stack itself.
