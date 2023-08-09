### onboarding user case
1. onboarding hello_app in namespace: newapp-1
2. onboarding hello_app in namespace: newapp-2
3. create the needed objects/resources as below
- namespace
- role binding
- network policy
- service entry
- resource quota

### RBAC 
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
`default-deny-all-ingress` sits in the `base-apps` folder and leverage Kustomize in the cluster folder level to change the namespace
Each namespace has its own `allow-namespace-network-policy` to allow traffic from `asm-gateway` or other needed namespaces (such as newapp-1 to newapp-2 or newapp-2 to newapp-1).

Run `kubectl get netpol -n newapp-2 -o yaml` to see the Network Policy.

### Egress control
Apply `REGISTRY_ALL` to the egress traffic for the clusters in the default `outboundTrafficPolicy` from `ALLOW_ANY`. 

Run `kubectl get cm -n istio-system` and here is the results `istio-asm-managed-stable`. 

Run `kubectl edit configmap istio-asm-managed-stable -n istio-system -o yaml` and edit the file as below. 
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
And check by running `kubectl get configmap istio-asm-managed-stable -n istio-system -o yaml`.

Then no egress traffic can go into the external. 

### Service Entry 
Define the Service Entry for each namespace to whitelist the hosts such as `httpbin.org` with port number, protocol and name.

Run `kubectl get se -A` and get the results below. 
```
NAMESPACE   NAME                   HOSTS             LOCATION   RESOLUTION   AGE
newapp-1    allow-egress-httpbin   ["httpbin.org"]                           5h5m
newapp-2    allow-egress-httpbin   ["httpbin.org"]                           5h5m
```

### Without ASM 
Once deployment and service in each namespace are created, get into the pod and try to reach the service in other namespace. 

Run `kubectl exec -it -n newapp-2 svc-b-6664cc77ff-tb7z9 -- bash` and then `curl svc-a.newapp-1:8080/hello` or `curl svc-b.newapp-2:8080/hello` to reach each other.



### With ASM 
Once install ASM, it will restart the pods such as `kubectl rollout restart deployments -n newapp-2`


#### Virtual Service
Application team deploy the 



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
Create auto-provisioning network tage for the cluster

### Testing 
Run `curl '35.239.99.137/testcallseq?call1=http://svc-a.newapp-1:8080/testcallseq&call2=http://svc-b.newapp-2:8080/testcallseq&call3=http://httpbin.org/get' --header "host: a2.helloapp.judyzwu.joonix.net" -m 10` and the result is 

### apps for testing 
Get the user email to leverage Docker pull that (image)[https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling]

Probably needs to authenticate into the system using Google ID and then pull and push to its own environment. 


                          
