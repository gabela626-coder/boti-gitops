# GitOps Bot Template

Template manifests for adding a new Telegram bot to the cluster.

## Usage

```bash
# 1. Copy template to a new directory
cp -r apps/_template apps/my-bot

# 2. Replace all BOT_NAME placeholders
cd apps/my-bot
sed -i 's/BOT_NAME/my-bot/g' deployment.yaml sealedsecret.yaml kustomization.yaml application.yaml

# 3. Generate SealedSecret with real token
kubectl create secret generic my-bot-secret \
  -n infra \
  --from-literal=BOT_TOKEN=<TOKEN_FROM_BOTFATHER> \
  --dry-run=client -o yaml \
  | kubeseal --cert infra/keys/sealed-secrets-public.pem -o yaml \
  > apps/my-bot/sealedsecret.yaml

# 4. Set the correct image tag in deployment.yaml
sed -i 's/CHANGE_ME/<short-sha>/g' apps/my-bot/deployment.yaml

# 5. Copy application.yaml to app-definitions (for root-app)
cp apps/my-bot/application.yaml argocd/app-definitions/my-bot.yaml

# 6. Add to app-definitions/kustomization.yaml
echo "  - my-bot.yaml" >> argocd/app-definitions/kustomization.yaml

# 7. Commit and push
git add apps/my-bot argocd/app-definitions/my-bot.yaml argocd/app-definitions/kustomization.yaml
git commit -m "feat: add my-bot to GitOps"
git push
```

ArgoCD will pick up the changes and deploy automatically.
