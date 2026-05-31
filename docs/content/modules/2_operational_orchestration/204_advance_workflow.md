# Workflows with State and Integrations

## Overview

In this section you will implement a stateful, real-world use case for developer self-service using the Orchestrator: **requesting a namespace on an OpenShift cluster**.

Namespaces have limited resources defined by [ResourceQuotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) and [LimitRanges](https://kubernetes.io/docs/concepts/policy/limit-range/). Namespaces come in three sizes: **small**, **medium**, and **large**.

To illustrate the approval capabilities of the Orchestrator plugin, requests for a **large** namespace require an approval step before the namespace is provisioned.

By the end of this section you will have:

- Deployed a production-ready stateful workflow
- Executed a simple (small/medium) namespace request end-to-end
- Tested the full approval flow for a large namespace request using an issue tracker

-----

## Part 1: Understanding the Workflow

The serverless workflow used in this section is stored in [redhat-ads-tech/orchestrator-self-service-workflows](https://github.com/redhat-ads-tech/orchestrator-self-service-workflows) on GitHub. The repository structure is as follows:

|Path                       |Purpose                                                                              |
|---------------------------|-------------------------------------------------------------------------------------|
|`create-ocp-namespace-swt/`|Workflow definition and dependencies — OpenAPI specs and JSON schemas                |
|`scripts/`                 |Build scripts for the container image and deployment manifest generation             |
|`charts/`                  |Helm Chart for deploying the workflow (numeric-prefixed templates are auto-generated)|
|`resources/`               |Dockerfile used to build the workflow container image                                |

GitHub Actions builds and releases new versions of the container image and Helm Chart whenever the version in `create-ocp-namespace-swt/pom.xml` is updated.

-----

### Workflow Functions

View the [serverless workflow file: create-ocp-namespace-swt.sw.yaml](https://github.com/redhat-ads-tech/orchestrator-self-service-workflows/blob/main/create-ocp-namespace-swt/src/main/resources/create-ocp-namespace-swt.sw.yaml).

The workflow defines a set of **functions** used to:

- Interact with a Git issue tracker via its REST API — see `specs/gitlab-openapi.yaml`
- Run Software Templates and send notifications to Developer Portal users — see `specs/notifications-openapi.yaml`
- Evaluate results and perform data transformation using [jq expressions](https://jqlang.org/) for final workflow output — note `result.message` matching `workflow-output-schema.json`

-----

### Workflow States

A serverless workflow is a [Finite State Machine](https://en.wikipedia.org/wiki/Finite-state_machine) — a set of states and rules for transitioning between them.

The workflow starts at `NamespaceSizeSwitch`, which routes to:

- `LaunchSoftwareTemplate` — for small/medium namespaces (no approval needed)
- `GetIssueTrackerProjectId` — for large namespaces (triggers the approval flow)

For a **large** namespace, the first transition calls `getProjectId` and stores the result for downstream states:

```yaml
- name: GetIssueTrackerProjectId
  actionMode: sequential
  actions:
    - name: GetIssueTrackerProjectId
      actionDataFilter:
        toStateData: .getProjectIdResult
        useResults: true
      functionRef:
        arguments:
          search: orchestrator-approvals
          simple: true
        invoke: sync
        refName: getProjectId
  transition:
    nextState: CreateIssue
```

The `actionDataFilter` tells the workflow to update overall state (`useResults: true`) and store the result under `getProjectIdResult`.

The workflow then creates an approval issue in the issue tracker and continuously polls between **PollIssue** and **CheckIssueStatus** states until an `approved` or `denied` label is detected. **PollIssue** sleeps for 60 seconds between each poll (`PT60S` in ISO 8601 duration format).

-----

### Workflow Output

When a terminal state is reached (e.g. **CloseIssueDenied**), it sets `end.terminate: true`:

```yaml
- name: CloseIssueDenied
  type: operation
  actionMode: sequential
  actions:
    - actionDataFilter:
        useResults: true
      functionRef:
        invoke: sync
        refName: errorDeniedResult
      name: setOutput
    - actionDataFilter:
        toStateData: .closeIssueResult
        useResults: true
      functionRef:
        arguments:
          id: .getProjectIdResult[0].id|tostring
          issue_iid: .createIssueResult.iid
          state_event: close
        invoke: sync
        refName: closeIssue
      name: CloseIssue
  end:
    terminate: true
  metadata:
    errorMessage: '"Creation of namespace " + .namespace + " denied"'
```

The `errorDeniedResult` expression sets `result.message` to “Creation of namespace <name> denied” and attaches the issue link in the `outputs` block.

-----

## Part 2: Configure Your Developer Portal

### Step 1: Enable the Notifications Plugin

1. Navigate to the Developer Portal namespace in the OpenShift Web Console.
1. Open the YAML editor for the dynamic plugins ConfigMap (e.g. `<your-rhdh-instance>-dynamic-plugins`).
1. Verify these notification plugins are present and set to `disabled: false`:
   
   ```yaml
   - disabled: false
     package: ./dynamic-plugins/dist/backstage-plugin-notifications
   - disabled: false
     package: ./dynamic-plugins/dist/backstage-plugin-notifications-backend-dynamic
   ```
1. Save the ConfigMap.

### Step 2: Enable Token-Based API Access

The workflow calls the Developer Portal backend API using [static token authentication](https://backstage.io/docs/auth/service-to-service-auth/#static-tokens).

1. Open the YAML editor for the app config ConfigMap (e.g. `<your-rhdh-instance>-app-config`).
1. Add `externalAccess` to the `backend.auth` section — do not remove existing properties:
   
   ```yaml
   backend:
     auth:
       externalAccess:
         - type: static
           options:
             token: ${BACKEND_SECRET}
             subject: Orchestrator
   ```
1. Save the ConfigMap.
1. Wait for the Developer Portal pod to redeploy and reach Running status.

> **ℹ️ Note**
> `BACKEND_SECRET` is typically already used as a session signing secret. In production, provision a dedicated secret with a securely generated token:
> 
> ```bash
> openssl rand 24 | base64 | cut -c1-32
> ```

-----

## Part 3: Import the Namespace Software Template

The workflow triggers a Software Template to provision the namespace. Import it before deploying the workflow.

1. Log in to your Developer Portal as an administrator.
1. Click the **+** (Create) icon in the top navigation bar.
1. Click **Import an existing Git repository**.
1. Enter the following URL and click **Analyze**:
   
   ```
   https://github.com/redhat-ads-tech/orchestrator-self-service-templates/blob/main/namespace/template.yaml
   ```
1. Review the entities to import and click **Import**.
1. Verify the import by clicking **+** again and filtering by the `orchestrator` tag. You should see the **OpenShift Namespace Request** template.

> **ℹ️ Note**
> This template is not intended to be used directly — it is invoked by the Orchestrator workflow. In production, use RBAC rules to hide it from the self-service catalog. It is annotated with `backstage.io/managed-by: orchestrator` and tagged `orchestrator`.

-----

## Part 4: Deploy the Advanced Workflow

### Step 1: Gather Prerequisites

Collect the credentials the workflow needs. Run these commands in a terminal with an active `oc` session, replacing secret and namespace names with your own:

```bash
# Retrieve the Developer Portal backend token
export BACKSTAGE_TOKEN=$(oc get secret <your-rhdh-env-secret> -n <your-portal-namespace> \
  -o jsonpath='{.data.BACKEND_SECRET}' | base64 -d)

# Retrieve the issue tracker token (e.g. GitLab personal access token)
export ISSUE_TRACKER_TOKEN=$(oc get secret <your-issue-tracker-secret> -n <your-issue-tracker-namespace> \
  -o jsonpath='{.data.token}' | base64 -d)
```

> **⚠️ Warning**
> In production, create dedicated tokens with the minimum required permissions and store them in OpenShift Secrets.

### Step 2: Install via Helm

1. Open the OpenShift Web Terminal (**>_**) or use a local terminal with `oc` and `helm`.
1. Set your project context:
   
   ```bash
   oc project <your-developer-portal-namespace>
   ```
1. Add the Helm repository:
   
   ```bash
   helm repo add advanced-workflows https://redhat-ads-tech.github.io/orchestrator-self-service-workflows/
   ```
1. Install the workflow with your environment-specific values:
   
   ```bash
   helm install request-ns advanced-workflows/create-ocp-namespace-swt \
     -n <your-developer-portal-namespace> \
     --set env.backstageBackendUrl="https://<your-developer-portal-url>" \
     --set env.backstageBackendBearerToken="$BACKSTAGE_TOKEN" \
     --set env.gitlabUrl="https://<your-issue-tracker-url>" \
     --set env.gitlabToken="$ISSUE_TRACKER_TOKEN"
   ```

> **ℹ️ Note**
> Replace `<your-developer-portal-url>` and `<your-issue-tracker-url>` with your environment’s actual hostnames. If using a different issue tracker (e.g. Jira, ServiceNow), update the Helm values accordingly.
1. Wait for the **create-ocp-namespace-swt** pod to appear in the Topology view and reach Running status.

-----

## Part 5: Execute the Workflow — Small Namespace

1. Log in to your Developer Portal as a developer user.
1. Select **Orchestrator** from the left-hand navigation menu.
1. Click the **Create OpenShift Namespace** workflow.

> **ℹ️ Note**
> If the workflow is not listed, delete the Developer Portal pod to force a refresh.
1. Click **Run** (top-right) to start a workflow instance.
1. Fill in the form for a small namespace:
   
   |Field             |Example Value                  |
   |------------------|-------------------------------|
   |Namespace name    |`<your-username>-small`        |
   |Issue Tracker Host|`<your-issue-tracker-hostname>`|
   |Requester         |`<your-username>`              |
   |Size              |`small`                        |
   |Reason            |(optional)                     |
   |Recipients        |`user:default/<your-username>` |
1. Click **Next**, then **Run**.
1. After a few seconds the status should update to **Run completed**.

### Verify the Result

1. Select **Notifications** from the left-hand menu. Confirm a notification was received showing the namespace was created.
1. Click the notification link to view the new component in the Developer Portal catalog.
1. In the OpenShift Web Console, navigate to **Home → Projects** and find your namespace.
1. Click **Administration → ResourceQuotas** and **Administration → LimitRanges** to confirm governance policies were applied.

-----

## Part 6: Test the Approval Flow — Large Namespace

1. Return to **Orchestrator** in your Developer Portal.
1. Click **Run** on the **Create OpenShift Namespace** workflow.
1. Fill in the form for a large namespace:
   
   |Field             |Example Value                         |
   |------------------|--------------------------------------|
   |Namespace name    |`<your-username>-large`               |
   |Issue Tracker Host|`<your-issue-tracker-hostname>`       |
   |Requester         |`<your-username>`                     |
   |Size              |`large`                               |
   |Reason            |`Required for a production deployment`|
   |Recipients        |`user:default/<your-username>`        |
1. Click **Next** and **Run**.
1. Select **Notifications**. After a few seconds you should see a notification that an approval issue was created.
1. Click the notification link to open the issue in your issue tracker.

### Approve the Request

1. Log in to your issue tracker as an approver.
1. Locate the approval issue opened by the workflow.
1. Add the **Approved** label to the issue.

> **ℹ️ Note**
> The workflow polls for label changes every 60 seconds. Once `approved` is detected, it automatically provisions the namespace and closes the issue.
1. Return to your Developer Portal. After one or two polling cycles, a notification should confirm the namespace was created.
1. Verify the namespace exists in OpenShift and that the issue was automatically closed.

### Test a Denied Request

1. Run the workflow again for a new large namespace (e.g. `<your-username>-large-2`).
1. Open the approval issue and apply the **Denied** label.
1. Return to your Developer Portal. You should receive a denial notification and **no namespace should be created** in OpenShift.

### Verify Software Template Task History

1. Navigate to **Create → Task List** (or `/create/tasks`) in your Developer Portal.
1. Apply the **All** filter.
1. You should see the **OpenShift Namespace Request** template invocations triggered by the Orchestrator — one per approved workflow run.

-----

## Workflow State Diagram

```
Start
  ↓
NamespaceSizeSwitch
  ├── small/medium ──→ LaunchSoftwareTemplate ──→ NotifySuccess/Failure ──→ End
  │
  └── large ──→ GetIssueTrackerProjectId
                    ↓
               CreateIssue (opens approval ticket)
                    ↓
               PollIssue (waits 60s)
                    ↓
               CheckIssueStatus
                    ├── approved ──→ CloseIssueApproved ──→ LaunchSoftwareTemplate ──→ NotifySuccess ──→ End
                    ├── denied   ──→ CloseIssueDenied   ──→ NotifyDenied ──→ End
                    └── pending  ──→ PollIssue (loop)
```

-----

## Key Concepts Demonstrated

|Concept                       |How It’s Implemented                                                                   |
|------------------------------|---------------------------------------------------------------------------------------|
|Stateful long-running process |Workflow polls an issue tracker over minutes/hours using `PT60S` sleep states          |
|Human-in-the-loop approval    |Approver applies a label to an issue; workflow detects it on next poll                 |
|External system integration   |Issue tracker REST API called via OpenAPI spec functions                               |
|Developer Portal notifications|Backend API called by workflow at key milestones                                       |
|Software Template invocation  |Workflow triggers a template programmatically — developer never fills the form directly|
|Governance via resource quotas|Each namespace size tier maps to predefined ResourceQuota and LimitRange values        |
|Audit trail                   |Every workflow instance tracked in the Developer Portal with full state history        |

-----

## Conclusion

In this module you learned how the Orchestrator plugin, combined with Serverless Workflow on OpenShift, enables complex self-service workflows beyond what Software Templates can do natively.

The namespace request workflow demonstrated:

- **Conditional logic** — different paths for different namespace sizes
- **Stateful waiting** — polling an external system until an approval decision is made
- **External integrations** — calling an issue tracker API and the Developer Portal notifications API
- **End-to-end automation** — from developer form submission to a provisioned, governed namespace

This pattern can be adapted to any approval-gated self-service request in your organisation — quota upgrades, environment promotions, access requests, and beyond.

-----

## Further Reading

- [Orchestrator project documentation](https://www.rhdhorchestrator.io/)
- [orchestrator-self-service-workflows repository](https://github.com/redhat-ads-tech/orchestrator-self-service-workflows)
- [orchestrator-self-service-templates repository](https://github.com/redhat-ads-tech/orchestrator-self-service-templates)
- [SonataFlow GitOps deployment profile](https://sonataflow.org/serverlessworkflow/main/cloud/operator/gitops-profile.html)
- [Backstage static token authentication](https://backstage.io/docs/auth/service-to-service-auth/#static-tokens)
- [Backstage Notifications plugin](https://backstage.io/docs/notifications/)
- [Kubernetes ResourceQuotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
