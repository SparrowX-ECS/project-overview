# Delivery and rollback model

## Build Once, Promote Many

1. A pull request runs change detection, the appropriate Python or Node tests, and a Docker build path when relevant.
2. A successful build uses the commit SHA as the immutable image tag and skips rebuilding if that tag already exists.
3. Trivy scans the image before image metadata is published.
4. A merge to `main` resolves the built artifact, deploys that exact tag to `dev`, and runs an environment-specific smoke test.
5. Only after the smoke test succeeds is the tag and digest written as the production candidate in SSM Parameter Store.
6. A manually confirmed production workflow resolves the candidate, verifies it in the development ECR repository, copies it by digest into the production ECR repository, deploys the same tag, and smoke-tests production.

This separates build identity from environment deployment and gives production a verifiable artifact lineage.

## Three rollback levels

### 1. ECS deployment rollback

The reusable CloudFormation service template enables the ECS deployment circuit breaker with rollback. If a rolling deployment fails its health checks or cannot stabilize, ECS can return the service to the previous task definition automatically.

### 2. Git revert rollback

When the desired state or application configuration is wrong, reverting the offending commit causes the normal CI/CD path to produce and deploy a corrective commit. This preserves an auditable source-history explanation for the change.

### 3. Manual quicker image rollback

Each service has a manually triggered production rollback workflow. An operator types `ROLLBACK`, selects a previously known-good image tag, redeploys it through the shared deployment workflow, runs the production smoke test, and publishes the resulting deployed metadata. No rebuild is required.

These paths cover automatic failure recovery, source-controlled correction, and urgent operator-led recovery.
