# GitOps Platoform Generator

## Overview

This project is a GitOps platform generator that simplifies the creation of GitOps projects using Helm. It allows you to define your GitOpsProject abstraction and generates the necessary Kubernetes resources for ArgoCD.

The primary goal is to provide a clean and structured way to manage GitOps projects, making it easier to deploy and manage applications in a Kubernetes environment.

The goal is to breakdown the higher-level, GitOpsProject abstraction into smaller composable components using Helm. to make the resources GitOps compatible, individual resources for Kubernetes Namespaces, ArgoCD Projects and ArgoCD Application Sets are generated. These files will be generated into a `base` direcotory structure for a tenant and then `kustomize` will be used to build the final manifests for each environment.

## Project Layout

1. Folder Structure for GitOps Platform

The basic folder structure for the GitOps platform generator is as follows:

```bash
.
├── README.md
├── base
│   ├── tenant-a
│   │   └── gitops-project
│   │       └── templates
│   │           ├── applicationset.yaml
│   │           ├── argocd-project.yaml
│   │           └── namespace.yaml
│   ├── tenant-b
│   └── tenant-c
└── platform-generator
    ├── Chart.yaml
    ├── rendered
    │   └── gitops-project
    │       └── templates
    │           ├── applicationset.yaml
    │           ├── argocd-project.yaml
    │           └── namespace.yaml
    ├── templates
    │   ├── applicationset.yaml
    │   ├── argocd-project.yaml
    │   └── namespace.yaml
    ├── tenants
    │   └── tenant-a
    │       └── values.yaml
    └── values.yaml
```

A few items of note:

- The `platform-generator` directory contains the Helm chart that generates the GitOps project resources. 

- This `platform-generator/templates` folder is where you would define your templates to produce specific Kubernetes resources. In this case, we have templates for `namespace.yaml`, `argocd-project.yaml`, and `argocd-applications-set.yaml`. Future templates such as `network-policy`can be added here as needed.

- The `platform-generator/tenants` directory contains the values.yaml files for each tenant. This is where you would define the specific values for each tenant's GitOps project.

- The `base` directory contains the rendered output of the GitOps project for each tenant. This is the output directory for Helm charts. The `base/tenant-a` folder would contain the rendered output for the `tenant-a` GitOps project and can be further enhanced with `kustomize` to build the final manifests for each environment.


## A Look At En Example **values.yaml** File

The `values.yaml` file is where you would define the specific values for youar GitOps project. The `project` is a composition of multiple components, including the project name, description, environments, repositories, and applications.

Here is an example of what a `values.yaml` file might look like:

```yaml

projectName: foo-portal
projectShortName: foo
projectDescription: "Foo Portal GitOps Project"

environments:
  - dev
  - nonprod
  - prod

repositories:
  - https://dev.azure.com/acme/Biz-Portal_git/foo-leave-service-k8s-manifests
  - https://dev.azure.com/acme/Biz-Portal_git/foo-intake-service-k8s-manifests

applications:
  - name: leave-service
    repoURL: https://dev.azure.com/acme/Biz-Portal_git/foo-leave-service-k8s-manifests
  - name: intake-service
    repoURL: https://dev.azure.com/acme/Biz-Portal_git/foo-intake-service-k8s-manifests

```

## Template: namespace.yaml

This template creates a Kubernetes namespace. The rendered namepsace will be further processed by `kustomize` to build the final manifests for each environment of dev, nonprod and prod.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ $.Values.projectShortName }}-gitops-{{ . }}

```

## Template: argocd-project.yaml

This template creates an ArgoCD project. The project name is derived from the `projectShortName` value in the `values.yaml` file. The project description and source repositories are also defined here.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: {{ .Values.projectShortName }}
  namespace: argocd
spec:
  description: {{ .Values.projectDescription }}
  sourceRepos:
  {{- range .Values.repositories }}
    - {{ . }}
  {{- end }}
  destinations:
  {{- range .Values.environments }}
    - namespace: {{ $.Values.projectShortName }}-gitops-{{ . }}
      server: https://kubernetes.default.svc
  {{- end }}
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
```

## Template: argocd-application-set.yaml

This assumes use of ApplicationSet with a matrix generator.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: {{ .Values.projectShortName }}-apps
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
              {{- range .Values.environments }}
                - env: {{ . }}
              {{- end }}
          - list:
              elements:
              {{- range .Values.applications }}
                - name: {{ .name }}
                  repoURL: {{ .repoURL }}
              {{- end }}
  template:
    metadata:
      name: '{{`{{name}}-{{env}}`}}'
    spec:
      project: {{ .Values.projectShortName }}
      source:
        repoURL: '{{`{{repoURL}}`}}'
        targetRevision: HEAD
        path: overlays/{{`{{env}}`}}
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{`{{$.Values.projectShortName}}-gitops-{{env}}`}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true

```

## Rendering Each Resource to Separate Files

To render each resource into a separate file, you can use the `helm template` command with the `--output-dir` flag. This will create a directory structure that mirrors the Helm chart structure, allowing you to easily manage and version control the generated resources.
Use helm template with the --output-dir flag:

```bash
helm template gitops-project ./platform-generator \ 
  --values ./platform-generator/tenants/tenant-a/values.yaml  \
  --output-dir ./base/tenant-a
```

### Explanation of the command

`helm template`: This command generates Kubernetes manifests from a Helm chart without installing them into a Kubernetes cluster.
gitops-project:

The release name for the Helm chart. It is a label used to identify the generated resources.

`./platform-generator`: The path to the Helm chart directory. This is where the chart's Chart.yaml and templates are located.

`--values ./platform-generator/tenants/tenant-a/values.yaml`: Specifies a custom values.yaml file to override default values in the chart. In this case, it uses the values specific to "tenant-a."

`--output-dir ./base/tenant-a`: Specifies the directory where the rendered Kubernetes manifests will be saved. The output will be written to `./base/tenant-a`

This will render each resource into a separate file in the specified output directory. For example:

```bash
wrote ./base/tenant-a/gitops-project/templates/namespace.yaml
wrote ./base/tenant-a/gitops-project/templates/argocd-project.yaml
wrote ./base/tenant-a/gitops-project/templates/applicationset.yaml
```

This creates a structure like:

```bash
├── base
│   ├── tenant-a
│   │   └── gitops-project
│   │       └── templates
│   │           ├── applicationset.yaml
│   │           ├── argocd-project.yaml
│   │           └── namespace.yaml
```

## Kustomize

Kustomize will be used to build the final manifests for each environment. The `kustomization.yaml` file will be created in each environment directory, and it will reference the rendered resources.
The `kustomization.yaml` file will look something like this:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - argocd-project.yaml
  - applicationset.yaml
```
