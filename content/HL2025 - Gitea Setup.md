This is part 3 of [[Homelab Plans - Summer 2025]]
# Installation
I referenced the [Gitea helm chart configuration details](https://gitea.com/gitea/helm-gitea). *Note: Anything added from the [Cheat Sheet](https://docs.gitea.com/administration/config-cheat-sheet) can be under `gitea.config`*
```yaml
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: ""
  hostAliases: []
namespace: ""
replicaCount: 1

strategy:
  type: "RollingUpdate"
  rollingUpdate:
    maxSurge: "100%"
    maxUnavailable: 0
clusterDomain: cluster.local

image:
  registry: "docker.gitea.com"
  repository: gitea
  tag: ""
  digest: ""
  pullPolicy: IfNotPresent
  rootless: true
  fullOverride: ""
imagePullSecrets: []

podSecurityContext:
  fsGroup: 1000
containerSecurityContext: {}
securityContext: {}
podDisruptionBudget: {}

service:
  http:
    type: ClusterIP
    port: 3000
    clusterIP:
    loadBalancerIP:
    nodePort:
    externalTrafficPolicy:
    externalIPs:
    ipFamilyPolicy:
    ipFamilies:
    loadBalancerSourceRanges: []
    annotations: {}
    labels: {}
    loadBalancerClass:
  ssh:
    type: ClusterIP
    port: 22
    clusterIP:
    loadBalancerIP:
    nodePort:
    externalTrafficPolicy:
    externalIPs:
    ipFamilyPolicy:
    ipFamilies:
    hostPort:
    loadBalancerSourceRanges: []
    annotations: {}
    labels: {}
    loadBalancerClass:

ingress:
  enabled: false
  className: ""
  pathType: Prefix
  annotations: {}
  hosts:
    - host: git.stevenchen.one
      paths:
        - path: /
  tls: []

resources: {}
schedulerName: ""
nodeSelector: {}
tolerations: []
affinity: {}
topologySpreadConstraints: []
dnsConfig: {}
priorityClassName: ""
deployment:
  env: []
  terminationGracePeriodSeconds: 60
  labels: {}
  annotations: {}

serviceAccount:
  create: false
  name: ""
  automountServiceAccountToken: false
  imagePullSecrets: []
  annotations: {}
  labels: {}

persistence:
  enabled: true
  create: true
  mount: true
  claimName: gitea-shared-storage
  size: 1Gi
  accessModes:
    - ReadWriteOnce
  labels: {}
  storageClass:
  subPath:
  volumeName: ""
  annotations:
    helm.sh/resource-policy: keep
extraContainers: []
extraVolumes: []
extraContainerVolumeMounts: []
extraInitVolumeMounts: []
extraVolumeMounts: []

initPreScript: ""
initContainersScriptsVolumeMountPath: "/usr/sbinx"
initContainers:
  resources:
    limits: {}
    requests:
      cpu: 100m
      memory: 128Mi

signing:
  enabled: false
  gpgHome: /data/git/.gnupg
  privateKey: ""
  existingSecret: ""

gitea:
  admin:
    existingSecret:
    username: steven
    password: r8sA8CPHD9!bt6d
    email: "sc0.steven.chen@gmail.com"
    passwordMode: keepUpdated
  metrics:
    enabled: false
    token:
    serviceMonitor:
      enabled: false
      interval: ""
      relabelings: []
      scheme: ""
      scrapeTimeout: ""
      tlsConfig: {}
  ldap: []
  oauth: []
  config:
    server:
      SSH_PORT: 22
      SSH_LISTEN_PORT: 2222
      PROTOCOL: http
      ROOT_URL: "https://gitea.cluster.stevenchen.one"
    webhook:
      ALLOWED_HOST_LIST: "woodpecker.cluster.stevenchen.one"
  additionalConfigSources: []
  additionalConfigFromEnvs: []
  podAnnotations: {}
  ssh:
    logLevel: "INFO"
  livenessProbe:
    enabled: true
    tcpSocket:
      port: http
    initialDelaySeconds: 200
    timeoutSeconds: 1
    periodSeconds: 10
    successThreshold: 1
    failureThreshold: 10
  readinessProbe:
    enabled: true
    tcpSocket:
      port: http
    initialDelaySeconds: 5
    timeoutSeconds: 1
    periodSeconds: 10
    successThreshold: 1
    failureThreshold: 3
  startupProbe:
    enabled: false
    tcpSocket:
      port: http
    initialDelaySeconds: 60
    timeoutSeconds: 1
    periodSeconds: 10
    successThreshold: 1
    failureThreshold: 10

valkey-cluster:
  enabled: true
  usePassword: false
  usePasswordFiles: false
  cluster:
    nodes: 2
    replicas: 0
  service:
    ports:
      valkey: 6379
  persistence:
    enabled: true
    size: 1Gi

postgresql-ha:
  global:
    postgresql:
      database: gitea
      password: gitea
      username: gitea
  enabled: true
  postgresql:
    repmgrPassword: changeme2
    postgresPassword: changeme1
    password: changeme4
  pgpool:
    adminPassword: changeme3
    srCheckPassword: changeme4
  service:
    ports:
      postgresql: 5432
  persistence:
    size: 1Gi

checkDeprecation: true
test:
  enabled: true
  image:
    name: busybox
    tag: latest
extraDeploy: []
```

To install with helm:
```sh
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
helm upgrade --install \
    gitea gitea-charts/gitea \
    --namespace gitea \
    --create-namespace \
    --values=chart-values.yaml
```

For the `HTTPRoute`:
```sh
kubectl label namespaces gitea shared-gateway-access=true
kubectl apply -f http-route.yaml
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gitea-route
  namespace: gitea
spec:
  parentRefs:
  - name: main-gateway
    namespace: gateway-system
  hostnames:
  - "gitea.cluster.stevenchen.one"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: gitea-http
      port: 3000
```

With Gitea up, I mirrored my repositories from Github.