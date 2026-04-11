This is part 2 of [[Homelab Plans - Summer 2025]]
# Introduction
To access the services that I want to host, I need to understand networking (and storage?) in Kubernetes...
## Plan
- Setup the Cilium CNI
- Setup the bare metal load balancer
- Deploy a test site with certificates from [cert-manager](https://cert-manager.io/)
- Setup the Longhorn CSI?
## Notes
- Since I am running a bare metal cluster, I will need to provide my own load balancer
- Cilium has support for a bare metal load balancer, but will need extra configuration
- GatewayAPI vs Ingress?
# Cilium
*Originally the CNI setup was part of the Terraform modules but I have since moved it here*

From part 1, the `kubectl` binary should be available. We also need to install `helm` using [homebrew](https://formulae.brew.sh/formula/helm#default).

Talos comes default with [Flannel](https://github.com/flannel-io/flannel) plus kube-proxy, but I opted to disable them and instead install [Cilium](https://cilium.io/). I saw that Cilium supported the functionality that [MetalLB](https://metallb.io/) provided and also has a way to create a `GatewayClass`. *[Docs for the GatewayAPI](https://gateway-api.sigs.k8s.io/concepts/api-overview/)*

**Note: As of version 1.17.5, [Cilium does not seem to support the `TCPRoute` CRD](https://github.com/cilium/cilium/issues/21929), I think this might be needed later for when I configure SSH for Gitea**
## Installation
In the Talos machine configurations in [[HL2025 - Kubernetes on Talos (p1)]], I had already disabled the default CNI and kube-proxy.

*These two steps were written into the `apply-cilium.sh` script.*
- I installed the CRDs for the GatewayAPI from [here](https://gateway-api.sigs.k8s.io/guides/#install-standard-channel).
- To install Cilium, I utilized helm.
```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml

helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade --install \
    cilium cilium/cilium \
    --version 1.17.5 \
    --namespace kube-system \
    --values=chart-values.yaml
```
For the values, I grabbed most of the values from the [Talos guide](https://www.talos.dev/v1.10/kubernetes-guides/network/deploying-cilium/#method-1-helm-install).
```yaml
ipam:
  mode: kubernetes
kubeProxyReplacement: true
securityContext:
  capabilities:
    ciliumAgent:
      - CHOWN
      - KILL
      - NET_ADMIN
      - NET_RAW
      - IPC_LOCK
      - SYS_ADMIN
      - SYS_RESOURCE
      - DAC_OVERRIDE
      - FOWNER
      - SETGID
      - SETUID
    cleanCiliumState:
      - NET_ADMIN
      - SYS_ADMIN
      - SYS_RESOURCE
cgroup:
  autoMount:
    enabled: false
  hostRoot: /sys/fs/cgroup
k8sServiceHost: localhost
k8sServicePort: 7445
gatewayAPI:
  enabled: true
  enableAlpn: true
  enableAppProtocol: true
l2announcements:
  enabled: true
k8sClientRateLimit:
  qps: 20
  burst: 40
```

AFAIK, the two ways to expose a service externally in Kubernetes is either by using a [`NodePort`](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) or by using a [`LoadBalancer`](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer). The `NodePort` should open a port on all the nodes, and route the traffic internally to the pod, while the `LoadBalancer` should give me a single point of entry and route it to the pod.

For a bare metal load balancer, there seems to be two ways to achieve a stable IP address:
- L2 ARP
- BGP
While my PfSense router does support BGP through an additional package, I chose to just use the L2 ARP. 

I modified the example from the following Cilium docs:
- [L2 Announcements / L2 Aware LB](https://docs.cilium.io/en/latest/network/l2-announcements/)
- [LoadBalancer IP Address Management (LB IPAM)](https://docs.cilium.io/en/stable/network/lb-ipam/)

I added this to the helm values. *already included in the yaml above*
```yaml
l2announcements:
  enabled: true
k8sClientRateLimit:
  qps: 20
  burst: 40
```

I then applied the following file:
```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-policy
spec:
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  interfaces:
  - ^eth[0-9]+
  externalIPs: true
  loadBalancerIPs: true
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "pool"
spec:
  blocks:
  - cidr: "10.0.10.224/28"
```

The first block specifies the L2 announcement policy. From the example, it excludes the control plane nodes from the ARP process, and uses regex to match on all network interfaces that are enumerated with the `eth` prefix. The first second creates the pool that the load balance can use, which we can verify by running `kubectl get ippools`. 
## Testing
In my current lab, I have an internal reverse proxy that terminates all the TLS. I wanted to be able to replicate this with this new cluster.
### Installing cert-manager
I will be referencing the [official docs](https://cert-manager.io/docs/installation/) for this.

Installation will be done through helm:
```sh
helm repo add jetstack https://charts.jetstack.io --force-update
helm upgrade --install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.18.2 \
    --set config.apiVersion="controller.config.cert-manager.io/v1alpha1" \
    --set config.kind="ControllerConfiguration" \
    --set config.enableGatewayAPI=true \
    --set crds.enabled=true
```
*I have added the extra options from [here](https://cert-manager.io/docs/usage/gateway/) to get support for Gateway API CRDs*

### Configuring the ClusterIssuer
I am using the [ACME issuer](https://cert-manager.io/docs/configuration/acme/). List of issuer can be found [here](https://cert-manager.io/docs/configuration/issuers/).

I will be using the `DNS01` challenge instead of `HTTP01`, which will require ACME to have access to my DNS provider. I am using Cloudflare currently.

To obtain the API Token from Cloudflare:
- Log into Cloudflare
- On the top right:
	- Click the avatar
	- Select `Profile`
- On the left side:
	- Select `API Tokens`
- Select `Create Token`
- At the bottom:
	- Select `Create Custom Token`
- Give the token a name
- For `Permissions`:
	- Zone:
		- Zone:
			- Read
		- DNS
			- Edit
- For `Zone Resources`:
	- Include:
		- All zones
- Save the token somewhere safe

To create the secret:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-key: keykeykeykeykeykeykey
```

To create the ClusterIssuer:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cloudflare-clusterissuer
spec:
  acme:
    email: emailemailemail@example.com
    profile: tlsserver
    # server: https://acme-staging-v02.api.letsencrypt.org/directory
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-key
    solvers:
      - dns01:
          cloudflare:
            email: emailemailemail@example.com
            apiTokenSecretRef:
              name: cloudflare-api-key-secret
              key: api-key
```

Notes:
- I commented out the staging server in place for the production one, for testing purposes it is recommended to use staging before prod to prevent running into rate limits.
- I used a `ClusterIssuer` instead of an `Issuer` because Issuers are tied down their namespace (I think)
- The docs use `apiKeySecretRef`, which won't work. I had to replace it with `apiTokenSecretRef`. [Github Issue](https://github.com/cert-manager/cert-manager/issues/2384#issuecomment-575301692).

### Whoami
Now to deploy a test application. I will be using the [whoami](https://github.com/traefik/whoami).

 Using this yaml:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: whoami-example
  labels:
    shared-gateway-access: true
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: whoami-cert
  namespace: whoami-example
spec:
  secretName: whoami-cert
  dnsNames:
    - "whoami.cluster.stevenchen.one"
  issuerRef:
    name: cloudflare-clusterissuer
    kind: ClusterIssuer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: whoami-example
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          env:
            - name: WHOAMI_PORT_NUMBER
              value: "8080"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: whoami-example
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: whoami
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: whoami
  namespace: whoami-example
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    port: 443
    protocol: HTTPS
    tls:
      certificateRefs:
      - kind: Secret
        name: whoami-cert
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: whoami
  namespace: whoami-example
spec:
  parentRefs:
  - name: whoami
  hostnames:
  - "whoami.cluster.stevenchen.one"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: whoami
      port: 80
```

To visit the website, I had to first find the IP of the gateway using `kubectl get gateway`, then add the entry to my `/etc/hosts` file.
### Whoami (Wildcard)
From the [gateway usage page](https://cert-manager.io/docs/usage/gateway/), we can tell cert-manger to generate a certificate for the gateway using annotations.

Here is the same `whoami` app but with the annotations:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: whoami-example
  labels:
    shared-gateway-access: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: whoami-example
spec:
  replicas: 2
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          env:
            - name: WHOAMI_PORT_NUMBER
              value: "8080"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: whoami-example
spec:
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: whoami
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: whoami
  namespace: whoami-example
  annotations:
    cert-manager.io/cluster-issuer: cloudflare-clusterissuer
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    hostname: "*.cluster.stevenchen.one"
    port: 443
    protocol: HTTPS
    tls:
      certificateRefs:
      - kind: Secret
        name: wildcard-cert
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: whoami
  namespace: whoami-example
spec:
  parentRefs:
  - name: whoami
  hostnames:
  - "whoami.cluster.stevenchen.one"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: whoami
      port: 80
```

I use the wildcard cert for my current internal services. I will use the same for all my services hosted on Kubernetes.
### Main Gateway
I created a main gateway for all my services with this yaml:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gateway-system
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway  
  namespace: gateway-system
  annotations:
    cert-manager.io/cluster-issuer: cloudflare-clusterissuer
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    hostname: "*.cluster.stevenchen.one"
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            shared-gateway-access: "true"
    tls:
      certificateRefs:
      - kind: Secret
        name: wildcard-cert
```
This should allow me to create an `HTTPRoute` for all my apps. Note that the namespace that the `HTTPRoute` is in needs to have the `shared-gateway-access` label set to `true`. I found this [here](https://gateway-api.sigs.k8s.io/guides/multiple-ns/#shared-gateway).
### Hubble
To expose the Hubble web ui, I used this `HTTPRoute`:
```sh
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hubble-route
  namespace: kube-system
spec:
  parentRefs:
  - name: main-gateway
    namespace: gateway-system
  hostnames:
  - "hubble.cluster.stevenchen.one"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: hubble-ui
      port: 80
```

I applied the label and route with:
```sh
kubectl label namespaces kube-system shared-gateway-access=true
kubectl apply -f httproute.yaml
```
### Personal Notes
Notes:
- To check for any issues with ACME, I used this **very helpful** [resource](https://cert-manager.io/docs/troubleshooting/acme/).
- Current understanding of this:
	- For the certificate:
		- We create the `Certificate`
		- cert-manager creates a `CertificateRequest`
		- ACME issuer creates an `Order`
		- The `Order` creates the `Challenge`
		- cert-manager solves the `Challenge` using the `ClusterIssuer`
		- We get the certificate?
	- For the app:
		- Cilium creates the `GatewayClass`
		- We create a `Gateway` under the `GatewayClass`
		- We apply the app `Deployment`
			- The `Deployment` contains the `ReplicaSet`
				- The `ReplicaSet` contains the `Pod`
		- We create a `Service` that references the `Deployment` which gives us a `ClusterIP`
		- We create the `HTTPRoute` which references the `Gateway` and the `Service`
# Longhorn
I have not explored Longhorn yet, but installation steps were taken from the [official docs](https://longhorn.io/docs/1.9.0/advanced-resources/os-distro-specific/talos-linux-support/):

*Originally the CSI setup was part of the Terraform modules but I have since moved it here, all the machine configurations were already applied by now*

I need to create the privileged namespace:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: longhorn-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

Then I install with helm using default values:
```sh
helm repo add longhorn https://charts.longhorn.io
helm repo update
kubectl apply -f longhorn-namespace.yaml
helm upgrade --install \
    longhorn longhorn/longhorn \
    --version 1.9.0 \
    --namespace longhorn-system
```

To expose the web UI, I used this `HTTPRoute`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: longhorn-route
  namespace: longhorn-system
spec:
  parentRefs:
  - name: main-gateway
    namespace: gateway-system
  hostnames:
  - "longhorn.cluster.stevenchen.one"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: longhorn-frontend
      port: 80

```

I applied the label and route with:
```sh
kubectl label namespaces longhorn-system shared-gateway-access=true
kubectl apply -f httproute.yaml
```