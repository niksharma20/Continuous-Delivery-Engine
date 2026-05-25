# Orchestrator Installation and Setup

## Overview

Since the Orchestrator plugin for your Developer Portal relies on Serverless Workflow, it requires the **OpenShift Serverless Logic Operator** to provide the underlying primitives.

This Operator provides the necessary [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) for **SonataFlow**, which the Orchestrator uses to facilitate the execution of custom workflows.

-----

## Step 1: Install the OpenShift Serverless Logic Operator

Confirm whether the Operator is already installed by visiting the **Installed Operators** section of the OpenShift Web Console.

> **ℹ️ Note**
> You must be logged in as a cluster administrator to manage Operators. Ensure you have the appropriate credentials before proceeding.

If the Operator is **not** installed, install it via OperatorHub:

1. Navigate to **OperatorHub** in the OpenShift Web Console.
1. Search for **OpenShift Serverless Logic** and select the **OpenShift Serverless Logic Operator** from the Red Hat catalog.
1. Select the appropriate version (e.g. `1.36.0` or the latest stable release available in your environment).
1. On the install form:
- Keep the default namespace (`openshift-serverless-logic` or as recommended)
- Set **Update Approval** to `Automatic`
1. Click **Install** and wait for the installation to complete.
1. Confirm the Operator status shows **Succeeded** in the Installed Operators list.

-----

## Step 2: Enable the Orchestrator Plugins in Your Developer Portal

The Orchestrator requires the following dynamic plugins to be enabled in your Developer Portal configuration:

|Plugin                                                                   |Purpose                                                                |
|-------------------------------------------------------------------------|-----------------------------------------------------------------------|
|`@redhat/backstage-plugin-orchestrator`                                  |Frontend UI — workflow listing and execution                           |
|`@redhat/backstage-plugin-orchestrator-backend-dynamic`                  |Backend — connects to SonataFlow; manages workflow state               |
|`@redhat/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic`|Scaffolder integration — allows Software Templates to trigger workflows|
|`@redhat/backstage-plugin-orchestrator-form-widgets`                     |UI form components for workflow input rendering                        |

### Locate the Dynamic Plugins ConfigMap

1. Open the OpenShift Web Console and navigate to **Workloads → ConfigMaps**.
1. Select the namespace where your Developer Portal is deployed.
1. Find and open the dynamic plugins ConfigMap (typically named `<your-rhdh-instance>-dynamic-plugins` or similar).

### Verify Plugin Configuration

1. Review the `plugins` array in the ConfigMap YAML and confirm all four Orchestrator plugins listed above are present.
1. Inspect the `@redhat/backstage-plugin-orchestrator-backend-dynamic` entry. It should include a `dependencies` property pointing to SonataFlow:
   
   ```yaml
   - package: '@redhat/backstage-plugin-orchestrator-backend-dynamic'
     disabled: false
     dependencies:
       - ref: sonataflow
   ```

> **ℹ️ Note**
> The `dependencies.ref: sonataflow` key instructs the Developer Portal Operator to automatically provision a **SonataFlowPlatform** Custom Resource in the same namespace. This is what connects your portal to the workflow engine.
1. Inspect the `@redhat/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic` entry. It should include a `pluginConfig` section specifying the Data Index Service URL:
   
   ```yaml
   - package: '@redhat/backstage-plugin-scaffolder-backend-module-orchestrator-dynamic'
     disabled: false
     pluginConfig:
       orchestrator:
         dataIndexService:
           url: http://sonataflow-platform-data-index-service
   ```

> **ℹ️ Note**
> The `dataIndexService.url` tells the plugin where to fetch available workflows from. The default URL assumes SonataFlow is deployed in the same namespace as the Developer Portal. If you deploy SonataFlow in a separate namespace or cluster, update this URL accordingly.

### Enable the Plugins

