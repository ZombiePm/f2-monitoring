# f2-monitoring

ArgoCD-managed manifests на f2 (argocd.csjob.ru) для мониторинга
s-Loki (https://loki.aeve.ru) из Grafana на f2.

## Состав

- `grafana-datasource-s-loki.yaml` — ConfigMap с datasource `Loki-s` →
  s-Loki через NPM proxy с basic_auth (логин promtail), плюс Secret
  `s-loki-credentials` с паролем.
- `grafana-dashboards-config.yaml` — два dashboard:
  - `homelab-hosts-logs` — rate(lines) per host / container
  - `homelab-watchdog` — OOM (systemd + Docker) и Disk errors
- `application.yaml` — ArgoCD Application `f2-mon-s-loki`,
  source = https://github.com/ZombiePm/f2-monitoring.git

## Deploy

```bash
# Создать repo (один раз):
gh repo create ZombiePm/f2-monitoring --public --description "f2 Grafana monitoring stack"
git push -u origin main

# Применить ArgoCD Application:
scp application.yaml root@f2:/tmp/
ssh root@f2 "kubectl apply -f /tmp/application.yaml -n argocd"

# Одноразовый фикс DNS для пода Grafana (CoreDNS NodeHosts):
ssh root@f2 'kubectl patch cm -n kube-system coredns --type merge -p "{\\"data\\":{\\"NodeHosts\\":\\"95.81.99.178 firstvds40.fvds.ru\\n2a0c:db40:0:82d9::2 firstvds40.fvds.ru\\n213.171.6.146 loki.aeve.ru s.aeve.ru vault.aeve.ru\\n192.168.1.153 s.lan loki.lan\\"}}"'
ssh root@f2 "kubectl rollout restart -n kube-system deploy/coredns"
```

## Vault rotation

При смене пароля в Vault нужно обновить password в Secret `s-loki-credentials`
и в ConfigMap `s-loki-datasource.secureJsonData.basicAuthPassword`:

```bash
NEW_PASS=$(ssh s 'grep ^promtail: /etc/promtail/credentials | cut -d: -f2')
sed -i "s|password: \".*\"|password: \"$NEW_PASS\"|" grafana-datasource-s-loki.yaml
sed -i "s|basicAuthPassword: \".*\"|basicAuthPassword: \"$NEW_PASS\"|" grafana-datasource-s-loki.yaml
git add -A && git commit -m "rot: loki password" && git push
# ArgoCD auto-sync подхватит изменения за <60s
```
