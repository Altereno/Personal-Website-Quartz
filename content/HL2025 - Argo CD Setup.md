This is part 5 of [[Homelab Plans - Summer 2025]]
# Installation
I referenced the [Argo CD getting started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/).

I needed to create the namespace first and add a label to allow gateway access
```sh
kubectl create namespace argocd
kubectl label ns argocd shared-gateway-access=true
```

I applied the manifest.
```sh
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

To proxy the website through the gateway, I had to enable the insecure options and restart the deployment to apply changes.
```sh
kubectl patch configmap argocd-cmd-params-cm -n argocd --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl rollout restart deployment argocd-server -n argocd
```

The initial admin password is stored in a secret, I had to extract the secret and base64 decode it.
```sh
kubectl get secrets -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode
```

For the `HTTPRoute`:
```sh
kubectl apply -f http-route.yaml
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: argocd-route
  namespace: argocd
spec:
  parentRefs:
  - name: main-gateway
    namespace: gateway-system
  hostnames:
  - "argocd.cluster.stevenchen.one"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: argocd-server
      port: 80
```

# Configuration
The goal is to have my website automatically deploy when changes are made. Before writing the Argo CD application manifest, I wrote all the necessary configurations for my website, as if I were to manually deploy it.

Here is the `Deployment`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: personal-website
  namespace: website
spec:
  selector:
    matchLabels:
      app: personal-website
  replicas: 2
  template:
    metadata:
      labels:
        app: personal-website
    spec:
      containers:
        - name: nginx
          image: gitea.cluster.stevenchen.one/steven/website:6b3993618d3e3a90bb1647561ba1243521e1369f
          ports:
            - containerPort: 80

```

Here is the `Service`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: personal-website-service
  namespace: website
spec:
  selector:
    app: personal-website
  ports:
    - protocol: TCP
      port: 80
  type: ClusterIP
```

And finally the `HTTPRoute`:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: personal-website-route
  namespace: website
spec:
  parentRefs:
    - name: main-gateway
      namespace: gateway-system
  hostnames:
    - "steven.cluster.stevenchen.one"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: personal-website-service
          port: 80
```

This is my current `Application` for Argo CD:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: website
  namespace: argocd
spec:
  destination:
    namespace: website
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://gitea.cluster.stevenchen.one/steven/website-argo.git
    targetRevision: main
    path: apps/website
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    managedNamespaceMetadata:
      labels:
        shared-gateway-access: "true"
    automated:
      selfHeal: true
      prune: true
```
This should automatically sync with the repository when I make changes. After applying this, the Argo CD web ui should show the application being deployed.

To verify the initial deployment, I just visited the site I configured in the `HTTPRoute`.
# Conclusion
The final flow for my personal website pipeline is:
- Make change to code
- Push to repository on Gitea
- Woodpecker CI runs and builds the NGINX image
- Woodpecker CI pushes image to Gitea registry and updates the GitOps repository
- Argo CD notices that the application is out of sync and starts syncing
- My personal website is updated