# AWS Terraform Plan

## Goal
Use Terraform to provision the AWS infrastructure required for the production-grade version of this stack, while keeping Helm responsible for Kubernetes application deployment.

## What Terraform Should Own
- VPC and subnets
- Security groups
- EKS cluster and node groups
- S3 or EBS-backed storage choices
- IAM roles and policies
- Secrets integration
- Helm releases if you want Terraform to orchestrate the full cluster install

## What Helm Should Own
- Kafka
- Kafka UI
- Producer and consumer services
- Config maps and chart values
- Environment-specific deployment settings

## Step-by-Step Process
1. Define the AWS network and EKS baseline in Terraform.
2. Create a Terraform module for cluster bootstrap.
3. Add Terraform-managed Kubernetes namespaces and secrets.
4. Install core platform charts with Helm or Terraform `helm_release`.
5. Deploy Kafka and Kafka UI into EKS.
6. Deploy the Spring Boot services into the cluster.
7. Add SSL termination and internal TLS paths.
8. Add Schema Registry and schema validation.
9. Add monitoring, alerts, and failure drills.
10. Promote from local values to staging and production values.

## Terraform Module Layout
- `infra/terraform/modules/network/`
- `infra/terraform/modules/eks/`
- `infra/terraform/modules/k8s-base/`
- `infra/terraform/modules/helm-releases/`
- `infra/terraform/envs/dev/`
- `infra/terraform/envs/stage/`
- `infra/terraform/envs/prod/`

## Guardrails
- Use remote state.
- Separate state by environment.
- Do not hardcode secrets in Terraform variables.
- Use IAM least privilege.
- Keep Helm chart values versioned alongside Terraform.

## Exit Criteria
- EKS is reproducible from Terraform.
- Helm charts deploy cleanly into AWS.
- TLS and schema validation work in-cluster.
- The production topology matches the local and Minikube plan closely enough to reduce drift.
