# Lab 1 Report: Cloud Account Security, Identity & Access Management

**Name:** [Your Full Name]
**Subject:** Cloud Computing Security Essentials
**Code:** IKB 42603
**Date:** 5 August 2026

---

## Purpose

This report documents the implementation and verification of identity governance controls across two environments: a simulated AWS account running on LocalStack, and a local Kubernetes cluster provisioned with `kind`. Screenshots taken during each task serve as evidence of successful execution.

## Environment

- **OS:** Windows 11, commands executed through Git Bash
- **LocalStack:** started with an auth token to unlock the Pro edition (activation is required as of 23 March 2026)
- **Cluster tooling:** `kind` v0.31.0, `kubectl` client v1.36.3

---

## 1. LocalStack Startup & Identity Verification

LocalStack was started as a Docker container with the licence auth token injected as an environment variable, activating the Pro edition rather than the free Community image:

```
docker run -d --name localstack \
  -p 4566:4566 -p 4510-4559:4510-4559 \
  -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
  -v /var/run/docker.sock:/var/run/docker.sock \
  localstack/localstack
```

Activation and health were confirmed:

```
curl http://localhost:4566/_localstack/info
{"edition": "pro", "is_license_activated": true, ...}
```

The AWS CLI was pointed at the container using dummy credentials, since LocalStack does not validate them against a real account:

