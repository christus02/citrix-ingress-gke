# Create a Citrix Ingress in Google Managed Kubernetes Engine (GKE) using gcloud API

This guide describes how to do a complete deployment of a Citrix Ingress in Google's Managed Kubernetes Engine using gcloud without using Google Cloud's UI console.

For more information on Citrix's Ingress solution, please read [Citrix Kubernetes Ingress Controller](https://github.com/citrix/citrix-k8s-ingress-controller)

### Pre-requisites:

* Install kubectl on your client
* Install gcloud SDK for your client
* Authenticate to your Google account using gcloud API `gcloud auth login`
* Set your default project for Google cloud using gcloud API `gcloud config set project <PROJECT-NAME>`

## Create a Google Managed Kubernetes (GKE) Cluster 

### Create 3 VPC networks, subnets and respective firewall rules

**Create a VPC for Management traffic**
```
gcloud compute --project=netscaler-networking-k8 networks create k8s-mgmt --subnet-mode=custom
gcloud compute --project=netscaler-networking-k8 networks subnets create k8s-mgmt-subnet --network=k8s-mgmt --region=asia-south1 --range=172.17.0.0/16
gcloud compute firewall-rules create k8s-allow-mgmt --network k8s-mgmt --allow tcp:443,tcp:80,tcp:22
```

**Create a VPC for Server side or Private communication with the Kubernetes cluster**
```
gcloud compute --project=netscaler-networking-k8 networks create k8s-server --subnet-mode=custom
gcloud compute --project=netscaler-networking-k8 networks subnets create k8s-server-subnet --network=k8s-server --region=asia-south1 --range=172.18.0.0/16
gcloud compute firewall-rules create k8s-allow-server --network k8s-server --allow tcp:443,tcp:80
```

**Create a VPC for Client traffic**
```
gcloud compute --project=netscaler-networking-k8 networks create k8s-client --subnet-mode=custom
gcloud compute --project=netscaler-networking-k8 networks subnets create k8s-client-subnet --network=k8s-client --region=asia-south1 --range=172.19.0.0/16
gcloud compute firewall-rules create k8s-allow-client --network k8s-client --allow tcp:443,tcp:80
```

### Create a 3 node GKE cluster

```
gcloud beta container --project "netscaler-networking-k8" clusters create "test-1" --zone "asia-south1-a" --username "admin" --cluster-version "1.11.7-gke.12" --machine-type "n1-standard-2" --image-type "COS" --disk-type "pd-standard" --disk-size "100"  --num-nodes "3" --network "projects/netscaler-networking-k8/global/networks/k8s-server" --subnetwork "projects/netscaler-networking-k8/regions/asia-south1/subnetworks/k8s-server-subnet" --addons HorizontalPodAutoscaling,HttpLoadBalancing
```

### Connect to the created Kubernetes cluster and create a cluster-admin role for your Google Account

```
gcloud container clusters get-credentials test-1 --zone asia-south1-a --project netscaler-networking-k8 
```
Now your kubectl client is updated with the credentials required to login to the newly created Kubernetes cluster

```
kubectl create clusterrolebinding cpx-cluster-admin --clusterrole=cluster-admin --user=<email of the gcp account>
```

### Deploying a Citrix ADC VPX instance on Google Cloud

Follow the guide [Deploy a Citrix ADC VPX instance on Google Cloud Platform](https://docs.citrix.com/en-us/citrix-adc/12-1/deploying-vpx/deploy-vpx-google-cloud.html) to download the VPX from Citrix Downloads, uploading to Google Cloud's storage and to create a Citrix ADC VPX image out of it.

**Create a VPX instance (assuming you have already created the VPX image)**
```
gcloud compute --project=netscaler-networking-k8 instances create vpx-frontend-ingress --zone=asia-south1-a --machine-type=n1-standard-4 --network-interface subnet=k8s-mgmt-subnet --network-interface subnet=k8s-server-subnet --network-interface subnet=k8s-client-subnet --image=vpx-gcp-12-1-51-16 --image-project=netscaler-networking-k8 --boot-disk-size=20GB
```

**IMPORTANT**
After executing the above command, it would return both the private IPs and the Public IPs of the newly created VPX instance. Make a note of it.

**Example:**

```
# gcloud compute --project=netscaler-networking-k8 instances create vpx-frontend-ingress --zone=asia-south1-a --machine-type=n1-standard-4 --network-interface subnet=k8s-mgmt-subnet --network-interface subnet=k8s-server-subnet --network-interface subnet=k8s-client-subnet --image=vpx-gcp-12-1-51-16 --image-project=netscaler-networking-k8 --boot-disk-size=20GB
Created [https://www.googleapis.com/compute/v1/projects/netscaler-networking-k8/zones/asia-south1-a/instances/vpx-frontend-ingress].
NAME                  ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP                       EXTERNAL_IP                                  STATUS
vpx-frontend-ingress  asia-south1-a  n1-standard-4               172.17.0.2,172.18.0.5,172.19.0.2  35.200.253.191,35.200.146.53,35.200.250.239  RUNNING
```

Explanation of the IP address captured from previous output:

| Private IP |    Public IP   |   Network  |              Comments             |
|:----------:|:--------------:|:----------:|:---------------------------------:|
| 172.17.0.2 | 35.200.253.191 | Management | NSIP                              |
| 172.18.0.5 | 35.200.146.53  | Server     | SNIP (to communicate with K8s)    |
| 172.19.0.2 | 35.200.250.239 | Client     | VIP (to receive incoming traffic) |


