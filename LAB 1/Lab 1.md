# Lab 1: Cloud Account Security, Identity and Access Management

Course: IKB42603 Cloud Computing Security Essentials
Lab: Lab 1
Topic: Identity governance, least privilege, LocalStack IAM and Kubernetes RBAC
Environment: LocalStack on localhost:4566 and kind Kubernetes cluster `ccse-lab1`
Name: Nur Shafiqa binti Ab Rahim

## Lab Summary // Objective

This report walks through two hands-on exercises in cloud identity management:

- Using **LocalStack** to practise core AWS IAM operations — setting up users, groups, policies, and access keys without touching a real AWS account.
- Using a local **Kubernetes** cluster to see access control actually enforced, by defining a Role, binding it to an identity, and testing the boundary with live commands.

---

## Evidence Folder

Screenshots referenced throughout this report are kept in the `evidence lab1` folder.

| Evidence File | Purpose |
|---|---|
| `01-caller-identity.png` | `sts get-caller-identity` result confirming the CLI is pointed at LocalStack |
| `02-admin-group-setup.png` | Group + admin user creation, membership added, and `get-group` verification (Task 2) |
| `03-analyst-user-policy.png` | `Analyst_piqa` created with only `AmazonS3ReadOnlyAccess` attached (Task 3) |
| `04a-access-key-created.png` | Access key created for `Analyst_piqa`, status `Active` (Task 4, before rotation) |
| `04b-access-key-rotated.png` | Access key deactivated (rotated), confirmed via `list-access-keys` (Task 4, after rotation) |
| `05-kind-cluster-setup.png` | `kind` cluster verified up and node `Ready` |
| `06-namespaces-created.png` | `dev` and `prod` namespaces created (Task 5) |
| `07-role-rolebinding-setup.png` | ServiceAccount, Role, and RoleBinding all created (Task 6) |
| `08-auth-can-i-results.png` | The three `kubectl auth can-i` test results (Task 7) |
| `09-rolebinding-yaml.png` | RoleBinding exported as YAML (verification) |

## Overview

The lab runs across two sessions:

- **Session A:** Cloud identity basics on LocalStack — avoiding root, creating scoped users and groups, managing access keys.
- **Session B:** Kubernetes RBAC, where permissions are not just declared but actually checked and enforced against a running cluster.

Everything here stays local — LocalStack simulates the AWS API on `localhost:4566`, and `kind` spins up a disposable Kubernetes cluster inside Docker. No real cloud account is touched at any point.

---

## Session A — Cloud Identity with LocalStack

## Environment Setup

```
docker --version
```
**Explanation:** A quick sanity check that Docker itself is installed and the daemon can be reached, before anything container-related is attempted.

```
docker run -d --name localstack \
  -p 4566:4566 -p 4510-4559:4510-4559 \
  -e LOCALSTACK_AUTH_TOKEN=$LOCALSTACK_AUTH_TOKEN \
  -v /var/run/docker.sock:/var/run/docker.sock \
  localstack/localstack
```
**Explanation:** Starts the LocalStack container in the background, exposing port 4566 (the single endpoint all emulated AWS services sit behind). As of 23 March 2026, LocalStack's standard image will not boot without a valid licence auth token, so the token is passed in as an environment variable here to activate it.

```
curl http://localhost:4566/_localstack/info
```
**Explanation:** A quick way to confirm the token actually worked — the response should show `is_license_activated: true` before moving on to anything IAM-related.

```
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```
**Explanation:** The AWS CLI won't run without some credentials configured, even against an emulator. LocalStack doesn't check whether these values are real, so any placeholder string is enough.

```
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```
**Explanation:** The `--endpoint-url` flag is what actually redirects the CLI to the local container instead of real AWS. `sts get-caller-identity` then confirms which identity the CLI is currently operating under.

**Output:**
```
{
    "UserId": "000000000000",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```
The repeated `000000000000` is LocalStack's standard placeholder account number — a clear sign this ran against the local emulator rather than a genuine AWS account.

**Evidence:**

![sts get-caller-identity output](evidence%20lab1/01-caller-identity.png)

---

## Task 1 — Map the Cloud Identity Landscape

