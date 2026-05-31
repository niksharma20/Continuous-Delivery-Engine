# Namespace as a Service: Self-Service Provisioning with GitOps

## Introduction

In this exercise you will learn how to work with Software Templates in a Developer Portal (such as Red Hat Developer Hub or Backstage) and create a new OpenShift Project (also referred to as a **namespace**) with custom configurations via a self-service workflow.

This lab uses an existing Software Template to create OpenShift Projects. You will work on increasing functionality by updating the template to enforce resource quotas — a key governance control in a Namespace as a Service (NaaS) platform.

During this lab, you will explore different project customizations and understand how GitOps maintains the desired state across environments.

-----

## Scenario

Imagine you need to fulfill a request from a development team — for example, a Python or AI prototyping team — who needs the ability to create new namespaces to start prototyping, outside of the standard Software Development Lifecycle. These namespaces should be:

- Isolated from the rest of the cluster
- Subject to resource limits to prevent runaway consumption
- Provisioned without manual platform team intervention
- Sufficient in resources for experimentation

This is the self-service namespace pattern in action.

-----

## Explore the Project Template in Your SCM

1. Navigate to your Software Configuration Management (SCM) system (e.g. GitLab, GitHub, Gitea) and locate the namespace Software Template repository.
1. Review the following key files to understand the template structure:

**`template.yaml`**
Defines the UI experience for the end user in the Developer Portal — fields, dropdowns, and selectors to configure the namespace. Also defines the programmatic steps executed during component creation: creating repositories, creating GitOps objects, and more.

**`skeleton/catalog-info.yaml`**
A descriptor file used to define and register entities in the Developer Portal. Contains information displayed in the component overview (source code links, tech docs, etc.). Placeholders (e.g., `${{ values.component_id }}`) are replaced with actual values when the template is used.

**`manifests/argocd/`**
Contains the Argo CD Application definitions and secrets required to read from the SCM. The `argo-app-<env>.yaml` file is the Argo CD Application pointing to the Kubernetes manifest folder. GitOps maintains the desired state defined in these manifests as the actual state in the cluster.

**`manifests/helm/app/`**
Contains a Helm chart used to create the namespace. OpenShift GitOps uses this Helm chart to provision the namespace by combining user-supplied values with template files (e.g. `quota.yaml`). When these files are updated in the user’s Git repo, OpenShift GitOps automatically syncs the changes.

-----

## Explore the Project Template in the Developer Portal

1. Access your Developer Portal instance.
1. Select the **Create** icon (plus icon or **Create** button) in the top navigation to create a new component.
1. Click **Import an existing Git repository**.
1. Enter the URL of your `template.yaml` file in the **Select URL** field and click **Analyze**.
1. Click **Import**.

> **ℹ️ Note**
> You now have a new Software Template available in your Developer Portal. End users can self-provision namespaces without filing a ticket or contacting the platform team.

### Explore the End-User Experience

1. Navigate to **Catalog → Self-service** in the Developer Portal.
1. Locate the namespace template (e.g., **OpenShift Project Medium Size** or your equivalent template name).
1. Click **Choose**.
1. Review and fill out the form with sample data until you reach the review screen.

> **⚠️ Important**
> **Do not click Create yet.** We will complete the provisioning after updating the template in the next section.

-----

## Implement Changes in the Software Template

To enforce governance, we will update the `quota.yaml` file to set resource limits on the namespace, preventing teams from accidentally consuming excessive cluster resources.

### Edit the Quota Template

1. In your SCM, navigate to the template repository and open:
   
   ```
   manifests/helm/app/templates/quota.yaml
   ```
1. Edit the file and update the `ResourceQuota` limits. For example:
   
   ```yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: {{ .Values.namespace }}-quota
   spec:
     hard:
       limits.cpu: "2"
       limits.memory: 2Gi
       requests.cpu: "1"
       requests.memory: 1Gi
       pods: "10"
   ```
   
   Adjust values to match your organization’s sizing standards for the tier (small / medium / large).
