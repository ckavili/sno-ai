# SNO-AI Menu Selection

This directory contains the menu configuration for SNO-AI deployments.

## How It Works

The `config.yaml` file controls which RHOAI menu option is deployed via the ApplicationSet controller.

### Current Configuration

```yaml
selectedMenu: "1"  # Which menu option to deploy
git:
  repoURL: https://github.com/ckavili/sno-ai.git
  branch: main
```

### Available Menu Options

| Value | Menu Option | Description |
|-------|-------------|-------------|
| `"1"` | Full RHOAI | Complete RHOAI with workbenches, pipelines, and KServe |
| `"2"` | Model Serving Focus | Model serving with Granite model |
| `"3"` | LLM Serving Focus | vLLM-based LLM serving with GPU support |
| `"4"` | Custom | User-defined configuration |

## Switching Menu Options

### Method 1: Edit and Commit (Recommended)

```bash
# Edit the config file
vi gitops/menu/config.yaml

# Change selectedMenu to desired option
selectedMenu: "2"

# Commit and push
git add gitops/menu/config.yaml
git commit -m "Switch to menu option 2 (Model Serving)"
git push

# ArgoCD will automatically detect the change and:
# 1. Delete the old menu option's resources
# 2. Deploy the new menu option's resources
```

### Method 2: GitHub Web UI

1. Navigate to `gitops/menu/config.yaml` in GitHub
2. Click "Edit this file"
3. Change `selectedMenu` value
4. Commit directly to main branch
5. ArgoCD will sync automatically

### Method 3: CLI (One-liner)

```bash
# Switch to option 3
cat > gitops/menu/config.yaml <<EOF
selectedMenu: "3"
git:
  repoURL: https://github.com/ckavili/sno-ai.git
  branch: main
EOF

git add gitops/menu/config.yaml
git commit -m "Switch to menu option 3 (LLM Serving)"
git push
```

## Verification

After changing the menu option:

```bash
# Check ApplicationSet status
oc get applicationset snoai-menu-router -n openshift-gitops

# Check which Application was generated
oc get application -n openshift-gitops -l app.kubernetes.io/part-of=snoai-platform

# Watch the sync progress
oc get application snoai-<menu-name> -n openshift-gitops -w

# Verify RHOAI components
oc get datasciencecluster -n redhat-ods-operator
oc get pods -n redhat-ods-applications
```

## How the ApplicationSet Works

The `menu-router-appset.yaml` uses a **matrix generator** that:

1. **Git Generator**: Reads `config.yaml` from the Git repo
   - Exposes: `{{.selectedMenu}}`, `{{.git.repoURL}}`, `{{.git.branch}}`

2. **List Generator**: Defines all available menu options
   - Each option has: `menu`, `path`, `name`

3. **Selector**: Filters the matrix to only include the selected menu
   - Only generates an Application for the option matching `selectedMenu`

4. **Template**: Creates the ArgoCD Application with:
   - Source from Git (repo URL and branch from config)
   - Path to the specific menu option
   - Auto-sync enabled with prune

## Troubleshooting

### ApplicationSet not generating Application

```bash
# Check ApplicationSet status
oc get applicationset snoai-menu-router -n openshift-gitops -o yaml

# Check ApplicationSet controller logs
oc logs -n openshift-gitops -l app.kubernetes.io/name=argocd-applicationset-controller -f
```

### Multiple Applications created

If you see multiple Applications (all menu options), check:
- The selector in the ApplicationSet
- Go template syntax is correct (`{{.selectedMenu}}` with dot notation)

### Application not syncing

```bash
# Check Application health
oc get application snoai-full-rhoai -n openshift-gitops -o yaml

# Check ArgoCD application controller logs
oc logs -n openshift-gitops -l app.kubernetes.io/name=argocd-application-controller -f
```

## For Seed Image Creation

When creating the seed image, this configuration will be baked in. The deployed clusters will:

1. Boot with the seed image
2. OpenShift GitOps reads the `selectedMenu` from the Git repo
3. Deploys the corresponding RHOAI configuration

To change the menu option in a deployed cluster:
- Update `config.yaml` in your Git repo
- ArgoCD will automatically switch configurations
