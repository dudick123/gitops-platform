# GitOps Platoform Generator

## Overview

This project is a GitOps platform generator that simplifies the creation of GitOps projects using Helm. It allows you to define your GitOpsProject abstraction and generates the necessary Kubernetes resources for ArgoCD.

The primary goal is to provide a clean and structured way to manage GitOps projects, making it easier to deploy and manage applications in a Kubernetes environment.

The goal is to breakdown the higher-level, GitOpsProject abstraction into smaller composable components using Helm. to make the resources GitOps compatible, individual resources for Kubernetes Namespaces, ArgoCD Projects and ArgoCD Application Sets are generated. These files will be generated into a `base` direcotory structure for a tenant and then `kustomize` will be used to build the final manifests for each environment.

## Project Layout

1. Folder Structure for Helm Chart

Create a Helm chart that maps parts of the GitOpsProject spec to Kubernetes/ArgoCD primitives.

gitops-project-chart/
├── templates/
│   ├── namespace.yaml
│   ├── argocd-project.yaml
│   ├── applicationset.yaml
├── values.yaml
Chart.yaml



⸻

2. values.yaml

Pass your high-level GitOps abstraction via values.yaml:

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



⸻

3. Template: namespace.yaml

{{- range .Values.environments }}
apiVersion: v1
kind: Namespace
metadata:
  name: {{ $.Values.projectShortName }}-gitops-{{ . }}
---
{{- end }}



⸻

4. Template: argocd-project.yaml

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



⸻

5. Template: applicationset.yaml

This assumes use of ApplicationSet with a matrix generator.

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



⸻

6. Rendering Each Resource to Separate Files

Use helm template with the --output-dir flag:

```bash
helm template gitops-project ./platform-generator \ 
  --values ./platform-generator/tenants/tenant-a/values.yaml  \
  --output-dir ./base/tenant-a
```
This will render each resource into a separate file in the specified output directory. For example:

```bash
wrote ./base/tenant-a/gitops-project/templates/namespace.yaml
wrote ./base/tenant-a/gitops-project/templates/argocd-project.yaml
wrote ./base/tenant-a/gitops-project/templates/applicationset.yaml
This creates a structure like:

rendered/
└── gitops-project/
    ├── templates/
        ├── namespace.yaml
        ├── argocd-project.yaml
        ├── applicationset.yaml

You can check in each rendered file into a GitOps repo (e.g., with ArgoCD watching the folder).

⸻

Bonus Tips
	•	Use Helm subcharts if you want to package each component (e.g., namespace, argocd-project, appset) as its own reusable Helm chart.
	•	Add conditions in Chart.yaml to toggle rendering components.

⸻