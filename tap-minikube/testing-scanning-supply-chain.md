## Configuring ootb\_supply\_chain\_testing\_scanning


**Note:** This guide assumes you have succesfully configured`ootb_supply_chain_basic` already.

Make sure all the correct packages are installed. The example `tap-values.yml` in this repo has the full profile with some packages excluded to make it still run on a local minikube. 

Also adjust the `supply_chain` from `basic` to `testing_scanning`. See `tap-values.yml` in this repo for examples.

Update your TAP deployment:

```
tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.0.0 --values-file tap-values-pvanderbleek.yml -n tap-install
```

Once all the packages have reconciled, ensure you have the correct supply chain:

```
$ tanzu apps cluster-supply-chain list
NAME                      READY   AGE   LABEL SELECTOR
source-test-scan-to-url   Ready   48m   apps.tanzu.vmware.com/has-tests=true,apps.tanzu.vmware.com/workload-type=web
```

Create your Tekton pipeline using the example in this repo:

```
kubect apply -f ./pipeline-test.yml
```

Create a Scanning Policy using the example in this repo:

```
kubectl apply -f ./scan-policy.yml
```

Check if the developer namespace (where you create your workloads) has the proper scantemplates:


```
$ kubectl get scantemplates 
NAME                           AGE
blob-source-scan-template      55m
private-image-scan-template    55m
private-source-scan-template   55m
public-image-scan-template     55m
public-source-scan-template    55m
```

You can now create your workload but make sure you add `--label apps.tanzu.vmware.com/has-tests=true` to the workload like in the following example:

```
tanzu apps workload create go-places \
  --git-repo https://github.com/pvdbleek/go-places \
  --git-branch main \
  --type web \
  --label apps.tanzu.vmware.com/has-tests=true \
  --label app.kubernetes.io/part-of=go-places \
  --env "MARIADB_USER=dbuser" \
  --env "MARIADB_PASS=secretpass" \
  --env "MARIADB_HOST=mariadb" \
  --yes
```

If you now tail the workload it should go through the following steps:

1. Pull source code from your git repo
2. Run tests defined in your pipeline
3. Scan the source code and create a BOM
4. Build a container image 
5. Apply conventions (if any)
6. Scan the container image for CVE's and create BOM
7. Deploy the app or push a `delivery.yml` (depends if you use the gitops workflow)

The BOM are not stored besides in the logging you see when tailing the workload.
If you want to store it for querying/reporting purposes, you can install [https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-scst-store-overview.html](https://docs.vmware.com/en/Tanzu-Application-Platform/1.0/tap/GUID-scst-store-overview.html)