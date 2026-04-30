# OpenShift Log Forwarding → AWS CloudWatch

Forwards OpenShift audit logs to AWS CloudWatch using the Cluster Logging Operator (CLO).
Deployed as a Helm chart via ArgoCD. Designed for clusters running in **Manual credentials
mode** with STS/`ccoctl`.

Audit logs cover API server, node, and OAuth audit trails. This configuration is scoped to
audit logs only to satisfy STIG compliance requirements. Infrastructure and application logs
can be added later with a one-line change — see [Adding More Log Types](#adding-more-log-types).

---

## Prerequisites

- OpenShift 4.21+ with Manual credentials mode (`ccoctl`/STS)
- `ccoctl` installed and configured with AWS credentials that can create IAM roles
- ArgoCD installed on the cluster
- `oc` and `helm` CLIs

---

## Configuration

Edit `chart/values.yaml` before deploying:

```yaml
clusterName: "my-cluster"   # Used as the CloudWatch log group prefix
awsRegion: "us-east-1"      # AWS region where logs will be written
```

The resulting CloudWatch log group will be: `<clusterName>.audit`

To list available CLO channels on your cluster:
```bash
oc get packagemanifest cluster-logging -n openshift-marketplace \
  -o jsonpath='{.status.channels[*].name}'
```

---

## Deployment

### Step 1 — Deploy via ArgoCD

Set `spec.source.repoURL` in `argocd/application.yaml` to your Git remote, then apply:

```bash
oc apply -f argocd/application.yaml
```

ArgoCD syncs in waves, ensuring correct ordering:

| Wave | Resources |
|---|---|
| 0 | Namespace, OperatorGroup, Subscription |
| 1 | ServiceAccount, ClusterRoleBinding |
| 2 | ClusterLogForwarder |

The `ClusterLogForwarder` (wave 2) will remain degraded until the AWS credentials Secret
exists. ArgoCD will keep retrying — this is expected.

### Step 2 — (Optional) Inspect the CredentialsRequest

Once CLO is installed, check if the operator created a `CredentialsRequest` you can use as
a reference or replace the one in this repo:

```bash
oc get credentialsrequest -n openshift-cloud-credential-operator -o yaml | grep -A10 -i logging
```

### Step 3 — Create the IAM role with ccoctl and apply the Secret

`ccoctl` reads `credentials-requests/cloudwatch-logging.yaml`, creates an IAM role in AWS
scoped to the cluster's OIDC provider, and writes a Kubernetes Secret to `_generated/`.

```bash
OIDC_ENDPOINT=$(oc get authentication cluster -o jsonpath='{.spec.serviceAccountIssuer}')

ccoctl aws create-iam-roles \
  --credentials-requests-dir=credentials-requests/ \
  --output-dir=_generated/ \
  --name=<cluster-name> \
  --region=<aws-region> \
  --oidc-endpoint=${OIDC_ENDPOINT}

oc apply -f _generated/manifests/openshift-logging-cloudwatch-credentials-credentials.yaml
```

Once the Secret is in place, ArgoCD self-heals and the `ClusterLogForwarder` reconciles.

---

## Verification

```bash
# Confirm collector pods are running
oc get pods -n openshift-logging -l component=collector

# Check ClusterLogForwarder status
oc get clusterlogforwarder instance -n openshift-logging \
  -o jsonpath='{.status.conditions}' | jq .

# Tail collector logs for errors
oc logs -n openshift-logging -l component=collector --tail=50

# Confirm the log group exists in AWS
aws logs describe-log-groups \
  --log-group-name-prefix "<clusterName>.audit" \
  --region <awsRegion>

# Confirm log events are arriving
aws logs describe-log-streams \
  --log-group-name "<clusterName>.audit" \
  --region <awsRegion> \
  --order-by LastEventTime \
  --descending
```

---

## AWS Resources

Nothing needs to be manually created in AWS ahead of time:

| Resource | How it's created |
|---|---|
| IAM OIDC Identity Provider | Already exists — created at cluster install |
| IAM Role | Created by `ccoctl` in Step 2 |
| CloudWatch Log Group | Auto-created by the collector on first log write |
| Encryption | AWS-managed key (`aws/logs`) — default server-side encryption |

> **Upgrading to production:** If your STIG baseline requires a Customer Managed Key (CMK),
> the log group must be pre-created with the KMS key association and `kms:GenerateDataKey` /
> `kms:Decrypt` added to the IAM policy in `credentials-requests/cloudwatch-logging.yaml`.

---

## Adding More Log Types

To forward infrastructure or application logs in addition to audit, add them to `inputRefs`
in `chart/templates/clusterlogforwarder-instance.yaml`:

```yaml
pipelines:
  - name: audit-to-cloudwatch
    inputRefs:
      - audit
      - infrastructure   # add this
      - application      # and/or this
    outputRefs:
      - cloudwatch-audit
```

Commit the change — ArgoCD syncs it automatically.
