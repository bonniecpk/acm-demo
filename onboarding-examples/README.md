## Onboarding Applications - Demo
### Platform Team 
Platform team is in charge of provisioning Kubernetes resources/objects while onboarding new applications or applying changes to applications.

### Scenarios
This example demonstrates the scenario that platform team will onboard two applications and each in their own namespace (newapp-1 and newapp-2). The needed objects/resources are below:
- Namespace
- ClusterRoles
- Roles
- ClusterRoleBinding
- RoleBinding
- NetworkPolicy
- ServiceEntry
- ResourceQuota - Optional
### Setup Instructions 
Step 1 - Ensure you have copied the folders below into your own repository. 

- `/acm-config/base-apps`
- `/acm-config/namespaces/newapp-1`
- `/acm-config/namespaces/newapp-1`
- `/acm-config/overlays/sandbox/tenant/roles`
- `/onboarding-examples`

Step 2 - Edit the files below to include the added folders. 

In `acm-config/overlays/sandbox/hn1-gke-europe-west1-cluster2/kustomization.yaml` and `acm-config/overlays/sandbox/hn1-gke-us-central1-cluster1/kustomization.yaml` files, please ensure that they have included the configuration below: 

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
...
-  ../../../namespaces/newapp-1
-  ../../../namespaces/newapp-2
-  ../tenant/roles
```

Step 3 - Edit the Google Service Accounts
Make sure to change the service account in the following files: 
- `acm-config/namespaces/newapp-1/deployer-role-binding.yaml`
- `acm-config/namespaces/newapp-1/namespace-role-binding.yaml`
- `acm-config/namespaces/newapp-1/viewer-role-binding.yaml`
- `acm-config/namespaces/newapp-2/deployer-role-binding.yaml`
- `acm-config/namespaces/newapp-2/namespace-role-binding.yaml`
- `acm-config/namespaces/newapp-2/viewer-role-binding.yaml`

Look for the comments saying `##Needs to be changed` to ensure attaching the corresponding deployer and viewer service accounts to the namespace.


Step 4 - Check the format

Run below from the repo root:

`nomos hydrate --no-api-server-check --source-format=unstructured --output=../dev-output-cluster2 --path=acm-config/overlays/sandbox/hn1-gke-europe-west1-cluster2` 

`nomos hydrate --no-api-server-check --source-format=unstructured --output=../dev-output-cluster1 --path=acm-config/overlays/sandbox/hn1-gke-us-central1-cluster1` 

Please use a dedicated directory for stoing the output.

### Deployment Instructions 

Step 1 - Download the Sample app for testing 

Leverage Artifact Registry

