## Onboarding Applications 
### Platform Team 
Platform team is in charge of provisioning Kubernetes resources/objects while onboarding new applications or applying changes to applications.

### Scenarios
This example demonstrates the scenario that platform team will onboard two applications and each in their own namespace. The needed objects/resources are below:
- Namespace
- ClusterRoles
- Roles
- ClusterRoleBinding
- RoleBinding
- NetworkPolicy
- ServiceEntry
- ResourceQuota
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

Now go change the deployment files image path into `us-central1-docker.pkg.dev/cn-datahub-logs/hlf-acm/hello_app:0.0.1`. 

### RBAC 
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

### Network Policy
In the `/base` fodler, Network Policy - `default-deny-all-ingress` sits in the `/base-apps` folder so by default all ingress traffic are denied for all namespaces. 
Each namespace has its own Network Policy - `allow-namespace-network-policy` to allow traffic from `asm-gateway` or other needed namespaces (such as `newapp-1` to `newapp-2` or `newapp-2` to `newapp-1`).

Run `kubectl get netpol -n newapp-2 -o yaml` in the cluster to see the Network Policy.

### Egress control
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

### Service Entry 
Define the Service Entry for each namespace to whitelist the hosts such as `httpbin.org` with port number, protocol and name for each namespace to allow egress traffic into the internet for specific hosts.

Run `kubectl get se -A` and get the results below. 
```
NAMESPACE   NAME                   HOSTS             LOCATION   RESOLUTION   AGE
newapp-1    allow-egress-httpbin   ["httpbin.org"]                           5h5m
newapp-2    allow-egress-httpbin   ["httpbin.org"]                           5h5m
```

### Without ASM 
Once deployment and service in each namespace are created, get into the pod and try to reach the service in other namespaces. 

Get the pod name by running `kubectl get pods -n newapp-1` and get the pod name below.
Run `kubectl exec -it -n newapp-2 svc-b-6664cc77ff-tb7z9 -- bash` and then `curl svc-a.newapp-1:8080/hello` or `curl svc-b.newapp-2:8080/hello` to reach each other.

### With ASM 
Once install ASM, it will restart the pods such as `kubectl rollout restart deployments -n newapp-2` and for each pod it will also inject a proxy. 

## Application Team 
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