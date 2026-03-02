# Argo CD Notifications for Kustomize Apps

This folder contains the manifests required to notify GitHub whenever an Argo CD
Application syncs successfully. The solution uses the Argo CD notifications
controller to send a `repository_dispatch` event pointing at a GitHub Actions
workflow.

## Prerequisites

- Argo CD installed in your cluster (tested with `quay.io/argoproj/argocd:v3.1.9`).
- The Argo CD Notifications controller running in the same namespace (`argocd`).
- A personal access token (PAT) in GitHub with permission to trigger
  `repository_dispatch` events (`repo` scope is enough).
- Your Applications must expose the following annotations:
  - `notify.github.owner`
  - `notify.github.repo`
  - `notify.github.event_type` (value consumed by your Actions workflow)
  - `notifications.argoproj.io/subscribe.on-deployed-dispatch.github-dispatch: ""`

## Deploy the manifests

```powershell
# Apply the ConfigMap that defines the webhook template
kubectl -n argocd apply -f argo/apps/argocd-notifications-cm.yaml

# Create or update the Secret with a short-lived GitHub token
kubectl -n argocd create secret generic argocd-notifications-secret `
  --from-literal=github-token=REPLACE_WITH_GITHUB_TOKEN `
  --dry-run=client -o yaml | kubectl apply -f -

# Apply the sample Application (points at kustom-webapp/overlays/dev)
kubectl -n argocd apply -f argo/apps/kustom-app-dev.yaml
```

> ℹ️ Replace `REPLACE_WITH_GITHUB_TOKEN` with a fresh PAT before applying. Avoid
> committing real secrets to Git.

## How it works

1. `argocd-notifications-cm.yaml` registers a `github-dispatch` webhook service
   and a template that builds the JSON payload expected by GitHub.
2. `argocd-notifications-secret.yaml` provides the `github-token` the template
   references when authenticating against the GitHub API.
3. Each Application adds the annotations that point to the desired repo and
   event type. When the app is `Synced` and `Healthy`, the controller posts to
   `https://api.github.com/repos/<owner>/<repo>/dispatches`.
4. GitHub Actions workflows listening to `repository_dispatch` can then react to
   deployments triggered by Argo CD.

## Applying to additional apps

- Copy the annotation block from `kustom-app-dev.yaml` and adjust owner, repo
  and `event_type` for each new Application.
- Ensure the app carries an `env` label (or adjust the template if you prefer a
  different name).
- After updating annotations, force a reprocess to avoid deduplication delays:

  ```powershell
  kubectl -n argocd annotate app <app-name> meta.touch="$(Get-Date -UFormat %s)" --overwrite
  ```

## Troubleshooting

- Check controller logs: `kubectl -n argocd logs deploy/argocd-notifications-controller`.
- Errors like `webhook is not supported` usually indicate the controller image
  does not bundle the generic webhook notifier. Upgrade to an image that does,
  or mirror `quay.io/argoproj/argocd-notifications:<tag>` to an accessible
  registry.
- Errors mentioning `notify.github.owner annotation is required` mean the
  Application is missing annotations expected by the template.
