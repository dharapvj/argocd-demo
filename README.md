# ArgoCD Deployment and demo

## Objectives
* Use upstream ArgoCD docker image. No custom docker image.
* Use ArgoCD helm-chart. No customized helm-chart to maintain. Only customized values files.
* Be able to deploy application workloads via helm or raw kubernetes manifests
* Be able to use decrypt the manifests for application workloads e.g. secret manifests, secrets in values.yaml etc and deploy them. The encrypted manifests would be available from a standard git repository (for both helm and raw k8s scenarios).

## Tools used
* [Age](FIXME:)
    * used by SOPS to encrypt sensitive files
    * used to encrypt the master secret used by ArgoCD helm chart to store a master private key used to decrypt encrypted values by ArgoCD (internally by helm-sops or sops)
* [SOPS](FIXME:)
* [helm-sops](FIXME:)
* [ArgoCD helm chart](FIXME:)
    * Custom initContainer to include sops and helm-sops in the ArgoCD
    * ArgoCD custom Plugin to decrypt sops encrypted raw kubernetes manifest files on the fly (required for secrets in raw k8s manifests)

## How does the entire setup works?
* Standard ArgoCD Deployment
    * To deploy ArgoCD, we use standard ArgoCD helm chart as outlined above.
    * Once ArgoCD is deployed, you should be able to use to deploy regular of-the-shelf apps e.g nginx-ingress-controller helm chart / raw k8s manifests from git repo etc.