| Concept | AWS term | Purpose |
|---|---|---|
| All-powerful owner | **Root user** | The original identity tied to the account itself, with no restrictions on what it can touch. Because a leak of this identity means total compromise, it should stay locked away and unused for everyday work. |
| Human/app identity | **IAM User** | A dedicated identity for one person or one application. Giving each actor its own identity means access can be tracked, adjusted, or revoked individually rather than everyone sharing a single login. |
| Permission bundle | **IAM Policy** | The actual rulebook — a JSON document spelling out precisely which actions are permitted or blocked, and on what. Nothing is granted until a policy says so. |
| Collection of users | **IAM Group** | A bucket for related users so a policy only needs to be set once. Add someone to the group and they inherit its access; remove them and it's gone — no per-person edits needed. |
| Temporary identity | **IAM Role** | An identity with nothing permanent attached to it. Something else — a user, an app, a service — borrows it for a limited window and gets short-lived credentials in return, which is safer than handing out something that never expires. |

---

## Task 2 — Create a Least-Privilege Admin

```
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws $EP iam create-user --user-name CloudAdmin_piqa
aws $EP iam add-user-to-group --group-name Admins \
  --user-name CloudAdmin_piqa
aws $EP iam get-group --group-name Admins
```
**Explanation:** The `AdministratorAccess` policy is attached to the `Admins` group itself, not to any single user — this is the whole point of the exercise. `CloudAdmin_piqa` is created as a dedicated identity to replace root for daily admin work, then added to the group so it inherits admin access purely through membership. The final `get-group` call confirms the user really landed inside the group, and that the group's policy is correctly attached.

**Output (trimmed):**
```
{
    "Users": [
        {
            "UserName": "CloudAdmin_piqa",
            "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_piqa"
        }
    ],
    "Group": {
        "GroupName": "Admins",
        "Arn": "arn:aws:iam::000000000000:group/Admins"
    }
}
```
This confirms `CloudAdmin_piqa` sits inside `Admins`, drawing its permissions from the group rather than holding them directly.

**Evidence:**

![Admins group and CloudAdmin_piqa setup and verification](evidence%20lab1/02-admin-group-setup.png)

---

## Task 3 — Enforce Least Privilege with a Scoped Policy

```
aws $EP iam create-user --user-name Analyst_piqa
aws $EP iam attach-user-policy --user-name Analyst_piqa \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws $EP iam list-attached-user-policies --user-name Analyst_piqa
```
**Explanation:** A second identity is built for someone who only needs to look at data, never change it. `AmazonS3ReadOnlyAccess` only permits get/list-style S3 actions — nothing that writes or deletes. The final `list-attached-user-policies` check confirms exactly one policy is attached, and that it's the read-only one, with no broader access snuck in.

**Output:**
```
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonS3ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        }
    ]
}
```

**Evidence:**

![Analyst_piqa user created with S3 read-only policy attached](evidence%20lab1/03-analyst-user-policy.png)

### Short-Answer: Blast-Radius Reduction

> **If the Analyst account were stolen, why is the damage limited compared to a stolen admin account?**

`Analyst_piqa` only carries `AmazonS3ReadOnlyAccess`, so a stolen credential hands an attacker nothing more than the ability to read S3 objects — no deleting, no writing, no reach into any other service. A stolen `CloudAdmin_piqa` credential is a completely different situation: it carries `AdministratorAccess`, meaning the attacker could do anything at all across the account. Keeping the Analyst identity tightly scoped is what keeps a worst-case breach small and contained rather than total — that's what "blast radius" is describing.

---

## Task 4 — Credential Hygiene & Access Keys

```
aws $EP iam create-access-key --user-name Analyst_piqa
aws $EP iam list-access-keys --user-name Analyst_piqa
aws $EP iam update-access-key --user-name Analyst_piqa \
  --access-key-id <AccessKeyId> --status Inactive
aws $EP iam list-access-keys --user-name Analyst_piqa
```
**Explanation:** A key pair is generated so `Analyst_piqa` can authenticate programmatically instead of through a console login. The secret value only appears once at creation time and cannot be retrieved again afterward, so it needs to be captured immediately if it's ever needed (and kept out of screenshots submitted for this report — the `AccessKeyId` and `SecretAccessKey` are redacted below). Deactivating rather than deleting the key mimics a real rotation workflow — a new key would normally take over first, with the old one switched off afterward. Keeping keys short-lived limits how much damage a leaked one could do before anyone notices.

