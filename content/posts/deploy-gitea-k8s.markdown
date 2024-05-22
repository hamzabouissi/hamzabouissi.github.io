+++
title = 'Deploying gitea into kubernetes with custom domain'
date = 2023-05-20T16:12:38+01:00
tags = ['Kubernetes','giea','dns','helm','proxmox','bind9']
draft = false

+++

## Introudction

Hello, lately I have been trying to deploy a custom Docker image into my local Kubernetes cluster. It turned out I needed to host my Docker image on a container registry, either Docker Hub, which is not suitable for my use case, or deploy and use a local registry. During my research, I found Gitea, which I liked as it allows me to deploy all my projects on it and also host the containers.

## Prerequisite

    * kubernetes cluster
    * external server(S3,NFS) for dynamic provisioning
    * metallb installed

## Create PVC for NFS Server

With the help of Proxmox, I created a VM and configured it as an NFS server on 192.168.1.109. To use this server in our Kubernetes cluster, we need to create a StorageClass and then create a PVC that points to that class so pods can use it.

![pvc](/img/pvc.png)

```bash
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --create-namespace \
  --namespace nfs-provisioner \
  --set nfs.server=192.168.1.109 \
  --set nfs.path=/srv/public/nfs
```

Then we create a PVC, linking it to the storageClassName:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-test
  labels:
    storage.k8s.io/name: nfs
    storage.k8s.io/part-of: kubernetes-complete-reference
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 15Gi

```

If we want the deployment to store its volume in NFS, we define volumes with the argument persistentVolumeClaim = nfs-test. Here is an example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: nfs-test
  ...
```

## Deploy Gitea + PostgreSQL + Redis

To facilitate deploying resources into Kubernetes, we use Helm. With one command and changing a few values, we can deploy our resources. Before deploying, when I was reading the Gitea chart, I noticed Gitea requires PostgreSQL and Redis to be deployed alongside it to save its configs and states.

So let's create a file 'values-gitea.yaml' and add the default chart values.

Then change the following values:

```yaml
persistence:
  create: false
  claimName: nfs-test

postgresql-ha:
  enabled: false

postgresql:
  enabled: true

```

I disabled the deployment of Postgres-HA because I didn't need it for my use case. However, if you're deploying for your organization where multiple users are pushing and pulling, you may keep it enabled.

Now to deploy the Helm release with the modifications, run:

``` bash
  helm install gitea gitea-charts/gitea --values values-gitea.yaml
```

> **_NOTE:_** You need a cluster with an internet connection to pull the Docker images.

> **_NOTE:_** If you don't specify the namespace, it will choose the 'default' namespace.

Now, we wait until the images get pulled and deployed. You can watch the pods by running:

``` bash
  kubectl get pods -w
```

![k9s pods](/img/k9s_screenshoot.png)

After all the pods are deployed, to access Gitea, we need to either open a port or create an ingress. Let's try the port-forwarding mechanism for now to test the app:

``` bash
kubectl port-forward service/gitea-http 3000:3000
```