* Handling secrets and passwords etc in values.yaml (in helm)
    * Age
        * Most crucial piece of software is age which handles encryption and decrytion of the text.
        * Age is similar to asymetric key cryptographic softwares like RSA.
        * We use age-keygen utility on local to create a public-private key pair. Call this master keypair. You do ___not___ share this private key with anyone.
        * We create one more keypair using age-keygen which we will embed in ArgoCD installation.
        * Look for `enc.age-key-file-secret.yaml` file at root folder of this repo for the secret that we install before installing ArgoCD helm chart. This file contains the private key for argocd specific key-pair (but in encrypted format).
        * This file itself has been encrypted locally using the master keypair. So we can safely place this file in a public repository. No one can decrypt it and know your sops specific private key.
    * SOPS
        * SOPS is a tool developed by Mozilla which makes it easy to use any of the standard multiple encryption tools (including cloud based encryption tools) to encrypt certain set of files.
        * It also allows partial encryption of files.
        * This makes it a great choice for files like secret definition yaml manifests, values.yaml containing sensitive data etc.
        * We use age as our encryption tool within the purvue of SOPS.
        * But you can use pgp, vault, azure keyvault, aws kms and other such tools, refere to SOPS github page to know how to integrate with other tools.
        * for SOPS and age to integrate, we need `SOPS_AGE_KEY_FILE` environment variable pointing to Age private key.
        * So, if you look at ArgoCD helm-chart values.yaml (`values-argo.yaml` in root folder), you will see this environment variable pointing to a `key.txt` file. You will also notice in the values-argo.yaml that, `key.txt` file is mounted via the `volume` called `age-key` which mounts the `secret` `age-key` which we have installed as indicated above in age tools section.
    * Helm-SOPS
        * When we deploy helm charts using ArgoCD, ArgoCD internally uses `helm template` command to generate final raw k8s manifests which are then deployed by ArgoCD.
        * So, if we want to decrypt the secrets / values.yaml files, and then use them via helm-template command, you would need to wrap helm to run a decrpytion first. This is where helm-sops comes into play. helm-sops can run as "helm" when original helm binary is in path with name "_helm".
        * If you see the initContainer section of `values-argo.yaml`, you will see that we are downloading helm-sops and renaming existing helm as _helm and helm-sops as helm.
        * We are mounting such renamed binaries in ArgoCD repo-server so that it will use helm-sops (named helm) to process helm manifest files and thereby decrypt any encrypted content on the fly. For decryption, helm-sops looks for sops binary. SOPS looks for `SOPS_AGE_KEY_FILE` env variable for private key to do decryption of the encrypted content.
        * So flow to get decrypted content is ArgoCD --> Helm (actually helm-sops) --> SOPS --> Age.
    * ArgoCD helm-chart
        * ArgoCD allows you to pass an arbitary initContainer which can [add custom tools to ArgoCD](https://argoproj.github.io/argo-cd/operator-manual/custom_tools/#adding-tools-via-volume-mounts).
        * We are making use of this feature to push, helm-sops and sops binaries and also to rename the helm to _helm and mounting that in repo-server.
        * ArgoCD also allows you to make use of [custom plugins](https://argoproj.github.io/argo-cd/user-guide/config-management-plugins/) to "transform" k8s manifests generated by ArgoCD.
        * We make use of custom-plugins to decrypt any "encrypted" manifest files using sops binary.
        * Look at custom plugin defined in values-argo.yaml for this.
        * __Please note__: Current plugin code expects you to have all your encrypted manifests in a dedicated `enc` directory. But you can adjust plugin source in values-argo.yaml, if you want different way to select all encrypted files. Just make sure that encrypted files are not getting selected via `cat *.yaml` stream.
    * The `age-key` secret deployment
        * One key piece that was not explained so far is, how is the `age-key` secret deployed in k8s, if the content of it was encrypted by private key, which is not available in git repo.
        * This secret was installed manually, from local environment using following command
        ```
        export KUBECONFIG=<kubeconfig_of_your_cluster>
        export SOPS_AGE_KEY_FILE=<PATH TO YOUR age-master-key.txt>
        sops -d enc.age-key-file-secret.yaml | kubectl apply -f - 
        ```
        * For above command to work, you have to export SOPS_AGE_KEY_FILE environment variable with master private key which we discussed in the Age tool above.
        * This way, sops decrypts the encrypted file and applies decrypted file content in argocd namespace in your cluster

## Install ArgoCD using helm and encrypted secret
```
export KUBECONFIG=<kubeconfig_of_your_cluster>
export SOPS_AGE_KEY_FILE=<PATH TO YOUR age-master-key.txt>
sops -d enc.age-key-file-secret.yaml | kubectl apply -f - 

# Assumed that you have added argocd helm repo in your local already.
# if not.. please run below command first
# helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade --install argocd -f values-argo.yaml --version 3.17.6 --namespace argocd --create-namespace  argo/argo-cd
```

## Application Deployment via ArgoCD

Now that ArgoCD is deployed along with secret decryption capabilities, its time to deploy sample apps. Two sample apps here are available in this repo itself. a) helm-chart based app in `helm-chart` directory. A raw k8s manifest app in `raw-k8s` directory. Both the apps showcase encrypted content. Content has been encrypted using same application encryption key which was provided to ArgoCD via age-key secret.

### Deployment of helm-app using secret
To deploy helm-app as well as raw-k8s app, you can use included manifest file which uses ArgoCD CR called `Application`
```
kubectl apply -f ./hello-world-app.yaml`
```

This will install both apps - which you can then sync from ArgoCD UI to get actually deployed in your cluster.

You can see an encrypted content in `helm-chart/secrets.yaml`. This content will get decrypted by age-key private key and will get used by helm to add decrypted content in `index.html`. You can port-forward the service `helm-chart` to see the index.html file in browser.

### Deployment of raw kubernetes manifests (including secret)
This deployment simply creates a secret named `demo` in `hello-world` namespace. You can describe the secret to see the value is getting deployed properly

## Updating secret values in application
```
export SOPS_AGE_KEY_FILE=<PATH TO YOUR application-age-key.txt>
sops ./helm-chart/secret.yaml
```
Above command will open an editor like vi for you with decrypted content. You can change values as per your desire and save and close the editor for getting an updated encrypted file.

## Controlling which parts are considered as _encrypted_ by SOPS
Look at `.sops.yaml` file helm-chart as well as raw-k8s directory to understand how we can control which files and which parts of those files are considered sensitive and therefore should be encrypted. More information about this can be found in sops project readme.

## References and Thanks
This project was largely derived from 
* https://github.com/camptocamp/argocd-helm-sops-example

More details about above example can be read in camp-to-camp article on dev.to.
* https://dev.to/camptocamp-ops/argo-cd-secrets-management-using-sops-1eke

I also use forked helm-sops from teejaded as it has more better features which are not merged into upstream helm-sops project.
* https://github.com/teejaded/helm-sops
