# Orchestrator for Your Developer Portal

## Overview

Other modules demonstrate how to use Software Templates as a means for developers to scaffold new codebases that follow organisational standards, and self-service certain aspects of their daily activities.

However, developer self-service requests might require more than what a Software Template can provide natively. For example, certain requests might require an approval workflow, while others might involve **long-running, stateful processes** — this is where the Orchestrator feature of your Developer Portal can help.

The Orchestrator feature addresses the fact that regular Software Templates are stateless “run and done” processes. It extends your Developer Portal with support for workflows created using [SonataFlow](https://sonataflow.org/), an implementation of the [Serverless Workflow](https://serverlessworkflow.io/) specification.

The Orchestrator project website and documentation is available at [rhdhorchestrator.io](https://www.rhdhorchestrator.io/).

-----

## SonataFlow and Serverless Workflow

> *SonataFlow is a tool for building cloud-native workflow applications. You can use it to do the services and events orchestration and choreography.* — [Apache KIE](https://kie.apache.org/)

SonataFlow is part of a broader ecosystem of projects within [Apache KIE](https://kie.apache.org/), including Drools, jBPM, and Kogito. Specifically, SonataFlow is an implementation of the [Serverless Workflow](https://serverlessworkflow.io/) model.

Serverless Workflow is a cloud-native workflow engine that allows teams to design, deploy, and execute arbitrarily complex workflows that integrate with external systems through several mechanisms. Teams define workflows in the Serverless Workflow format, and SonataFlow handles the build, state management, and serverless deployment aspects automatically.

### Example Workflow Definition

The following workflow definition demonstrates functions, input validation using schemas, and behavior modeling using [state machine](https://en.wikipedia.org/wiki/Finite-state_machine) semantics. A workflow like this can be built and deployed on OpenShift as a Quarkus-based application container using the SonataFlow Operator.

```yaml
id: greeting
version: '1.0'
specVersion: '1.0'
name: Greeting workflow
description: YAML based greeting workflow
annotations:
  - "workflow-type/infrastructure"

# Workflows use schemas, in JSON Schema format, to validate inputs
dataInputSchema:
  schema: schemas/greeting.sw.input-schema.json
  failOnValidationErrors: true

# Reusable logic can be defined using functions
functions:
  - name: greetFunction
    type: custom
    operation: sysout

  - name: successResult
    type: expression
    operation: '{ "result": { "message": "Greeting workflow completed successfully", "outputs":[ { "key":"Selected language", "value": .language }, { "key":"Greeting message", "value": .greeting } ] } }'

start:
  stateName: ChooseOnLanguage

states:
  - dataConditions:
      - condition: .language == "English"
        transition:
          nextState: GreetInEnglish
      - condition: .language == "Spanish"
        transition:
          nextState: GreetInSpanish
    defaultCondition:
      transition:
        nextState: GreetInEnglish
    name: ChooseOnLanguage
    type: switch

  - data:
      greeting: Hello from YAML Workflow
    name: GreetInEnglish
    transition:
      nextState: GreetPerson
    type: inject

  - data:
      greeting: Saludos desde YAML Workflow
    name: GreetInSpanish
    transition:
      nextState: GreetPerson
    type: inject

  - actionMode: sequential
    actions:
      - actionDataFilter:
          useResults: true
        functionRef:
          arguments:
            message: .greeting
          invoke: sync
          refName: greetFunction
        name: greetAction
      - actionDataFilter:
          useResults: true
        functionRef:
          invoke: sync
          refName: successResult
        name: setOutput
    end:
      terminate: true
    name: GreetPerson
    type: operation
```

> **ℹ️ Note**
> To learn more about the workflow YAML/JSON format, see the [workflow JSON schema on serverlessworkflow.io](https://serverlessworkflow.io/schemas/1.0.0/workflow.json).

A [VSCode Extension to assist workflow development](https://sonataflow.org/serverlessworkflow/latest/tooling/serverless-workflow-editor/swf-editor-vscode-extension.html) is supported by the SonataFlow project. This extension renders a visual representation of the workflow directly in your editor.

-----

## SonataFlow and the Orchestrator

The Orchestrator feature effectively acts as an interface between your Developer Portal and SonataFlow-powered workflows, providing a listing and the ability to view and run workflows directly from the portal UI.

Additionally, long-running workflows can integrate with the [Backstage Notifications](https://backstage.io/docs/notifications/) plugin, allowing users or groups to be notified of milestones during workflow execution. This is particularly useful for long-running workflows where it is impractical for a user to wait at the screen for completion.

### Key Capabilities

|Capability           |Description                                                                |
|---------------------|---------------------------------------------------------------------------|
|Workflow listing     |Browse all available SonataFlow workflows from within the Developer Portal |
|Workflow execution   |Trigger workflows directly from the portal UI with a form-based input      |
|Status tracking      |Monitor the progress of running or completed workflow instances            |
|Notifications        |Notify users or groups at key milestones in long-running workflows         |
|Approval flows       |Model multi-step human approval processes as stateful workflows            |
|External integrations|Connect workflows to external systems via REST, events, or custom functions|

-----

## Architectural Overview

The following describes the high-level architecture of the Orchestrator:

```
Developer Portal (UI)
        ↓
Orchestrator Plugin
        ↓  ↑ (status / notifications)
SonataFlow Operator (OpenShift)
        ↓
Workflow Instances (Quarkus containers)
        ↓
External Systems (APIs, ITSM, Approval Services, etc.)
```

|Component                      |Role                                                                   |
|-------------------------------|-----------------------------------------------------------------------|
|Developer Portal               |Provides the self-service UI for browsing and triggering workflows     |
|Orchestrator Plugin            |Bridges the portal and SonataFlow; surfaces workflow state and outputs |
|SonataFlow Operator            |Manages the lifecycle of workflow deployments on OpenShift             |
|Serverless Workflow (YAML/JSON)|Defines workflow logic, states, functions, and integrations            |
|Backstage Notifications        |Delivers milestone alerts to users or groups during execution          |
|External Systems               |Target systems invoked by workflow steps (e.g. ITSM, Git, Vault, Slack)|

-----

## Relation to Phase 2: Namespace as a Service

In the context of a Namespace as a Service platform, the Orchestrator enables workflows that go beyond what a Software Template can do on its own:

|Use Case                                    |Why Orchestrator                                                |
|--------------------------------------------|----------------------------------------------------------------|
|Namespace provisioning with manager approval|Stateful: waits for human approval before proceeding            |
|Multi-step onboarding (Vault + Quay + ITSM) |Long-running: coordinates multiple external systems sequentially|
|TTL extension requests                      |Approval gate: notifies owner, waits for response, then acts    |
|Quota upgrade requests                      |Audit trail: captures who approved what and when                |

This makes the Orchestrator the natural complement to GitOps — GitOps handles the *desired state*, the Orchestrator handles the *process to get there* when that process requires coordination, approvals, or time.

-----

## Conclusion

The Orchestrator feature enables teams to create stateful, long-running workflows using open standards (Serverless Workflow / SonataFlow), and provide them in a self-service manner through their internal developer portal.

Where Software Templates are stateless and “run and done”, the Orchestrator introduces:

- **State** — workflows pause, wait, and resume
- **Approvals** — human-in-the-loop steps at any point
- **Coordination** — multi-system integrations in a defined sequence
- **Visibility** — status and notifications surfaced in the portal

This is the foundation for building self-service platforms that handle real-world complexity — not just the happy path.

-----

## Further Reading

- [Orchestrator project documentation](https://www.rhdhorchestrator.io/)
- [SonataFlow documentation](https://sonataflow.org/)
- [Serverless Workflow specification](https://serverlessworkflow.io/)
- [Backstage Notifications plugin](https://backstage.io/docs/notifications/)
- [Apache KIE project](https://kie.apache.org/)
