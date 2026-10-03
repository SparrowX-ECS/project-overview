# Scope and goals

## Scope

The project models the migration of a small SaaS company to AWS ECS/Fargate. It covers the platform boundary from source repositories to running services in isolated `dev` and `prod` environments.

### In scope

- VPC networking with public and private subnets and configurable NAT gateways.
- Environment-specific CloudFormation root stacks and nested modules.
- ECR repositories, ECS clusters, Service Connect, ALB routing, RDS PostgreSQL, Secrets Manager, and CloudWatch Logs.
- A reusable service CloudFormation template driven by each service’s `ecs-parameters-<environment>.yaml`.
- Shared GitHub Actions workflows for change detection, tests, Docker builds, Trivy image scanning, deployment, image promotion, metadata, and smoke testing.
- Independent delivery of four database-backed APIs, one stateless API, and one frontend.
- Artifact promotion and three rollback paths.

### Deliberately out of scope

- Kubernetes, Terraform, Helm, or a multi-account landing zone.
- Product-grade authentication, authorization, billing integrations, and asynchronous business workflows.
- Claiming high availability or production scale from a low-cost demonstration configuration.

## Goals

1. Make service onboarding configuration-driven and repeatable.
2. Keep application teams responsible for service code while centralizing AWS delivery conventions.
3. Isolate development and production resources without duplicating the platform templates.
4. Preserve artifact provenance from commit SHA through both environments.
5. Make deployment health visible and rollback practical.
6. Explain the trade-offs and remaining hardening work honestly.

## Success criteria

A service is considered onboarded when it can run its tests, build an immutable image, deploy through the shared workflow using its environment parameters, pass its smoke test, and be promoted or rolled back without rebuilding the artifact.
