# gratitudeapp Helm Chart

This chart packages the full Gratitude App stack from this repository.

## Install

```bash
helm upgrade --install gtapp ./gratitudeapp \
  --namespace default \
  --create-namespace
```

## Validate rendering

```bash
helm lint ./gratitudeapp
helm template gtapp ./gratitudeapp > /tmp/gtapp-rendered.yaml
```

## Common overrides

```bash
helm upgrade --install gtapp ./gratitudeapp \
  --set openai.secret.apiKey="<real-key>" \
  --set database.secret.password="<postgres-password>"
```

```bash
helm upgrade --install gtapp ./gratitudeapp \
  --set filesService.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::<account-id>:role/<role-name>"
```