1. If any of the Orchestrator plugins are set to `disabled: true`, change them to `disabled: false`.
1. Save the ConfigMap.
1. The Developer Portal Operator will detect the change and reconcile — expect the pods to restart. Wait for all pods to return to a healthy (Running) state before proceeding.

-----

## Step 3: Verify the SonataFlowPlatform

Once the plugins are enabled and the pods have stabilised, confirm that the **SonataFlowPlatform** Custom Resource was created and is healthy.

### Check the Topology View

1. In the OpenShift Web Console, navigate to the **Topology** view for your Developer Portal namespace.
1. Wait for the SonataFlow pods to show a dark blue ring (Running/Ready). This may take a few minutes as the SonataFlow components depend on the portal being fully initialised with the newly enabled plugins.
1. You should see a **SonataFlowPlatform** (abbreviated **SFP**) deployment in the topology.

### Inspect the SonataFlowPlatform CR

1. Click the three-dot menu next to the `sonataflow-platform` resource and select **Edit SonataFlowPlatform**.
1. In the YAML view, confirm the following:
   
   **`ownerReferences`** — Should reference your Developer Portal instance. This confirms the portal is managing the SonataFlowPlatform lifecycle.
   
   **`spec.services`** — Should show a database connection configuration. SonataFlow uses your Developer Portal’s PostgreSQL database to manage and persist workflow state across executions.
   
   ```yaml
   spec:
     services:
       dataIndex:
         persistence:
           postgresql:
             secretRef:
               name: <your-db-secret>
               userKey: POSTGRESQL_USER
               passwordKey: POSTGRESQL_PASSWORD
             serviceRef:
               name: <your-postgresql-service>
               port: 5432
               databaseName: sonataflow
   ```

> **💡 Tip**
> Adjust the `secretRef` and `serviceRef` values to match your environment’s PostgreSQL deployment. If your Developer Portal was installed via the RHDH Operator, the PostgreSQL instance and secret are typically provisioned automatically.

### Verify Data Index Service

1. Navigate to **Workloads → Pods** in the Developer Portal namespace.
1. Confirm the following pods are Running:
- `sonataflow-platform-data-index-service-*`
- `sonataflow-platform-jobs-service-*`
1. If pods are in a `CrashLoopBackOff` or `Pending` state, check Events and Logs for database connectivity issues.

-----

## Troubleshooting

|Symptom                                        |Likely Cause / Resolution                                                                                                    |
|-----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
|Orchestrator menu not visible in portal        |Plugins still disabled in ConfigMap — re-check `disabled: false` for all four plugins                                        |
|SonataFlowPlatform pods not starting           |Operator not installed or database secret missing — verify Operator status and PostgreSQL connectivity                       |
|Workflows not appearing in portal UI           |`dataIndexService.url` misconfigured — confirm the Data Index pod is running and the URL is reachable from the backend plugin|
|Portal pods restarting after enabling plugins  |Normal during reconciliation — wait 3–5 minutes for stabilisation                                                            |
|`ownerReferences` missing on SonataFlowPlatform|`dependencies.ref: sonataflow` not present in backend plugin config — re-check ConfigMap                                     |

-----

## Conclusion

You have now:

- Installed the **OpenShift Serverless Logic Operator** to provide SonataFlow Custom Resources
- Enabled and configured the **four Orchestrator dynamic plugins** in your Developer Portal
- Verified that the **SonataFlowPlatform** is healthy and connected to your portal’s database

With the Orchestrator installed, your Developer Portal is now capable of hosting, managing, and executing stateful, long-running workflows. The next step is to build and deploy your first workflow.

-----

## Further Reading

- [Orchestrator project documentation](https://www.rhdhorchestrator.io/)
- [SonataFlow documentation](https://sonataflow.org/)
- [Serverless Workflow specification](https://serverlessworkflow.io/)
- [Red Hat Developer Hub documentation](https://docs.redhat.com/en/documentation/red_hat_developer_hub)
- [OpenShift Serverless documentation](https://docs.redhat.com/en/documentation/openshift_serverless)
