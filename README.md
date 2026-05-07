# RHDH Software Template -- Multi-Agent Loan Origination

An RHDH (Red Hat Developer Hub) Software Template that deploys the
[multi-agent loan origination system](https://github.com/rrbanda/multi-agent-loan-origination)
on OpenShift with full GitOps CI/CD. When a developer runs this template it creates:

* An **application repo** forked from the upstream source with a Backstage catalog entry
* A **GitOps repo** with custom Helm values, a multi-source ArgoCD Application manifest, and catalog metadata
* A **Tekton CI pipeline** triggered by GitHub webhooks that builds both API and UI container images
* An **ArgoCD Application** for continuous deployment via Helm
* **Backstage catalog entries** with CI (Tekton), CD (ArgoCD), and Kubernetes tabs

## Architecture

```
Developer
    |
    v
 RHDH Template ──> GitHub (app repo + gitops repo + webhook)
                        |
        push to main ───┘
                        |
                        v
              Tekton EventListener
                        |
                        v
              Tekton Pipeline (loan-origination-ci)
                |           |           |              |
              clone    build-api    build-ui       clone-gitops
                       (buildah)   (buildah)           |
                                                  push-gitops
                                                       |
                                                       v
                                                ArgoCD syncs
                                                       |
                                                       v
                                           ┌───────────────────────┐
                                           │   OpenShift Cluster   │
                                           │                       │
                                           │  ┌─────┐  ┌────┐     │
                                           │  │ API │  │ UI │     │
                                           │  └──┬──┘  └──┬─┘     │
                                           │     │        │       │
                                           │  ┌──┴────────┴──┐   │
                                           │  │  PostgreSQL   │   │
                                           │  │  + pgvector   │   │
                                           │  └───────────────┘   │
                                           │  ┌───────┐ ┌───────┐ │
                                           │  │ MinIO │ │Keyclk │ │
                                           │  └───────┘ └───────┘ │
                                           └───────────────────────┘
```

## How It Works

1. The template **fetches the full upstream source** from `rrbanda/multi-agent-loan-origination`
2. It **overlays a `catalog-info.yaml`** with Backstage annotations for K8s/Tekton/ArgoCD tabs
3. It **publishes** the source to a new GitHub repo under the user's org
4. It **generates a GitOps repo** containing:
   - `values.yaml` with Helm overrides customized to user inputs (LLM, namespace, etc.)
   - A **multi-source ArgoCD Application** manifest (chart from source repo, values from GitOps repo)
   - A Backstage Resource catalog entry
5. It **creates an ArgoCD Application** pointing to the Helm chart in the source repo
6. It **wires a GitHub webhook** to trigger Tekton CI on every push to `main`
7. It **registers** both repos in the Backstage catalog

### Post-Deploy: Upgrade to Custom Values

The template initially creates an ArgoCD app using default chart values.
To apply your custom Helm values from the GitOps repo, run:

```bash
oc apply -f manifests/argocd-app.yaml   # from within the GitOps repo clone
```

This upgrades the ArgoCD app to a **multi-source** configuration that pulls
the Helm chart from the source repo and custom values from the GitOps repo.

## What Gets Deployed

| Component        | Description                                          | Port  |
|------------------|------------------------------------------------------|-------|
| API              | FastAPI backend + 5 LangGraph agents                 | 8000  |
| UI               | React 19 frontend (nginx)                            | 8080  |
| PostgreSQL       | pgvector-enabled database                            | 5432  |
| MinIO            | S3-compatible object storage                         | 9000  |
| Keycloak         | OIDC identity provider (optional)                    | 8080  |
| MCP Risk Server  | Risk assessment MCP server (reuses API image)        | 8081  |

## Quick Start

The template is pre-configured for:

- **GitHub org:** `rrbanda`
- **Cluster domain:** `apps.ocp.v7hjl.sandbox2288.opentlc.com`
- **RHDH:** `backstage-developer-hub-rhdh-operator.apps.ocp.v7hjl.sandbox2288.opentlc.com`

To use a different cluster or org, update these locations:

| Value             | Where                                               |
|-------------------|-----------------------------------------------------|
| GitHub org        | `template.yaml` (allowedOwners), `trigger-template.yaml` (gitops-repo-url) |
| Cluster domain    | `template.yaml` (clusterDomain default)             |

---

## Prerequisites

The following operators must be installed on the OpenShift cluster:

| Operator                     | Purpose                            |
|------------------------------|------------------------------------|
| Red Hat Developer Hub (RHDH) | Internal developer portal          |
| OpenShift Pipelines (Tekton) | CI pipeline execution              |
| OpenShift GitOps (ArgoCD)    | Continuous deployment              |
| OpenShift Dev Spaces         | Cloud-based development (optional) |

You also need a GitHub account with a personal access token that has
`repo`, `admin:repo_hook`, and `workflow` scopes.

---

## Installation Guide

### 1. Create the CI namespace (separate from deployment namespace)

The Tekton pipeline infrastructure runs in its own namespace (`loan-origination-ci`)
so it does not interfere with application deployments. Each template run
creates a fresh application namespace (default: `loan-origination-demo`).

```bash
oc new-project loan-origination-ci
```

### 2. Create RBAC for the pipeline service account

```bash
oc adm policy add-role-to-user edit \
  system:serviceaccount:loan-origination-ci:pipeline -n loan-origination-ci
oc adm policy add-role-to-user system:image-builder \
  system:serviceaccount:loan-origination-ci:pipeline -n loan-origination-ci
```

### 3. Create the shared Tekton Pipeline

```bash
oc apply -f tekton/pipeline.yaml -n loan-origination-ci
```

### 4. Create the GitHub webhook secret

```bash
oc create secret generic github-webhook-secret \
  --from-literal=webhook-secret=pac-webhook-shared-secret \
  -n loan-origination-ci
```

### 5. Create the GitHub basic-auth secret for GitOps pushes

```bash
oc create secret generic github-basic-auth \
  --from-literal=.git-credentials="https://<GITHUB_USERNAME>:<GITHUB_TOKEN>@github.com" \
  --from-literal=.gitconfig='[credential "https://github.com"]
    helper = store
' \
  -n loan-origination-ci

oc annotate secret github-basic-auth \
  "tekton.dev/git-0=https://github.com" \
  -n loan-origination-ci
```

### 6. Create the Tekton Triggers

```bash
oc apply -f tekton/trigger-binding.yaml -n loan-origination-ci
oc apply -f tekton/trigger-template.yaml -n loan-origination-ci
oc apply -f tekton/event-listener.yaml -n loan-origination-ci
```

### 7. Expose the EventListener

```bash
oc create route edge el-loan-origination-listener \
  --service=el-loan-origination-listener \
  --insecure-policy=Redirect \
  -n loan-origination-ci
```

Verify the webhook URL:

```bash
WEBHOOK_URL=$(oc get route el-loan-origination-listener \
  -n loan-origination-ci -o jsonpath='{.spec.host}')
echo "Webhook URL: https://$WEBHOOK_URL"
```

---

## RHDH Configuration

### 1. Secret: `rhdh-secrets`

Create a secret with all credentials the plugins need:

```bash
oc create secret generic rhdh-secrets \
  --from-literal=GITHUB_TOKEN=<github-pat> \
  --from-literal=K8S_SA_TOKEN=<sa-token> \
  --from-literal=ARGOCD_USERNAME=admin \
  --from-literal=ARGOCD_PASSWORD=<argocd-admin-password> \
  --from-literal=ARGOCD_AUTH_TOKEN=<argocd-auth-token> \
  -n rhdh-operator
```

**How to get `K8S_SA_TOKEN`:**

```bash
oc create sa backstage-k8s-plugin -n rhdh-operator
oc create clusterrolebinding backstage-k8s-reader-binding \
  --clusterrole=backstage-k8s-reader \
  --serviceaccount=rhdh-operator:backstage-k8s-plugin
oc create token backstage-k8s-plugin -n rhdh-operator --duration=8760h
```

### 2. Add the template to RHDH catalog

Add this entry to your `app-config-rhdh` ConfigMap:

```yaml
catalog:
  locations:
    - type: url
      target: https://github.com/rrbanda/rhdh-loan-origination-template/blob/main/location.yaml
      rules:
        - allow: [Template]
```

### 3. Required RHDH dynamic plugins

Ensure these plugins are enabled in your `dynamic-plugins-rhdh` ConfigMap:

```yaml
plugins:
  # GitHub scaffolder (publish:github, github:webhook)
  - package: ./dynamic-plugins/dist/backstage-plugin-scaffolder-backend-module-github-dynamic
    disabled: false

  # Kubernetes backend (cluster API access)
  - package: ./dynamic-plugins/dist/backstage-plugin-kubernetes-backend-dynamic
    disabled: false
    pluginConfig:
      kubernetes:
        serviceLocatorMethod:
          type: multiTenant
        clusterLocatorMethods:
          - type: config
            clusters:
              - name: local-cluster
                url: https://kubernetes.default.svc
                authProvider: serviceAccount
                skipTLSVerify: true
                serviceAccountToken: ${K8S_SA_TOKEN}
        customResources:
          - group: 'tekton.dev'
            apiVersion: 'v1'
            plural: 'pipelineruns'
          - group: 'tekton.dev'
            apiVersion: 'v1'
            plural: 'taskruns'

  # Kubernetes frontend (Kubernetes tab)
  - package: ./dynamic-plugins/dist/backstage-plugin-kubernetes
    disabled: false

  # Tekton CI tab
  - package: ./dynamic-plugins/dist/backstage-community-plugin-tekton
    disabled: false

  # ArgoCD scaffolder (argocd:create-resources action)
  - package: ./dynamic-plugins/dist/roadiehq-scaffolder-backend-argocd-dynamic
    disabled: false
    pluginConfig:
      argocd:
        username: ${ARGOCD_USERNAME}
        password: ${ARGOCD_PASSWORD}
        appLocatorMethods:
          - type: config
            instances:
              - name: openshift-gitops
                url: https://openshift-gitops-server-openshift-gitops.apps.ocp.v7hjl.sandbox2288.opentlc.com
                token: ${ARGOCD_AUTH_TOKEN}

  # ArgoCD backend (CD tab data)
  - package: ./dynamic-plugins/dist/roadiehq-backstage-plugin-argo-cd-backend-dynamic
    disabled: false
    pluginConfig:
      argocd:
        username: ${ARGOCD_USERNAME}
        password: ${ARGOCD_PASSWORD}
        appLocatorMethods:
          - type: config
            instances:
              - name: openshift-gitops
                url: https://openshift-gitops-server-openshift-gitops.apps.ocp.v7hjl.sandbox2288.opentlc.com
                token: ${ARGOCD_AUTH_TOKEN}

  # ArgoCD frontend (CD tab UI)
  - package: oci://ghcr.io/redhat-developer/rhdh-plugin-export-overlays/backstage-community-plugin-argocd:bs_1.45.3__2.4.3
    disabled: false
```

### 4. Verify available scaffolder actions

Visit your RHDH instance to confirm the required actions are installed:

```
https://backstage-developer-hub-rhdh-operator.apps.ocp.v7hjl.sandbox2288.opentlc.com/create/actions
```

Look for:
- `fetch:plain`
- `fetch:template`
- `publish:github`
- `github:webhook`
- `argocd:create-resources`
- `catalog:register`

If `publish:github` or `github:webhook` is missing, enable the GitHub scaffolder plugin.
If `argocd:create-resources` is missing, enable the Roadie ArgoCD scaffolder plugin.

---

## Template Parameters Reference

The template presents a three-page form:

### Page 1: Service Details

| Parameter     | Required | Description                          | Default                                          |
|---------------|----------|--------------------------------------|--------------------------------------------------|
| `name`        | Yes      | Instance name (kebab-case)           | --                                               |
| `description` | No       | Short description                    | Multi-agent AI loan origination system...        |
| `owner`       | No       | Backstage entity owner               | --                                               |
| `companyName` | No       | UI header display name               | Red Hat AI Quickstart Demo                       |

### Page 2: LLM Configuration

| Parameter           | Required | Description                     | Default            |
|---------------------|----------|---------------------------------|--------------------|
| `llmBaseUrl`        | No       | OpenAI-compatible endpoint      | http://vllm:8000/v1 |
| `llmModel`          | No       | Model name                      | gpt-4o-mini        |
| `llmApiKey`         | No       | API key                         | not-needed         |
| `visionModel`       | No       | Vision model name               | (empty)            |
| `visionBaseUrl`     | No       | Vision endpoint                 | (empty)            |
| `embeddingProvider` | No       | Local or openai_compatible      | (empty = local)    |

### Page 3: Deployment Configuration

| Parameter         | Required | Description                        | Default                                        |
|-------------------|----------|------------------------------------|-------------------------------------------------|
| `namespace`       | No       | OpenShift namespace                | loan-origination                                |
| `clusterDomain`   | No       | Cluster apps domain                | apps.ocp.v7hjl.sandbox2288.opentlc.com         |
| `imageRegistry`   | No       | Container image registry           | quay.io                                         |
| `imageRepository` | No       | Registry namespace                 | rh-ai-quickstart                                |
| `imageTag`        | No       | Image tag                          | latest                                          |
| `repoUrl`         | Yes      | GitHub repo location (picker)      | --                                              |
| `enableKeycloak`  | No       | Deploy Keycloak                    | true                                            |
| `enableSeedData`  | No       | Run seed job                       | false                                           |
| `enableMLflow`    | No       | MLflow RBAC resources              | false                                           |
| `authDisabled`    | No       | Skip auth for dev                  | false                                           |

---

## Customization

### Changing the target namespace

Update these locations:

* `template.yaml` -- `namespace` parameter default
* All Tekton resources in `tekton/` -- `metadata.namespace` fields
* `trigger-template.yaml` -- `deploy-namespace` param and `gitops-repo-url`

### Changing the GitHub organization

Update these locations:

* `template.yaml` -- `allowedOwners` in the `RepoUrlPicker`
* `trigger-template.yaml` -- `gitops-repo-url` value

### Adding optional components

To enable LlamaStack, NeMo Guardrails, or Kagenti, edit the `values.yaml`
in the GitOps repo after scaffolding and set the corresponding
`enabled: true` flags.

---

## Troubleshooting

### Template fails at "Fetch Upstream Application Source"

The `fetch:plain` action downloads the upstream repo content. Verify that
your RHDH instance can reach `https://github.com` and that the GitHub
integration is configured with a valid token.

### Template fails at "Create Application Repository"

Ensure `backstage-plugin-scaffolder-backend-module-github-dynamic` is
enabled. The GitHub token needs `repo` and `admin:repo_hook` scopes.

### Template fails at "Create ArgoCD Application"

The `argocd:create-resources` action requires the
`roadiehq-scaffolder-backend-argocd-dynamic` plugin. Check scaffolder logs:

```bash
oc logs deploy/backstage-developer-hub -n rhdh-operator -c backstage-backend \
  | grep "actions enabled"
```

### CI tab is blank

* Verify PipelineRuns have label `backstage.io/kubernetes-id: <component-name>`
* Verify the Component has annotation `janus-idp.io/tekton: <component-name>`
* Verify `pipelineruns` and `taskruns` are listed under `customResources`
  in the Kubernetes backend plugin config

### CD tab is blank

* Verify the Component has annotation `argocd/app-name: <component-name>`
* Verify the ArgoCD plugin is configured with the correct instance URL and token
* Verify the ArgoCD Application exists in `openshift-gitops` namespace

### Build fails due to disk space

The monorepo requires more workspace storage than a simple MCP server.
The TriggerTemplate requests 5Gi for the source workspace. Increase if needed.

---

## Repository Structure

```
rhdh-loan-origination-template/
├── README.md                                    # This file
├── LICENSE                                      # Apache-2.0
├── .gitignore
├── location.yaml                                # Backstage Location entity
├── tekton/
│   ├── pipeline.yaml                            # Shared Tekton Pipeline (2 images)
│   ├── trigger-binding.yaml                     # Extract webhook payload values
│   ├── trigger-template.yaml                    # PipelineRun template
│   └── event-listener.yaml                      # GitHub webhook receiver
└── templates/
    └── multi-agent-loan-origination/
        ├── template.yaml                        # RHDH scaffolder template (9 steps)
        ├── skeleton/
        │   └── catalog-info.yaml                # Backstage Component + API entity
        └── skeleton-gitops/
            ├── catalog-info.yaml                # Backstage Resource entity
            ├── values.yaml                      # Templated Helm values
            └── manifests/
                ├── argocd-app.yaml              # Multi-source ArgoCD Application
                └── namespace.yaml               # Target namespace
```

## Related

* [multi-agent-loan-origination](https://github.com/rrbanda/multi-agent-loan-origination) -- Application source
* [rhdh-mcp-template](https://github.com/rrbanda/rhdh-mcp-template) -- Simpler template for FastMCP servers
* [Build your first Software Template for Backstage](https://developers.redhat.com/articles/2025/08/12/build-your-first-software-template-backstage) -- Tutorial
* [Backstage Software Templates docs](https://backstage.io/docs/features/software-templates/writing-templates)
