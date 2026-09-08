# Junior DevOps Engineer – Practical Assessment

## Task

Select a simple existing open-source web application and prepare it for deployment to a Kubernetes environment.

The application itself is not part of the assessment. Choose a lightweight application that allows you to focus on the DevOps work. Include the source repository and briefly explain why you selected it.

Your solution should include:

1. **Docker**
   - Containerize the application.
   - Build a clean, deployable image.
   - Tag the image so it can be traced back to the source commit.

2. **CI/CD**
   - Create a CI pipeline that builds the Docker image automatically.
   - Push the image to a private container registry.
   - Handle registry credentials securely.

3. **Kubernetes and Helm**
   - Deploy the application to Kubernetes using a Helm chart.
   - Configuration that is expected to vary between environments should be exposed through Helm values.
   - Expose the application using an Ingress or LoadBalancer suitable for your environment.

4. **Terraform / Infrastructure as Code**
   - Create one small Terraform configuration for a simple infrastructure component related to the deployment, such as object storage, DNS/networking, or access-control infrastructure.
   - `terraform fmt`, `terraform validate`, and a clean `terraform plan` are sufficient.
   - Running `terraform apply` against paid cloud infrastructure is not required.

5. **Monitoring and Logging**
   - Configure Prometheus, Loki, and Grafana for the Kubernetes environment.
   - Show at least one useful metric in Grafana using Prometheus.
   - Show application or container logs in Grafana using Loki.

## Things to Consider

1. A simple, working solution is preferred over unnecessary complexity.
2. The structure and maintainability of your Docker, CI/CD, Helm, Terraform, and monitoring configuration will be reviewed.
3. Configuration and credentials should be handled appropriately and must not be committed to the repository.
4. Your Git commit history should show meaningful, incremental progress.
5. Your README should allow another engineer to understand and reproduce your solution.
6. If something is incomplete, document what you attempted, what blocked you, and what you would do next.

## Rules / Requirements

1. Local Kubernetes environments such as kind, minikube, or k3d are acceptable.
2. Use Git for version control and submit the assessment through a Git repository.
3. Do not commit passwords, tokens, API keys, private keys, or other sensitive information.
4. Include a `README.md` in the repository root covering:
   - the source of the selected application and why you chose it,
   - the architecture and deployment flow,
   - setup and deployment steps,
   - important assumptions or trade-offs,
   - any known limitations or incomplete work.
5. Include enough screenshots or command output to demonstrate that the main parts of the solution work.

## Bonus

The following are optional and are not required to complete the assessment:

- Argo CD or another GitOps approach
- Horizontal Pod Autoscaler
- Improved Kubernetes secret management
- CI validation such as `helm lint` or `terraform validate`
- Terraform module structure
- Additional monitoring dashboards or alerting
- Additional container or Kubernetes security improvements

## Submission

Submit the Git repository link by email.
