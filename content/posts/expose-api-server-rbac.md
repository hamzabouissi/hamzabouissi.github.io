+++
title = 'Expose Api-Server with Dex and Traefik'
date = 2026-07-08T15:24:10+01:00
draft = false
tags = ['Kubernetes', 'kube-apiserver', 'Dex', 'OIDC', 'Traefik', 'RBAC', 'kubeadm', 'kubectl']
[cover]
    image = 'img/kube-api-server-cover.jpg'
+++

## **Prologue**

For most GlueOps clusters, `kube-apiserver` sits behind a private network and the only way in is through our internal tooling. That works well until the moment a customer says: *“I just want to `kubectl get pods` on my own cluster.”*

At that point we have three bad options: ship them a long-lived `ServiceAccount` token (a credential we can never really rotate), give them a shared admin `kubeconfig` (auditing nightmare), or ask them to VPN in every time (friction we don’t want to charge them for).

What we actually want is boring and correct: **the customer logs in with their existing SSO identity, `kubectl` gets a short-lived token, `kube-apiserver` validates it, and RBAC decides what they’re allowed to touch.** No shared secrets, no long-lived credentials, and the public surface is small enough that we can sleep at night.

This post walks through how we wired that up on top of our existing GlueOps stack — **Traefik** as the ingress on the edge, **Dex** as the OIDC provider federating the customer’s identity, and **kubeadm** driving the api-server configuration. Every piece is boring on its own; the interesting part is how they fit together.

## **Opening the Gate: A TCP Passthrough to kube-apiserver**

The first instinct when exposing anything through Traefik is to reach for an `IngressRoute` and let Traefik terminate TLS for us. That instinct is wrong here.

`kube-apiserver` speaks TLS with its own certificate — the same certificate that `kubectl`, kubelets and every controller in the cluster already trust. If Traefik terminated TLS and re-encrypted, we’d either have to teach every client to trust Traefik’s cert, or run api-server without TLS behind the proxy. Both are worse than what we started with.

The right primitive is `IngressRouteTCP` with `tls.passthrough: true`. Traefik becomes a dumb L4 forwarder that routes by SNI and hands the raw TLS stream to `kube-apiserver`. The api-server’s own certificate remains the one the client validates, and nothing about the client-side trust story changes.

Before we open the gate, we bolt on an IP allowlist. Even though every request must still authenticate through OIDC and pass RBAC, we don’t want the public internet spraying login attempts and CVE probes at the api-server. `MiddlewareTCP` gives us a simple source-range filter:

```yaml
apiVersion: traefik.io/v1alpha1
kind: MiddlewareTCP
metadata:
  name: kube-apiserver-ip-allowlist
  namespace: traefik
spec:
  ipAllowList:
    sourceRange:
      - "x.x.x.x/32"
```

Replace `x.x.x.x/32` with whatever CIDRs the customer actually connects from — you can list as many as you need. Then the `IngressRouteTCP` that ties it all together:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: kube-apiserver-passthrough
  namespace: traefik
spec:
  entryPoints:
    - websecure
  routes:
    - match: HostSNI(`kube-api.domain`)
      middlewares:
        - name: kube-apiserver-ip-allowlist
          namespace: traefik
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
  namespace: traefik
spec:
  ports:
    - name: https
      port: 443
      protocol: TCP
      targetPort: 443
```

Two subtle points worth calling out:

1. The `HostSNI` match works precisely because we set `tls.passthrough: true`. Traefik peeks at the ClientHello, sees the SNI, and routes accordingly — it never breaks the TLS envelope open.
2. The `Service` in this snippet has no selector. In our clusters we point it at the api-server’s endpoints; if you’re running vanilla kubeadm, you can either add a selector matching the api-server pods or manage the `Endpoints`/`EndpointSlice` object directly.

At this point, the door is open — but there’s nothing behind it yet that knows how to check IDs.

## **A Herald at the Door: Registering kubectl in Dex**

Dex is already the OIDC broker for the rest of our platform (Argo CD, Grafana, Pomerium, etc.). All we need to do is teach it that `kubectl` is a legitimate client that will come knocking.

In our Dex helm values:

```yaml
staticClients:
  - id: kubectl
    name: kubectl
    public: true
