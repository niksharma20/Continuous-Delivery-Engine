This is where provision ClusterRole, service account, Role binding, generate token for AAP.
User will git clone this repo and run the yamls and store the decoded secret.

# EDA

## Step 1: Create the ClusterRole

**File:** [01_eda-clusterrole.yaml]

The ClusterRole defines exactly what the EDA service account is allowed to do — **read-only access** to the resource types the rulebooks need to watch.

```bash
oc apply -f ocp/01_eda-clusterrole.yaml
```
-----

## Step 2: Create the ServiceAccount

**File:** [ocp/02_eda-serviceaccount.yaml](ocp/02_eda-serviceaccount.yaml)

The ServiceAccount is the machine identity the EDA controller presents when connecting to OpenShift. It is created in a dedicated namespace (e.g. `aap`) to keep it isolated from workload namespaces.  

**Apply:**

```bash
# Create the namespace first if it doesn't exist
oc create namespace aap

# Create the ServiceAccount
oc apply -f ocp/02_eda-serviceaccount.yaml
```
-----

## Step 3: Bind the ServiceAccount to the ClusterRole

**File:** [ocp/03_eda-rolebinding.yaml](ocp/03_eda-rolebinding.yaml)

The ClusterRoleBinding connects the ServiceAccount (Step 2) to the ClusterRole (Step 1), granting the identity its read-only permissions.

**Apply:**

```bash
oc apply -f ocp/eda-rolebinding.yaml
```
-----
## Step 4: Generate a Persistent Long-Lived Token

**File:** [ocp/04_eda-token-secret.yaml](ocp/04_eda-token-secret.yaml)

OpenShift does not auto-generate permanent tokens for ServiceAccounts. Since the EDA controller lives outside the cluster, you must manually create a Secret to house a permanent bearer token.

**Apply:**

```bash
oc apply -f ocp/04_eda-token-secret.yaml
```

-----

## Step 5: Extract the Token

Once the Secret is created, extract and decode the token for use in AAP:

```bash
oc get secret eda-token-secret -n aap \
  -o jsonpath='{.data.token}' | base64 --decode
```

Copy the output. You will paste it into AAP when configuring the OpenShift credential in the next section.


# AAC 

## Step 1: Create the ClusterRole

**File:** [01_aac-clusterrole.yaml](content/rbac/aac/aac-ocp-sa/01_aac-clusterrole.yaml)

The ClusterRole defines exactly what the AAC service account is allowed to do — **read-only access** to the resource types the rulebooks need to watch.

**Apply:**

```bash
oc apply -f rbac/aac/aac-ocp-sa/01_aac-clusterrole.yaml
```

-----

## Step 2: Create the ServiceAccount

**File:** [02_aac-serviceaccount.yaml](content/rbac/aac/aac-ocp-sa/02_aac-serviceaccount.yaml)

The ServiceAccount is the machine identity the EDA controller presents when connecting to OpenShift. It is created in a dedicated namespace (e.g. `aap`) to keep it isolated from workload namespaces.  

**Apply:**

```bash
# Create the namespace first if it doesn't exist
oc create namespace aap

# Create the ServiceAccount
oc apply -f rbac/aac/aac-ocp-sa/02_aac-serviceaccount.yaml
```

-----

## Step 3: Bind the ServiceAccount to the ClusterRole

**File:** [03_aac-rolebinding.yaml](content/rbac/aac/aac-ocp-sa/03_aac-rolebinding.yaml)

The ClusterRoleBinding connects the ServiceAccount (Step 2) to the ClusterRole (Step 1), granting the identity its read-only permissions.

**Apply:**

```bash
oc apply -f rbac/aac/aac-ocp-sa/03_aac-rolebinding.yaml
```

-----

## Step 4: Generate a Persistent Long-Lived Token

**File:** [04_aac-token-secret.yaml](content/rbac/aac/aac-ocp-sa/04_aac-token-secret.yaml)

OpenShift does not auto-generate permanent tokens for ServiceAccounts. Since the EDA controller lives outside the cluster, you must manually create a Secret to house a permanent bearer token.

**Apply:**

```bash
oc apply -f rbac/aac/aac-ocp-sa/04_aac-token-secret.yaml
```

-----

## Step 5: Extract the Token

Once the Secret is created, extract and decode the token for use in AAP:

```bash
oc get secret aac-token-secret -n aap \
  -o jsonpath='{.data.token}' | base64 --decode
```

Copy the output. You will paste it into AAP when configuring the OpenShift credential in the next section.



!!! success "Congratulations!"
    You have successfully completed this section.