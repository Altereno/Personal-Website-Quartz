This is part 4 of [[Homelab Plans - Summer 2025]]
# Installation
I referenced the [official docs](https://woodpecker-ci.org/docs/intro) and the [helm chart values](https://github.com/woodpecker-ci/helm/blob/main/charts/woodpecker/values.yaml).
*Note: Gitea specific setup is [here](https://woodpecker-ci.org/docs/administration/configuration/forges/gitea). Specifically, I needed to create the Oauth2 Application under `/user/settings/applications` and also add the Woodpecker subdomain under `gitea.config.webhook.ALLOWED_HOSTS_LIST` in the Gitea helm values*
```yaml
server:
  env:
    WOODPECKER_HOST: "https://woodpecker.cluster.stevenchen.one"
    WOODPECKER_ADMIN: "steven"
    WOODPECKER_OPEN: true
    WOODPECKER_GITEA: true
    WOODPECKER_GITEA_URL: "https://gitea.cluster.stevenchen.one"
    WOODPECKER_GITEA_CLIENT: "clientclientclientclientclient"
    WOODPECKER_GITEA_SECRET: "secretsecretsecretsecretsecret"
  persistentVolume:
    size: "1Gi"
agent:
  env:
    WOODPECKER_BACKEND_K8S_VOLUME_SIZE: "1Gi"
  persistence:
    size: "1Gi"
```

I needed to create a privileged namespace since I wanted to use docker-in-docker (`dind`) to build my images. I don't fully understand Kubernetes security yet.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: woodpecker
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

I installed with helm:
```sh
kubectl apply -f namespace.yaml
helm upgrade --install \
    woodpecker oci://ghcr.io/woodpecker-ci/helm/woodpecker \
    --version 3.1.2 \
    --namespace woodpecker \
    --values=chart-values.yaml
```

And to expose the application:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: woodpecker-route
  namespace: woodpecker
spec:
  parentRefs:
  - name: main-gateway
    namespace: gateway-system
  hostnames:
  - "woodpecker.cluster.stevenchen.one"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: woodpecker-server
      port: 80
```

```sh
kubectl label namespaces woodpecker shared-gateway-access=true
kubectl apply -f http-route.yaml
```

# Pipeline configuration
Once the repository is enabled on the woodpecker web ui, it should create a webhook in Gitea. The default place it will check for pipeline configuration are:
- `.woodpecker/*.yaml`
- .woodpecker.yaml
*Note that `.yaml` and `.yml` both work*

Since I am using `dind`, I have to enable `Security` under:
`myrepo -> Settings -> Project -> Trusted`
This should allow Woodpecker to spin up a privileged pod to build my app.

The following is my current pipeline for my personal website:
```yaml
when:
  - event: push
    branch: main

steps:
  - name: Pull Quartz repo
    image: alpine/git
    commands:
      - git clone https://gitea.cluster.stevenchen.one/steven/quartz.git
  
  - name: Quartz generate and move
    image: node:24-bookworm-slim
    commands:
      - mv $CI_WORKSPACE/writeups/quartz/* quartz/content/
      - cd quartz
      - npm i
      - npx quartz build
      - mv public/* $CI_WORKSPACE/writeups/quartz/

  - name: Build NGINX and push to registry
    image: docker:24-dind
    privileged: true
    environment:
      DOCKER_TLS_CERTDIR: ""
      DOCKER_HOST: tcp://127.0.0.1:2375
      IMAGE: gitea.cluster.stevenchen.one/steven/website
      REGISTRY_USERNAME:
        from_secret: REGISTRY_USERNAME
      REGISTRY_TOKEN:
        from_secret: REGISTRY_TOKEN
    commands:
      - dockerd-entrypoint.sh &
      - |
        while ! docker info >/dev/null 2>&1; do
          echo "Waiting for Docker daemon..."
          sleep 1
        done
      - echo "$REGISTRY_TOKEN" | docker login gitea.cluster.stevenchen.one -u "$REGISTRY_USERNAME" --password-stdin
      - docker build -t $IMAGE:$CI_COMMIT_SHA .
      - docker push $IMAGE:$CI_COMMIT_SHA

  - name: Update gitops repo
    image: alpine/git
    environment:
      GITEA_TOKEN:
        from_secret: GITEA_TOKEN
    commands:
      - git config --global user.name "woodpecker"
      - git config --global user.email "ci@gitea.cluster.stevenchen.one"

      - git clone https://$GITEA_TOKEN:x-oauth-basic@gitea.cluster.stevenchen.one/steven/website-argo.git
      - cd website-argo/apps/website

      - apk update
      - apk add yq
      - yq -i '.spec.template.spec.containers[0].image = "gitea.cluster.stevenchen.one/steven/website:${CI_COMMIT_SHA}"' deployment.yaml

      - git commit -am "chore - update website image to ${CI_COMMIT_SHA}"
      - git push origin main

```

An overview of pipeline:
- Clone my personal website repo
- Clone the [Quartz](https://quartz.jzhao.xyz/) repo
- Move all the Markdown files into the Quartz repo
- Use `npm` to generate all the static files
- Use the Dockerfile to build the NGINX image
- Tag the image with the commit hash and push to the Gitea registry
- Update the hash in the GitOps repo for ArgoCD to pull

The `REGISTRY_USERNAME`, `REGISTRY_TOKEN` and `GITEA_TOKEN` are environment variables which is configurable under:
`myrepo -> Settings -> Secrets`
The rest of the environment variables were built-in. The list is found [here](https://woodpecker-ci.org/docs/usage/environment).

For the configurable secrets, I generated two tokens from Gitea, one with read/write for `package` and one with read/write for `repository`. The username is just the same my Gitea username. *Generation of tokens can be found under `/user/settings/applications`*

[Here](https://docs.gitea.com/usage/packages/container) are the docs for connecting to the Gitea registry.
