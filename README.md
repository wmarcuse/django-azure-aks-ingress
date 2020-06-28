# Deploying a Django application in Azure AKS with an Ingress Controller and TLS

TL;DR approach, read the full acrticle on [Medium](https://medium.com/@wmarcuse/deploying-a-django-application-in-azure-aks-with-an-ingress-controller-and-tls-266a1e2dc697).

## Build and run the Django app with Docker

    $ docker build -f Dockerfile -t django-aks:v1.0.0 .
    $ docker run -it -p 8010:8010 django-aks:v1.0.0

Check the deployment at: https://127.0.0.1:8010

## Tag and push the image to ACR

    $ docker tag django-aks:v1.0.0 yourregistry.azurecr.io/django-aks:v1.0.0
    $ docker push yourregistry.azurecr.io/django-aks:v1.0.0
    
## Create NGINX Ingress controller
Replace `PUBLIC_IP` with your Azure public ip resource address.

    $ helm install nginx-ingress stable/nginx-ingress \     
      --namespace yournamespace \     
      --set controller.replicaCount=2 \     
      --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
      --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux \
      --set controller.service.loadBalancerIP="PUBLIC_IP" \
      --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"="your-dns-label"
    
## Replace the host in the files

Replace the current FQDN in all the `k8s` files with your own FQDN that you created in the previous step.

Add the FQDN in `app/app/settings.py` in the `ALLOWED_HOSTS` list.

Rebuild the Docker image and push it to your ACR registry.
    
## Install cert-manager

    $ helm repo add jetstack https://charts.jetstack.io
    $ helm repo update
    $ helm install \
      cert-manager jetstack/cert-manager \
      --namespace yournamespace \
      --version v0.15.1 \
      --set installCRDs=true

## Apply the manifests to your cluster

    $ kubectl apply -f cluster-issuer.yaml --namespace yournamespace
    $ kubectl apply -f webapp-deployment.yaml --namespace yournamespace
    $ kubectl apply -f webapp-service.yaml --namespace yournamespace
    $ kubectl apply -f ingress-routing.yaml --namespace yournamespace