**Output (before rotation):**
```
{
    "AccessKey": {
        "UserName": "Analyst_piqa",
        "AccessKeyId": "[redacted]",
        "Status": "Active",
        "SecretAccessKey": "[redacted]",
        "CreateDate": "2026-08-05T11:36:32.476158+00:00"
    }
}
```

**Output (after rotation):**
```
{
    "AccessKeyMetadata": [
        {
            "UserName": "Analyst_piqa",
            "AccessKeyId": "...",
            "Status": "Inactive",
            "CreateDate": "2026-08-05T11:36:32.476158+00:00"
        }
    ]
}
```

**Evidence:**

![Access key created, status Active before rotation](evidence%20lab1/04a-access-key-created.png)

![Access key deactivated, status confirmed Inactive after rotation](evidence%20lab1/04b-access-key-rotated.png)

*End of Session A.*

---

## Session B — Enforced Access Control with Kubernetes RBAC

LocalStack shows what IAM structures look like, but it doesn't stop you from doing something you shouldn't. Kubernetes RBAC is different — it actively checks and blocks disallowed actions, which this session sets out to prove directly.

```
kind create cluster --name ccse-lab1
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```
**Explanation:** `kind` (Kubernetes-in-Docker) builds a genuine, disposable Kubernetes cluster inside Docker containers, entirely on the local machine — no cloud account or billing involved. The `cluster-info` and `get nodes` checks confirm the control plane responds and the node is actually ready before anything is deployed into the cluster.

**Result:** cluster `ccse-lab1` came up cleanly, node status `Ready`.

**Evidence:**

![kind cluster and node status verified Ready](evidence%20lab1/05-kind-cluster-setup.png)

---

## Task 5 — Separate Environments with Namespaces

```
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```
**Explanation:** Namespaces split one cluster into isolated pockets. Having `dev` and `prod` side by side sets up the test in Task 7 — proving that access granted in one doesn't leak into the other.

**Result:** both namespaces created, both `Active`.

**Evidence:**

![dev and prod namespaces created and Active](evidence%20lab1/06-namespaces-created.png)

---

## Task 6 — Define a Role and Bind It

```
kubectl create serviceaccount dev-user -n dev

kubectl create role pod-reader -n dev \
  --verb=get,list,watch --resource=pods

kubectl create rolebinding dev-user-binding -n dev \
  --role=pod-reader --serviceaccount=dev:dev-user
```
**Explanation:** A ServiceAccount stands in for a non-human identity — here, representing a developer — scoped from the outset to the `dev` namespace only. The Role `pod-reader` defines a namespaced permission set limited to read-type actions on pods, with nothing that allows creating, deleting, or editing. A Role alone does nothing until something is bound to it — the RoleBinding is what connects `pod-reader` to `dev-user`, and because RoleBindings are namespace-scoped, the grant stays confined to `dev`.

**Result:** service account, Role, and RoleBinding all created successfully.

**Evidence:**

![ServiceAccount, Role, and RoleBinding created](evidence%20lab1/07-role-rolebinding-setup.png)

---

## Task 7 — Test That Access Control Works

```
SA=system:serviceaccount:dev:dev-user

kubectl auth can-i list pods -n dev --as=$SA
kubectl auth can-i delete pods -n dev --as=$SA
kubectl auth can-i list pods -n prod --as=$SA
```
**Explanation:** `kubectl auth can-i --as=<identity>` checks what an identity would be allowed to do, without actually performing the action. Running all three side by side shows the boundary from every angle: what's allowed, what verb is blocked, and what namespace is off-limits.

**Results:**
- `list pods -n dev` → **yes** — covered directly by `pod-reader`.
- `delete pods -n dev` → **no** — `delete` was never part of the Role's verb list.
- `list pods -n prod` → **no** — the RoleBinding never extended past `dev`.

