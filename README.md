# Migrating and Onboarding SparrowX to AWS ECS

This project demonstrates how a small company could migrate a containerized SaaS platform to Amazon ECS/Fargate and onboard its independently deployable services onto a consistent delivery platform.

SparrowX is fictional. The company, team, product, and workloads exist to make the platform engineering problem concrete. The six services are not intended to be a complete production SaaS product; they are demonstration workloads that prove how different application shapes can be onboarded and deployed smoothly to multiple environments.

The project’s central question is:

> How can a small platform team give application developers a reliable path from source code to a healthy service on AWS, without asking every team to design its own networking, ECS, IAM, ECR, database, and deployment process?

## The fictional company

SparrowX is a small B2B SaaS company with a six-person development team. Each developer owns one workload and should be able to deliver changes independently while the platform team provides the shared AWS and CI/CD capabilities.

The product supports customer management, internal task tracking, notifications, billing, reporting, and a browser-based portal. The product domain is intentionally simple; its purpose is to exercise the platform’s traffic and deployment paths.

| Workload | Team responsibility | Runtime role | Persistence / communication |
| --- | --- | --- | --- |
| `customer-api` | Customer experience | Customer CRUD and search API | Private PostgreSQL RDS |
| `notification-api` | Communications | Notification creation and status tracking | Private PostgreSQL RDS |
| `task-api` | Operations | Operational task management | Private PostgreSQL RDS |
| `billing-api` | Finance | Invoices and payment state | Private PostgreSQL RDS |
| `reporting-api` | Analytics | Read-only cross-service reports | Stateless; calls other APIs through ECS Service Connect |
| `web-portal` | Product experience | React/Vite frontend | Stateless; calls APIs through the ALB |

The workloads demonstrate three important traffic paths:

1. **Frontend to backend:** the React web portal calls the backend APIs through the environment’s Application Load Balancer.
2. **Backend to backend:** `reporting-api` calls `customer-api`, `task-api`, and `billing-api` privately through ECS Service Connect.
3. **Backend to database:** the four stateful APIs connect to their own private PostgreSQL databases on Amazon RDS using credentials injected from AWS Secrets Manager.

## The migration and onboarding challenge

The starting point is a small company with independently owned services and a need to move from local/container-based development to AWS ECS. The company needs more than a cluster. It needs a platform that makes the safe path the easy path:

- application repositories should contain service code, tests, a Dockerfile, and a small ECS configuration file;
- shared infrastructure should provide the AWS foundation and reusable service deployment primitive;
- shared workflows should standardize testing, image building, security scanning, deployment, promotion, and smoke testing;
- development and production should be isolated and tracked independently;
- production should receive the exact artifact tested in development;
- deployment failure and urgent recovery should have clear rollback paths;
- onboarding a new service should be documented and repeatable.

The project deliberately focuses on this platform boundary. The microservices are evidence that the onboarding contract works across database-backed APIs, a stateless aggregation API, and a frontend.

## Goals

1. Build a reusable AWS ECS/Fargate platform for a small multi-service company.
2. Separate `dev` and `prod` resources and deployment histories.
3. Give each service a predictable onboarding contract.
4. Implement a reusable CloudFormation service template instead of hand-authoring an ECS stack per service.
5. Centralize CI/CD conventions in reusable GitHub Actions workflows.
6. Apply Build Once, Promote Many so production uses the same immutable artifact tested in development.
7. Demonstrate frontend-to-backend, backend-to-backend, and backend-to-database traffic.
8. Make deployment health visible and rollback practical.
9. Document the trade-offs and remaining gaps honestly.

## Scope

### Included

- AWS VPC networking with public and private subnets.
- Configurable NAT gateways and environment-specific network parameters.
- Amazon ECR repositories and environment-qualified image namespaces.
- Amazon ECS clusters running Fargate tasks without public IP addresses.
- ECS Service Connect for private service discovery.
- Application Load Balancer path-based routing and health checks.
- Private Amazon RDS PostgreSQL databases for stateful services.
- AWS Secrets Manager-generated database credentials.
- IAM task execution and application task roles.
- CloudWatch log groups and deployment feedback.
- Optional CloudFront edge distribution configuration.
- Reusable CloudFormation service deployment.
- Reusable GitHub Actions workflows for Python and Node.js workloads.
- GitHub OIDC access to AWS.
- Trivy container image scanning.
- Environment smoke tests after deployment.
- Three rollback levels: ECS automatic rollback, Git revert, and manual image rollback.

### Deliberately excluded

- Kubernetes, Terraform, Helm, and a Kubernetes-style control plane.
- Product-grade authentication and authorization.
- A real payment provider or notification provider integration.
- Claiming production scale, multi-account governance, or full self-service registration.
- Treating the sample business APIs as the primary achievement; the platform and onboarding experience are the focus.

## What this project is

This is a multi-repository platform engineering demonstration. The repository set represents a GitHub organization:

