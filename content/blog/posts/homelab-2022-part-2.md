+++
title = "Homelab 2022 Part 2"
date = 2022-03-18T17:01:01+01:00
images = []
tags = ["devops", "kubernets", "helm", "ansible"]
categories = ["projects"]
draft = false
type = "posts"
+++

- [Storage](#storage)
- [Monitoring and logs](#monitoring-and-logs)
  - [Metrics](#metrics)
  - [Logs](#logs)
- [CD - argocd](#cd---argocd)
  - [How to setup argocd?](#how-to-setup-argocd)
  - [Configure access to repository](#configure-access-to-repository)
  - [Add argo apps](#add-argo-apps)
- [Ingress](#ingress)
- [Secrets management - vault](#secrets-management---vault)
  - [Installation of bank-vault - operator](#installation-of-bank-vault---operator)
  - [Installation of vault](#installation-of-vault)
  - [Accessing secrets from vault](#accessing-secrets-from-vault)
- [Access from the Internet](#access-from-the-internet)
- [What next?](#what-next)


In previous [post](/blog/posts/homelab-2022-part-1) I have described hardware and network setup. Today I will talk about software which I have installed on my cluster

Repository with configuration described in this post can be found [here](https://github.com/aldor007/homelab)


# Storage

In my case for storage I'm using Synology NAS server. It is enough for me

```yaml
# helmfile
  - name: nfs-nas
    namespace: nfs-provisioner
    chart: ./charts/nfs-subdir-external-provisioner
    values:
      - values/nfs-nas.yaml
```

Configuration for storage

```yaml
# nfs-nas.yaml
nfs:
  server: 10.39.38.109
  path: /volume2/data

# storage class config
storageClass:
  create: true
  provisionerName: nas
  defaultClass: false
  name: nas
  allowVolumeExpansion: true
  reclaimPolicy: Delete
  archiveOnDelete: true
  accessModes: ReadWriteMany
```


# Monitoring and logs

## Metrics

For monitoring I've decided to use [prometheus-operator](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
It is well known monitoring solution for a Kubernetes cluster so I don't think that I need to add more. My whole configuration can be found [here](https://github.com/aldor007/homelab/blob/master/helmfile/values/prometheus-operator.yaml)

| ![example grafana dashboard](https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy8yMDIyL2dyYWZhbmFfMTEzMDJiNzI3NC5QTkc/photo_grafana_big.jpg) |
|:--:|
| *Grafana dashboard* |

I always have one issue with prometheus operator - I always need to add label

```yaml
# https://github.com/aldor007/homelab/blob/c496358ca5ea22c8662ae971bf2f903096d19161/helmfile/values/prometheus-operator.yaml#L1794

      serviceMonitorSelector:
        matchLabels:
          monitoring: prometheus
      ## Example which selects ServiceMonitors with label "prometheus" set to "somelabel"
      # serviceMonitorSelector:
      #   matchLabels:
      #     prometheus: somelabel

      ## Namespaces to be selected for ServiceMonitor discovery.
      ## See https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/api.md#namespaceselector for usage
      ##
      serviceMonitorNamespaceSelector:
        matchLabels:
          monitoring: prometheus
```

Each namespace in which there is a `ServiceMonitor` has to have `monitoring: prometheus` label

To make my life easier I've written script which is adding this label

``` bash
# https://github.com/aldor007/homelab/blob/master/helmfile/scripts/patch-ns-monitoring.sh
NS_LIST=$(kubectl get ns |  awk "{ print \$1 }" | grep -iv name | grep -v kube )

for ns in ${NS_LIST}; do
    kubectl get ns ${ns} -o yaml | grep labels
    if [[ "$?" == "1" ]]; then
        kubectl patch ns ${ns} --type=json -p='[{"op": "add", "path": "/metadata/labels", "value": {} }]'
    fi
    kubectl get ns ${ns} -o yaml | grep labels -A 10 | grep "monitoring: promethes" >/dev/null || kubectl patch ns ${ns} --type=json -p='[{"op": "add", "path": "/metadata/labels/monitoring", "value": "prometheus" }]'
done
```

And used it in [helmfile](https://github.com/roboll/helmfile) `presync` hook
```yaml
# https://github.com/aldor007/homelab/blob/master/helmfile/helmfile.yaml#L68

    - events: ["presync"]
      command: "/bin/sh"
      args: ["-c", "./scripts/patch-ns-monitoring.sh"]
```
## Logs

For centralized logging solution I'm using [Loki](https://grafana.com/oss/loki/). It works very well with grafana stack. As I don't have big traffic is good enough for me. Configuration is [here](https://github.com/aldor007/homelab/blob/master/helmfile/values/prometheus-operator.yaml#L2267). There is one additional feature "added" by me - extra container for proxing gRPC requests as [greebo](https://github.com/aldor007/greebo) can use Loki as backend.


| ![example loki logs](https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy8yMDIyL2xva2lfOWZjNDgyMDcwMS5QTkc/photo_loki_big.jpg)
|:--:|
| *Loki logs* |

# CD - argocd

In cloud native environment ease of deployment is very important factor. In my commercial experience I have used tools such like Jenkins ans Spinnaker but IMO the greatest one
that can be used for deploying to k8s is [argocd](https://argo-cd.readthedocs.io/en/stable/)

| ![argocd dashboard for mkaciuba.pl](https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy8yMDIyL2FyZ29jZF9lOWQyMzJjNmVjLlBORw/photo_argocd_big.jpg)
|:--:|
| *Argocd dashboard for mkaciuba.pl app* |

## How to setup argocd?

Example based on my homelab


Install argocd server from official helm chart

```yaml
# https://github.com/aldor007/homelab/tree/master/helmfile/charts/argocd-m39
# https://github.com/aldor007/homelab/blob/c496358ca5ea22c8662ae971bf2f903096d19161/helmfile/helmfile.yaml#L125
  - name: argocd
    namespace: argocd
    chart:  ./charts/argocd-m39
    values:
      - values/argocd.yaml.gotmpl

```

## Configure access to repository

```yaml
# https://github.com/aldor007/homelab/blob/c496358ca5ea22c8662ae971bf2f903096d19161/helmfile/values/argocd.yaml.gotmpl#L523
    config:
      # Argo CD's externally facing base URL (optional). Required when configuring SSO
      url: https://argocd.k8s.m39
      # Argo CD instance label key
      application.instanceLabelKey: argocd.argoproj.io/instance
      repositories: |
        - url: git@github.com:aldor007/homelab.git
          name: github-argo-monorepo
          sshPrivateKeySecret: # ssh key used for access can be also HTTP token
            name: argocd-github-sshkey
            key: sshPrivateKey
```

## Add [argo apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications)

Example below

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mkaciuba
  namespace: argocd # argo namespace
spec:
  project: default
  source:
    repoURL: 'git@github.com:aldor007/homelab.git'
    targetRevision: HEAD
    path: argo-apps.mkaciuba
  destination:
    server: https://kubernetes.default.svc
    namespace: mkaciuba
  syncPolicy:
    automated:
      prune:  false # Specifies if resources should be pruned during auto-syncing ( false by default ).
      selfHeal: true
      allowEmpty: false
    retry:
      limit: 5 # number of failed sync attempt retries; unlimited number of attempts if less than 0
      backoff:
        duration: 5s # the amount to back off. Default unit is seconds, but could also be a duration (e.g. "2m", "1h")
        factor: 2 # a factor to multiply the base duration after each failed retry
        maxDuration: 3m # the maximum amount of time allowed for the backoff strategy
```

And that it, you application is ready to be deployed by argo

# Ingress

At beginning of my homelab as ingress I have used [traefik](https://github.com/traefik/traefik) (it is installed by default by k3s) but after some time I came to the conclusion that it is not enough for my
use case. I had a issue with caching of graphql response from strapi. I couldn't modify response headers from strapi (I'm using v3, in v4 is seems that it would be possible)
so I need to do it on proxy/ingress level. I've tried to find a solution how to do it in traefik but without luck. After this I've came up with another solution - use [kong](https://github.com/kong/kong). In kong it
is very easy to add lua scrips on given ingress. Below a few lines of yaml that solved the issue

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: post-header
  namespace: {{ .Release.Namespace }}
config:
  header_filter:
    -  |-
      --- if there is no cache-control header add no-cache
      if kong.response.get_header("cache-control") == nil then
        kong.response.set_header("cache-control", "no-cache")
      end
plugin: post-function
```

Kong agility is something that is required by me (writing plugins in lua is fast)


Recently I have enabled Kong correlation plugin (adding request id header) something not complicated but very powerful
```yaml
apiVersion: configuration.konghq.com/v1
kind: KongClusterPlugin
metadata:
  name: request-id
  annotations:
    kubernetes.io/ingress.class: kong
  labels:
    global: "true"   # optional, if set, then the plugin will be executed
                     # for every request that Kong proxies
                     # please note the quotes around true
config:
  header_name: x-request-id
  echo_downstream: true
plugin: correlation-id
```

Distributed tracing here I come! :)

# Secrets management - vault

Most of my projects are open source but still I need to use some kind of tool to store my credentials for S3 etc
I do not store them in git, for secrets store I'm using [Hashicorp vault](https://www.vaultproject.io/) to be precise Banzai version of it
[bank vault](https://github.com/banzaicloud/bank-vaults)

##  Installation of bank-vault - operator

```yaml
# https://github.com/aldor007/homelab/blob/c496358ca5ea22c8662ae971bf2f903096d19161/helmfile/helmfile.yaml#L111
  - name: vault-operator
    namespace: vault
    chart: banzai/vault-operator
    version: 1.9.2
    values:
      - values/vault-operator.yaml
  - name: vault-secrets-webhook
    namespace: vault
    chart: banzai/vault-secrets-webhook
    version: 1.11.2
```

## Installation of vault

```yaml
# https://github.com/aldor007/homelab/blob/master/helmfile/charts/vault/templates/vault.yaml
apiVersion: vault.banzaicloud.com/v1alpha1
kind: Vault
metadata:
  name: vault
spec:
  size: 1
  image: vault:1.6.1
  bankVaultsImage: ghcr.io/banzaicloud/bank-vaults:1.9.0
  annotations:
    konghq.com/protocol: https

  serviceAccount: vault
  ingress:
    annotations:
      konghq.com/protocols: https
    # Override the default Ingress specification here
    # This follows the same format as the standard Kubernetes Ingress
    # See: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#ingressspec-v1beta1-extensions
    spec:
      ingressClassName: kong
      rules:
        - host: vault.k8s.m39
          http:
            paths:
              - path: "/"
                backend:
                  serviceName: vault
                  servicePort: 8200
      tls:
        - hosts:
            - vault.k8s.m39
          secretName: vault-tls
  # Use local disk to store Vault file data, see config section.
  volumes:
    - name: vault-file
      persistentVolumeClaim:
        claimName: vault-file

  volumeMounts:
    - name: vault-file
      mountPath: /vault/file

  # Support for distributing the generated CA certificate Secret to other namespaces.
  # Define a list of namespaces or use ["*"] for all namespaces.
  caNamespaces:
    - "*"

  # Describe where you would like to store the Vault unseal keys and root token.
  unsealConfig:
    options:
      # The preFlightChecks flag enables unseal and root token storage tests
      # This is true by default
      preFlightChecks: true
    kubernetes:
      secretNamespace: vault

  # A YAML representation of a final vault config file.
  # See https://www.vaultproject.io/docs/colfiguration/ for more information.
  config:
    storage:
      file:
        path: /vault/file
    listener:
      tcp:
        address: 0.0.0.0:8200
        tls_cert_file: /vault/tls/server.crt
        tls_key_file: /vault/tls/server.key
    ui: true

  # See: https://banzaicloud.com/docs/bank-vaults/cli-tool/#example-external-vault-configuration
  # The repository also contains a lot examples in the deploy/ and operator/deploy directories.
  externalConfig:
    policies:
      - name: mort
        rules: |
          path "mort/*" {
            capabilities = ["read","list"]
          }
      - name: mkaciuba
        rules: |
          path "mkaciuba/*" {
            capabilities = ["read","list"]
          }
      - name: insti
        rules: |
          path "insti/*" {
            capabilities = ["read","list"]
          }
    auth:
      - type: kubernetes
        roles:
          # Allow every pod in the default namespace to use the secret kv store
          - name: mort
            bound_service_account_names: ["mort", "vault"]
            bound_service_account_namespaces: ["*"]
            policies: ["mort", "default"]
            ttl: 1h
          - name: mkaciuba
            bound_service_account_names: ["mkaciuba", "vault", "purge-api"]
            bound_service_account_namespaces: ["*"]
            policies: ["mkaciuba", "default"]
            ttl: 1h
          - name: insti
            bound_service_account_names: ["insti", "vault"]
            bound_service_account_namespaces: ["*"]
            policies: ["insti", "default"]
            ttl: 1h

    secrets:
      - path: mort
        type: kv
        description: General secrets.
        options:
          version: 2
```

Everything to configure vault is done by Kubernetes CR. The only manual process is adding secrets by my

| ![vault ](https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy8yMDIyL3ZhdWx0X2FhOTkzZTk3NzcuUE5H/photo_vault_big.jpg) |
|:--:|
| *vault secrets* |

## Accessing secrets from vault

To access secrets defined in vault you need to add annotations for pod like below

```yaml
  vault.security.banzaicloud.io/vault-addr: https://vault.vault.svc.cluster.local:8200
  vault.security.banzaicloud.io/vault-tls-secret: "vault-tls"
  vault.security.banzaicloud.io/vault-path: "kubernetes"
  vault.security.banzaicloud.io/vault-role: "mort"
```

Retrieval process looks like below. You can have env variable with `vault` prefix after which there is a path to secret

```
secrets:
  enabled: true
  env:
    MORT_ASSETS_ACCESS_KEY: "vault:mort/data/buckets#assets.accessKey"
    MORT_ASSETS_SECRET_ACCESS_KEY: "vault:mort/data/buckets#assets.secretAccessKey"
```

# Access from the Internet

There is improvement in this factor from my previous post. Now instead of using OpenVPN I'm using [cloudflare tunnel](https://www.cloudflare.com/products/tunnel/).

First off all you need to generate Cloudflare credentials using [cloudflared](https://github.com/cloudflare/cloudflared) or [terraform](https://github.com/khuedoan/homelab/tree/master/external)
Next step is to deploy cloudflared into cluster - [chart](https://github.com/khuedoan/charts/tree/master/charts/cloudflared)

Example:

```yaml
# helmfile
  - name: cloudflared
    namespace: cloudflare
    chart:  ./charts/cloudflared
    version: 0.3.3
    values:
      - values/cloudflared.yaml
```

```yaml
# values.yaml
credentials:
  existingSecret: cloudflared-credentials
config:
  tunnel: homelab
  ingress:
    - hostname: '*.mkaciuba.com'
      service: http://kong-kong-proxy.ingress
      originRequest:
        noTLSVerify: true
    - hostname: '*.mkaciuba.pl'
      service: http://kong-kong-proxy.ingress
      originRequest:
        noTLSVerify: true
    - service: http_status:404

podMonitor:
  enabled: true
  metricsEndpoints:
    - port: http
```


# What next?

* backup of vault
* homepage for homelab - for example https://github.com/bastienwirtz/homer