```

The `public: true` bit is the important one. `kubectl` is a native binary running on the user’s laptop — there’s no server-side place to safely store a client secret. Marking it public tells Dex to accept the **OAuth 2.0 device-code flow** for this client and skip the client secret check. That’s the same flow the [`kubelogin`](https://github.com/int128/kubelogin) plugin uses: pop open a browser, complete the login, and drop a short-lived ID token back into `kubectl`’s credential exec.

No callback URLs, no client secret to rotate — the herald is announced.

## **Teaching the Guard to Read the Herald’s Seal: Kubeadm Authentication Config**

The api-server now needs two things: it has to be reachable at the public hostname, and it has to know how to validate the JWTs Dex hands out.

### The certificate SAN

Since we’re about to serve TLS on `kube-api.domain`, the api-server’s serving certificate must include that name. Otherwise every `kubectl` call will fail the SNI/hostname check before authentication even runs. In the kubeadm `ClusterConfiguration`, we extend `certSANs`:

```yaml
apiServer:
  certSANs:
    - kube-api.domain
```

### The AuthenticationConfiguration

Historically you’d configure OIDC on the api-server with a pile of `--oidc-*` CLI flags. Starting from `AuthenticationConfiguration` (beta since 1.30), Kubernetes lets you point the api-server at a single YAML file that declares one or more JWT issuers, along with claim mappings and validation rules. We use it because it composes better with our GitOps flow and because the CEL validation rules let us enforce things like `email_verified` without writing a webhook.

Here’s the config we drop in — parameterized because Ansible fills in `domain_name` per environment:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AuthenticationConfiguration
jwt:
  - issuer:
      url: https://dex.{{ domain_name }}
      audiences:
        - kubectl
    claimMappings:
      username:
        claim: email
        prefix: "oidc:"
      groups:
        claim: groups
        prefix: "oidc:"
    claimValidationRules:
    - expression: "has(claims.email_verified) && claims.email_verified == true"
      message: "Access denied: Your email address must be verified in Dex."
```

A few things worth noting:

- `audiences: [kubectl]` mirrors the `client_id` we registered in Dex. If a token was issued for another client — say, Argo CD — the api-server will reject it. This scoping matters.
- The `oidc:` prefix on `username` and `groups` is what shows up in RBAC bindings later. It also prevents any collision with local ServiceAccount or system group names.
- The CEL `claimValidationRules` cleanly refuses tokens where `email_verified` isn’t `true`. If someone can register a fresh account in your IdP and hit the api-server before they’ve confirmed their email, that’s a real hole — this closes it.

### Wiring the file into the api-server

There’s a small operational catch: `kube-apiserver` runs as a static pod, and it can only read files that live inside a directory that’s already mounted into the pod. The two directories kubeadm mounts by default are `/etc/kubernetes/pki` and `/etc/kubernetes`. To keep the config close to the other trust material and avoid touching the static-pod manifest by hand, we drop the file at `/etc/kubernetes/pki/authentication-config.yaml`.

Then we tell the api-server about it via an `extraArgs` entry in the kubeadm config:

```yaml
apiServer:
  extraArgs:
    - name: "authentication-config"
      value: "/etc/kubernetes/pki/authentication-config.yaml"
```

On a fresh `kubeadm init`, that’s all you need — the resulting static-pod manifest will already include the flag and the mount. On an existing cluster, the config change alone isn’t enough; we still need the api-server certificate to be regenerated so the new SAN takes effect. That’s the next section.

## **The Ceremony of Rotation: Reissuing Certs on an Existing Cluster**

If you’re standing up a brand-new cluster, skip this section. `kubeadm init` will honor the new `certSANs` and `extraArgs` on the first pass.

