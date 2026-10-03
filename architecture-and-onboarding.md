# Architecture and onboarding model

## Runtime architecture

```text
Internet
   │
   ▼
ALB per environment ── path rules ──► ECS/Fargate services in private subnets
                                           │                 │
                                           │ Service Connect │
                                           ▼                 ▼
                                  reporting-api       private RDS databases
                                                               │
                                                        Secrets Manager
```

The portal and public API paths enter through the environment’s ALB. ECS tasks do not receive public IP addresses. Stateful services receive database host, port, name, and secret references from the deployment workflow; the task execution role reads only the required secret. The reporting service uses Service Connect names such as `customer-api:8000` for internal calls.

## Infrastructure composition

Each environment is deployed from the same template shape with different parameters. The root stack composes network, ECR, ECS cluster, ALB, PostgreSQL, SSM, and optional CloudFront modules. Service stacks are deployed separately so an application change does not require replacing the shared foundation.

Resource names, exports, log groups, ECR namespaces, and service stacks include the environment name. This allows `dev` and `prod` to coexist while keeping the templates reusable.

## Service onboarding contract

A service repository supplies:

1. Application code and tests.
2. A Dockerfile exposing the configured container port.
3. A `/health` endpoint for the target group.
4. `ecs-parameters-dev.yaml` and `ecs-parameters-prod.yaml`.
5. Thin repository workflows that call the shared workflow version.

The parameters file describes the service name, environment namespace, ECR repository, container size, desired count, ALB path and priority, health and smoke-test paths, and optional database or upstream settings. This keeps service-specific intent visible without copying deployment logic into every repository.

## Onboarding a seventh service

The next service would add its own repository and environment parameter files, create the required ECR/database entries in the infrastructure parameters, and reference the shared workflow templates. The current platform proves the contract across six workloads; it does not claim that service registration is fully self-service yet.
