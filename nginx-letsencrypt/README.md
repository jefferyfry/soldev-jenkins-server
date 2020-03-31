# soldev-jenkins-server 

## Create Your GKE Cluster

```
gcloud beta container clusters create soldev-jenkins \
    --zone us-central1-c \
    --release-channel regular \
    --machine-type=n1-standard-1 \
    --enable-autoscaling \
    --min-nodes=1 \
    --max-nodes=5
```

## Install NGINX K8s Controller

```
kubectl create namespace ingress-nginx

helm install --namespace ingress-nginx \
    --set defaultBackend.enabled=false \
    --set controller.scope.enabled=true \
    --set controller.scope.namespace=soldev-jenkins \
    nginx-ingress stable/nginx-ingress
```

Get your LoadBalancer IP.

```
LB_IP=$(kubectl -n ingress-nginx \
    get svc nginx-ingress-controller \
    -o jsonpath="{.status.loadBalancer.ingress[0].ip}")

echo $LB_IP
```

## Setup DNS
Configure the $LBIP to be static. IN GCP, this is done under [External IP Addresses](https://console.cloud.google.com/networking/addresses/list?organizationId=201145417873).

Use [Cloud DNS](https://console.cloud.google.com/net-services/dns/zones) to add a subdomain to jfrog.team and point to the the $LBIP.

## Create Jenkins Namespace
```
kubectl create namespace soldev-jenkins
```

## Set Up Storage
### Create a SSD Storage Class
Apply soldev-jenkins-storageclass-ssd.yaml to create an SSD storage class for Jenkins.

```
kubectl apply -f soldev-jenkins-storageclass-ssd.yaml
```

### Request Storage
Apply the persistent volume claim to set up Jenkins storage.

```
kubectl apply -f soldev-jenkins-pvc.yaml -n soldev-jenkins
```

## Set Up SSL
### Install K8s Certificate Manager

1. Create a namespace for the cert-manager.

```
kubectl create namespace cert-manager
```
2. Install cert-manager and CRDs.

```
helm repo add jetstack https://charts.jetstack.io

helm repo update

helm install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --version v0.13.1
```

3. Verify the installation.

```
kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6b5d76bf77-hgb9f              1/1     Running   0          101m
cert-manager-cainjector-7c5667645b-qhjvv   1/1     Running   0          101m
cert-manager-webhook-59846cdfb6-xncff      1/1     Running   1          101m
```
### Set Up the DNS Service Account for DNS01 Challenges

```
export PROJECT_ID=<project id>

gcloud iam service-accounts create dns01-solver --display-name "dns01-solver"

gcloud projects add-iam-policy-binding $PROJECT_ID \
   --member serviceAccount:dns01-solver@$PROJECT_ID.iam.gserviceaccount.com \
   --role roles/dns.admin

gcloud iam service-accounts keys create key.json \
   --iam-account dns01-solver@$PROJECT_ID.iam.gserviceaccount.com

kubectl create secret generic clouddns-dns01-solver-svc-acct \
   --from-file=key.json -n soldev-jenkins
```

### Set Up the Let's Encrypt Issuer
1. Updated the letsencrypt-cluster-issuer.yaml with your email address.

2. Apply the cluster issuer.

```
kubectl apply -f letsencrypt-issuer.yaml -n soldev-jenkins
```

3. Check the state of the cluster issuer.

```
kubectl get issuer -n soldev-jenkins

kubectl describe issuer -n soldev-jenkins
```

4. Update self-cert.yaml with the correct host and common name.

5. Apply certificate.

```
kubectl apply -f letsencrypt-cert.yaml -n soldev-jenkins
```
6. Check the status of the certificate.

```
kubectl get certificate -n soldev-jenkins
```

7. Get the details of the certificate.

```
kubectl describe certificate -n soldev-jenkins
```

8. If you are having issues, you can check the certificate request.

```
kubectl describe certificaterequest <id> -n soldev-jenkins
```

9. And if you continue to have certificate requst issues, check the cert-manager logs.

## Install Jenkins
Execute the following 'helm install' to install jenkins. Set your own admin username and password. Set $HOSTNAME to the FQDN set above.

If you have made changes and want to see what the resulting configuration will look like without actually executing, add --dry-run.

```
helm install --namespace soldev-jenkins --set master.hostName=$HOSTNAME --set master.adminUser=admin --set master.adminPassword=p@ssw0rd! -f soldev-jenkins-helm-config.yaml soldev-jenkins stable/jenkins
```
or

```
helm install --set master.hostName=$HOSTNAME --set master.adminUser=admin --set master.adminPassword=p@ssw0rd! -f soldev-jenkins-helm-config.yaml soldev-jenkins stable/jenkins --dry-run > dryrun.txt
```