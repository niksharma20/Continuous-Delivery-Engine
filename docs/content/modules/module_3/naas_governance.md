# EDA-Enabled Namespace Provisioning

## Overview  

In this section you will extend the namespace request workflow by enabling Event-Driven Ansible (EDA) governance on a namespace at provisioning time.

When a developer requests a **medium/large** namespace and with **Enable EDA Governance** option, the workflow applies a label to the namespace:  
```
type: "eda"
```  
This label acts as a signal to the EDA Decision Controller. The `juniper.eda.k8s` collection watches the cluster for events on labelled namespaces and triggers Ansible Automation Controller (AAC) job templates in response to detected drift, policy violations, or resource lifecycle events.

By the end of this section you will have:

- Provisioned an EDA-governed medium/large namespace via the Orchestrator workflow
- Observed how the EDA label activates namespace-scoped event polling
- Triggered and verified EDA-driven remediation for the governanced namespace .  
  Other scenarios to consider related this use case:
  - PersistentVolumeClaim
  - TLS certificates
  - NetworkPolicy



!!! warning "Prerequisites"

    ✅ **Module 1, Part 2 - completed successfull**
    ✅ **Module 2         - completed successfull**
    ✅ **Module 3, Part 1 - completed successfully**
    ✅ **Module 3, Part 2 - completed successfully**


# Step 1: Update the Software Template to Support the EDA Label
### 1b: Pass the Value Through the Workflow Step
### 1c: Apply the Label Conditionally in the Helm Chart
## Step 4: Provision an EDA-Governed Medium Namespace
### Verify the EDA Label Was Applied

## Further Reading


!!! success "Congratulations!"
    You have successfully completed this section.