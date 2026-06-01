# Developing, Managing, and Running Workflows

## Overview

In this section you will learn how to:

- Develop a workflow on OpenShift using the SonataFlow development profile
- Deploy a production-ready workflow via Helm using a pre-built example
- Run a workflow using the Orchestrator feature in your Developer Portal

-----

## Part 1: Developing Workflows on OpenShift

It is impractical to cover all aspects of SonataFlow and Serverless Workflow development in this module. However, you will get hands-on with a basic example that demonstrates an Operator-based development flow. After this, you will learn how to make the workflow available in your Developer Portal.

> **ℹ️ Note**
> To learn about developing workflows locally using the `kn` CLI and the `workflow` extension, see the [relevant SonataFlow documentation](https://sonataflow.org/serverlessworkflow/1.45.0-SNAPSHOT/getting-started/create-your-first-workflow-service-with-kn-cli-and-vscode.html#proc-creating-app-with-kn-cli).

-----

### Step 1: Create a Workflow in Dev Mode

1. Open the OpenShift Web Console and navigate to the namespace where your Developer Portal is deployed.
1. Click the plus (**+**) icon in the top navigation bar and choose **Import YAML**.
1. Paste the following YAML and click **Create**:
   
   ```yaml
   apiVersion: sonataflow.org/v1alpha08
   kind: SonataFlow
   metadata:
     name: greeting
     annotations:
       # Dev profile runs the Quarkus server in development mode,
       # enabling the Dev UI for testing and debugging
       sonataflow.org/profile: dev
       sonataflow.org/description: Greeting example on OpenShift!
       sonataflow.org/version: 0.0.1
       app.kubernetes.io/part-of: sonataflow-platform
   spec:
     flow:
       start: ChooseOnLanguage
       functions:
         - name: greetFunction
           type: custom
           operation: sysout
       states:
         - name: ChooseOnLanguage
           type: switch
           dataConditions:
             - condition: "${ .language == \"English\" }"
               transition: GreetInEnglish
             - condition: "${ .language == \"Spanish\" }"
               transition: GreetInSpanish
           defaultCondition: GreetInEnglish
         - name: GreetInEnglish
           type: inject
           data:
             greeting: "Hello from JSON Workflow, "
           transition: GreetPerson
         - name: GreetInSpanish
           type: inject
           data:
             greeting: "Saludos desde JSON Workflow, "
           transition: GreetPerson
         - name: GreetPerson
           type: operation
           actions:
             - name: greetAction
               functionRef:
                 refName: greetFunction
                 arguments:
                   message: ".greeting+.name"
           end: true
   ```

> **ℹ️ Note**
> The `namespace` field has been intentionally omitted — set it to your target namespace before applying, or rely on your current `oc project` context.
1. A new **SonataFlow** resource will appear in the Topology view. Click it and wait for the Pod to enter **Running** status.
1. Once running, click the route arrow next to the Pod in the Topology view. Append `/q/dev-ui` to the URL to access the [Quarkus Dev UI](https://quarkus.io/guides/dev-ui).

> **ℹ️ Note**
> Quarkus is the application framework that exposes REST endpoints, performs input validation, and provides extensions such as `sysout`. The Serverless Logic Operator uses [kogito-codegen](https://mvnrepository.com/artifact/org.kie.kogito/kogito-codegen) to generate Java code from your workflow definition and JSON schemas.

The key element in the `SonataFlow` CR above is the `sonataflow.org/profile: dev` annotation. This instructs the Serverless Logic Operator to start the Quarkus application in development mode, which is what makes the Quarkus Dev UI available.

-----

### Step 2: Experiment with the Quarkus Dev UI

1. In the Quarkus Dev UI, click the **Workflows** link inside the **Serverless Workflows Tools** box.
1. Click the **Workflow Definitions** tab. You should see the *greeting* workflow listed.
1. Click the **play** button under the **Actions** column for the **greeting** workflow.
1. Enter the following JSON in the **Start Workflow Data** field and click **Start**:
   
   ```json
   { "language": "English" }
   ```

> **💡 Tip**
> Click the up arrow (^) at the bottom of the Quarkus Dev UI to expand the application log pane. This provides additional visibility into workflow execution.
1. Navigate to the **Workflow Instances** tab. Your completed workflow run will be listed.
1. Click the **greeting** link to view the run details. Confirm the English branch was executed and that `"Hello from JSON Workflow"` appears in the `workflowdata` output.

-----

### Step 3: Edit the Workflow Live

One of the key benefits of dev mode is live reloading — changes to the `SonataFlow` CR are picked up without redeploying.

1. Return to the OpenShift Web Console Topology view for your Developer Portal namespace.
1. Click the three-dot menu on the **SF** (SonataFlow) item and select **Edit SonataFlow**.
1. Switch to the YAML editor and make the following changes:
- In `ChooseOnLanguage` conditions, update the Spanish condition to check for `"Irish"` and set `transition` to `GreetInIrish`
- Rename the `GreetInSpanish` state to `GreetInIrish`
- Update `data.greeting` to: `Háigh ó JSON workflow,`
1. Click **Save**.
1. Open the Pod logs for the **greeting** Pod. You should see Quarkus restarting (the Quarkus logo printed in the logs) — this confirms live reload was triggered.
1. Return to the Quarkus Dev UI and test the workflow again, passing `"Irish"` as the language. Observe that the Irish greeting branch is now executed.

-----

## Part 2: Integrate a Workflow with the Orchestrator

When it is time to deploy a production-ready workflow, you need to:

1. Build the workflow into a container image
1. Run it using the `gitops` SonataFlow profile

This is outlined in the [SonataFlow Deployment Profiles Guide](https://sonataflow.org/serverlessworkflow/main/cloud/operator/gitops-profile.html). In this section you will use a pre-built image to save time.

> **ℹ️ Note**
> The source code and build scripts for the sample workflow used in this section can be found in [redhat-ads-tech/orchestrator-workflows](https://github.com/redhat-ads-tech/orchestrator-workflows) on GitHub.

-----

### Step 1: Remove the Development Workflow

Before deploying the production version, delete the development `SonataFlow` resource:

1. Navigate to the Topology view of your Developer Portal namespace.
1. Click the three-dot menu on the **SF** item and select **Delete SonataFlow**.
1. Confirm the deletion.

-----

### Step 2: Deploy the Production Workflow via Helm

Use the OpenShift Web Terminal or any terminal with `oc` and `helm` access:

1. Open the Web Terminal (**>_** icon in the top navigation) or use your local terminal with an active `oc` session.
1. Set your project context to the Developer Portal namespace:
   
   ```bash
   oc project <your-developer-portal-namespace>
   ```
1. Add the Helm repository containing the sample workflows:
   
   ```bash
   helm repo add workflows https://redhat-ads-tech.github.io/orchestrator-workflows/
   ```
1. Install the greeting workflow:
   
   ```bash
   helm install greeting-workflow workflows/greeting -n <your-developer-portal-namespace>
   ```
1. The new **greeting** service will appear in the Topology view. Wait for the Pod to reach Running status.

-----

### Step 3: Verify the Workflow Appears in Your Developer Portal

1. Log in to your Developer Portal instance.
1. Navigate to the **Orchestrator** section (typically accessible via the left navigation).
1. Confirm that the **Greeting workflow** is now listed.

> **ℹ️ Note**
> If the greeting workflow does not appear, delete the Developer Portal Pod to force a cache refresh. If the issue persists, check the backend plugin logs and verify all Pods are healthy.

-----

### Step 4: Run the Workflow from the Developer Portal

1. Click the **play** button on the Greeting workflow listing.
1. Select a language when prompted by the input form.
1. Click **Next**, review the parameters, then click **Run**.
1. A workflow details page will display the result — including the **Greeting Message** determined by your chosen language.

> **ℹ️ Note**
> The input form is automatically generated from the workflow’s `dataInputSchema`. The schema files referenced by the workflow definition drive the form rendering in the Developer Portal — no manual UI configuration required.

-----

## How It Works: Production vs Development Profiles

|Aspect           |Dev Profile                     |GitOps (Production) Profile                   |
|-----------------|--------------------------------|----------------------------------------------|
|Annotation       |`sonataflow.org/profile: dev`   |`sonataflow.org/profile: gitops`              |
|How it runs      |Quarkus in dev mode; live reload|Pre-built container image deployed by Operator|
|Dev UI available |Yes (`/q/dev-ui`)               |No                                            |
|State persistence|In-memory                       |PostgreSQL via SonataFlowPlatform             |
|Use case         |Local iteration and debugging   |Stable, versioned production deployments      |
|Deployment method|Apply CR directly               |Helm chart / GitOps pipeline                  |

-----

## Workflow Lifecycle Summary

```
Write workflow YAML
        ↓
Apply SonataFlow CR (dev profile) → Test in Quarkus Dev UI
        ↓
Iterate on workflow logic (live reload)
        ↓
Build container image (gitops profile)
        ↓
Package as Helm chart → Deploy to OpenShift
        ↓
Workflow appears in Developer Portal via Orchestrator plugin
        ↓
Developers run workflows self-service via portal UI
```

-----

## Conclusion

In this section you:

- Created and deployed a **SonataFlow** workflow in development mode on OpenShift
- Used the **Quarkus Dev UI** to test and iterate on the workflow with live reload
- Deployed a **production-ready workflow** via Helm using the `gitops` profile
- Ran the workflow end-to-end **from the Developer Portal** using the Orchestrator feature

This completes the core Orchestrator workflow loop. Your Developer Portal is now capable of surfacing and executing SonataFlow-based workflows in a fully self-service manner.

-----

## Further Reading

- [SonataFlow documentation](https://sonataflow.org/)
- [SonataFlow GitOps deployment profile](https://sonataflow.org/serverlessworkflow/main/cloud/operator/gitops-profile.html)
- [Quarkus Dev UI guide](https://quarkus.io/guides/dev-ui)
- [Sample orchestrator workflows repository](https://github.com/redhat-ads-tech/orchestrator-workflows)
- [Serverless Workflow specification](https://serverlessworkflow.io/)
