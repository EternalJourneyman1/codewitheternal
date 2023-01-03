---
layout: ../../layouts/PostLayout.astro
url: posts/01-deploying-local-service-to-internet
title: Exposing Local Kubernetes (k3s) services to the Internet using an External Domain Name
description: Showing how to expose a self-hosted application to the internet using a custom domain name.
author: Demetrious Robinson
publishDate: "02 Jan, 2023"
---

# Exposing Local Kubernetes (k3s) services to the Internet using an External Domain Name

#pre-reqs
- A kubernetes cluster. I am using a k3s cluster but you can use whatever you like.

- Access to your router's control panel

- An External Domain Name. If you don’t have you can usually get one for $1 for your first year from [ionos.com](http://ionos.com)** (You can still expose your service without a domain name but it will be expose as your public ip address)


This article assumes you have an existing kubernetes cluster. If you just care about exposing a local service from your home network to the internet with an external domain name feel free to skip to the end to the  Port forwarding section.

First things first. We need a load balancer and for bare-metal we might as well go with MetalLB. I chose MetalLB for it is easy to install and manage. (If you already have a load balancer feel free to continue using that and skip to the next part). Whenever we deploy a kubernetes service of type LoadBalancer MetalLB  will allocate an external IP address from a pre-configured address range and add that external IP address to the host network. Which is then accessible by any client on our local network.

## Installing MetalLb
Run the following command in a terminal to install MetalLB to our cluster.

`kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml`

After installing MetalLB We have to configure an address pool or a range of ip addresses for metallb to be able to grant IP address to our services of type LoadBalancer.

### Configuring our IP address Pool
We can configure this however we want but in this instance I am going to allocate ip addresses ending in 240 to 250 on my local network. You can even use VLAN addresses  but for the purpose of this article we won’t get into that.

## Create the following file and apply it. 
#### Make sure to substitute your desired local ip range in the addresses section

--- metallb-addresspool.yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: staging
  namespace: metallb-system
spec:
  addresses:
     - 192.168.1.240-192.168.1.250
  autoAssign: true
```
`kubectl apply - metallb-addresspool.yaml`

Once we create the pool we have to alert the cluster to the address pool. We do this by deploying a Layer 2 Advertisement or L2Advertisement. Let's create this file and apply it as well.

---  metallb-addresspool-advertisement.yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: staging
  namespace: metallb-system
spec:
  ipAddressPools:
  - staging
```
`kubectl apply -f metallb-addresspool-advertisement.yaml`

Now that we have our loadbalancer configured to assign ip addresses let's deploy an application. Create the following file and apply it. 

--- test-nginx-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-nginx-app
  namespace: test
spec:
  selector:
    matchLabels:
      name: test-nginx
  template:
    metadata:
      labels:
        name: test-nginx
    spec:
      containers:
        - name: backend
          image: docker.io/nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
```
`kubectl apply -f test-nginx-deployment.yaml`


## Service
--- test-nginx-service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-nginx-service
  namespace: test
spec:
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    name: test-nginx
```
`kubectl apply -f test-nginx-service.yaml`
Now let’s apply the files we’ve just created.

We can now list our services and our whoami service should have ask external IP address assigned to it. This app is now accessible over your local network.

Kubectl get service


Now that we have made our app accessible throughout our local network lets work on getting our app accessible to the greater World Wide Web.  First lets go in our Control Panel where we have our external domain name. My Domain is from Ionos but if you have GoDaddy or any other domain name provider the following still applies.  Go Into your DNS settings for your domain name and create an A record. If hostname is left blank it assumes the whole domain instead of a subdomain. For points to add the domain name for your home network. If you don’t know your public IP address you can visit. https://www.whatsmyip.org/.


** provide pictures here of Ionos dashboard


Now we have to install our Ingress Controller to control incoming traffic as our reverse proxy load balancer. We’re going to be using the Nginx Ingress Controller.



Install Ingress Controller


kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/baremetal/deploy.yaml

The Nginx Ingress Controller sits dormant until you deploy a resource. Let’s create an Ingress rule to connect our custom domain to our test-nginx application.


--- test-nginx-ingress.yaml
## expose service  with Ingress (actually already exposed)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-nginx-ingress
  namespace: test
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: codewitheternal.net
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: test-nginx-service
                port:
                  number: 80
```
`kubectl apply -f test-nginx-ingress.yaml`  

Alright so we’ve pointed our custom domain to our public IP address and now our Ingress Controller knows to route traffic it receives for our domain to our test-nginx application  but we’re still missing something. Our home router is actually the one receiving all the traffic for our custom domain right now.
<insert image wtf 0.o> 


We have to go into our router’s control panel and route all traffic on port 80 to our machine’s local IP address with our kubernetes master node.

## route port 80 to our kubernetes master node
** provide pictures here of router for Port-Forwarding

That’s it. We should now be able to open our web browser and navigate to our custom domain.  We should see
The Nginx Welcome message.
![](/nginx.png)  

I hope you’ve enjoyed this post as much as I enjoyed figuring out how to expose my local app :D