# f2-monitoring

ArgoCD-managed manifests на f2 (argocd.csjob.ru) для мониторинга
s-Loki (https://loki.aeve.ru) из Grafana на f2.

## Состав

- `grafana-datasource-s-loki.yaml` — ConfigMap с datasource `Loki-s` →
  s-Loki через NPM proxy с basic_auth (логин promtail).
- `grafana-dashboards-config.yaml` — два dashboard:
  - Homelab: Per-Host Log Volume — rate(lines) per host, container
  - Homelab: Watchdog Status — OOM/Docker OOMKilled/Disk errors
- `application.yaml` — ArgoCD Application с auto-sync, source = git.

## Vault rotation

При смене пароля в Vault нужно обновить password в Secret `s-loki-credentials`
и в ConfigMap `s-loki-datasource.secureJsonData.basicAuthPassword`:

```bash
NEW_PASS=$(ssh s 'grep ^promtail: /etc/promtail/credentials | cut -d: -f2')
sed -i "s|password: \".*\"|password: \"$NEW_PASS\"|" grafana-datasource-s-loki.yaml
sed -i "s|basicAuthPassword: \".*\"|basicAuthPassword: \"$NEW_PASS\"|" grafana-datasource-s-loki.yaml
git add -A && git commit -m "rot: loki password" && git push
```

## Deploy

```bash
git clone ssh://z@s:2222/media/s/data/git/f2-monitoring.git
cd f2-monitoring
kubectl apply -f application.yaml   # или через argocd CLI
```
