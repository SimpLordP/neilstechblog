---
title: "What the Helm?"
date: 2026-06-24
draft: false
tags: ["Kubernetes", "Helm", "DevOps", "K3s", "Homelab"]
series: ["Homelab Hero"]
description: "Breaking down what Helm charts actually are, why raw kubectl apply doesn't scale, and how to package, template, and version your Kubernetes deployments the right way."
---

So you're probably wondering, **why do I need Helm when kubectl apply already works fine?** I would say this is exactly the question I asked myself the first time someone told me to "just Helm install it." kubectl apply does work fine, right up until you have more than one environment, more than one developer, or more than one version of your app running at the same time.

I hit this wall on my own K3s cluster the moment I tried deploying the same app to a dev namespace and a prod namespace with slightly different resource limits. I had two nearly identical YAML files, copy-pasted, with three lines different between them, and I already knew that was going to be a maintenance nightmare six months from now.

# Gameplan

To understand why Helm solves this, we need to break it down into three pieces: what a chart actually is, how templating replaces copy-pasted YAML, and how versioning and rollbacks work once your app is packaged as a chart instead of a pile of manifests. I'll walk through creating a chart from scratch using a real app on my Pi cluster as the example.

## What a Chart Actually Is

A Helm chart is just a directory with a specific structure that Helm knows how to read. At its simplest:

```text
my-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

**Chart.yaml** holds metadata about the chart itself, things like name and version. **values.yaml** holds the configuration values you actually want to change between environments. **templates/** holds your actual Kubernetes manifests, but written with placeholders instead of hardcoded values.

That's it. No magic, just a folder convention and a templating engine on top of plain YAML.

## Creating Your First Chart

Let's scaffold one. Helm gives you a generator for this:

```bash
helm create my-app
```

This creates a full directory structure with a working example deployment already inside it. Go ahead and look inside `templates/deployment.yaml` and you'll see something like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

Notice the double curly braces, `{{ .Values.replicaCount }}`. That's a template placeholder, and the actual value gets pulled from `values.yaml` at deploy time.

## values.yaml Is Where the Magic Happens

Open up `values.yaml` and you'll see the defaults that fill in those placeholders:

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: latest

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

Here's where the dev versus prod problem I mentioned earlier actually gets solved. Instead of two separate YAML files, I create two values files:

```bash
touch values-dev.yaml values-prod.yaml
```

`values-dev.yaml`:

```yaml
replicaCount: 1
resources:
  limits:
    cpu: 100m
    memory: 128Mi
```

`values-prod.yaml`:

```yaml
replicaCount: 3
resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

Same chart, same templates, two different override files. The actual deployment logic never gets duplicated.

## Installing the Chart

To deploy to your dev namespace:

```bash
helm install my-app-dev ./my-app -f values-dev.yaml --namespace dev --create-namespace
```

And to prod, using the exact same chart:

```bash
helm install my-app-prod ./my-app -f values-prod.yaml --namespace prod --create-namespace
```

**install**: Refers to the Helm command for a brand new deployment that doesn't exist yet.
**my-app-dev** / **my-app-prod**: The release name, this is how Helm tracks this specific deployment instance going forward.
**-f**: The values file to layer on top of the chart's defaults.
**--namespace**: Which Kubernetes namespace this release lives in.

## Upgrading Without Tears

This is where Helm actually separates itself from raw kubectl apply. Let's say you bump the image tag in `values-prod.yaml`. Instead of re-applying a YAML file and hoping nothing else changed unexpectedly, you run:

```bash
helm upgrade my-app-prod ./my-app -f values-prod.yaml --namespace prod
```

Helm calculates the diff between what's currently running and what your updated values produce, and only changes what actually needs to change.

## Rollbacks, the Feature That Sold Me

Here's the part that actually made me a Helm believer. Every install and upgrade gets a revision number:

```bash
helm history my-app-prod --namespace prod
```

```text
REVISION  UPDATED                   STATUS      CHART          APP VERSION
1         Mon Jun 16 14:02:11 2026  superseded  my-app-0.1.0   1.0.0
2         Mon Jun 23 09:14:55 2026  deployed    my-app-0.1.1   1.0.1
```

If revision 2 turns out to be broken, rolling back to the last known good state is one command:

```bash
helm rollback my-app-prod 1 --namespace prod
```

No reconstructing what the old YAML looked like from memory or git history. Helm already knows exactly what revision 1 was, and puts it back instantly.

# Resources

**Scaffold a new chart**

```bash
helm create my-app
```

**Install to a specific environment**

```bash
helm install my-app-dev ./my-app -f values-dev.yaml --namespace dev --create-namespace
```

**Upgrade an existing release**

```bash
helm upgrade my-app-prod ./my-app -f values-prod.yaml --namespace prod
```

**Check revision history**

```bash
helm history my-app-prod --namespace prod
```

**Roll back to a previous revision**

```bash
helm rollback my-app-prod 1 --namespace prod
```

# Conclusion

And that's pretty much it. We covered what a Helm chart actually is under the hood, how values files eliminate copy-pasted YAML across environments, and why revision history and rollback turn a stressful incident into a one-line command. kubectl apply still has its place for quick one-off tests, but the moment you have more than one environment to manage, Helm earns its place in the stack fast.

# FAQ

**Do I need Helm for a single-node homelab cluster?**
Not strictly, but even on a single K3s node, the moment you're running more than one or two apps, the templating and rollback benefits show up faster than you'd expect.

**What's the difference between a Helm chart and a Kustomize overlay?**
Kustomize patches existing YAML without templating syntax, while Helm templates generate the YAML from scratch using values. Both solve the multi-environment problem, just with a different philosophy. Helm tends to win out once you're distributing a chart to other people, since it packages cleanly as a single versioned artifact.

**Can I find pre-built Helm charts instead of writing my own?**
Yes, most popular self-hosted apps already publish official charts through Helm repositories like Artifact Hub. Writing your own chart matters most for your own custom applications, not for things like Postgres or Redis that already have a maintained chart available.
