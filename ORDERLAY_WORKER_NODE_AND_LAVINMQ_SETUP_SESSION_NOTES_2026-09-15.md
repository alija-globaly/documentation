# orderlay-staging: dedicated worker node join + LavinMQ setup — session notes (2026-09-15)

Scope: this note covers two connected pieces of work from the same debugging
session — (1) discovering this cluster had **no real worker node** and
properly joining a dedicated one, and (2) standing up a self-hosted LavinMQ
instance for `backend_v2`, including the full user/permission debugging saga
that followed. Real command output from the session is included throughout,
not paraphrased summaries.

---

## Table of Contents

1. [Background: how the gap was discovered](#1-background-how-the-gap-was-discovered)
2. [Bug: ALB target group had zero registered targets](#2-bug-alb-target-group-had-zero-registered-targets)
3. [Decision: label the master, or join a real worker?](#3-decision-label-the-master-or-join-a-real-worker)
4. [Finding the actual join automation](#4-finding-the-actual-join-automation)
5. [Bug: the worker's first join attempt had already failed silently](#5-bug-the-workers-first-join-attempt-had-already-failed-silently)
6. [Root cause: the same DNS Firewall issue, blocking the join itself](#6-root-cause-the-same-dns-firewall-issue-blocking-the-join-itself)
7. [Fix: local /etc/hosts override, then re-run the join](#7-fix-local-etchosts-override-then-re-run-the-join)
8. [Confirming the join for real](#8-confirming-the-join-for-real)
9. [Building LavinMQ: operator vs plain Deployment](#9-building-lavinmq-operator-vs-plain-deployment)
10. [Mirroring production's real LavinMQ manifest](#10-mirroring-productions-real-lavinmq-manifest)
11. [Reusing the generic backend chart instead of writing a new one](#11-reusing-the-generic-backend-chart-instead-of-writing-a-new-one)
12. [Exposing the management UI: the primary/secondary port swap](#12-exposing-the-management-ui-the-primaryecondary-port-swap)
13. [Bug: backend_v2 — "User orderlay-admin not found"](#13-bug-backend_v2--user-orderlay-admin-not-found)
14. [False lead: vhost encoding (%2F)](#14-false-lead-vhost-encoding-2f)
15. [Real root cause: password mismatch, not vhost](#15-real-root-cause-password-mismatch-not-vhost)
16. [Fix confirmed — backend_v2 finally Running](#16-fix-confirmed--backend_v2-finally-running)
17. [Bug: dashboard login "Authentication failure" despite correct password](#17-bug-dashboard-login-authentication-failure-despite-correct-password)
18. [Summary table — every bug found this session](#18-summary-table--every-bug-found-this-session)

---

## 1. Background: how the gap was discovered

While testing `backend_v2`'s deployment on the new ArgoCD/Gateway-based
`orderlay-staging` cluster, the pod was found running on
`root-master-nginx-ingress-172-34-21-146` — the cluster's **only** node. A
`kubectl get nodes -L type` check confirmed there was no separate worker at
all:

```
ubuntu@root-master-nginx-ingress-172-34-21-146:~$ kubectl get nodes -L type
NAME                                      STATUS   ROLES                  AGE    VERSION   TYPE
root-master-nginx-ingress-172-34-21-146   Ready    control-plane,worker   5d2h   v1.35.7
```

The `TYPE` column is empty — this node carries no `type=` label at all. Every
`targetGroupConfiguration` block written for this project (backend_v2,
web-v2, LavinMQ, etc.) selects ALB targets by `type=default-server-worker`.
With no node carrying that label, every target group in the cluster was
silently getting zero targets.

## 2. Bug: ALB target group had zero registered targets

**Symptom:** `https://lavinmq.orderlay-test.agentcis.com` returned a plain
`503 Service Temporarily Unavailable`, even though the LavinMQ pod itself was
`1/1 Running` with clean startup logs.

**Diagnosis — went straight to the AWS API rather than guessing:**

```
$ aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:ap-south-1:381491939487:targetgroup/k8s-orderlay-lavinmqh-ec08e445be/7b2f14e7eed436a7 --region ap-south-1
{
    "TargetHealthDescriptions": []
}
```

Not "unhealthy" — a genuinely **empty array**. No node in the cluster matched
the target group's node selector, so there was nothing to register at all.

**Comparison that confirmed it wasn't LavinMQ-specific** — agentcis's own
staging cluster, for contrast:

```
ubuntu@root-master-nginx-ingress-172-34-6-25:~$ kubectl get nodes -L type
NAME                                    STATUS   ROLES                  AGE   VERSION   TYPE
default-server-worker-172-34-13-92      Ready    worker                 87d   v1.35.7   default-server-worker
default-server-worker-172-34-16-76      Ready    worker                 87d   v1.35.7   default-server-worker
root-master-nginx-ingress-172-34-6-25   Ready    control-plane,worker   88d   v1.35.7
```

agentcis has two dedicated, correctly-labeled workers. orderlay-staging had
never had one at all — this was a foundational gap affecting **every**
service's ALB routing on this cluster, not something introduced by the
LavinMQ work.

## 3. Decision: label the master, or join a real worker?

The fastest fix would have been `kubectl label node root-master-... type=default-server-worker`.
This was explicitly rejected: the goal was to keep application workloads off
the control-plane node entirely, matching agentcis's real topology, and to
use the already-provisioned (but never-joined) EC2 instance
`i-03591e1ad07746b49` (`[orderlay-staging]-default-worker-instance`) —
created earlier by Terraform's own auto-scaling group but sitting unused.

## 4. Finding the actual join automation

Rather than hand-writing `kubeadm join` commands, the existing infra repo was
searched first. Key findings:

- This cluster uses **Canonical Kubernetes (the `k8s` snap)**, not raw
  kubeadm.
- The join logic already exists as a shared, S3-hosted script:
  `k8s-cluster-join-worker.sh`, run identically by both `agentcis` and
  `orderlay` (same Terraform module, different parameters).
- The `type=default-server-worker` label convention comes directly from this
  script's own final step: `sudo k8s kubectl label node ${FINAL_HOSTNAME} type=${TAG}`.
- The EC2 instance was created by Terraform's `default-worker` ASG
  (`default_auto_scaling = true`, desired capacity 1) — this explains why
  `i-03591e1ad07746b49` already existed before any manual join step.

## 5. Bug: the worker's first join attempt had already failed silently

Checking the instance directly revealed the automatic user-data join (which
runs once at first boot) had already failed, five days earlier, and never
retried:

```
ubuntu@ip-172-34-30-247:/var/log$ cat kubernetes.log
[2026-09-10 10:10:34] | function 1 : Master health check Start
Master node unreachable (Status: 000)
[2026-09-10 10:10:34] | attempt 1  ->  Master node health check timeout
```

```
ubuntu@ip-172-34-30-247:/var/log$ sudo k8s status
Error: The node is not part of a Kubernetes cluster. You can bootstrap a new cluster with:

  sudo k8s bootstrap
```

Two things worth calling out:
- The instance's hostname was still the AWS default (`ip-172-34-30-247`),
  never renamed — confirming the script bailed out at the very first health
  check step, before ever reaching the actual join or hostname-setting logic.
- `sudo k8s bootstrap` is **not** the fix for this — it creates a brand-new,
  standalone cluster. Running it would not join the existing
  `orderlay-staging` cluster; it was explicitly flagged and avoided.

## 6. Root cause: the same DNS Firewall issue, blocking the join itself

`Status: 000` means the health check couldn't connect at all — not a
reachable-but-erroring server, a total connection failure. This matched the
still-unresolved DNS Firewall problem investigated earlier the same day for
`nfs.orderlay-staging.internal` (see the NFS/DNS investigation notes). Tested
directly from the worker instance:

```
ubuntu@ip-172-34-30-247:/var/log$ dig @172.34.0.2 k8s-root-master.orderlay-staging.internal +short
ubuntu@ip-172-34-30-247:/var/log$
```

Empty result — confirmed: the same `orderlay-staging.internal` zone is
unreachable VPC-wide (not just for the NFS record), and it was blocking the
master's own hostname resolution too, which is exactly what the worker join
script depends on.

## 7. Fix: local /etc/hosts override, then re-run the join

Same workaround pattern already used for the NFS server — bypass DNS
entirely with a local hosts-file entry, since a durable fix (the DNS Firewall
rule) was still pending from the senior:

```bash
echo "172.34.21.146 k8s-root-master.orderlay-staging.internal" | sudo tee -a /etc/hosts
```

Then re-ran the exact same join script by hand (safe/idempotent — identical
to what the failed user-data run had already attempted):

```bash
curl -fsSL https://globalyhub-kubernetes-aws-provider.s3.ap-southeast-2.amazonaws.com/scripts/k8s-cluster-join-worker.sh | tr -d '\r' | \
  bash -s -- --worker default-server-worker --master k8s-root-master.orderlay-staging.internal
```

This time it ran to completion:

```
User input recieved
Configuring hostname
Hostname is default-server-worker-172-34-30-247
Generating token and joining cluster...
Joining the cluster. This may take a few seconds, please wait.
Cluster services have started on "default-server-worker-172-34-30-247".
Node successfully joined the cluster!
==========================================================================
[2026-09-15 13:26:28] | Master cluster join Finish
...
[2026-09-15 13:26:33] | Kubelet ECR config generation Finish
...
🔄 Reloading K8s node: default-server-worker-172-34-30-247
🔒 Cordon node (prevent new pods)
🗑️ Force deleting node object
node "default-server-worker-172-34-30-247" force deleted
Stopped.
Started.
Checking K8s status
Error: Failed to retrieve the cluster status.
The error was: wait check failed: failed after potential retry: wait check failed: failed to GET /k8sd/cluster: this action is restricted on workers
K8s failed to reach ready state
[2026-09-15 13:26:50] | function 10 : Node labelling Start
node/default-server-worker-172-34-30-247 labeled
[2026-09-15 13:26:51] | default-server-worker-172-34-30-247 Node label Finish to type => default-server-worker
==========================================================================
[2026-09-15 13:26:51] | All tasks completed successfully!
```

Two intermediate errors turned out to be non-fatal, worth noting so they
aren't mistaken for real failures next time:
- **AWS CLI install failed** (`Permission denied` writing `awscliv2.zip`) —
  caused by running the script from `/var/log`, a non-writable directory for
  `ubuntu`. Harmless: the separate **ECR credential provider** binary (what
  kubelet actually needs to pull images) installed successfully in an earlier
  step regardless.
- **`K8s failed to reach ready state` / "this action is restricted on
  workers"** — Canonical Kubernetes deliberately refuses a cluster-wide
  status query when run from a plain worker node (by design, not a bug). The
  script's own health-check step doesn't distinguish node roles for this
  particular check, producing a scary-looking but expected error. The script
  still proceeded and finished successfully afterward.

## 8. Confirming the join for real

Trusted only the master's own view, not the worker's local output:

```
ubuntu@root-master-nginx-ingress-172-34-21-146:~$ kubectl get nodes -L type
NAME                                      STATUS   ROLES                  AGE    VERSION   TYPE
default-server-worker-172-34-30-247       Ready    worker                 100s   v1.35.7   default-server-worker
root-master-nginx-ingress-172-34-21-146   Ready    control-plane,worker   5d3h   v1.35.7
```

`STATUS: Ready`, `ROLES: worker`, `TYPE: default-server-worker` — genuinely
joined and correctly labeled. No values-file changes were needed afterward;
the AWS Load Balancer Controller picked the new node up automatically on its
next reconcile.

---

## 9. Building LavinMQ: operator vs plain Deployment

Before building anything, the shared infra charts repo was checked for an
existing LavinMQ chart. Two were found:
`helm-charts/public/lavinmq-operator` + `lavinmq-instance` (a full
operator/CRD pattern requiring **etcd** and a dedicated NFS `StorageClass`,
neither of which exist in this project), and nothing simpler.

Checking the actual **running production** LavinMQ pod first
(`kubectl get pod -A` on the old cluster) showed it's just a single plain
pod — `lavinmq-orderlay-6df4f8cd55-6l57r` — with no operator, no etcd, no
CRDs anywhere in that cluster. Decision made explicitly: mirror the simple,
proven production setup rather than adopt the heavier, unproven operator path
(agentcis's own copy of that chart sits in a folder literally named
`disabled`) — staging should match what's actually running in prod.

## 10. Mirroring production's real LavinMQ manifest

Pulled directly from the old cluster rather than guessed:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lavinmq-orderlay
  namespace: orderlay-self-hosted
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: lavinmq
        image: cloudamqp/lavinmq:2.3.0
        ports:
        - containerPort: 5672
          name: amqp
        - containerPort: 15672
          name: http
        volumeMounts:
        - name: lavinmq-data
          mountPath: /var/lib/lavinmq
        livenessProbe:
          tcpSocket:
            port: 5672
        readinessProbe:
          httpGet:
            path: /
            port: 15672
      volumes:
      - name: lavinmq-data
        persistentVolumeClaim:
          claimName: nfs-lavinmq-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: lavinmq-svc
  namespace: orderlay-self-hosted
spec:
  selector:
    app: lavinmq
  ports:
  - name: amqp
    port: 5672
    targetPort: 5672
  - name: http
    port: 15672
    targetPort: 15672
  type: ClusterIP
```

## 11. Reusing the generic backend chart instead of writing a new one

The existing `helm-charts/self-managed/backend` chart (already used for
`backend_v2`) turned out to be fully generic — nothing in its templates is
backend_v2-specific. Reused it for LavinMQ instead of writing a new chart:

- `backend.name: "lavinmq"` — the chart derives the Service name as
  `{name}-svc`, so this had to be exactly `lavinmq` to produce `lavinmq-svc`,
  matching `backend_v2`'s hardcoded `RABBITMQ_URL` host.
- `service.type: ClusterIP` initially (not `NodePort`) — LavinMQ's AMQP port
  is only ever reached pod-to-pod by `backend_v2`, never through the ALB.
- `startupCommand.enabled: false` — production's manifest has no
  command/args override either; the image's own entrypoint handles startup.
- `livenessProbe`/`readinessProbe` via `httpGet` on `15672` for both — the
  chart only supports `httpGet`, not production's `tcpSocket` check on
  `5672`; reusing the readiness endpoint for both was judged close enough.
- New PV (`persistent-volume/lavinmq.yml`) reusing the already-exported
  `pods-storage` NFS path (`/nfs-staging/pods-storage/lavinmq`) rather than
  creating a new NFS export.
- The `1-nfs-directory-create.yml` CronJob was updated to also create
  `lavinmq/` under `pods-storage` — otherwise the PV mount would have hit the
  exact same "No such file or directory" error already solved for
  `backend_v2`'s own volumes.

## 12. Exposing the management UI: the primary/secondary port swap

To reach the LavinMQ dashboard through the same Gateway/ALB used everywhere
else, a real chart constraint had to be worked around: `aws-http-route.yml`'s
`backendRefs[].port` is hardcoded to `service.svcport` (the "primary" port) —
it can never target a `multiplePorts` entry. Since `5672` was primary, `15672`
(the UI) could never be routed to externally as configured.

**Fix:** swapped which port is primary — `15672` became `containerport`/
`svcport`, `5672` moved into the secondary `multiplePorts` list. `backend_v2`'s
AMQP connection was unaffected either way, since Kubernetes Services expose
every declared port regardless of which one is "primary" in this chart's
terms. `service.type` was also switched from `ClusterIP` to `NodePort` at
this point (same instance-mode target-group requirement as every other
externally-reachable service), and a new `httproute` was added using the
already-issued wildcard cert/DNS (`lavinmq.orderlay-test.agentcis.com`) — no
new Cloudflare/ACM step needed.

---

## 13. Bug: backend_v2 — "User orderlay-admin not found"

Once LavinMQ was reachable, `backend_v2` still failed to boot:

```json
{"level":"error","message":"Failed to connect to RabbitMQ","error":"getaddrinfo ENOTFOUND lavinmq-svc.orderlay-self-hosted.svc.cluster.local"}
```

This first error was simply "LavinMQ doesn't exist in this cluster yet" —
resolved once the Deployment above actually landed. The very next failure,
after LavinMQ was up, was a real auth problem:

```
ERROR lmq.auth_handler Basic authentication failed: Missing hash key: "orderlay-admin"
WARN  lmq.amqp.connection_factory[address: "10.1.0.58:36028"] User "orderlay-admin" not found
```

This was actually good news at the time — it proved network connectivity
(DNS, `ClusterIP` routing) fully worked; the only remaining gap was that this
fresh LavinMQ instance had no such user yet. Created it:

```
root@lavinmq-deployment-59db7bd74c-w2bxr:/var/lib/lavinmq# lavinmqctl add_user orderlay-admin xxxx
root@lavinmq-deployment-59db7bd74c-w2bxr:/var/lib/lavinmq# lavinmqctl set_permissions orderlay-admin ".*" ".*" ".*"
```

`xxxx` here was a **literal placeholder typed into the command**, not a
redacted real password — this turned out to matter a lot (see §15).

## 14. False lead: vhost encoding (%2F)

After the user was created, the error changed to a generic protocol-level
refusal:

```json
{"level":"error","message":"Failed to connect to RabbitMQ","error":"Handshake terminated by server: 403 (ACCESS_REFUSED) with message \"ACCESS_REFUSED - \"\""}
```

`ACCESS_REFUSED` at this stage commonly means a vhost permission problem.
The `.env`'s `RABBITMQ_URL` ended in a bare trailing slash:

```
RABBITMQ_URL=amqp://orderlay-admin:xx@lavinmq-svc.orderlay-self-hosted.svc.cluster.local:5672/
```

Per the AMQP URI spec, a bare trailing slash means vhost = **empty string**,
not the default vhost `/` — the correct encoding for the actual default vhost
is `%2F`. The URL was corrected to
`.../5672/%2F`, and `set_permissions` was re-run explicitly scoped to `/`:

```
kubectl exec -it lavinmq-deployment-59db7bd74c-w2bxr -n orderlay-self-hosted -- lavinmqctl set_permissions -p / orderlay-admin ".*" ".*" ".*"
```

**This did not fix it.** Same `ACCESS_REFUSED` persisted after confirming
(via a masked `grep`) that the `.env` file genuinely had the corrected
`%2F` value. This was a real, plausible, well-reasoned theory that turned out
to be addressing a symptom, not the cause — worth keeping in the note as a
reminder that a technically-correct fix can still be the wrong fix.

## 15. Real root cause: password mismatch, not vhost

The breakthrough came from reconciling two log lines that looked
contradictory: `backend_v2`'s own error said `ACCESS_REFUSED`, while
LavinMQ's own server-side log, at the exact same timestamp, said:

```
kubectl logs --tail=30 -n orderlay-self-hosted lavinmq-deployment-59db7bd74c-w2bxr
2026-09-15T13:58:38... WARN lmq.amqp.connection_factory[address: "10.1.1.162:...] User "orderlay-admin" not found
```

```
kubectl logs -n orderlay-backend orderlay-backend-v2-deployment-f8c5695c5-cssnn --previous
{"level":"error","message":"Failed to connect to RabbitMQ","timestamp":"2026-09-15T13:58:38.337Z",...
"error":"Handshake terminated by server: 403 (ACCESS_REFUSED) with message \"ACCESS_REFUSED - \"\""...}
```

Same moment, two different-sounding messages — reconciled as: AMQP brokers
deliberately return a **generic** `ACCESS_REFUSED` to the client for *any*
auth failure (wrong user, wrong password, wrong vhost alike), specifically so
a client can't tell which part was wrong. The broker's own internal log is
the authoritative one, and it said "not found" — which, for an
already-confirmed-existing user, points to a **password mismatch** being
reported the same generic way.

This traced straight back to §13: `xxxx` had been typed as the literal
password when the user was created, while the `.env` held the real password
the whole time.

## 16. Fix confirmed — backend_v2 finally Running

```
root@lavinmq-deployment-59db7bd74c-w2bxr:/var/lib/lavinmq# lavinmqctl add_user orderlay-admin orderlay-admin
```

(matching the real `.env` value:
`RABBITMQ_URL=amqp://orderlay-admin:orderlay-admin@lavinmq-svc.orderlay-self-hosted.svc.cluster.local:5672/%2F`)

Pod deleted to force a fresh connection attempt:

```
NAME                                              STATUS    RESTARTS   AGE
orderlay-backend-v2-deployment-f8c5695c5-k4v5n    1/1 Running   0        27s
```

The stale old ReplicaSet (`68997bc778`), which had been stuck alongside the
current one for hours because neither had ever passed health checks, was
automatically scaled down and removed by Kubernetes the moment the current
one finally became healthy — no manual cleanup needed.

An end-to-end HTTP test through the real Gateway/ALB confirmed the whole
pipeline, not just the pod:

```
$ curl https://backendv2.orderlay-test.agentcis.com/
Cannot GET /
```

`Cannot GET /` is Express's normal response for a REST API with no root
route — proof that DNS, TLS (the wildcard cert), the ALB, the `HTTPRoute`,
the `Service`, and the pod itself all worked correctly together for the
first time on this cluster.

## 17. Bug: dashboard login "Authentication failure" despite correct password

With AMQP now working, the LavinMQ web dashboard
(`lavinmq.orderlay-test.agentcis.com`) still refused browser login for
`orderlay-admin` with the exact same (now-correct) password. Cause: the
management UI requires a user to carry the `administrator` tag to be allowed
to log in at all — a separate authorization check from AMQP protocol auth.
`lavinmqctl list_users` had shown this plainly all along:

```
name            tags
guest           administrator
orderlay-admin
```

`orderlay-admin` had no tags. Fixed with:

```
kubectl exec -it lavinmq-deployment-59db7bd74c-w2bxr -n orderlay-self-hosted -- lavinmqctl set_user_tags orderlay-admin administrator
```

Confirmed immediately afterward — dashboard loaded fully, logged in as
`orderlay-admin`, `Queues` page rendering (empty, as expected — nothing
published yet).

---

## 18. Summary table — every bug found this session

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | ALB target group `[]` empty, 503 on every externally-routed service | No node in the cluster carried the `type=default-server-worker` label — the master was the only node, unlabeled | Joined a real, dedicated worker node instead of labeling the master |
| 2 | Worker's automatic join never completed; hostname stuck at AWS default | First join attempt (5 days earlier) failed at the master health check with `Status: 000` and was never retried | Diagnosed via the worker's own `/var/log/kubernetes.log` |
| 3 | Health check `Status: 000` | Same unresolved DNS Firewall issue blocking `*.orderlay-staging.internal` VPC-wide, now also blocking the join | Confirmed via `dig` from the worker itself returning empty |
| 4 | — | — | `/etc/hosts` override for the master's hostname, then re-ran the join script; succeeded end-to-end |
| 5 | `backend_v2`: `ENOTFOUND lavinmq-svc...` | LavinMQ didn't exist in this cluster yet | Deployed a plain Deployment/Service/PVC mirroring production exactly, via the reused generic `backend` chart |
| 6 | LavinMQ UI 503 even though the pod was healthy | Same target-group-empty root cause as #1 | Resolved automatically once the worker node joined |
| 7 | `backend_v2`: `User "orderlay-admin" not found` | User genuinely didn't exist yet on the fresh LavinMQ instance | `lavinmqctl add_user` + `set_permissions` |
| 8 | `backend_v2`: `403 ACCESS_REFUSED` | Looked like a vhost encoding issue (bare `/` vs `%2F`) — this was a red herring | Fixed the URL encoding anyway (technically correct), but it did not resolve the error |
| 9 | Same `403 ACCESS_REFUSED` persisted | Real cause: password mismatch — `xxxx` had been typed literally as the password during user creation, not substituted with the real one | Re-ran `add_user` with the actual matching password from `.env` |
| 10 | LavinMQ dashboard: "Authentication failure" with the correct password | Management UI login requires the `administrator` user tag, separate from AMQP auth | `lavinmqctl set_user_tags orderlay-admin administrator` |
