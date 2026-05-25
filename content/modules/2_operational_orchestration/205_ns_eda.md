# Part 7: EDA-Enabled Namespace Provisioning

## Overview

In this section you will extend the namespace request workflow by enabling **Event-Driven Ansible (EDA)** governance on a namespace at provisioning time.

When a developer requests a **medium** namespace and checks the **Enable EDA Governance** option, the workflow applies a label to the namespace:

```
eda-governed: "true"
```

This label acts as a signal to the EDA Decision Controller. The [juniper.eda.k8s](https://github.com/juniper/juniper.eda.k8s) collection watches the cluster for events on labelled namespaces and triggers Ansible Automation Controller (AAC) job templates in response to detected drift, policy violations, or resource lifecycle events.

By the end of this section you will have:

- Provisioned an EDA-governed medium namespace via the Orchestrator workflow
- Observed how the EDA label activates namespace-scoped event polling
- Triggered and verified EDA-driven remediation for three real-world scenarios:
  - A PersistentVolumeClaim created without a storage class label
  - A TLS certificate Secret missing a required annotation
  - A NetworkPolicy deleted from the namespace

---

## Architecture: How the EDA Label Connects the Layers

```
Developer checks "Enable EDA Governance" in workflow form
          ↓
Orchestrator workflow provisions namespace via Software Template
          ↓
Software Template applies label: eda-governed=true to Namespace CR
          ↓
EDA Decision Controller (juniper.eda.k8s) detects labelled namespace
          ↓
EDA begins polling events scoped to that namespace
          ↓
Event detected (PVC / Certificate / NetworkPolicy change)
          ↓
EDA rulebook fires matching rule
          ↓
Ansible Automation Controller runs remediation Job Template
          ↓
Namespace returns to desired state — Git is the audit trail
```

| Layer | Component |
|---|---|
| Self-service UI | Developer Portal (Orchestrator workflow form) |
| Provisioning | SonataFlow workflow → Software Template → GitOps (ArgoCD) |
| Label application | Namespace CR annotation applied by Helm chart at sync time |
| Event source | `juniper.eda.k8s` collection watching labelled namespaces |
| Decision engine | Ansible EDA Decision Controller (rulebooks) |
| Remediation | Ansible Automation Controller job templates (playbooks) |
| Audit | AAC job logs + Git history of namespace manifests |

---

## Step 1: Update the Software Template to Support the EDA Label

### 1a: Add the EDA Checkbox to the Template Form

In your namespace Software Template (`template.yaml`), add the following parameter to the `spec.parameters` section:

```yaml
- title: Governance Options
  properties:
    edaGovernance:
      title: Enable EDA Governance
      type: boolean
      default: false
      description: >
        When enabled, applies the eda-governed=true label to the namespace.
        The EDA Decision Controller will begin polling events on this namespace
        and trigger remediation playbooks for detected drift or policy violations.
  dependencies: {}
```

### 1b: Pass the Value Through the Workflow Step

In the `steps` section of `template.yaml`, pass `edaGovernance` as a value to the Helm chart action:

```yaml
- id: create-namespace-manifests
  name: Create Namespace Manifests
  action: fetch:template
  input:
    url: ./skeleton
    values:
      namespace: ${{ parameters.namespaceName }}
      size: ${{ parameters.size }}
      edaGovernance: ${{ parameters.edaGovernance }}
```

### 1c: Apply the Label Conditionally in the Helm Chart

In `manifests/helm/app/templates/namespace.yaml`, add the EDA label conditionally:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace }}
  labels:
    environment: {{ .Values.environment | default "dev" }}
    managed-by: gitops
    {{- if .Values.edaGovernance }}
    eda-governed: "true"
    {{- end }}
  annotations:
    provisioned-by: orchestrator
    {{- if .Values.edaGovernance }}
    eda.ansible.com/watch-events: "PersistentVolumeClaim,Secret,NetworkPolicy"
    {{- end }}
