# Conclusion: Continuous Delivery Engine

## What You Have Built  
By working through this workshop, you have assembled the complete **event-driven governance layer** for a Namespace on Openshift — this simple is very use case but the integration quite Powerful.  

-----

## Objectives: Completed

-----

## Extending This Platform

The patterns in this repository are intentionally generic. Common extensions include:

|Extension                      |What to Add                                                                              |
|-------------------------------|-----------------------------------------------------------------------------------------|
|Watch additional resource types|Add new `kinds:` entries to the rulebook source block                                    |
|Add a new governance rule      |Add a new `rules:` entry in the rulebook + a new Job Template + a new playbook           |
|Support multiple clusters      |Create one Rulebook Activation per cluster, each with its own OpenShift credential       |
|Add TTL enforcement            |Add a rule matching namespace TTL annotations + a playbook that opens a Git PR           |
|Integrate with ITSM            |Extend remediation playbooks to create ServiceNow / Jira tickets via their APIs          |
|Add quota breach alerting      |Source from Alertmanager in addition to `k8s.eda` — combine event sources in one rulebook|

-----


## Further Reading

- [Event-Driven Ansible documentation](https://www.ansible.com/products/event-driven-ansible)
- [Juniper k8s.eda collection](https://github.com/Juniper/k8s.eda)
- [AAP Automation Decisions](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/latest/html/using_automation_decisions)
- [kubernetes.core collection](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/index.html)
- [OpenShift RBAC](https://docs.openshift.com/container-platform/latest/authentication/using-rbac.html)
- [ArgoCD documentation](https://argo-cd.readthedocs.io/en/stable/)
- [SonataFlow documentation](https://sonataflow.org/)
