# Swapping preview over to other cluster

We have the ability to create another preview cluster on demand.
We don't run with two at a time as they're 75 node clusters and that would be quite expensive.

To create the cluster:

* Go to the [release pipeline](https://dev.azure.com/hmcts/CNP/_release?_a=releases&view=mine&definitionId=16)
* Create a new release, updating the Clusters variable for Preview, to include either 00, or 01 whichever one you want to build
* Click 'Deploy' for the Preview stage

Then:

* Wait for the build to finish
* Check the system helm releases and pods are up: flux, flux-helm-operator, tunnelfront, coredns, nodelocaldns, aad-pod-identity, traefik
* Check an external IP has been assigned: `kubectl get service -n admin |awk '$4 ~ /^[0-9]/'` 
* Check OSBA is running, `kubectl get pods -n osba`, `kubectl get pods -n catalog`

Once you're happy with the cluster:

* Change Jenkins to use the other cluster, e.g.  [cnp-flux-config#4348](https://github.com/hmcts/cnp-flux-config/pull/4348).
* Swap external DNS active cluster, e.g. [cnp-flux-config#4606](https://github.com/hmcts/cnp-flux-config/pull/4606)
* Create a test PR, normally we just make a README change to [rpe-pdf-service](https://github.com/hmcts/rpe-pdf-service).

If, there are any issues with dns records still conflicting : 

* Delete all ingress on the old cluster to ensure external-dns deletes it's existing records:

```command
$ kubectl delete ingress --all-namespaces -l app.kubernetes.io/managed-by=Helm
```

* Delete any orphan records that external-dns might have missed:

_Replace 10.12.79.250 with the loadbalancer IP of the cluster you want to cleanup_
```command
$ az network private-dns record-set a list --zone-name service.core-compute-preview.internal -g core-infra-intsvc-rg --subscription DTS-CFTPTL-INTSVC --query "[?aRecords[0].ipv4Address=='10.12.79.250'].[name]" -o tsv | xargs -I {} -n 1 -P 8 az network private-dns record-set a delete --zone-name service.core-compute-preview.internal -g core-infra-intsvc-rg --subscription DTS-CFTPTL-INTSVC --yes --name {}
```

Once swap over is fully complete then delete the older cluster,

* Create a new release in the release pipeline, setting Delete.Cluster to true and 
updating the Clusters variable to contain just the cluster you want deleted.