| Repository | Responsibility |
| --- | --- |
| [`ecs-infrastructure`](https://github.com/SparrowX-ECS/ecs-infrastructure) | Shared CloudFormation foundation, environment stacks, and reusable ECS service template |
| [`workflows-templates`](https://github.com/SparrowX-ECS/workflows-templates) | Reusable GitHub Actions workflows |
| [`customer-api`](https://github.com/SparrowX-ECS/customer-api) | Customer CRUD API |
| [`notification-api`](https://github.com/SparrowX-ECS/notification-api) | Notification lifecycle API |
| [`task-api`](https://github.com/SparrowX-ECS/task-api) | Operational task API |
| [`billing-api`](https://github.com/SparrowX-ECS/billing-api) | Invoice and billing API |
| [`reporting-api`](https://github.com/SparrowX-ECS/reporting-api) | Stateless cross-service reporting API |
| [`web-portal`](https://github.com/SparrowX-ECS/web-portal) | React frontend |

### ECS infrastructure

The [`ecs-infrastructure`](https://github.com/SparrowX-ECS/ecs-infrastructure) repository composes modular CloudFormation resources into separate `dev` and `prod` root stacks. It provisions networking, ECR, ECS, ALB, RDS, Secrets Manager, CloudWatch, Service Connect, exports, and optional CloudFront resources.

The reusable `cloudformation/service.yaml` template is the application onboarding primitive. It creates an ECS task definition, Fargate service, target group, listener rule, service security group, IAM roles, Service Connect configuration, health checks, and CloudWatch log group from service parameters.

### Reusable workflow templates

The [`workflows-templates`](https://github.com/SparrowX-ECS/workflows-templates) repository centralizes the CI/CD building blocks used by all application repositories:

- change detection;
- Python or Node.js tests, with optional PostgreSQL;
- Docker image build and ECR push;
- immutable Git-SHA image tags;
- Trivy vulnerability scanning;
- CloudFormation/ECS deployment;
- deployment smoke tests;
- image metadata publication to SSM Parameter Store;
- candidate image resolution;
- digest-verified image copying between environment ECR repositories.

Each application repository keeps only thin workflow composition and service-specific parameter files.

## Multi-environment delivery

Every workload has separate `ecs-parameters-dev.yaml` and `ecs-parameters-prod.yaml` files. The environments have separate ECS stacks, ECR namespaces, URLs, deployment metadata, and GitHub deployment histories.

The delivery flow is:

```text
Pull request
    │
    ├── change detection
    ├── tests
    ├── Docker build with commit-SHA tag
    ├── Trivy image scan
    └── publish image metadata

Merge to main
    │
    ├── resolve immutable image
    ├── deploy to dev
    ├── run dev smoke test
    └── publish tag and digest as prod candidate

Manual production promotion
    │
    ├── resolve candidate tag and digest
    ├── copy exact image by digest: dev ECR → prod ECR
    ├── deploy to prod
    ├── run prod smoke test
    └── publish deployed prod metadata
```

This is **Build Once, Promote Many**. The production image is not rebuilt. The tag and digest promoted to production are the artifact that passed development validation.

## Rollback options

The project has three rollback levels:

1. **ECS deployment rollback:** the reusable ECS service template enables the deployment circuit breaker with rollback. An unhealthy rolling deployment can return to the previous task definition.
2. **Git revert rollback:** revert the problematic application or deployment commit and let the normal CI/CD pipeline validate and deploy the corrective state.
3. **Manual quicker image rollback:** open the repository’s GitHub **Deployments** tab, select the `prod` environment, open the desired previous deployment, copy its image tag, then run `rollback-prod.yaml` with confirmation `ROLLBACK` and that tag. The workflow redeploys the immutable image and runs a production smoke test without rebuilding.

## Security and reliability measures

- ECS tasks and RDS databases run in private subnets.
- Database credentials are generated and stored in Secrets Manager, not committed to Git.
- Security groups restrict ALB-to-task and task-to-database traffic.
- ECS execution and application task roles are separate.
- GitHub Actions uses OIDC rather than long-lived AWS access keys.
- Images use immutable commit-SHA tags.
- Trivy scans images before deployment metadata is published.
- ALB and ECS health checks run during deployments.
- CloudWatch logs are created per environment and service.
- ECS deployment circuit breakers provide automatic failure rollback.
- RDS storage is encrypted through the infrastructure templates.

Application security can be strengthened further by adding SAST to pull-request pipelines and DAST after deployment smoke tests. Those are intentionally identified as next improvements rather than claimed as already implemented.

## Hosted demonstration

The platform has been hosted through separate environment URLs:

- Development: [https://sparrowx-dev.mo2cloud.com](https://sparrowx-dev.mo2cloud.com)
- Production: [https://sparrowx-prod.mo2cloud.com](https://sparrowx-prod.mo2cloud.com)

The AWS environment may be taken down after a demonstration to avoid ongoing resource costs, so the URLs may not be live when you visit them. Because the environment is defined as code, it can be recreated when needed. If you would like to see the platform live, discuss the architecture, or work with me on a future project, feel free to reach out, I would be glad to bring it online for a demo.

## Documentation map

- [Architectural Decisions](architectural-decisions.md)
- [Onboarding a New Service](onboarding-new-service-guide.md)
- [Scope and goals](scope-and-goals.md)
- [Architecture and onboarding model](architecture-and-onboarding.md)
- [Delivery and rollback model](delivery-and-rollback.md)

The original [`the-platform-challenge-ecs`](../the-platform-challenge-ecs) directory is retained as historical context. This directory is the current project overview.

## License

This is a proprietary portfolio project. It is publicly viewable but not open source. All rights are reserved. See [LICENSE.md](LICENSE.md).