Now go to [http://localhost:3000](http://localhost:3000) and test, you can login as gitea_admin, password: r8sA8CPHD9!bt6d

To access Gitea from another pod, CoreDNS provides a default resolving mechanism in the form: service_name.namespace.svc.cluster.local. In our example, the DNS for Gitea is: gitea-http.default.

## Deploy Controller Nginx

For testing purposes, port-forwarding may be a good solution, but if we want a more reliable solution and even attach a domain with HTTPS to the Gitea service, we need an ingress.

To start with ingress, an ingress controller is needed. We will choose the most popular one: nginx-ingress.

following this [article](https://docs.nginx.com/nginx-ingress-controller/installation/installing-nic/installation-with-helm/), choosing helm install, I got the below command to run:

``` bash
  helm install my-release oci://ghcr.io/nginxinc/charts/nginx-ingress --version 1.2.1
```

If everything works as expected, you should see an ingress-controller service with an IP address from your load-balancer (MetalLB) pool of IP addresses.

![nginx-ingress service](/img/nginx_controller.png)

Great, to expose Gitea, we will change the previous chart file 'values-gitea.yaml' values:

```yaml
  ingress:
    enabled: true
    className: nginx
    hosts:
      - host: gitea.homelab.local
        paths:
          - path: /
            pathType: Prefix
```

here we instructed to create an ingress rule:

    - className is needed if you have multiple ingress controllers installed
    - the host part tell ingress to accept any request with domain or  host header : gitea.homelab.local and forward it to gitea instance  

To redeploy the release with the new configuration, we run:

``` bash
helm upgrade gitea gitea-charts/gitea --values values-gitea.yaml
```

If we check the ingresses, we can find Gitea ingress has been created. To test it, we will query the IP address of the ingress, supplying a custom host header: gitea.homelab.local.

``` bash
curl --header 'Host: gitea.homelab.local' ingress_ip_address
```

## Deploy bind9

You may notice that accessing Gitea from a browser isn't possible because the local DNS server doesn't have knowledge of the domain: homelab.local. The solution is either to modify the /etc/hosts file or create a CT in Proxmox and host a DNS server there.

I went for the second option, hosting a DNS server because my homelab may require a variety of services in the future, and I want them to be mapped to a domain for all the connected devices in my network.

For the DNS server, Pi-hole may be the most popular option for ad-blocking and adding DNS records, but I experienced a few bugs with serving DNS, so I went with the second option: Bind9.

I read this [article](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-20-04#step-2-configuring-the-primary-dns-server)

I created a CT in proxmox and assigned a static ip 192.168.1.114, don't use dhcp because it may change if CT restarted. So here are my configuration

**my local ip address**: 192.168.1.104, **ingress ip address**: 192.168.1.148

filename: /etc/bind/named.conf.options
```text
acl "trusted" {
  192.168.1.0/24;
};
options {
  directory "/var/cache/bind";

  // If there is a firewall between you and nameservers you want
  // to talk to, you may need to fix the firewall to allow multiple
  // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

  // If your ISP provided one or more IP addresses for stable
  // nameservers, you probably want to use them as forwarders.
  // Uncomment the following block, and insert the addresses replacing
  // the all-0's placeholder.
  recursion yes;
  allow-recursion { trusted; };
  listen-on { 192.168.1.114;};
  allow-transfer { none; };

  // forwarders {
  //      0.0.0.0;
  // };

  //========================================================================
  // If BIND logs error messages about the root key being expired,
  // you will need to update your keys.  See https://www.isc.org/bind-keys
  //========================================================================
  dnssec-validation auto;

  listen-on-v6 { any; };
};
```

filename: /etc/bind/named.conf.local
```text
zone "homelab.local" {
    type master;
    file "/etc/bind/zones/db.homelab.local";
};
zone "168.192.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.192.168";  # 192.168.0.0/24 subnet
};
```

filename: zones/db.homelab.local
```text 

;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     homelab.local. admin.homelab.local. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
; name servers - NS records
    IN      NS      ns1.homelab.local.

; name servers - A records
ns1.homelab.local.            IN      A       192.168.1.114


; name servers - A records
gitea.homelab.local.          IN      A       192.168.1.148
```

filename: /etc/bind/zones/db.168.192
```text
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns1.homelab.local. admin.homelab.local. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; name servers - NS records
      IN      NS      ns1.homelab.local.

; PTR Records
114.1   IN      PTR     ns1.homelab.local.    ; 192.168.1.114
148.1   IN      PTR     gitea.homelab.local.  ; 192.168.1.148

```

once the the bind9 configured and it's working, we need to add the dns server ip address as an additional one, am using NetworkManager:

![network_manager](/img/network_manager_dns.png)

## Add TLS

Now, if we try to login into to the Gitea registry using the below command:
``` bash
docker login gitea.homelab.local
```

It will return an error claiming the registry domain needs a TLS certificate. We can work around that by adding the registry domain to /etc/docker/daemon.json, but it would be more useful if we create a TLS certificate and append it to the domain.

We will start first by creating the cert. I chose mkcert because my first search led to it 😄.

```bash
mkcert gitea.homelab.local
```

It will generate two PEM files: a public key and a private key.

We will create a TLS secret and append the two created files from mkcert:

``` bash
kubectl create secret tls gitea-secret \
    --key gitea.homelab.local-key.pem \
    --cert gitea.homelab.local.pem
```

Finally, we append the gitea-secret into the ingress by changing the gitea-values.yaml file:

```yaml
tls:
    - secretName: gitea-secret
      hosts:
       - gitea.homelab.local
```

Now, we can visit gitea.homelab.local and login to gitea registry without issues.

![docker-login](/img/docker_login.png)

## Change nginx config for pushing the image

We deployed Gitea with one main purpose in mind: pushing containers to the registry. However, if we try building a local image and pushing it, you may face an error saying: "413 Request Entity Too Large"!

![413_push_image](/img/docker_push_413.png)

This is because by default Nginx imposes a limit of 1MB for uploading media files. To change that, we add an annotation for ingress to remove the limit:

```yaml
annotations:
    nginx.org/client-max-body-size: "0"
```

then we update the release chart

```bash
helm upgrade gitea gitea-charts/gitea --values values-gitea.yaml
```

Now, we can push the image: gitlab.homelab.local/gitea_admin/app:latest

![pushed_image](/img/pushed_image.png)

if you have created another user instead of gitea_admin, you can replace it in the above command.

## Add Bind9 Server in CoreDNS

We have done everything from deploying to adding TLS cert, but if we tried to create a deployment with the deployed image as an example

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: app
spec:
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: gitea.homelab.local/gitea_admin/app:latest
        ports:
        - containerPort: 80

```

after applying the yaml, if you run kubectl describe deployment/app_name
you may notice in the events section that it's stating pulling the image has failed, that's logical because kubernetes cluster doesn't know about our custom domain: **homelab.local**.

So to let kubernetes DNS server: CoreDNS, acknowledge our domain we gonna need a litle tweak into the CoreDNS config

we run the following command to open the editor with configmap:

```bash
kubectl edit configmap -n kube-system coredns
```

and then we add the reference to homelab.local

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 172.16.0.1
        cache 30
        loop
        reload
        loadbalance
    }
    homalb.local:53 {
        errors
        cache 30
        forward . 192.168.1.114
    }  

```

and for the CoreDNS to take effect, we will restart it with :
```bash
kubectl rollout restart -n kube-system deployment/coredns
```

Now, to test things out you can redeploy the previous deployed yaml or just run an alpine with nslookup

```bash
kubectl run --image=alpine:3.5 -it alpine-shell-1 -- nslookup gitea.homelab.local
```

it should return the ip address of the ingress.