```
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

```
{
    "UserId": "000000000000",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

The all-zero account number is expected — it is LocalStack's fixed placeholder identity, not a real AWS account number.

- **Result:** the CLI is confirmed to be talking to the local container (not real AWS), operating under the simulated root identity before any scoped users exist.
- **Evidence:**

`[insert screenshot: sts get-caller-identity output]`

---

## 2. Task 1 — Mapping the Cloud Identity Landscape

Before creating any identities, the core building blocks of cloud IAM were mapped out conceptually:

| Concept | AWS Term | Purpose |
|---|---|---|
| All-powerful owner | Root user | The default, unrestricted identity created with the account. Has full access to everything and should be locked away rather than used for daily work. |
| Human/app identity | IAM User | A named identity tied to one person or application, holding only the permissions assigned to it. |
| Permission bundle | IAM Policy | A JSON document listing exactly which actions are allowed or denied against which resources. |
| Collection of users | IAM Group | A container that groups multiple users together so permissions can be managed once, at the group level. |
| Temporary identity | IAM Role | An identity that is assumed rather than owned, issuing short-lived credentials instead of a permanent password or key. |

- **Result:** the conceptual foundation for the rest of the lab — every subsequent task builds directly on these five building blocks.

---

## 3. Task 2 — Creating a Least-Privilege Admin

Rather than operating as root, an `Admins` group was created first, with the `AdministratorAccess` policy attached to the **group** — not to any individual user:

```
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

A personal admin identity, `CloudAdmin_piqa`, was then created and placed inside that group:

```
aws $EP iam create-user --user-name CloudAdmin_piqa
aws $EP iam add-user-to-group --group-name Admins --user-name CloudAdmin_piqa
aws $EP iam get-group --group-name Admins
```

```
{
    "Users": [{ "UserName": "CloudAdmin_piqa", ... }],
    "Group": { "GroupName": "Admins", ... }
}
```

- **Result:** `CloudAdmin_piqa` inherits `AdministratorAccess` through group membership. Because the policy lives on the group, changing everyone's access later only requires editing the group once — not every user individually.
- **Evidence:**

`[insert screenshot: get-group Admins output showing CloudAdmin_piqa]`

---

## 4. Task 3 — Enforcing Least Privilege with a Scoped Policy

A second, deliberately restricted identity was created to represent a teammate who should never be able to modify data:

```
aws $EP iam create-user --user-name Analyst_piqa
aws $EP iam attach-user-policy --user-name Analyst_piqa \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws $EP iam list-attached-user-policies --user-name Analyst_piqa
```

```
{
    "AttachedPolicies": [
        { "PolicyName": "AmazonS3ReadOnlyAccess", ... }
    ]
}
```

- **Result:** `Analyst_piqa` holds exactly one policy, and it is read-only. No write, delete, or IAM-management permissions exist on this identity at all.
- **Evidence:**

`[insert screenshot: list-attached-user-policies showing only AmazonS3ReadOnlyAccess]`

### Short-Answer: Blast-Radius Reduction

> **If the Analyst account were stolen, why is the damage limited compared to a stolen admin account?**

If `Analyst_piqa`'s credentials leak, an attacker inherits only what `AmazonS3ReadOnlyAccess` allows — reading objects out of S3. They cannot delete data, alter IAM, or touch any other service. A stolen `CloudAdmin_piqa` credential, by contrast, carries `AdministratorAccess`, meaning full control over the entire account, including the ability to create new privileged users or tear down existing infrastructure. Scoping the Analyst tightly means a compromise there stays contained to "someone read some files," instead of "someone owns the account" — that containment is the practical meaning of blast-radius reduction.

---

## 5. Task 4 — Credential Hygiene & Access Keys

An access key was generated for `Analyst_piqa` to simulate programmatic access, then rotated to demonstrate credential hygiene:

```
aws $EP iam create-access-key --user-name Analyst_piqa
aws $EP iam list-access-keys --user-name Analyst_piqa
aws $EP iam update-access-key --user-name Analyst_piqa \
  --access-key-id <AccessKeyId> --status Inactive
aws $EP iam list-access-keys --user-name Analyst_piqa
```

```
{
    "AccessKeyMetadata": [
        { "AccessKeyId": "...", "Status": "Inactive", ... }
    ]
}
```

- **Result:** the key's status changed from `Active` to `Inactive` after rotation, confirming that deactivation works as an incident-response step. In a real environment, long-lived static keys like this carry more risk than short-lived role-based credentials, since a leaked key stays valid indefinitely until someone notices and rotates it.
- **Evidence:**

`[insert screenshot: list-access-keys showing Status: Inactive]`

---

## 6. Kubernetes Cluster Provisioning

Session B shifts from LocalStack's simulated IAM (which does not actually block anything) to Kubernetes RBAC, which enforces access control in real time. A disposable cluster was created with `kind`:

```
kind create cluster --name ccse-lab1
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```

```
NAME                       STATUS   ROLES           AGE   VERSION
ccse-lab1-control-plane    Ready    control-plane   67s   v1.35.0
```

- **Result:** a single-node cluster is up, with the control plane healthy and the node reporting `Ready`.
- **Evidence:**

`[insert screenshot: kind create cluster + get nodes output]`

---

## 7. Task 5 — Namespace Isolation

Two namespaces were created to represent separate environments living in the same cluster:

```
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

- **Result:** both `dev` and `prod` report `Active`, giving a clean boundary to test cross-namespace access against in Task 7.

---

## 8. Task 6 — Defining a Role and Binding It

A service account was created to represent a developer identity, paired with a `Role` restricted to read-only pod access, and a `RoleBinding` connecting the two:

```
kubectl create serviceaccount dev-user -n dev

kubectl create role pod-reader -n dev \
  --verb=get,list,watch --resource=pods

kubectl create rolebinding dev-user-binding -n dev \
  --role=pod-reader --serviceaccount=dev:dev-user
```

```
serviceaccount/dev-user created
role.rbac.authorization.k8s.io/pod-reader created
rolebinding.rbac.authorization.k8s.io/dev-user-binding created
```

- **Result:** `dev-user` now has exactly `get`, `list`, and `watch` on pods, scoped to the `dev` namespace only — nothing more, and nowhere else.
- **Evidence:**

`[insert screenshot: serviceaccount / role / rolebinding creation output]`

---

## 9. Task 7 — Testing That Access Control Actually Works

Unlike LocalStack's IAM simulation, `kubectl auth can-i` queries the live RBAC engine directly, so the results reflect real enforcement:

```
SA=system:serviceaccount:dev:dev-user

kubectl auth can-i list pods -n dev --as=$SA
kubectl auth can-i delete pods -n dev --as=$SA
kubectl auth can-i list pods -n prod --as=$SA
```

| Action | Namespace | Result |
|---|---|---|
| list pods | dev | **yes** |
| delete pods | dev | **no** |
| list pods | prod | **no** |

- **Result:** the boundary holds exactly as configured — read access inside `dev`, no destructive action anywhere, and no reach into `prod` at all, even though both namespaces live in the same cluster.
- **Evidence:**

`[insert screenshot: three kubectl auth can-i results]`

---

## 10. Final Verification

The deployed `RoleBinding` was exported as YAML to confirm the live cluster state matches what was intended:

```
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```

- **Result:** the binding's `roleRef` and `subjects` match exactly what was configured in Task 6 — no drift between intent and the actual cluster state.
- **Evidence:**

`[insert screenshot: rolebinding YAML output]`

---

## Lab Report Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Attaching a policy to a group means every current and future member inherits it automatically, and a single edit at the group level updates access for everyone at once. Attaching policies user-by-user instead causes permissions to drift over time — as people join, leave, or change roles, it becomes difficult to audit who actually holds what, and stale access is easy to leave behind unnoticed.

### Q2. What is the difference between an IAM User and an IAM Role?

An IAM User is a persistent identity with permanent credentials tied to one person or application — it exists until explicitly deleted. An IAM Role carries no credentials of its own; it is assumed temporarily by a trusted user, service, or application, which then receives short-lived tokens for the duration of that session. Roles are preferable wherever access does not need to be permanent, since there is no long-lived secret sitting around waiting to leak.

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

Least privilege means granting an identity only the access it needs to do its job, and nothing beyond that. `Analyst_piqa` was scoped to `AmazonS3ReadOnlyAccess` alone, so even in the worst case of credential theft, the attacker's capability is capped at reading S3 objects — no pivoting to other services, no privilege escalation, no destructive action. The blast radius of a compromise is bounded by the policy attached to the identity, not by what the account could theoretically do.

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?

A `Role` defines a set of permissions — which verbs (get, list, delete, and so on) are allowed on which resources, scoped to a namespace. On its own, a `Role` grants nothing to anyone. A `RoleBinding` is what actually assigns that `Role` to a subject (a user, group, or service account), activating the permissions for that specific identity within that namespace.

### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?

`dev-user`'s `RoleBinding` exists only in the `dev` namespace — there is no equivalent binding granting it any permissions in `prod`. Kubernetes RBAC is deny-by-default: unless a binding explicitly grants an action within a given namespace, that action is refused. This is least privilege enforced at the platform level, not just described on paper — the service account can do exactly what it was bound to do, and nothing beyond that, even though `dev` and `prod` sit in the same cluster.

---

## Conclusion

All seven tasks across Session A (LocalStack IAM) and Session B (Kubernetes RBAC) were completed and verified end-to-end. Identity boundaries — root avoidance, group-based admin access, and a scoped read-only Analyst — were established and confirmed in LocalStack. Namespace-scoped RBAC was then shown to actively block unauthorized actions in a live cluster, rather than merely describing them in a policy document. Both environments were torn down after all evidence was collected.

## Challenges Encountered & Lessons Learned

The lab manual's LocalStack startup command assumed the free Community image, but the environment in use required an auth token to activate (a licensing change introduced after the manual was written) — this meant adapting the setup command before Session A could begin at all. A second recurring issue was that environment variables such as the auth token and the `$EP` endpoint alias do not persist across terminal sessions in Git Bash, so they had to be re-set at the start of every new session; forgetting this caused a misleading `InvalidClientTokenId` error when the CLI silently fell back to contacting real AWS instead of LocalStack. The main lesson was to verify the environment state (`docker ps -a`, `curl .../info`) before running lab commands, rather than assuming a previous session's setup was still active.
