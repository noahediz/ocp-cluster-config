# GitOps structure: OCP cluster configuration

## Introduction
This repository manages the cluster-level configuration for my multi-platform homelab environment. Using **Kustomize** and **ArgoCD**, it implements a "Single Source of Truth" for infrastructure components without storing user applications or sensitive business logic.

My interpretation of the GitOps pattern focuses on a three-axis configuration strategy: **Components** (What is installed), **Tenants** (Which platform flavor), and **Environments** (Which lifecycle stage).

## Cluster Inventory
| Cluster Name | Provider/Tenant | Environment |
| :--- | :--- | :--- |
| `homelab-k8s-dev01` | Generic K8s | Development |
| `homelab-k8s-dev02` | Generic K8s | Development |
| `homelab-k8s-prod01` | Generic K8s | Production |
| `aws-eks-dev01` | AWS EKS | Development |
| `aws-eks-prod01` | AWS EKS | Production |

---

## Repository Principles
* **No Environment Branches**: We use a single `main` branch. Configuration drift is managed via folders and Kustomize overlays, not Git merges.
* **Infrastructure Only**: This repo is dedicated to cluster-admin level configurations (Operators, StorageClasses, RBAC, Networking). No user-facing apps live here.
* **Tenant-Based Abstraction**: The `tenants/` folder acts as a "Platform Flavor" layer. 
    * `homelab-k8s` handles on-premise logic (e.g., MetalLB).
    * `aws-eks` handles cloud-provider logic (e.g., AWS Load Balancer Controller).
* **DRY (Don't Repeat Yourself)**: If a config is used in more than one place, it belongs in `components/`.

---

### The "Recursive Overlay" structure
Every layer of the hierarchy (**Tenants**, **Environments**, and **Clusters**) follows a strict internal structure: `overlays/{name}/components/`. 

**Why we do this:**
* **No Base Duplication**: The root `/components` folder is the **only** place allowed to store base manifests. All other directories are forced to be overlays.
* **Structural Predictability**: By nesting `components` inside every overlay, we create a clear map. If you want to see how the "Monitoring" component is modified for EKS, you know exactly where to look without searching the whole tree.
* **Granular Control**: This allows us to apply a "patch on top of a patch." We can define a generic monitoring setup, patch it for AWS-specific storage in the Tenant layer, and then patch it again for lower retention in the Dev layer.

---

## Folder Layout
### ├── `/components`
**The Source of Truth.** This is the only directory containing raw, "naked" Kubernetes manifests. 
* *Constraint:* No environment-specific or platform-specific logic allowed here.

### ├── `/tenants/overlays/{tenant}/components`
**The Platform Flavor.** Handles provider-specific requirements.
* **homelab-k8s**: Overlays for on-premise logic (e.g., MetalLB, Local CSI).
* **aws-eks**: Overlays for cloud-provider logic (e.g., AWS Load Balancer Controller, EBS CSI).

### ├── `/environments/overlays/{env}/components`
**The Lifecycle State.** Handles scaling and stability requirements.
* **dev**: Patches for cost-savings (e.g., 1 replica, minimal resource requests).
* **prod**: Patches for High Availability (e.g., Pod Anti-Affinity, persistent storage backups).

### ├── `/clusters/overlays/{cluster}/components`
**The Final Realization.** This is the leaf node that defines a specific cluster instance.
> **Logic:** `Cluster = Component Base + Tenant Overlay + Environment Overlay`

---

## Promotion Workflow
To promote a change from Dev to Prod:
1.  Modify the base manifest in `components/` or a specific tenant in `tenants/`.
2.  ArgoCD will automatically detect the change and show an "OutOfSync" status.
3.  Apply/Sync to `dev` clusters first.
4.  Once verified, sync `prod` clusters.
*For strict pinning, use Git SHAs or Tags in the Kustomize `resources` fields.*

## References
This structure is an evolution of the [Red Hat Canada GitOps Standards](https://github.com/gnunn-gitops/standards), adapted for a hybrid-cloud homelab environment.