```

> **ℹ️ Note**
> The `eda.ansible.com/watch-events` annotation is optional but useful — it documents which resource types the EDA rulebook monitors for this namespace and can be read by tooling to auto-generate rulebook scope.

Commit these changes to your template repository. ArgoCD will sync the updated template on next reconciliation.

---

## Step 2: Configure the EDA Rulebook

The EDA Decision Controller uses a rulebook to match events from the `juniper.eda.k8s` event source and dispatch the appropriate AAC job template.

Create the following rulebook in your EDA rulebooks repository (e.g. `rulebooks/namespace-governance.yml`):

```yaml
---
- name: EDA Namespace Governance
  hosts: all

  sources:
    - juniper.eda.k8s:
        api_version: v1
        kind: Namespace
        label_selectors:
          - "eda-governed=true"
        # Watch these resource types within governed namespaces
        watched_resources:
          - api_version: v1
            kind: PersistentVolumeClaim
          - api_version: v1
            kind: Secret
          - api_version: networking.k8s.io/v1
            kind: NetworkPolicy

  rules:

    # ── Rule 1: PVC missing required storage class label ──────────────────────
    - name: PVC missing storage class label
      condition: >
        event.type == "ADDED" and
        event.resource.kind == "PersistentVolumeClaim" and
        event.resource.metadata.labels["storage-class-approved"] is not defined
      action:
        run_job_template:
          name: "NaaS - Remediate PVC Labels"
          organization: Platform
          job_args:
            extra_vars:
              namespace: "{{ event.resource.metadata.namespace }}"
              pvc_name: "{{ event.resource.metadata.name }}"

    # ── Rule 2: TLS Secret missing required certificate annotation ────────────
    - name: TLS Secret missing certificate annotation
      condition: >
        event.type == "ADDED" and
        event.resource.kind == "Secret" and
        event.resource.type == "kubernetes.io/tls" and
        event.resource.metadata.annotations["cert-manager.io/certificate-name"] is not defined
      action:
        run_job_template:
          name: "NaaS - Remediate TLS Certificate Annotation"
          organization: Platform
          job_args:
            extra_vars:
              namespace: "{{ event.resource.metadata.namespace }}"
              secret_name: "{{ event.resource.metadata.name }}"

    # ── Rule 3: NetworkPolicy deleted from governed namespace ─────────────────
    - name: NetworkPolicy deleted
      condition: >
        event.type == "DELETED" and
        event.resource.kind == "NetworkPolicy"
      action:
        run_job_template:
          name: "NaaS - Restore NetworkPolicy"
          organization: Platform
          job_args:
            extra_vars:
              namespace: "{{ event.resource.metadata.namespace }}"
              policy_name: "{{ event.resource.metadata.name }}"
```

> **ℹ️ Note**
> The `juniper.eda.k8s` collection must be installed in your EDA Decision Controller environment:
> ```bash
> ansible-galaxy collection install juniper.eda.k8s
> ```

---

## Step 3: Create the AAC Remediation Job Templates

In Ansible Automation Controller, create the following Job Templates. Each corresponds to a rule in the rulebook above.

### Job Template 1: Remediate PVC Labels

```yaml
# playbooks/remediate-pvc-labels.yml
---
- name: Remediate PVC — Apply Required Storage Class Label
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Apply storage-class-approved label to PVC
      kubernetes.core.k8s:
        state: patched
        api_version: v1
        kind: PersistentVolumeClaim
        name: "{{ pvc_name }}"
        namespace: "{{ namespace }}"
        definition:
          metadata:
            labels:
              storage-class-approved: "true"

    - name: Write audit annotation
      kubernetes.core.k8s:
        state: patched
        api_version: v1
        kind: PersistentVolumeClaim
        name: "{{ pvc_name }}"
        namespace: "{{ namespace }}"
        definition:
          metadata:
            annotations:
              eda.ansible.com/last-remediated: "{{ ansible_date_time.iso8601 }}"
              eda.ansible.com/remediation-reason: "Missing storage-class-approved label"
