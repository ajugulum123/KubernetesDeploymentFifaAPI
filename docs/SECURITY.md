# Security posture

What this platform locks down today and what would need cloud-side or CNI-side work to take further.

## In place today

| Control | Where |
|---|---|
| Default-deny ingress on `arc-runners` | `kubernetes/policies/arc-runners-egress.yaml` |
| Allow-list egress for runners (DNS, K8s API, BuildKit, MinIO, HTTPS to internet) | same |
| ResourceQuota + LimitRange capping runner consumption | `kubernetes/policies/resourcequota-arc-runners.yaml` |
| Per-namespace RBAC for the deployer SA | `kubernetes/app/<env>/rbac.yaml` |
| App pods: non-root, read-only root FS, all caps dropped, RuntimeDefault seccomp | `kubernetes/app/<env>/deployment.yaml` |
| BuildKit rootless, no privileged, isolated namespace | `kubernetes/buildkit/` |
| Pod Security Admission (`restricted`) on app namespaces | namespace labels in `kubernetes/app/<env>/namespace.yaml` |
| GitHub App (no static PAT) | initial install, scoped App permissions |
| Secret stored in Kubernetes, not Git | `arc-github-app-secret`, `fifa-api-secrets`, `minio-credentials` |
| gitleaks on every PR | `Fifa_API/.github/workflows/ci.yml` |
| Trivy scan of the built image, fail on CRITICAL/HIGH | same |
| TLS termination at Traefik Ingress, app does plain HTTP internally | `kubernetes/app/<env>/ingress.yaml` |

## Documented follow-ups

### FQDN-scoped egress for runners

Stock NetworkPolicy cannot match egress by hostname. Today we allow `0.0.0.0/0:443` from runner pods. This is acceptable on a laptop demo. For a real deployment use one of:

- **Cilium** with `ToFQDNs` rules listing the exact GitHub and GHCR hostnames.
- A cluster-internal egress proxy (Squid, Tinyproxy) and force runners through it.
- The published GitHub IP CIDRs from `https://api.github.com/meta` rendered into NetworkPolicy CIDRs by a CronJob.

### OIDC federation to a cloud provider

The custom runner image is the right place to wire in cloud OIDC (no static cloud creds). When you decide which cloud:

#### AWS

1. Create an IAM OIDC identity provider for `https://token.actions.githubusercontent.com`.
2. Create a role with a trust policy that conditions on `repo:ajugulum123/Fifa_API:*` and `repo:ajugulum123/KubernetesDeploymentFifaAPI:*`.
3. Attach a least-privilege policy (e.g. `s3:Put*` on a build-cache bucket).
4. In the workflow:
   ```yaml
   - uses: aws-actions/configure-aws-credentials@v4
     with:
       role-to-assume: arn:aws:iam::<acct>:role/fifa-runner
       aws-region: us-east-1
   ```

#### GCP

1. Create a Workload Identity Federation pool + provider for GitHub Actions.
2. Bind a service account with the right roles to the pool.
3. In the workflow:
   ```yaml
   - uses: google-github-actions/auth@v2
     with:
       workload_identity_provider: projects/<n>/locations/global/workloadIdentityPools/github/providers/github-actions
       service_account: fifa-runner@<project>.iam.gserviceaccount.com
   ```

The hard work is the trust policy. Once it is in place the runner image and workflows pick it up automatically.

### Image signing

Add `cosign sign` after `buildctl ... push` in the CD workflow. Verify on admission with the `cosign` admission controller or with policy-controller (Sigstore). Out of scope for v1.

### External Secrets

Move JWT secrets, admin password and registry creds into a real secret backend (Vault, AWS Secrets Manager) via `external-secrets-operator`. The app side does not change because we already inject via `envFrom: [secretRef]`.

### Network observability

Add Hubble (if running Cilium) or Cilium Tetragon for runtime network and process visibility. NetworkPolicy declares intent. Hubble shows you what is actually happening.
