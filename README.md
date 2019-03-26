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

### Basic configurtion of VPX

Login to the newly created VPX instance and do some basic configs

```
ssh nsroot@35.200.253.191
<input the password>

clear config -force full
add ns ip 172.18.0.5 255.255.0.0 -type snip -mgmt enabled
enable ns feature MBF
```
Now the VPX instance is ready

### Overview of Deployment

* For the demo, we would deploy a basic apache microservice that would service traffic on the root http path (/)
* These apache pods would be load-balanced by CPX Ingress sitting inside the Kubernetes cluster (Tier-2 LB)
* VPX (Tier-1 LB) which is present outside the Kubernetes cluster would load-balance the CPX Ingress
* We would use [Ingress class](https://github.com/citrix/citrix-k8s-ingress-controller/blob/master/docs/ingress-class.md) to segregate the Ingress between CPX and VPX
* [Citrix Ingress Controller](https://github.com/citrix/citrix-k8s-ingress-controller) is a separate pod that would automate the VPX configuration

### Deploying the Microservice:
```
kubectl create -f https://raw.githubusercontent.com/christus02/citrix-ingress-gke/master/manifests/apache.yaml
```

### Deploying CPX Ingress (Tier-2 LB):
```
kubectl create -f https://raw.githubusercontent.com/christus02/citrix-ingress-gke/master/manifests/cpx-ingress-gke.yaml
```

### Deploying the Citrix Ingress Controller for automating the VPX configuration:
```
wget https://raw.githubusercontent.com/christus02/citrix-ingress-gke/master/manifests/cic-gke.yaml
```
Modify the below variables inside the yaml file

* NS_IP should be replaced with the NSIP of the VPX instance. In my example it is "172.18.0.5"
* NS_VIP should be replaced with the VIP (Client side IP) of the VPX instance. In my example it is "172.19.0.2"
* NS_USER should be replaced with VPX's username. If it is not changed, ignore it.
* NS_PASSWORD should be replaced with VPX's password. If it is not changed, ignore it.

More information on these variables can be found in the [Citrix Ingress Controller Git Repo](https://github.com/citrix/citrix-k8s-ingress-controller/tree/master/deployment/baremetal)

After doing necessary modifications, apply the yaml
```
kubectl create -f cic-gke.yaml
```

### Configuring CPX Ingress (Tier-2):

**Note**
This ingress was created for example purpose and does not have TLS termination. You can enable TLS termination by creating Kubernetes secret and referring the same in the Ingress

```
kubectl create -f https://raw.githubusercontent.com/christus02/citrix-ingress-gke/master/manifests/ingress-apache.yaml
```

### Configuring VPX Ingress (Tier-1):

**Note**
This ingress was created for example purpose and does not have TLS termination. You can enable TLS termination by creating Kubernetes secret and referring the same in the Ingress

```
kubectl create -f https://raw.githubusercontent.com/christus02/citrix-ingress-gke/master/manifests/vpx-ingress.yaml
```

### Accessing the Microservice from Internet

Now the deployment is done. Let's test it out by sending some traffic.

Send a curl with hostname or access it via a web browser by adding necessary host entries in your client.

```
$ curl http://35.200.250.239/ -H 'Host: citrix-ingress-gke.com'
<html><body><h1>It works!</h1></body></html>
```

This is the response from the Apache microservice that is running inside the Kubernetes cluster.
The IP which was used to access is the VIP (Client side IP) of VPX