Set up a repository - [instructions](https://cloud.google.com/artifact-registry/docs/docker/store-docker-container-images) if not done so.

Authenticate to the Artifact Registry

`gcloud auth configure-docker us-central1-docker.pkg.dev` and change the region if not `us-central1`

Then run

`docker pull avinash2312/hello_app_2023:v1.0.0`

Then run 

`docker tag docker.io/avinash2312/hello_app_2023:v1.0.0 YOUR_CHOSEN_REGION-docker.pkg.dev/YOUR_PROJECT_NAME/YOUR_REPO_NAME/hello_app:0.0.1` 

Then push the image - for example as below: 

`docker push us-central1-docker.pkg.dev/cn-datahub-logs/hlf-acm/hello_app:0.0.1`

Change the image fields in the following files that are marked as `##needs to be changed` comments.

- `onboarding-examples/newapp-1/newapp-1-deployment.yaml` 
- `onboarding-examples/newapp-2/newapp-2-deployment.yaml`

Step 2 - Authenticate into the cluster as service account (deployer and viewer)
If you authenicate into the clusters via the bastion, please ensure that that service account has `Service Account Admin` access to create keys for the deployer and viewer service accounts. 

Run 

```
gcloud iam service-accounts keys create --iam-account=SERVICE_ACCOUNT_NAME@YOUR_PROJECT_NAME.iam.gserviceaccount.com newapp-1-deployer.json
``` 

```
gcloud auth activate-service-account SERVICE_ACCOUNT_NAME@YOUR_PROJECT_NAME.iam.gserviceaccount.com --key-file=newapp-1-deployer.json
```

Now you are authenticated as service account for the deployer.

Run below to ensure you are authenticated as the correct service account and then get credentials into the cluster again.
```
gcloud config set account SERVICE_ACCOUNT_NAME@YOUR_PROJECT_NAME.iam.gserviceaccount.com
``` 

Then deploy the resources in the `/onboarding-examples/newapp-1` by running
```
kubectl apply -f onboarding-examples/newapp-1
```

Ensure the deployments and services are running before authenticate to other clusters and deploy the same resources in `newapp-1`. 

Then run below to authenticate as the deployer for `newapp-2` namespace.
```
gcloud iam service-accounts keys create --iam-account=SERVICE_ACCOUNT_NAME@YOUR_PROJECT_NAME.iam.gserviceaccount.com newapp-2-deployer.json
``` 

```
gcloud auth activate-service-account SERVICE_ACCOUNT_NAME@YOUR_PROJECT_NAME.iam.gserviceaccount.com --key-file=newapp-2-deployer.json
```

Then deploy the resources in the `/onboarding-examples/newapp-2` by running
```
kubectl apply -f onboarding-examples/newapp-2
```

Test using `curl` and the call should fail.
```
curl 'https://cluster.sandbox.whereami.hbl.net/newapp-2/testcallseq?call1=http://svc-a.newapp-1:8080/testcallseq&call2=http://svc-b.newapp-2:8080/testcallseq&call3=http://httpbin.org/get' --header "host: a2.helloapp.judyzwu.joonix.net" -m 10
```

### Set Up Ingress Network Policy
Step 1 - Uncomment the following the file `allow-namespace-network-policy.yaml` in `acm-config/namespaces/newapp-1/kustomization.yaml` and in `acm-config/namespaces/newapp-2/kustomization.yaml` 

This allow ingress traffic from the `asm-gateway` namespace and namespaces for `newapp-1` from `newapp-2` and vice versa. 

Step 2 - Test
Test using `curl` and the call should succeed.
```
curl 'https://cluster.sandbox.whereami.hbl.net/newapp-2/testcallseq?call1=http://svc-a.newapp-1:8080/testcallseq&call2=http://svc-b.newapp-2:8080/testcallseq&call3=http://httpbin.org/get' --header "host: a2.helloapp.judyzwu.joonix.net" -m 10
```

### Control Egress traffic 
Step 1 - Apply `REGISTRY_ALL` to the egress traffic for the clusters in the default `outboundTrafficPolicy` from `ALLOW_ANY`. 

Run the following to see the ConfigMap the current istio release channel is such as `istio-asm-managed-rapid`.
```
kubectl get cm -n istio-system
```   

Run to edit the file to add the data field. 
```
kubectl edit configmap istio-asm-managed-rapid -n istio-system -o yaml
``` 
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-asm-managed-stable
  namespace: istio-system
data:
  mesh: |-
    outboundTrafficPolicy:
    mode: REGISTRY_ONLY
```

And check by running `kubectl get configmap istio-asm-managed-rapid -n istio-system -o yaml` to ensure add the `data` field as above. It disable all egress traffic into the external. 

Step 2 - Test
Test using `curl` and the call should fail.
```
curl 'https://cluster.sandbox.whereami.hbl.net/newapp-2/testcallseq?call1=http://svc-a.newapp-1:8080/testcallseq&call2=http://svc-b.newapp-2:8080/testcallseq&call3=http://httpbin.org/get' --header "host: a2.helloapp.judyzwu.joonix.net" -m 10
```

### Allow Specific hosts for egress 
Step 1 -  Uncomment the following the file `service-entry.yaml` in `acm-config/namespaces/newapp-1/kustomization.yaml` and in `acm-config/namespaces/newapp-2/kustomization.yaml` 


Step 2 - Test
Test using `curl` and the call should succeed.
```
curl 'https://cluster.sandbox.whereami.hbl.net/newapp-2/testcallseq?call1=http://svc-a.newapp-1:8080/testcallseq&call2=http://svc-b.newapp-2:8080/testcallseq&call3=http://httpbin.org/get' --header "host: a2.helloapp.judyzwu.joonix.net" -m 10
```

### More Information
#### RBAC 
The roles are defined under `/overlays/sandbox/tenant/roles`. Create a GCP custom role called `testMinConsoleGKE` with the following permissions: 
Permissions: 
- container.clusters.get
- container.clusters.getCredentials
- container.clusters.list
- gkehub.features.get
- gkehub.features.list
- gkehub.memberships.list

Create one service account for deployer and one viewer for each namespace with the custom role assigned. Then go to the `namespaces/newapp-1` and `namespaces/newapp-2` and change the service accounts in the role-binding YAML files. Look for comments on places that need to be changed.

- Deployer 
`Cluster Role` for deploying Kubernetes Resources 
`RoleBinding` for binding the permissions to the namespace only 

- Viewer 
`Cluster Role` for viewing Kubernetes Resources 
`RoleBinding` for binding the permissions to the namespace only 

- Namespace Viewer
`Cluster Role` for viewing all the namespace
`ClusterRoleBinding` for binding the permission to the whole cluster

#### Network Policy
In the `/base` fodler, Network Policy - `default-deny-all-ingress` sits in the `/base-apps` folder so by default all ingress traffic are denied for all namespaces. 
Each namespace has its own Network Policy - `allow-namespace-network-policy` to allow traffic from `asm-gateway` or other needed namespaces (such as `newapp-1` to `newapp-2` or `newapp-2` to `newapp-1`).

Run `kubectl get netpol -n newapp-2 -o yaml` in the cluster to see the Network Policy.

#### Egress control
Apply `REGISTRY_ALL` to the egress traffic for the clusters in the default `outboundTrafficPolicy` from `ALLOW_ANY`. 

Run `kubectl get cm -n istio-system` and here is the results `istio-asm-managed-rapid`. 

Run `kubectl edit configmap istio-asm-managed-rapid -n istio-system -o yaml` and edit the file as below. 
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-asm-managed-stable
  namespace: istio-system
data:
  mesh: |-
    outboundTrafficPolicy:
    mode: REGISTRY_ONLY
```
And check by running `kubectl get configmap istio-asm-managed-rapid -n istio-system -o yaml` to ensure add the `data` field as above. It disable all egress traffic into the external. 

#### Service Entry 
Define the Service Entry for each namespace to whitelist the hosts such as `httpbin.org` with port number, protocol and name for each namespace to allow egress traffic into the internet for specific hosts.

Run `kubectl get se -A` and get the results below. 
```
NAMESPACE   NAME                   HOSTS             LOCATION   RESOLUTION   AGE
newapp-1    allow-egress-httpbin   ["httpbin.org"]                           5h5m
newapp-2    allow-egress-httpbin   ["httpbin.org"]                           5h5m
```

#### Without ASM 
Once deployment and service in each namespace are created, get into the pod and try to reach the service in other namespaces. 

Get the pod name by running `kubectl get pods -n newapp-1` and get the pod name below.
Run `kubectl exec -it -n newapp-2 svc-b-6664cc77ff-tb7z9 -- bash` and then `curl svc-a.newapp-1:8080/hello` or `curl svc-b.newapp-2:8080/hello` to reach each other.

#### With ASM 
Once install ASM, it will restart the pods such as `kubectl rollout restart deployments -n newapp-2` and for each pod it will also inject a proxy. 

#### Application Team 
Application team will deploy the following resources. 
- Deployment
- Service 
- Virtual Service
Application team deploy the files in the `/onboarding-examples` folder and run `kubectl apply` in each namespace folder.
#### Virtual Service
It will require a hostname for the two virtual services or leverage path in the configuration to differentiate the entry points into the services. 

Configuration when using one domain name: 
```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: svc-b
  namespace: newapp-2
spec:
  hosts:
  - "cluster2.sandbox.whereami.hbl.net" 
  gateways:
  - "asm-gateway/test-smoke-gateway" 
  http:
  - route:
    - destination:
        port:
          number: 8080
        host: "svc-b.newapp-2.svc.cluster.local"
```

Configuration when using one: 
```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: svc-b
  namespace: newapp-2
spec:
  hosts:
  - "cluster.sandbox.whereami.avinashj.in" 
  gateways:
  - "asm-gateway/test-smoke-gateway" 
  http:
  - match:
    - uri:
        prefix: "/newapp-2"
      ignoreUriCase: true
    rewrite:
      uri: "/"
    route:
    - destination:
        port:
          number: 8080
        host: "svc-b.newapp-2.svc.cluster.local"
```

### Policy Controller
Start the root-sync for the policy bundles
```
kubectl create ns config-management-system && \
kubectl create secret generic git-creds \
  --namespace="config-management-system" \
  --from-literal=username=random-name \
  --from-literal=token=dfdfsfsdsf
```
Run `kubectl get secrets -n config-management-system` to check the `git-creds`.

(Constraint Template's target)[https://cloud.google.com/anthos-config-management/docs/how-to/write-custom-constraint-templates]

### Tag & Firewall Rules
Create auto-provisioning network tage for the cluster and ensure that you have the following firewall rules set up in your VPC for allowing the traffic. 

```
gcloud compute firewall-rules create YOUR_FIREWALL_RULE_NAME --network YOUR_NETWORK_NAME --description "" --allow tcp:10256 --source-ranges 130.211.0.0/22,209.85.152.0/22,209.85.204.0/22,35.191.0.0/16 --target-tags YOUR_CLUSTER_TAG --project YOUR_PROJECT_ID
```

```
gcloud compute firewall-rules create YOUR_FIREWALL_RULE_NAME --network YOUR_NETWORK_NAME --description "{\"kubernetes.io/service-name\":\"asm-gateway/istio-ingressgateway\", \"kubernetes.io/service-ip\":\"34.86.167.93\"}" --allow tcp:15021,tcp:443,tcp:80 --source-ranges 0.0.0.0/0 --target-tags YOUR_CLUSTER_TAG --project YOUR_PROJECT_ID
```

### Testing 
Run `curl '35.239.99.137/testcallseq?call1=http://svc-a.newapp-1:8080/testcallseq&call2=http://svc-b.newapp-2:8080/testcallseq&call3=http://httpbin.org/get' --header "host: a2.helloapp.judyzwu.joonix.net" -m 10` and the result is 

### apps for testing 
Get the user email to leverage Docker pull that (image)[https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling]

Probably needs to authenticate into the system using Google ID and then pull and push to its own environment. 

Leverage Artifact Registry

Set up a repository - (steps)[https://cloud.google.com/artifact-registry/docs/docker/store-docker-container-images]
Run `gcloud auth configure-docker us-central1-docker.pkg.dev`
Will need to change the region based on where the repo is created in. 

Copy from the public docker
Run `docker pull avinash2312/hello_app_2023:v1.0.0`
Then `docker tag docker.io/avinash2312/hello_app_2023:v1.0.0 us-central1-docker.pkg.dev/cn-datahub-logs/hlf-acm/hello_app:0.0.1` and then `docker push us-central1-docker.pkg.dev/cn-datahub-logs/hlf-acm/hello_app:0.0.1`
                          
 
`gcloud auth configure-docker us-central1-docker.pkg.dev`

#### Download the Sample app for testing 
Get the user email to leverage Docker pull that (image)[https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling]

Probably needs to authenticate into the system using Google ID and then pull and push to its own environment. 

Leverage Artifact Registry

Set up a repository - [instructions](https://cloud.google.com/artifact-registry/docs/docker/store-docker-container-images)

Run `gcloud auth configure-docker us-central1-docker.pkg.dev`
Will need to change the region based on where the repo is created in. 

Copy from the public docker
Run `docker pull avinash2312/hello_app_2023:v1.0.0`
Then `docker tag docker.io/avinash2312/hello_app_2023:v1.0.0 us-central1-docker.pkg.dev/cn-datahub-logs/hlf-acm/hello_app:0.0.1` and then `docker push us-central1-docker.pkg.dev/cn-datahub-logs/hlf-acm/hello_app:0.0.1`
                          
`gcloud auth configure-docker us-central1-docker.pkg.dev`

Now go change the deployment files image path into `us-central1-docker.pkg.dev/cn-datahub-logs/hlf-acm/hello_app:0.0.1`. 
