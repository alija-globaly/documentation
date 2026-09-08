# Junior DevOps

Globaly · Engineering · Exercise

**Level:** Junior DevOps Engineer  
**Format:** Independent, your own environment  
**Submission window:** 5 business days from receipt

---

## 1. Context

Our applications run as containerized services on Kubernetes, deployed through Helm charts, with visibility from Prometheus, Loki, and Grafana. Staging and production use the same charts with different values such as image tags, resource sizing, and routing.

This exercise is a small practical task based on that type of work, using an application you choose.

## 2. Pick the application

Choose a small, existing open-source web application. Do not build one from scratch.

Pick something simple enough to containerize and expose over HTTP. Avoid applications that require unnecessary external dependencies such as message queues, GPUs, or paid APIs.

In your README, include the source repository and briefly explain why you selected it. The application choice is part of the assessment, but application development is not.

## 3. What to deliver

How you implement the solution is your call — we're deliberately not prescribing the exact steps.

1. **A clean, deployable container image** — build the application image and tag it so it can be traced back to the source commit.

2. **A CI pipeline that builds and pushes the image** — trigger it on push to a branch you define and push the resulting image to a private container registry.

3. **A Helm chart that deploys the application to Kubernetes** — configuration that is expected to vary between environments should be exposed through Helm values.

4. **A reachable service** — expose the application through an Ingress or LoadBalancer appropriate for your Kubernetes environment.

5. **Basic observability** — configure:
   - at least one useful metric available through Prometheus and visible in Grafana
   - application or container logs available through Loki and queryable in Grafana

   Include at least one Grafana panel that shows real information about the running application or workload.

6. **Basic supporting infrastructure in Terraform** — create one small Terraform configuration representing infrastructure that could support the deployment, such as object storage, DNS/networking, or access-control infrastructure.

   `terraform fmt`, `terraform validate`, and a clean `terraform plan` are sufficient. You do not need to create paid cloud resources or run `terraform apply` against a real cloud environment.

## 4. Ground rules

- Local Kubernetes environments such as kind, minikube, or k3d are acceptable.
- If any part is incomplete, document what you attempted, what blocked you, and what you would do next.
- Do not commit credentials, tokens, API keys, private keys, or other sensitive information.
- Commit your work incrementally. Git history will be reviewed as part of the assessment.

## 5. Submission

Submit a Git repository containing the code, Docker configuration, CI pipeline, Helm chart, Terraform configuration, and monitoring/logging configuration used for the assessment.

Include a `README.md` at the repository root covering:

- the open-source application you selected and why,
- the overall architecture and deployment flow,
- how to run or reproduce the solution,
- important decisions or trade-offs,
- any incomplete items or known limitations.

Include enough screenshots or command output to demonstrate that the major parts of the solution work. You do not need to document every minor command.

## 6. What we're looking for

We are looking at how you approach the task, how you keep the solution appropriately scoped, how you use the required DevOps tools, and whether another engineer could understand and reproduce your work from the repository and documentation.
