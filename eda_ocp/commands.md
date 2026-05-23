# Commands

## Step 1: Create the ClusterRole  
This step defines exactly what resources your rulebook can view. It is limited to read-only API access (get, list, watch).  
[eda-clusterrole.yaml](eda-clusterrole.yaml)  
```bash  
oc apply -f eda-clusterrole.yaml
```  

## Step 2: Create the ServiceAccount  
This creates the machine identity that the EDA Controller will use to log into your cluster. Note: We will create this inside a dedicated namespace named aap.  
[eda-serviceaccount.yaml](eda-serviceaccount.yaml)  
```bash  
# Create the namespace first if it doesn't exist
oc create namespace aap

# Create the ServiceAccount
oc apply -f eda-serviceaccount.yaml
```  

## Step 3: Bind the ServiceAccount to the ClusterRole  
This step connects the machine identity (Step 2) to the read-only monitoring permissions (Step 1).  
[eda-rolebinding.yaml](eda-rolebinding.yaml)  
```bash
oc apply -f eda-rolebinding.yaml
```  
## Step 4: Generate a Persistent Long-Lived Token (Crucial for Outer App Access)  
OpenShift deployments do not auto-generate permanent tokens for security compliance. Since your EDA controller lives outside OpenShift, you must manually create a Secret to house a permanent access token.  
[eda-token-secret.yaml](eda-token-secret.yaml)  
```bash  
oc apply -f eda-token-secret.yaml
```  
## Step 5: Extract the Secret to Validate and Use in AAP  
To verify everything worked and to get the password your EDA controller needs, extract the token from the secret object and decode it.  
```bash  
# Extract the API Token string
oc get secret eda-token-secret -n aap -o jsonpath='{.data.token}' | base64 --decode
```  

## Step 6: Tag Target Namespaces/Pods with type=eda  
The Juniper plugin optimizes resource consumption by ignoring untagged resources. Ensure that any project namespace or specific resource you intend to monitor contains the type=eda label.
```bash
# Label a namespace for Juniper monitoring
oc label namespace default type=eda

# Label a specific target pod
oc label pod <your-pod-name> -n default type=eda
```  
#
#
##