If you’re modifying a cluster that’s already running — which is the usual case for us — kubeadm won’t retroactively add SANs to a cert that already exists on disk. You have to force a regeneration. Because we use Ansible to manage our kubeadm clusters, we wrapped the whole ceremony in a role. The full task file lives here: [rotate-certs-with-config.yaml](https://github.com/GlueOps/GlueKube/blob/main/ansible/roles/master/tasks/rotate-certs-with-config.yaml). The shape of it is:

```yaml
- name: Generate Kubeadm configuration
  ansible.builtin.template:
    src: "kubeadm-stacked-config.yaml.j2"
    dest: "{{ kubeadm_config_file }}"
    owner: root
    group: root
    mode: "0644"

- name: Generate Authentication Configuration
  ansible.builtin.template:
    src: "authentication-configuration.yaml.j2"
    dest: "/etc/kubernetes/pki/authentication-config.yaml"
    owner: root
    group: root
    mode: "0644"

- name: Remove existing API server cert files to force regeneration from config
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop:
    - /etc/kubernetes/pki/apiserver.crt
    - /etc/kubernetes/pki/apiserver.key

- name: Regenerate API server certificate from kubeadm config
  ansible.builtin.command: "kubeadm init phase certs apiserver --config {{ kubeadm_config_file }}"

- name: Upload kubeadm ClusterConfiguration to kubeadm-config ConfigMap
  ansible.builtin.command: "kubeadm init phase upload-config kubeadm --config {{ kubeadm_config_file }}"
  when: inventory_hostname == groups['masters'][0]

- name: Regenerate API server static pod manifest from kubeadm config
  ansible.builtin.command: "kubeadm init phase control-plane apiserver --config {{ kubeadm_config_file }}"

- name: Force restart of kube-apiserver static pod
  ansible.builtin.command: "mv /etc/kubernetes/manifests/kube-apiserver.yaml /opt/kubernetes/manifests/kube-apiserver.yaml"

- name: Wait until old kube-apiserver container is removed
  ansible.builtin.shell: "ctr -n k8s.io containers list | grep kube-apiserver || true"
  register: old_apiserver_container
  retries: 18
  delay: 5
  until: old_apiserver_container.stdout == ""
  changed_when: false

- name: Restore kube-apiserver manifest to recreate static pod
  ansible.builtin.command: "mv /opt/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml"
  args:
    removes: /opt/kubernetes/manifests/kube-apiserver.yaml

- name: Wait for API server to become ready
  ansible.builtin.shell: |
    export KUBECONFIG=/etc/kubernetes/admin.conf
    kubectl get --raw='/readyz'
  register: apiserver_ready
  retries: 18
  delay: 10
  until: apiserver_ready.rc == 0 and ('ok' in apiserver_ready.stdout)
  changed_when: false
```

The order matters more than it looks. In particular:

- We delete `apiserver.crt`/`apiserver.key` **before** running `kubeadm init phase certs apiserver`. Kubeadm otherwise sees an existing cert and no-ops.
- `upload-config kubeadm` runs on exactly one master (`groups['masters'][0]`). Without that guard, three masters race to write the same `kubeadm-config` ConfigMap.
- The manifest is moved out of `/etc/kubernetes/manifests` and then back in. The kubelet watches that directory as its trigger for static-pod restart; a plain `restart` of the container won’t pick up a new mounted file. Moving the manifest out kills the pod, moving it back in recreates it against the updated config.

Once `/readyz` returns `ok`, the api-server is running with the new SAN, the new `authentication-config`, and the OIDC gate is live.

## **The Scroll of Permissions: RBAC for OIDC Groups**

At this point a validly-authenticated Dex user can reach the api-server, but they can’t do anything — no bindings, no permissions. That’s intentional. RBAC is where per-customer policy actually lives.

Because our `claimMappings.groups` mapped Dex groups to `oidc:<group>`, and we bind against those group names directly. The default set we ship gives read-only access, plus targeted `exec` and `port-forward` on pods in the `default` namespace:

```yaml
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
  name: oidc:kubectl-reader
  apiGroup: rbac.authorization.k8s.io
```

Three separate groups, three separate `RoleBinding`s, all namespaced to `default`. That gives us fine-grained knobs — a customer that only needs to `kubectl logs -f` a pod gets slotted into `...-kubectl-reader` alone, and can’t suddenly `exec` into a pod because someone forgot to remove them from another group.

We use `ClusterRole` even for namespaced permissions because it’s reusable across bindings (the built-in `view` ClusterRole is the canonical example); the `RoleBinding` is what actually scopes the grant to the `default` namespace.

For customers that need broader access, we template out variations of this file — more namespaces, additional verbs, or a `ClusterRoleBinding` where they genuinely need cluster-wide read. The mental model stays the same: **Dex groups in, RBAC subjects out**.

## **Handing the Key to the Customer: The Kubeconfig**

The last piece is the artifact the customer actually uses. We generate it once on the cluster and hand it over — no shared secret embedded, no static token, just a pointer to the OIDC flow they’ll complete on their own machine.

First, they need `kubectl` with the [`oidc-login`](https://github.com/int128/kubelogin) plugin (installed via `krew`):

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

Then, on the cluster, we grab the CA data from an existing admin kubeconfig and emit a customer-facing one:

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

Two things fall out of this shape:

- The `exec` credential plugin means `kubectl` shells out to `kubectl oidc-login get-token` on every call, caches the token, and refreshes it via the OIDC flow when it expires. No renewals to babysit.
- `--grant-type=device-code` matches the `public: true` client we registered in Dex. The user sees a URL and a short code, completes the login in their browser, and control returns to their terminal.
- The `--oidc-extra-scope=groups` is not cosmetic — without it, Dex won’t include the `groups` claim, and every RBAC binding we set up in the previous section would silently miss.

The `kubeconfig.yaml` we send the customer contains no secrets. If they lose it, no one gains any access — the actual gate is still the Dex login. That’s the whole point.

## **Summary**

The story of exposing `kube-apiserver` for our customers came down to five moving parts, each doing exactly one job:

1. **Traefik + `IngressRouteTCP` with TLS passthrough** — the door. Traefik forwards raw TLS by SNI, so the api-server’s own cert stays the trust anchor. A `MiddlewareTCP` `ipAllowList` keeps the public surface small.
2. **Dex `staticClient` for `kubectl`** — the herald. A `public` client that speaks the OAuth device-code flow, no secrets to share.
3. **Kubeadm `AuthenticationConfiguration` + extended `certSANs`** — the guard. The api-server learns to trust JWTs from Dex, mapped to `oidc:` usernames and groups, with a CEL rule that enforces `email_verified`.
4. **RBAC `RoleBinding`s to `oidc:...` groups** — the scroll. Per-customer permissions are just group memberships in Dex; RBAC decides what each group is allowed to touch.
5. **A customer-facing kubeconfig using `oidc-login`** — the key. No embedded credentials; the customer authenticates through their own browser every time their token expires.

Every credential in this chain is short-lived, every permission is declarative, and the customer never holds a secret we can’t revoke by removing them from a group. That was the whole goal — and now `kubectl get pods` on a customer cluster just works, safely.

Referenced repositories and files:

- GlueKube: [https://github.com/GlueOps/GlueKube](https://github.com/GlueOps/GlueKube)
- Cert rotation Ansible task: [`ansible/roles/master/tasks/rotate-certs-with-config.yaml`](https://github.com/GlueOps/GlueKube/blob/main/ansible/roles/master/tasks/rotate-certs-with-config.yaml)
- `kubelogin` (`kubectl oidc-login`): [https://github.com/int128/kubelogin](https://github.com/int128/kubelogin)
- Kubernetes `AuthenticationConfiguration`: [https://kubernetes.io/docs/reference/access-authn-authz/authentication/#using-authentication-configuration](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#using-authentication-configuration)
