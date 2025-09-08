# Micro Pipeline Starter (Jenkins + DockerHub + Terraform EKS + Ansible + K8s)

This repo is a ready-to-run starter that builds and pushes Docker images, creates an EKS cluster with Terraform, and deploys three services (`api`, `web`, `worker`) to Kubernetes using Ansible.

## Prereqs
- Docker running on the Jenkins node
- Jenkins plugins: Credentials Binding, AWS Credentials, Docker, Docker Pipeline, AnsiColor
- Jenkins credentials:
  - `dockerhub-creds` (Username/Password)
  - `aws-creds` (Kind: AWS Credentials)
- CLIs available on the Jenkins node: `docker`, `terraform`, `ansible`, `kubectl`, `aws`

## How it works
1. Builds `api`, `web`, `worker` images via `docker-compose.ci.yml`.
2. Pushes images to DockerHub: `${DOCKERHUB_USER}/{api,web,worker}:${IMAGE_TAG}`.
3. Provisions VPC + EKS (2 t3.small nodes) with Terraform.
4. Fetches kubeconfig.
5. Applies K8s manifests with Ansible (namespace + 3 deployments; `web` is `LoadBalancer`).

## First Run (from Jenkins)
- Create a Pipeline job pointing to this repo.
- Parameters:
  - `IMAGE_TAG` (e.g., `v1`)
  - `AWS_REGION` (e.g., `us-east-1`)
  - `CLUSTER_NAME` (e.g., `micro-eks`)
- Run the build. After deploy:
  ```bash
  kubectl -n micro get svc web
  # open the EXTERNAL-IP in a browser
  ```

## Cleanup
```bash
cd infra/terraform
terraform destroy -auto-approve -var="region=us-east-1" -var="cluster_name=micro-eks"
```

## Notes
- The Dockerfiles are placeholders; replace with your real app.
- If DockerHub is private, add an imagePullSecret and reference it in the Deployments.
- If node group creation fails due to capacity, try a different instance type or region.