**Evidence:**

![Three auth can-i results: yes, no, no](evidence%20lab1/08-auth-can-i-results.png)

**Report answer — Authentication vs. Authorization:** All three checks pass authentication without issue — Kubernetes has no trouble recognising `system:serviceaccount:dev:dev-user` as a legitimate identity every time. The difference between the three results comes down to authorization instead. For `list pods -n dev`, the Role and RoleBinding line up with what's being asked, so it's approved. For the other two, the identity is still recognised, but nothing in its RBAC configuration covers `delete` or reaches into `prod` — so authorization is what refuses the request, not any failure to identify who's asking.

---

## Verification Command

```
kubectl get rolebinding dev-user-binding -n dev -o yaml
```
**Explanation:** Pulls the RoleBinding straight from the cluster as YAML — solid proof that what's configured matches what was intended, rather than relying on the creation commands alone.

**Output:**
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
This matches exactly what Task 6 set up — `dev-user-binding` ties `dev-user` to `pod-reader`, scoped to `dev`.

**Evidence:**

![RoleBinding exported as YAML](evidence%20lab1/09-rolebinding-yaml.png)

---

## Short-Answer Questions

**Q1. Why is attaching policies to groups better than attaching them directly to users?**
A policy on a group applies to everyone in it at once — one edit, and every member's access updates together. Doing it per-user means the same change has to be repeated for each individual, which gets harder to keep consistent and easier to lose track of as the number of users grows.

**Q2. What is the difference between an IAM User and an IAM Role?**
A User is a fixed identity with its own long-term login details, tied to a specific person or application until someone deletes it. A Role has no credentials of its own at all — something else assumes it temporarily and receives a short-lived token for that session only. Because nothing permanent is issued, there's no long-term secret sitting around that could eventually leak.

**Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.**
`Analyst_piqa` was only ever given `AmazonS3ReadOnlyAccess` — just enough to do its job, nothing extra. If those credentials were stolen, the attacker's options are limited to reading S3 data; there's no path to deleting, modifying, or reaching any other part of the account. That containment is exactly what least privilege buys you — the compromise stays small instead of turning into full account control, which is what would happen with a leaked admin identity instead.

**Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?**
A Role is just a description of permissions — which verbs apply to which resources, inside a given namespace. By itself it doesn't grant anyone anything. A RoleBinding is the step that actually connects that Role to a real identity (a user, group, or service account), which is what turns the description into an active grant.

**Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?**
`dev-user`'s only RoleBinding lives in `dev` — there's nothing equivalent set up for `prod`, so Kubernetes has no rule to grant access there and defaults to refusing it. This is least privilege in action at the platform level: an identity gets exactly the access it was explicitly given, and namespaces don't leak permissions into each other just because they share a cluster.

---

## Cleanup & Teardown

```
kind delete cluster --name ccse-lab1
docker stop localstack && docker rm localstack
```

## Security Best-Practices Checklist

- [x] Root was never used for day-to-day work — `CloudAdmin_piqa` was created as a dedicated admin identity instead.
- [x] Admin access came through the `Admins` group rather than being attached to the user directly.
- [x] A scoped-down identity, `Analyst_piqa`, was created with read-only S3 access only.
- [x] An access key was generated, listed, and deactivated to demonstrate rotation.
- [x] Kubernetes RBAC actually blocked both a disallowed verb (delete) and a cross-namespace request (prod).
- [x] `dev` and `prod` were kept in separate namespaces so permissions in one couldn't bleed into the other.

---

## Conclusion

Across both sessions, the same underlying idea kept showing up in different forms: identities should only ever get exactly the access they need, and that access should be structured so it's easy to manage and easy to audit. In LocalStack, that meant attaching admin rights to a group instead of a person, and carving out a separate read-only identity for a lower-trust role. In Kubernetes, the same principle was tested for real — `dev-user` could read pods in `dev` and nothing more, with both a disallowed action and a disallowed namespace refused automatically once the Role and RoleBinding were checked. LocalStack showed how the pieces of IAM fit together conceptually; the Kubernetes cluster showed what it actually looks like when those boundaries are enforced rather than just declared.
