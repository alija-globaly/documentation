# Junior DevOps

Globaly · Engineering · Task

**Level:** Junior DevOps Engineer  
**Format:** Independent, your own environment  

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

2. **A CI pipeline that builds and pushes the image** — trigger it automatically on push to an appropriate branch and push the resulting image to a private container registry.

3. **A Helm chart that deploys the application to Kubernetes** — configuration that is expected to vary between environments (image tag, resource sizing, routing, etc.) should be exposed through Helm values.

4. **A reachable service** — expose the application through an Ingress or LoadBalancer appropriate for your Kubernetes environment.

5. **Basic observability** — configure:
   - at least one useful metric available through Prometheus and visible in Grafana
   - application or container logs available through Loki and queryable in Grafana

   Include at least one Grafana panel that shows real information about the running application or workload.

6. **Basic supporting infrastructure in Terraform** — create one small Terraform configuration for a simple infrastructure component related to the deployment, such as a storage bucket, DNS/networking resource, or security/access-control resource.

## 4. Bonus points

None of these are required. They are optional ways to demonstrate additional knowledge.

- A GitOps-based deployment flow (e.g., Argo CD) instead of a manual/scripted deploy.
- A meaningful alerting rule wired up in Prometheus.
- A NetworkPolicy or other basic hardening around the application.
- Secret management via something other than plain Kubernetes Secrets (Sealed Secrets, SOPS, a cloud secrets manager).
- A CI step that validates your Helm chart or Terraform (`helm lint`, `terraform validate` as part of the pipeline).
- Anything else that demonstrates good judgment about production-readiness, as long as it doesn't come at the expense of the core deliverables.


## 5. Ground rules

- Local Kubernetes environments such as kind, minikube, or k3d are acceptable.
- If any part is incomplete, document what you attempted, what blocked you, and what you would do next.
- Do not commit credentials or other sensitive information.
- Commit your work incrementally; Git history will be reviewed.
- External references are allowed, but you should be able to explain your submission.

## 6. Submission

Submit a Git repository containing the code and configuration used for the assessment.

Include a `README.md` at the repository root covering:

- the open-source application you selected and why,
- the overall architecture and deployment flow,
- how to run or reproduce the solution,
- important decisions or trade-offs,
- any incomplete items or known limitations.

Include enough screenshots or command output to demonstrate that the major parts of the solution work. You do not need to document every minor command.

## 7. What we're looking for

We are looking at how you approach the task, the quality of your technical decisions, and how clearly your solution can be understood and reproduced.

A simple, well-reasoned solution is preferred over an unnecessarily complex one.
