## Setting up the GitOps workflow in TAP

Using the GitOps workflow, TAP is able to push the `delivery.yaml` file back to the Git repo where it downloaded the source code from.

This doc describe how to set that up using TAP 1.0 and the `ootb-supply-chain-basic` and github.com.

### Configuring ssh authentication

Create an ssh keypair to use for GitHub an add hostkeys:

```
ssh-keygen -t rsa
ssh-keyscan github.com > ./known_hosts
```

Use the generated public key to create an SSH key in Github at [https://github.com/settings/keys](https://github.com/settings/keys).

Create a k8s secret file (git-ssh.yaml):

```
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh
  annotations:
    tekton.dev/git-0: github.com
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: |
     <Your Base64 encoded private key here>
  ssh-publickey: |
     <Your Base64 encoded public key here>
  known_hosts: |
     <Base64 encoded contents of known_hosts>
```

Apply to the developer namespace (assuming default here):

```
kubectl apply -f ./git-ssh.yaml
```

Add the secret to the serviceaccount in the developer namespace:

```
kubectl patch serviceaccount default -p '{"secrets": [{"name": "git-ssh"}]}'
```

Adjust the `ootb-supply-chain-basic` configuration in `tap-values.yaml` to include the following (and adjust to your needs):

``` 
gitops:
    ssh_secret: "git-ssh"
    repository_prefix: git@github.com:pvdbleek/
    # The following parameters are optional:
    # username: ""
    # email: ""
    # branch: ""
    # commit_message: ""
```

Update your TAP installation:

```
tanzu package installed update tap -p tap.tanzu.vmware.com -v 1.0.0 --values-file tap-values.yaml -n tap-install
```

After all needed packages have been reconciled, create a new workload that is stored in github.com.
Instead of deploying the application on your TAP cluster, it will now push a `delivery.yml` to your Github repo that can be deployed to other clusters.
