# rosa-hcp-tf

ROSA with HCP using terraform

## Requirements 

* Terraform
* AWS CLI
* ROSA CLI
* OpenShift CLI

## Getting started : Deploy ROSA HCP

Base tutorial https://aws.amazon.com/blogs/containers/build-rosa-clusters-with-terraform/

### I. Prepare the deployment

1. Login onto AWS
2. Check if ELB role exists
```bash
# To check if the role exists for your account, run this command in your terminal:
aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing"

# If the role doesn't exist, create it by running the following command:
aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"

```
3. Login onto [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/token)
4. Generate an Offline token, click on "Load Token"
```bash
export RH_TOKEN=yourToken
rosa login --token=${RH_TOKEN}

rosa whoami

rosa verify quota --region="$AWS_REGION"

# this may fail due to org policy
rosa verify permissions --region="$AWS_REGION"

# TODO: check if this one is required:
rosa create account-roles --mode auto
```
5. Enable HCP ROSA on [AWS MarkePlace](https://docs.openshift.com/rosa/cloud_experts_tutorials/cloud-experts-rosa-hcp-activation-and-account-linking-tutorial.html)
    5.1 Navigate to the ROSA console : https://console.aws.amazon.com/rosa
    5.2 Choose Get started.
    5.3 On the Verify ROSA prerequisites page, select I agree to share my contact information with Red Hat.
    5.4 Choose Enable ROSA

Please note that **Only a single AWS account that will be used for service billing can be associated with a Red Hat account.**

### II. Deploy a cluster with terraform

```bash
export ADMIN_PASS="yourPassword!!138"
export ADMIN_USER="kubeadmin"
export CLUSTER_NAME="rosatest"

terraform init
terraform plan -out rosa.plan -var "cluster_name=$CLUSTER_NAME" -var "htpasswd_password=$ADMIN_PASS" -var "htpasswd_username=$ADMIN_USER" -var "offline_access_token=$RH_TOKEN"
terraform apply rosa.plan
```

### III. Retrieve cluster informations

1. In the output, you will have the created cluster id:
```bash
cluster_id = "2b3sq2r4geb7b6htaibb4uqk9qc9c3fa"
```
2. Describe the cluster
```bash
export CLUSTER_ID="2b3sq2r4geb7b6htaibb4uqk9qc9c3fa"

rosa describe cluster --output=json -c $CLUSTER_ID
```
3. Generate the kubeconfig:
```bash
export NAMESPACE="myNs"
export SERVER_API=$(rosa describe cluster --output=json -c "$CLUSTER_ID" | jq -r '.api.url')
oc login --username "$ADMIN_USER" --password "$ADMIN_PASS" --server=$SERVER_API

kubectl config rename-context $(oc config current-context) "$CLUSTER_NAME"
kubectl config use "$CLUSTER_NAME"

# create a new project
oc new-project "$NAMESPACE"
```


## Install C8 on the deployed OpenShift

TODO: align needed ressources to deploy C8 on OpenShift, minimal worker size is not sufficient for standard deployment

_Please note that this guide assumes that you have a working helm cli installed with a version > 3.1_

1. Install the helm chart with a specific version:
```bash
helm repo add camunda https://helm.camunda.io
helm repo update
helm pull camunda/camunda-platform --version 10.0.4 --untar --untardir ./tmp-helm
```
2. Please note that we use some values for:
    - the version of C8 that we deploy: [./test/c8-fixtures/values-latest.yaml](https://helm.camunda.io/camunda-platform/values/values-latest.yaml)
3. We need to [enforce some specific values for OpenShift](https://github.com/camunda/camunda-platform-helm/tree/main/charts/camunda-platform/openshift#prerequisite):
```bash
helm install camunda camunda/camunda-platform --skip-crds       \
    --values ./tmp-helm/camunda-platform/openshift/values.yaml        \
    --values ./test/c8-fixtures/values-latest.yaml         \
    --post-renderer bash --post-renderer-args ./tmp-helm/camunda-platform/openshift/patch.sh
```

## Improvements / TODO

- Use S3 to store the states (directly in the action)
- Setup weekly cleanup (in action)
- add to toolsversion (rosa and oc missing)