1. Commit your changes with a meaningful message, e.g.:
   
   ```
   chore: enforce resource quota limits for medium namespace tier
   ```
1. Return to the Developer Portal and trigger a **Schedule entity refresh** on the template to sync the catalog with your latest changes.

-----

## Test Your Changes: Provision a Namespace as a Developer

Now you will create a new Project using the updated Software Template.

1. From **Catalog**, select **Self-service**, then choose your namespace template.
1. Click **Choose**.
1. Complete the form fields:
   
   |Field         |Description                                              |
   |--------------|---------------------------------------------------------|
   |Namespace Name|A unique name for the project (e.g., `team-payments-dev`)|
   |Owner / Team  |The team or user owning this namespace                   |
   |Environment   |Target environment (dev / staging / prod)                |
   |Quota Tier    |The resource tier to apply (small / medium / large)      |
1. Review the summary and click **Create**.
1. Once provisioning completes, click **Open Component in Catalog** to view the new component.

### Verify the Namespace Was Created

1. Log in to the OpenShift Web Console.
1. Navigate to **Home → Projects**.
1. Search for the namespace name you created.
1. Select the namespace and scroll to **Resource Quotas**.
1. Click the quota name and confirm the limits you configured are applied.

> **💡 Tip**
> If the quota values do not reflect your changes, confirm that ArgoCD has completed its sync. Check the ArgoCD Application for any sync errors.

-----

## Architecture Overview

The following describes how each layer contributes to the self-service flow:

```
Developer submits form in Developer Portal
          ↓
Software Template creates Git repository with manifests
          ↓
ArgoCD detects new Argo Application → syncs Helm chart
          ↓
OpenShift creates Namespace + ResourceQuota + NetworkPolicy + RBAC
          ↓
Namespace is registered in the Developer Portal catalog
```

### Key Components

|Component                          |Role                                                     |
|-----------------------------------|---------------------------------------------------------|
|Developer Portal (RHDH / Backstage)|Self-service UI, template catalog, component registry    |
|Software Template (`template.yaml`)|Defines provisioning workflow and UI form                |
|Git / SCM                          |Source of truth for namespace manifests                  |
|ArgoCD / OpenShift GitOps          |Syncs desired state from Git to the cluster              |
|Helm Chart (`quota.yaml`, etc.)    |Renders namespace resources (quota, RBAC, network policy)|
|OpenShift                          |Target platform where namespaces are created             |

-----

## Extending This Pattern

This lab covers Phase 1 of a Namespace as a Service platform. Consider these extensions as your platform matures:

|Phase               |Extension                                                                                                                  |
|--------------------|---------------------------------------------------------------------------------------------------------------------------|
|Phase 1 *(this lab)*|Self-service namespace provisioning via Developer Portal + GitOps                                                          |
|Phase 2             |Automated side-effects: Vault secrets binding, Quay registry quotas, ITSM ticket creation via Ansible Automation Controller|
|Phase 3             |Event-driven governance: TTL enforcement, quota alerts, RBAC drift remediation via Ansible EDA (Event-Driven Ansible)      |

-----

## Conclusion

🎉 **Congratulations!** You have successfully implemented a self-service namespace provisioning workflow using Software Templates and GitOps.

In this lab you accomplished the following:

- **Enabled self-service at scale** — Teams can now instantly provision isolated namespaces without waiting for manual platform team intervention.
- **Implemented governance through code** — By updating the quota template, resource governance is embedded directly into the provisioning process.
- **Bridged development and operations** — Developers interact only with a form; GitOps handles the underlying infrastructure automatically.
- **Created a reusable, version-controlled asset** — The template can be refined, extended, and replicated across environments and teams.

This is the **Platform Engineering** model in action: turning complex infrastructure operations into simple, self-service experiences that accelerate innovation while maintaining control.

-----

## Further Reading

- [OpenShift Resource Quotas documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/building_applications/quotas)
- [ArgoCD documentation](https://argo-cd.readthedocs.io/en/stable/)
- [Backstage Software Templates documentation](https://backstage.io/docs/features/software-templates/)
- [Red Hat Developer Hub documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub)
