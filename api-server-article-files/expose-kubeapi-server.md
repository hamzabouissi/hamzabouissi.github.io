---
id: expose-kubeapi-server
title: "Expose Kubernetes API access publicly with OIDC and an IP allowlist"
---

# Expose Kubernetes API access publicly with OIDC and an IP allowlist

## Prerequisites

The client must have the following tools installed:

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./${KREW} install krew
)

# install kube-oidc-login plugin
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew install oidc-login

```

## Setup RBAC

for each customer we're going to have custom rbac permissions applied, here is the default manifest.

the following one allows: readonly, port-forward and exec permissions to pods under default namespace. 


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: port-forwarder
rules:
- apiGroups: [""]
  resources: ["pods", "pods/portforward"]
  verbs: ["get", "list", "create"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: oidc-kubectl-port-forward
  namespace: default 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: port-forwarder
subjects:
- kind: Group
  name: oidc:glueops-rocks:domain_glueops-kubectl-pfwd
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-exec
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create", "get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: oidc-kubectl-exec
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-exec
subjects:
- kind: Group
  name: oidc:glueops-rocks:domain_glueops-kubectl-exec
  apiGroup: rbac.authorization.k8s.io


---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: oidc-kubectl-readonly
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: Group
  name: oidc:glueops-rocks:domain_glueops-kubectl-reader
  apiGroup: rbac.authorization.k8s.io
```

## Set IngressRouteTCP with middleware

Now to expose kube-api to the public world, we need to use `IngressRouteTCP`, but first we must create a `MiddlewareTCP` to restrict access to certain IP addresses.

```yaml
apiVersion: traefik.io/v1alpha1
kind: MiddlewareTCP
metadata:
  name: kube-apiserver-ip-allowlist
  namespace: glueops-core-platform-traefik
spec:
  ipAllowList:
    sourceRange:
      - "x.x.x.x/32"
```

replace the `x.x.x.x` with IP address you want, note you can create as much IP address as you can

then let's have the `IngressRouteTCP` created

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: kube-apiserver-passthrough
  namespace: glueops-core-platform-traefik
  annotations:
    kubernetes.io/ingress.class: "platform-traefik"
    # ExternalDNS will create/update this DNS name to point at the Traefik LB target.
    external-dns.alpha.kubernetes.io/target: "platform-v2.domain"
spec:
  entryPoints:
    - websecure
  routes:
    - match: HostSNI(`kube-api.domain`)
      middlewares:
        - name: kube-apiserver-ip-allowlist
          namespace: glueops-core-platform-traefik
      services:
        - name: kube-apiserver-proxy
          port: 443
  tls:
    passthrough: true

---
apiVersion: v1
kind: Service
metadata:
  name: kube-apiserver-proxy
  namespace: glueops-core-platform-traefik
spec:
  ports:
    - name: https
      port: 443
      protocol: TCP
      targetPort: 443

```

## Setup config for kubectl access

In the current cluster, run the following. It will create a file `kubeconfig.yaml`, which you can then hand to the customer.

```bash

# 1. Grab the certificate data
CERT_DATA=$(kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')

# 2. Write the config to a file
cat <<EOF > kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
  - name: kubernetes
    cluster:
      server: https://kube-api.domain
      certificate-authority-data: "$CERT_DATA"
contexts:
  - name: kubectl-oidc@kubernetes
    context:
      cluster: kubernetes
      user: kubectl-oidc
current-context: kubectl-oidc@kubernetes
users:
  - name: kubectl-oidc
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1
        command: kubectl
        args:
          - oidc-login
          - get-token
          - --grant-type=device-code
          - --oidc-issuer-url=https://dex.domain
          - --oidc-client-id=kubectl
          - --oidc-extra-scope=profile
          - --oidc-extra-scope=email
          - --oidc-extra-scope=groups
EOF

```
