### Copy the hb-constraint folder 
All the deny mode constraints are targeting to the `constraint-test` namespace only. 

run `kubectl get configs -n gatekeeper-system` to check whether a Config has been deployed.

The step is required when leveraging [referential constraints](https://cloud.google.com/anthos-config-management/docs/how-to/creating-policy-controller-constraints#gatekeeper-config). It ensures that the policy controller is listening to certain resources.

### Test them out
Step 1: run 

```
kubectl apply -f constraint-test-namespace-without-network-policy-without-config.yaml
```

It will work because config is not created.

Step 2: Uncomment the `referential-constraint-config.yaml` to apply the config in `gatekeeper-system` via ACM.

Once ready, continue.

Step 3: run `constraint-test-namespace-without-network-policy.yaml`
It will fail.

Step 4: run `network-policy-for-namespace.yaml`

Step 5: rerun `constraint-test-namespace-without-network-policy.yaml`

It should succeed. 

Done verifying `K8sRequireNamespaceNetworkPolicies`

Step 1: run `pod-without-asm-sidecar-injection.yaml`
It will fail.

Step 2: change the value into `true` and rerun. It will succeed.

Done verifying `AsmSidecarInjection`

Step 1: run `container-has-bigger-limits.yaml`

Step 2: run `container-has-no-limits.yaml`

Both will fail.

Step 3: update the limit and rerun.

It should succeed.

Done verifying `K8sContainerLimits`

Step 1: run `pod-in-default-namespace.yaml`
It will fail.

Step 2: change the namespace into `constraint-test` and it should succeed.

Done verifying `K8sRestrictNamespaces`

### Clean up
Step 1: Delete everything in the namespaces and delete the namespace
Step 2: Delete the patch to make the constraints back to dryrun.