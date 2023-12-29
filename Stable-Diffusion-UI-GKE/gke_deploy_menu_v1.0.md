## deploy

### Before you begin
To deploy this solution, you will need the following applications installed on your workstation. If you use Cloud Shell to run these steps, those applications are already installed for you:
* A Google Cloud Project with a VPC network
* [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
* [Google Cloud CLI](https://cloud.google.com/sdk/docs/install)


## Enable APIs
Start a [Cloud Shell](https://cloud.google.com/shell/docs/run-gcloud-commands) instance to perform the following steps.

In Cloud Shell, enable the required Cloud APIs using the [gcloud CLI](https://cloud.google.com/sdk/docs).
```
gcloud services enable compute.googleapis.com artifactregistry.googleapis.com container.googleapis.com file.googleapis.com
```

## Initialize the environment
In Cloud Shell, set the default Compute Engine zone to the zone where you are going to create your GKE cluster.
```shell
export PROJECT=$(gcloud info --format='value(config.project)')
export GKE_CLUSTER_NAME="stable-diffusion-gke"
export REGION="us-central1"
export ZONE="us-central1-a"
export VPC_NETWORK="stable-diffion-gke-vpc"
export VPC_SUBNETWORK="stable-diffion-gke-vpc-subnet"
export CLIENT_PER_GPU=2
```

## Create GKE Cluster

```shell
#[option B] gcloud example for creating a zonal cluster
gcloud beta container --project ${PROJECT_ID} clusters create ${GKE_CLUSTER_NAME} --zone ${ZONE} \
    --no-enable-basic-auth --release-channel "None" \
    --machine-type "n1-standard-4" --accelerator "type=nvidia-tesla-t4,count=1" \
    --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "100" \
    --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/cloud-platform" \
    --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-private-nodes \
    --master-ipv4-cidr "172.16.1.0/28" --enable-ip-alias --network "projects/${PROJECT_ID}/global/networks/${VPC_NETWORK}" \
    --subnetwork "projects/${PROJECT_ID}/regions/${REGION}/subnetworks/${VPC_SUBNETWORK}" \
    --no-enable-intra-node-visibility --default-max-pods-per-node "110" --no-enable-master-authorized-networks \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,GcpFilestoreCsiDriver \
    --enable-autoupgrade --no-enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 \
    --enable-autoprovisioning --min-cpu 1 --max-cpu 64 --min-memory 1 --max-memory 256 \
    --autoprovisioning-scopes=https://www.googleapis.com/auth/cloud-platform --no-enable-autoprovisioning-autorepair \
    --enable-autoprovisioning-autoupgrade --autoprovisioning-max-surge-upgrade 1 --autoprovisioning-max-unavailable-upgrade 0 \
    --enable-vertical-pod-autoscaling --enable-shielded-nodes
```

## Create NAT and Cloud Router (Optional if your cluster is not private)
```shell
# create cloud router
gcloud compute routers create nat-router --network ${VPC_NETWORK} --region ${REGION}

# create nat 
gcloud compute routers nats create nat-gw --router=nat-router --region ${REGION} --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges
```

### Connect to your GKE cluster
```shell

# For zonal cluster
gcloud container clusters get-credentials ${GKE_CLUSTER_NAME} ---zone ${ZONE}
```

## Install NVIDIA GPU device drivers
```shell
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```

## Create Cloud Artifacts as Docker Repo
```
BUILD_REGIST=<replace this with your preferred Artifacts repo name>

gcloud artifacts repositories create ${BUILD_REGIST} --repository-format=docker \
--location=${REGION}
```

## Build Stable Diffusion Image
Build image with provided Dockerfile, push to repo in Cloud Artifacts \
Please note I have prepared two seperate Dockerfiles for inference and training, for inference, we don't include dreambooth extension for training.

```shell
cd gcp-stable-diffusion-build-deploy/Stable-Diffusion-UI-Novel/docker_inference

# Build Docker image locally (machine with at least 8GB memory avaliable)
gcloud auth configure-docker ${REGION}-docker.pkg.dev
docker build . -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${BUILD_REGIST}/sd-webui:inference
docker push 

# Build image with Cloud Build
gcloud builds submit --machine-type=e2-highcpu-32 --disk-size=100 --region=${REGION} -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${BUILD_REGIST}/sd-webui:inference 

```

## Create a Filestore instance
Use the below commands to create a Filestore instance to store model outputs and training data.

```
FILESTORE_NAME="stable-diffusion-fileStore"
FILESTORE_ZONE="us-central1-a"
FILESHARE_NAME="stable-diffusion-fileStore-FileShare"

gcloud filestore instances create ${FILESTORE_NAME} --zone=${FILESTORE_ZONE} --tier=BASIC_HDD --file-share=name=${FILESHARE_NAME},capacity=1TB --network=name=${VPC_NETWORK}
```

## Configure Cluster Autoscaling
```shell
# For zonal cluster
gcloud container clusters update ${GKE_CLUSTER_NAME} --enable-autoscaling --node-pool=default-pool --min-nodes=0 --max-nodes=5 ---zone ${ZONE}
```

## Apply deployment and service
```shell
cp ./Stable-Diffusion-UI-Novel/templates/* /Stable-Diffusion-UI-Novel/kubernetes/
# Edit variables in yaml files
# In practice you may have to deploy one deployment & service for each model
# different model share different NFS folder
kubectl apply -f ./Stable-Diffusion-UI-Novel/kubernetes/deployment.yaml
kubectl apply -f ./Stable-Diffusion-UI-Novel/kubernetes/service.yaml
```

## Enable Horizonal Pod autoscaling(HPA)
```shell
# Optional, just to ensure that you have necessary permissons to perform the following actions on the cluster
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user "$(gcloud config get-value account)"

kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml
```

If you use GCS instead of Filestore, Workload Identity in your cluster is enabled, so additional steps are necessary. In the commands below, use your Project ID as and Google Service Account. \
Make sure your has monitoring.viewer IAM role. \
-==============================
```
gcloud projects add-iam-policy-binding \
    ${PROJECT_ID} \
    --member="serviceAccount:${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/monitoring.viewer"
```
Create IAM Policy Binding:
```
gcloud iam service-accounts add-iam-policy-binding --role  roles/iam.workloadIdentityUser \
--member "serviceAccount:${PROJECT_ID}.svc.id.goog[custom-metrics/custom-metrics-stackdriver-adapter]" ${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com
```
Annotate the Custom Metrics - Stackdriver Adapter service account:
```
kubectl annotate serviceaccount --namespace custom-metrics \
  custom-metrics-stackdriver-adapter \
  iam.gke.io/gcp-service-account=${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com
```
GCS additional is over \
-==============================

Deploy horizonal pod autoscaler policy on the Stable Diffusion deployment
```shell
kubectl apply -f ./Stable-Diffusion-UI-Novel/kubernetes/hpa.yaml
# or below if time-sharing is enabled
kubectl apply -f ./Stable-Diffusion-UI-Novel/kubernetes/hpa-timeshare.yaml
```

## Clean up
```
gcloud container clusters delete ${GKE_CLUSTER_NAME} --region=${REGION_NAME}

gcloud filestore instances delete ${FILESTORE_NAME} --zone=${FILESTORE_ZONE}

gcloud artifacts repositories delete ${BUILD_REGIST} \
    --location=us-central1 --async

```