```

### Job Template 2: Remediate TLS Certificate Annotation

```yaml
# playbooks/remediate-tls-annotation.yml
---
- name: Remediate TLS Secret — Apply Required Certificate Annotation
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Read existing Secret
      kubernetes.core.k8s_info:
        api_version: v1
        kind: Secret
        name: "{{ secret_name }}"
        namespace: "{{ namespace }}"
      register: secret_info

    - name: Derive certificate name from secret name
      set_fact:
        cert_name: "{{ secret_name | regex_replace('-tls$', '') }}"

    - name: Apply cert-manager annotation to Secret
      kubernetes.core.k8s:
        state: patched
        api_version: v1
        kind: Secret
        name: "{{ secret_name }}"
        namespace: "{{ namespace }}"
        definition:
          metadata:
            annotations:
              cert-manager.io/certificate-name: "{{ cert_name }}"
              eda.ansible.com/last-remediated: "{{ ansible_date_time.iso8601 }}"
              eda.ansible.com/remediation-reason: "Missing cert-manager annotation on TLS Secret"
```

### Job Template 3: Restore NetworkPolicy

```yaml
# playbooks/restore-network-policy.yml
---
- name: Restore Deleted NetworkPolicy from Git Source of Truth
  hosts: localhost
  gather_facts: false

  vars:
    gitops_repo: "https://<your-gitops-repo>/namespaces/{{ namespace }}/networkpolicies"

  tasks:
    - name: Fetch NetworkPolicy definition from GitOps repo
      uri:
        url: "{{ gitops_repo }}/{{ policy_name }}.yaml"
        return_content: true
      register: policy_definition

    - name: Re-apply NetworkPolicy from Git
      kubernetes.core.k8s:
        state: present
        definition: "{{ policy_definition.content | from_yaml }}"
        namespace: "{{ namespace }}"

    - name: Annotate namespace with remediation event
      kubernetes.core.k8s:
        state: patched
        api_version: v1
        kind: Namespace
        name: "{{ namespace }}"
        definition:
          metadata:
            annotations:
              eda.ansible.com/last-remediated: "{{ ansible_date_time.iso8601 }}"
              eda.ansible.com/remediation-reason: "NetworkPolicy {{ policy_name }} was deleted and restored from Git"
```

> **💡 Tip**
> Replace `<your-gitops-repo>` with the actual URL of your GitOps repository where namespace manifests are stored. The playbook re-applies the policy from Git — maintaining Git as the single source of truth even during automated remediation.

---

## Step 4: Provision an EDA-Governed Medium Namespace

1. Log in to your Developer Portal as a developer user.

2. Select **Orchestrator** from the left-hand navigation menu.

3. Click **Run** on the **Create OpenShift Namespace** workflow.

4. Fill in the form as follows:

   | Field | Example Value |
   |---|---|
   | Namespace name | `<your-username>-medium` |
   | Issue Tracker Host | `<your-issue-tracker-hostname>` |
   | Requester | `<your-username>` |
   | Size | `medium` |
   | Reason | `EDA governance lab` |
   | Recipients | `user:default/<your-username>` |
   | **Enable EDA Governance** | ✅ **checked** |

5. Click **Next** — confirm **Enable EDA Governance** shows `true` in the review screen — then click **Run**.

6. Once the workflow completes, select **Notifications** and confirm the success notification was received.

### Verify the EDA Label Was Applied

1. Navigate to **Home → Projects** in the OpenShift Web Console and find your `<your-username>-medium` namespace.

2. Click the namespace and select the **YAML** tab.

3. Confirm the following label and annotation are present:

   ```yaml
   metadata:
     labels:
       eda-governed: "true"
     annotations:
       eda.ansible.com/watch-events: "PersistentVolumeClaim,Secret,NetworkPolicy"
   ```

---

## Step 5: Trigger and Verify EDA Remediation

### Scenario A: PVC Missing Storage Class Label

1. In the OpenShift Web Console, navigate to your `<your-username>-medium` namespace.

2. Click the **+** icon (Import YAML) and create a PVC without the required label:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: test-pvc
     namespace: <your-username>-medium
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
   ```

3. In your EDA Decision Controller, confirm the **PVC missing storage class label** rule fired.

4. In Ansible Automation Controller, confirm the **NaaS - Remediate PVC Labels** job ran successfully.

5. Return to the PVC in the OpenShift Web Console and verify:
   - Label `storage-class-approved: "true"` was applied
   - Annotation `eda.ansible.com/last-remediated` is present with a timestamp

---

### Scenario B: TLS Secret Missing Certificate Annotation

