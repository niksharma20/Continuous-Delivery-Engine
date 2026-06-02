# EDA-Enabled Namespace Provisioning

## Overview  

In this section you will extend the namespace request workflow by enabling Event-Driven Ansible (EDA) governance on a namespace at provisioning time.

When a developer requests a **medium/large** namespace and with **Enable EDA Governance** option, the workflow applies a label to the namespace:  
```
type: "eda"
```  
This label acts as a signal to the EDA Decision Controller. The `juniper.eda.k8s` collection watches the cluster for events on labelled namespaces and triggers Ansible Automation Controller (AAC) job templates in response to detected drift, policy violations, or resource lifecycle events.

By the end of this section you will have:

- Provisioned an EDA-governed medium namespace via the Orchestrator workflow
- Observed how the EDA label activates namespace-scoped event polling
- Triggered and verified EDA-driven remediation for a real-world scenarios.
  Other scenarios to consider:
  - A PersistentVolumeClaim created without a storage class label
  - A TLS certificate Secret missing a required annotation
  - A NetworkPolicy deleted from the namespace

## Architecture Flow  

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

## Prerequisites   
> Make sure module two has been successfully and completed and verified.
> 
>

# Step 1: Update the Software Template to Support the EDA Label
### 1b: Pass the Value Through the Workflow Step
### 1c: Apply the Label Conditionally in the Helm Chart
## Step 4: Provision an EDA-Governed Medium Namespace
### Verify the EDA Label Was Applied

## Further Reading


!!! success "Congratulations!"
    You have successfully completed this section.