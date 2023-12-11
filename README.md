# ArgoCD quick poc

The goal of this repository is to easily demonstrate that it is possible to have a helm registry with a template and have the values of it in a separate git repository.

To achieve this goal, we will try to deploy `vault` as an example.

The target is to deploy the application `vault` from the helm registry `https://helm.releases.hashicorp.com` but by applying the values of this git repository and more accurately the one from the file `mycustomvalues.yaml`. This values do implement the one from openshift and add a few more annotations we can then check once the app is deployed to validate that we did really apply the values from this git repository.

The file `values.yaml` is here just to see which values can be modify from the original values from the helm chart.

# Prerequisites

Install Openshift GitOps operator, stable channel.

# Quick setup

Get access to ArgoCD:
```bash
# If not already done, add the appropriate credentials so ArgoCD can deploy everywhere in the cluster
# Note that you can also deploy your own ArgoCD instance somewhere else, then you would need the credentials
# to access to it in the same way
oc project openshift-gitops
oc adm policy add-cluster-role-to-user -z openshift-gitops-argocd-application-controller cluster-admin
# Extract the secret of the admin user for ArgoCD
oc extract secret/openshift-gitops-cluster --to=-
# Get the route to navigate to the ArgoCD instance, or create a new one, as you wish
oc get route
```

Prepare app destination:
```bash
# Create a new project as a target for the application
oc new-project test-1
```

Update the configuration of ArgoCD:
```bash
# Move to the openshift-gitops project if you are using the default ArgoCD instance
oc project openshift-gitops
oc edit argocd <argocd_instance>
# Now you need to copy paste the line from the `manifests/patch_configManagementPlugin.yaml` file
# It needs to be at the same level of the "controller" entry, almost at the end of the manifest
# TODO: be more accurate with the path / provide a patch command
```

Deploy the app:
Navigate to the ArgoCD instance and deploy the app by creating a new application and copy pasting the content of the `manifests/app.yaml` file.
Note that the app will not be ready as you need to apply a few commands into the vault pod so it makes the probes healphy.
Here are the command to apply to the pod on the test-1 project:
```bash
# Exec to the vault pod
oc exec -it vault-0 -- /bin/sh
# Initialise the configuration of the vault, please refer to Vault documentation
# /!\ We decided to have 1 key and only 1 is mandatory to unseal the vault - not fine for Production /!\
vault operator init --tls-skip-verify -key-shares=1 -key-threshold=1
...
Unseal Key 1: <key-1>
Initial Root Token: <root-token>
# Please backup this root token because it will be useful in the next steps
# Export variables and unseal the vault ussing the key you get
export KEYS=<key-1>
export ROOT_TOKEN=<root-token>
export VAULT_TOKEN=${ROOT_TOKEN}
vault operator unseal -tls-skip-verify ${KEYS}
```

# Improvements
I have commited 2 more files on the manifests folder with `_2` suffix. This could be an improvement of the current setup to avoid using a hard coded helm registry.
However I did not spend the time to bootstrap them to see if it is working as intended. This could be more scallable as if you are running multiple template in your helm registry you could easily change that from one Application to another. 

# History / References / researches
The goal of this repository was to do so because some companies do have their own helm repository and they are using it as a template for several micro services.
That being said, I was looking for a solution to implement such a setup (having the values in one git repository while using the chart from another source).

During my researches, I came up with a few knowledge and here are the useful links:
- [Helm chart + values files from Git #2789](https://github.com/argoproj/argo-cd/issues/2789) from Dec 2, 2019 - basically asking for that feature to be natively covered by the product
- Comment on [the same thread](https://github.com/argoproj/argo-cd/issues/2789#issuecomment-574821873), from cronik + rbkaspr (Mar 26, 2020) which basically is the implementation we do in this git repository 
- [Multiple source reference as a target for 2.6](https://github.com/argoproj/argo-cd/issues/15032) - so it should be implemented on 2.6.3, but for some reason, it is not working properly (Aug 13, 2023)
- [Multiple source beta documentation page](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/) - bêta - so can not be recommanded for production
- [Config Management Plugins as a CRD approach documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/)
- Special warning for Red Hat Openshift GitOps operator which is not always following the latest version of ArgoCD: [link to the documentation](https://docs.openshift.com/gitops/1.10/release_notes/gitops-release-notes.html)
- [Environement variables implementation inside a plugin](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/#using-environment-variables-in-your-plugin), be careful this is changing from one version to another recently so if you update it, be aware that this might be broken and some prefix might need to be added to your `configManagementPlugin`.