1. In your `<your-username>-medium` namespace, create a TLS Secret without the cert-manager annotation:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: my-app-tls
     namespace: <your-username>-medium
   type: kubernetes.io/tls
   data:
     tls.crt: ""
     tls.key: ""
   ```

2. Confirm the **TLS Secret missing certificate annotation** rule fired in EDA.

3. Confirm the **NaaS - Remediate TLS Certificate Annotation** job ran in AAC.

4. Verify the Secret now has:
   - Annotation `cert-manager.io/certificate-name: my-app` applied
   - Annotation `eda.ansible.com/last-remediated` present

---

### Scenario C: NetworkPolicy Deleted

1. In your `<your-username>-medium` namespace, navigate to **Networking → NetworkPolicies**.

2. Delete the default deny-all or ingress NetworkPolicy that was applied at provisioning time.

3. Confirm the **NetworkPolicy deleted** rule fired in EDA.

4. Confirm the **NaaS - Restore NetworkPolicy** job ran in AAC.

5. Navigate back to **Networking → NetworkPolicies** and verify the policy was restored from Git.

6. Check the namespace YAML — confirm `eda.ansible.com/last-remediated` was updated with the remediation reason.

---

## What You Have Built: The Complete Governance Loop

```
Git PR merged
      ↓
ArgoCD syncs → Namespace + ResourceQuota + NetworkPolicy + RBAC
      ↓
EDA label present → juniper.eda.k8s begins event polling
      ↓
Developer adds PVC without label
      ↓
EDA rule fires → AAC job patches PVC with required label
      ↓
Developer deletes NetworkPolicy
      ↓
EDA rule fires → AAC job restores policy from Git
      ↓
TLS Secret created without annotation
      ↓
EDA rule fires → AAC job applies cert-manager annotation
      ↓
Every remediation annotated on the resource → full audit trail in cluster + AAC
```

This is a **self-governing namespace** — it continuously enforces its own desired state without human intervention.

---

## EDA Rule Reference

| Rule | Trigger | AAC Job Template | Collection |
|---|---|---|---|
| PVC missing storage class label | `ADDED` PVC without `storage-class-approved` label | NaaS - Remediate PVC Labels | `kubernetes.core` |
| TLS Secret missing annotation | `ADDED` TLS Secret without `cert-manager.io/certificate-name` | NaaS - Remediate TLS Certificate Annotation | `kubernetes.core` |
| NetworkPolicy deleted | `DELETED` NetworkPolicy in governed namespace | NaaS - Restore NetworkPolicy | `kubernetes.core` |
| *(extendable)* ResourceQuota deleted | `DELETED` ResourceQuota | NaaS - Restore ResourceQuota | `kubernetes.core` |
| *(extendable)* Namespace idle >30d | TTL annotation expiry | NaaS - Open TTL Extension PR | `ansible.builtin` |

---

## Conclusion

In this section you provisioned a medium namespace with the **EDA Governance** option enabled and observed the full self-governing loop in action.

The EDA checkbox at provisioning time is a simple mechanism with powerful downstream effects:

- It applies a label that scopes EDA event polling to that specific namespace
- It activates a set of pre-defined remediation rules without any additional configuration
- Every remediation is traceable — annotated on the resource and logged in AAC

Combined with Phase 1 (GitOps provisioning) and Phase 2 (Orchestrator approval workflows), this completes the **Continuous Delivery Engine for the Real World**:

| Phase | Capability | Tool |
|---|---|---|
| Phase 1 | Self-service provisioning via GitOps | Developer Portal + ArgoCD |
| Phase 2 | Stateful workflows and approval gates | Orchestrator + SonataFlow |
| Phase 3 | Continuous event-driven governance | EDA + AAC + juniper.eda.k8s |

---

## Further Reading

- [juniper.eda.k8s collection](https://github.com/juniper/juniper.eda.k8s)
- [Event-Driven Ansible documentation](https://www.ansible.com/products/event-driven-ansible)
- [Ansible Automation Controller Job Templates](https://docs.ansible.com/automation-controller/latest/html/userguide/job_templates.html)
- [Kubernetes ResourceQuotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [cert-manager documentation](https://cert-manager.io/docs/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
