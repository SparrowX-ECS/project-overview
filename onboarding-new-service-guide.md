# Onboarding a new service

This guide describes how to add a seventh service to the SparrowX ECS platform. It follows the current implementation and calls out the one remaining manual lifecycle bottleneck.

## Onboarding contract

A service repository should provide:

- application source code and tests;
- a Dockerfile;
- a process listening on the declared container port;
- `GET /health` for the ALB target group;
- a service-specific API smoke-test path;
- `ecs-parameters-dev.yaml` and `ecs-parameters-prod.yaml`;
- an `appStack.state` deployment switch in both environment parameter files;
- thin GitHub Actions workflows calling the reusable templates;
- repository/environment variables for AWS access, URLs, and release metadata.

For Python services, expose a Prometheus-compatible `/metrics` endpoint so the shared observability stack can discover and scrape the service.

## Step 1: Create the service repository

Create the repository in the SparrowX GitHub organization. Add the application, tests, Dockerfile, development dependencies, and a health endpoint.

For a Python API, confirm locally that the process starts on port `8000`, `/health` returns a successful response, and FastAPI documentation is available at `/docs`. For a frontend, use the appropriate Node build and container port.

## Step 2: Add environment parameter files

Create both files at the repository root:

```text
ecs-parameters-dev.yaml
ecs-parameters-prod.yaml
```

Start from the same schema as an existing service and change the service-specific values:

```yaml
project:
  name: sparrowx
  environment: dev

appStack:
  state: enabled

service:
  name: new-service

image:
  nameSpace: sparrowx/dev
  repository: newservice

container:
  port: 8000
  size: micro

deployment:
  desiredCount: 1

loadBalancer:
  pathPattern: /api/new-service/*
  listenerRulePriority: 160
  healthCheckPath: /health
  smokeTestPath: /api/new-service/health

database:
  enabled: false
```

For `prod`, change `project.environment` and `image.nameSpace` to `prod`. Choose a unique ALB listener priority. If the service has upstream dependencies, add the corresponding `upstream` values. If it needs persistence, enable `database` and declare the database name.

Set `appStack.state: disabled` when the application should not participate in deployments for that environment. The application switch works together with the infrastructure repository’s `RootStack.State`; the deployment guard allows downstream jobs to run only when both switches are `enabled`.

## Step 3: Add repository workflows

Copy the workflow shape from an existing service and update:

- reusable workflow release, such as `@v4.4.x`;
- `runtime` (`python` or `node`);
- whether tests require PostgreSQL;
- whether the build is a frontend build;
- the environment parameter file passed to each reusable workflow;
- the service’s paths in the `paths` filters.

Include the reusable `deployment-guard` job before deployment-dependent jobs. It reads the application parameter file and the selected infrastructure environment, then exposes the deployment decision to downstream jobs.

The service should have workflows for:

- pull-request CI;
- automatic deployment to `dev` after `main` changes;
- manual production promotion;
- manual production rollback.

## Step 4: Configure GitHub variables

Configure these repository or environment variables:

| Variable | Value |
| --- | --- |
| `AWS_ACCOUNT_ID` | AWS account containing the platform |
| `AWS_REGION` | Deployment region |
| `AWS_ROLE_NAME` | GitHub OIDC IAM role |
| `DEV_BASE_URL` | Protocol plus development domain, with no path |
| `DEV_DEPLOYED_PARAM_STORE_PATH` | SSM path for the last successful development image |
| `PROD_BASE_URL` | Protocol plus production domain, with no path |
| `PROD_CANDIDATE_PARAM_STORE_PATH` | SSM path for the candidate promoted from development |
| `PROD_DEPLOYED_PARAM_STORE_PATH` | SSM path for the last successful production image |

The smoke-test workflow appends the path from the selected ECS parameters file to the base URL.

## Step 5: Verify locally

Run the service tests and build the container locally:

```bash
pytest                       # or npm test
docker build -t new-service .
```

Verify the container port, `/health`, API documentation if applicable, database connectivity if enabled, and any upstream service configuration.

## Step 6: Open a pull request

Open a pull request containing the service code, tests, Dockerfile, environment parameter files, and workflows. Application CI will run tests, build the image, scan it with Trivy, and publish metadata if successful.

Review:

- listener priority uniqueness;
- database enablement and secret wiring, if applicable;
- image repository and environment namespace;
- smoke-test path and base URL;
- task CPU/memory and desired count.

## Step 7: Deploy and verify development

After the application change reaches `main`:

1. The deployment guard confirms that both application and infrastructure switches are enabled.
2. CI resolves the immutable image metadata.
3. The service deploys to the `dev` ECS stack.
4. CloudFormation and ECS wait for stability.
5. The smoke-test workflow calls the configured development URL and path.
6. Successful metadata is published as the production candidate.

Verify the ALB target is healthy, inspect CloudWatch logs, open the API documentation, exercise the service, and confirm upstream/database traffic.

## Step 8: Promote to production

Run the service’s production promotion workflow:

1. Type `PROMOTE` to confirm.
2. Resolve the candidate tag and digest from SSM.
3. Verify the image in the development ECR repository.
4. Copy the exact image by digest to the production ECR repository.
5. Deploy the same tag to the production ECS stack.
6. Run the production smoke test.
7. Publish production deployment metadata.

## Step 9: Roll back when necessary

There are four rollback methods:

- **ECS automatic rollback:** an unhealthy rolling deployment can be returned to the previous task definition by the ECS deployment circuit breaker.
- **Production-pipeline rollback:** if production smoke or API/frontend tests fail, the production workflow resolves and redeploys the previous deployed image.
- **Git revert:** revert the bad application or configuration commit, merge the revert, and let the normal pipeline deploy the correction.
- **Quick image rollback:** open **Deployments → prod**, select the desired previous deployment, copy its image tag, then run **Actions → Manual Rollback Production To Selected Image Tag** with `ROLLBACK` and that tag.

The manual rollback workflow redeploys the selected image, runs the production smoke test, and publishes deployment metadata.

If the service exposes Prometheus metrics, confirm that its `/metrics` endpoint is reachable from the private observability service and that the service appears in Prometheus service discovery.

## Current manual bottleneck

The current `ecs-infrastructure/scripts/cfn-destroy.sh` contains an explicit list of service stack names. This does not block onboarding or normal deployment, but the service must be added to that list when it becomes part of the environment cleanup set.

This is a known temporary limitation. The intended improvement is a validated service registry or safe CloudFormation stack discovery workflow that can calculate service stacks without relying on a hand-maintained destroy list.

## Onboarding completion checklist

- [ ] Repository created with service code, tests, and Dockerfile.
- [ ] Health endpoint and container port verified.
- [ ] API documentation available at `/docs` for backend services.
- [ ] `ecs-parameters-dev.yaml` added.
- [ ] `ecs-parameters-prod.yaml` added.
- [ ] Unique ECR repository and ALB listener priority selected.
- [ ] CI, dev deploy, prod promotion, and rollback workflows added.
- [ ] GitHub variables configured.
- [ ] Development deployment and smoke test verified.
- [ ] Production promotion and smoke test verified.
- [ ] Rollback procedure tested or documented.
- [ ] `appStack.state` verified for both `dev` and `prod`.
- [ ] `/metrics` exposed and verified for Prometheus scraping, where applicable.
