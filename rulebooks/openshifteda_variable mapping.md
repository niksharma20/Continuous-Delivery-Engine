---
- name: Listen for OpenShift events
  hosts: all
  sources:
    - name: Listen for OpenShift events
      juniper.eda.k8s:
        # --- THE FIX ---
        # Switch from kubeconfig mapping to direct variable mapping
        host: "{{ k8s_auth_host }}"
        api_key: "{{ k8s_auth_api_key }}"
        verify_ssl: false              # Set to true if your cluster has valid public certs
        kinds:
          - api_version: v1
            kind: Namespace
          - api_version: v1
            kind: Route
  rules:
    - name: Set Resource Quotas to a Namespace
      condition: event.resource.kind == "Namespace" and event.type == "ADDED"
      action:
        run_job_template:
          name: OpenShift Set Resource Quota on Namespace
          organization: Application Development
          job_args:
            extra_vars:
              namespace: "{{ event.resource.metadata.name }}"
