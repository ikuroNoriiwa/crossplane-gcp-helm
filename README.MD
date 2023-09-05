# Crossplane GCP learning 


## Installation 

### Install required tools 

This example required to have the following tools : 
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [helm](https://helm.sh/docs/intro/install/)
- [argocd](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- [gcloud](https://cloud.google.com/sdk/docs/install)

Please check the official documentation to install them 

### Add helm repo 

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm update
```

### Install Crossplane components 

```bash
helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace 
```

### Check crossplane installation 

```bash
kubectl get pods -n crossplane-system
```
Once all the resources are deployed, you can now configure the providers 

### Install providers 

This tutorial required the following providers : 
- provider.pkg.crossplane.io/provider-gcp-compute
- provider.pkg.crossplane.io/provider-gcp-servicenetworking
- provider.pkg.crossplane.io/provider-gcp-sql
- provider.pkg.crossplane.io/provider-gcp-storage

They are all specified in the `providers.yaml` file. To apply them you need just to apply the resources : 
```bash
kubectl apply -f providers.yaml
```
and wait all the resources are deployed. If the cluster is hosted on GKE autopilot, the nodes needs to be scale up, it can take more than 30 minutes. 

### Authorize acces to gcp 

Create a service account : 
```bash 
gcloud iam service-accounts create <SA_NAME> --description="Crossplane Service Account"
```

Give required permissions for the service account. For this tutorial, we need the following permissions : 
- roles/cloudsql.admin
- roles/compute.admin
- roles/compute.networkAdmin
- roles/storage.admin
- roles/storage.objectAdmin
- roles/storage.objectCreator

```bash
for role in roles/cloudsql.admin roles/compute.admin roles/compute.networkAdmin roles/storage.admin roles/storage.objectAdmin roles/storage.objectCreator; do
gcloud projects add-iam-policy-binding <PROJECT_NAME> --member="serviceAccount:<SA_NAME>@<PROJECT_NAME>.iam.gserviceaccount.com" --role="$role"
done
```

Create Service account Key file: 

```bash
gcloud iam service-accounts keys create gcp-credentials.json --iam-account=<SA_NAME>@<PROJECT_NAME>.iam.gserviceaccount.com
```

We can now create the Kubernetes secret : 
```bash
kubectl create secret generic gcp-secret -n crossplane-system --from-file=creds=./gcp-credentials.json
```

### Configure Provider access 

```bash 
cat <<EOF | kubectl apply -f -
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: <PROJECT_NAME>
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-secret
      key: creds
EOF
```

We can now start to deploy our cloud resources. 
