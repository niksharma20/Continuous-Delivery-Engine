# Continuous Delivery Engine - Technical Workshop

## Welcome to Continuous Delivery Engine for the Real World Technical Workshop

This hands-on comprehensive workshop provides **technical understanding** of Continuous Delivery Engine (CDE) components and their implementation. Designed for platform engineers, DevOps practitioners, and technical leaders, this hands-on learning experience will equip you with the expertise to deploy, configure, and optimize your production environments.

In this workshop you will build and experience a Continuous Delivery Engine using Red Hat Developer Hub and Ansible Automation Platform that delivers to containers, VMs, and other infrastructure components through a single, automated, policy-enforced pipeline — no environment left behind. Red Hat Developer Hub serves as the self-service abstraction layer; Ansible Automation Platform serves as the cross-platform automation engine. You will learn how to empower developers with a unified self-service experience while maintaining total operational control.  

> [!IMPORTANT]
>
> ## During the **Red Hat Tech Day** workshop we will only Implement Module 1 & 2.
>
> 

> [!NOTE]
> 
> ## **Workshop Goal**
> ### Build, Show, and Experience the integration between OCP, Ansible, and EDA. Users can take home this setup and explore different use cases.
> ### Today we will explore a simple use case "Namespace as Service" with EDA Goverance to understand the concept and integrations.
> ### You'll walk away with a working, end-to-end delivery pattern you can adapt to your own heterogeneous environment.  

> [!WARNING]
> 
> ## Prerequisites
> We will be using two CI from Red Hat Demo to provide us the baseline Infrastrature.  
> 1)  [Ansible 2.6 with EDA](https://catalog.demo.redhat.com/catalog/babylon-catalog-prod?item=babylon-catalog-prod/enterprise.aap-product-demos-cnv-aap25.prod&utm_source=webapp&utm_medium=share-link)
> 2)  [Openshift 4.18+ with Gitops, Pipeline, Monitoring Stack, RHDH with Orchestrator](https://catalog.demo.redhat.com/catalog/babylon-catalog-prod?item=babylon-catalog-prod/pert.redhat-rhads.prod&utm_source=webapp&utm_medium=share-link)
>
> 

## Workshop Structure

[**Module 1 - Declarative Infrastructure**](content/modules/1_declarative_infra)  

| Component | Description |
| :--- | :--- |
| **Red Hat Developer Hub (RHDH)** | Enterprise-grade developer portal providing convenient access to curated resources and promoting efficiency and collaboration |
| **Orchestrator** | Serverless Workflow |
| **Openshift** | Target Platform |  
| **Openshift Gitops** | GitOps |

[**Module 2 - Operational Orchestration**](content/modules/2_operational_orchestration)  

| Component | Description |
| :--- | :--- |
| **Red Hat Developer Hub (RHDH)** | Enterprise-grade developer portal providing convenient access to curated resources and promoting efficiency and collaboration |
| **Orchestrator** | Serverless Workflow |
| **Openshift** | Target Platform |
| **Openshift Gitops** | GitOps |
| **Ansible Automation Platform** | Automation Controller |
| **Event Driven Ansible** | Automation Decisions |  
| **GitHub** | Ticket Automation (Here we are using a Git but in real env it would be Jira or Service Now) |  

[**Module 3 - Reactive Intelligence**](content/modules/3_reactive_intelligence)

| Component | Description |
| :--- | :--- |
| **Red Hat Developer Hub (RHDH)** | Enterprise-grade developer portal providing convenient access to curated resources and promoting efficiency and collaboration |
| **Openshift** | Target Platform |
| **Openshift Gitops** | GitOps |
| **Openshift** | Monitoring Stack - Promethues & Alert Manager |
| **Ansible Automation Platform** | Automation Controller | 
| **Event Driven Ansible** | Automation Decisions |  

> [!TIP]
>
> ## Use Case to explore  
> 


## Prerequisites

## References
[View the Showroom Demo Event-Driven Ansible Modules](https://redhat-gpte-devopsautomation.github.io/showroom-demo-event-driven-ansible/modules/index.html)  

