# MariaDB no Kubernetes (Talos)

Instância MariaDB 11.8 usada **só pelo Nextcloud** (`nextcloud` namespace).
`Service` ClusterIP `mariadb.mariadb.svc.cluster.local:3306`, sem exposição externa.

## Estrutura

| Arquivo | Recurso |
|---|---|
| `namespace.yaml` | `Namespace/mariadb` (sem Pod Security Admission) |
| `configmap.yaml` | `ConfigMap/mariadb-config` (só `TZ`, via `envFrom` no pod) |
| `secret.example.yaml` | template do `Secret/mariadb-secret` (senha do root) |
| `secret.local.yaml` | valores em claro para gerar o sealed — **em `.gitignore`** |
| `sealed-secret.yaml` | `SealedSecret/mariadb-secret` (único segredo versionado) |
| `pvc.yaml` | `PersistentVolumeClaim/mariadb-data` (20Gi, `local-path`) |
| `deployment.yaml` | `Deployment/mariadb` (1 réplica, `strategy: Recreate`) |
| `service.yaml` | `Service/mariadb` ClusterIP :3306 |
| `networkpolicy.yaml` | default-deny + ingress só do `nextcloud` + egress só DNS |
| `kustomization.yaml` | junta tudo no namespace `mariadb` |
| `database.sql` | referência: DDL do database `nextcloud` (não é aplicado) |

## Aplicar

```bash
# 1. Segredo (só na 1ª vez ou ao rotacionar a senha):
cp secret.example.yaml secret.local.yaml
openssl rand -base64 24        # -> MARIADB_ROOT_PASSWORD em secret.local.yaml
kubeseal --format yaml --controller-namespace kube-system \
  < secret.local.yaml > sealed-secret.yaml     # commite só o sealed-secret.yaml

# 2. Aplicar:
kubectl apply -k .
kubectl -n mariadb rollout status deploy/mariadb
```

> **Migração do Secret (feita em 2026-09-03):** o `Secret/mariadb-secret` era
> criado à mão. Sequência: `kubectl -n mariadb delete secret mariadb-secret`,
> depois `kubectl apply -f sealed-secret.yaml`. Como o controller já tinha
> desistido do reconcile (retry esgotado enquanto o Secret não-gerenciado
> existia) e reaplicar o SealedSecret idêntico não força novo reconcile, foi
> preciso uma mudança real de `spec` — daí o `template.metadata.labels` neste
> arquivo. Resultado: `SYNCED=True`, Secret recriado e gerenciado pelo
> SealedSecret.

## Mudanças aplicadas ao migrar para este padrão (2026-09-03)

Partindo do manifesto antigo + estado real do cluster, o repo passou a ser a
fonte da verdade e foi aplicado:

| Item | Antes | Agora |
|---|---|---|
| `spec.strategy` | RollingUpdate (padrão) | **`Recreate`** — evita 2 mysqld no mesmo datadir |
| `--innodb-buffer-pool-size` | `512M` (== limite de RAM, risco OOM) | **`256M`** |
| `resources` | manifesto pedia `500m/512Mi`–`1/1Gi` | req `100m`/`256Mi` · lim `200m`/`512Mi` (valores reais do cluster) |
| Secret | `Secret` aplicado à mão | `SealedSecret` (ver nota acima) |
| `configmap.yaml` | `namespace:` hardcoded, não usado pelo pod | injetado por kustomize + `envFrom` no pod (TZ passa a valer) |
| `namespace.yaml` | criado imperativamente | versionado |
| NetworkPolicy | nenhuma | 3 policies (sem efeito enquanto o CNI for Flannel) |
| Labels `app.kubernetes.io/*` | nenhuma | bloco `labels:` no kustomization (Lens agrupa como "Application") |

> Ao aplicar a mudança do `configMapRef` o Deployment travou com
> `NewReplicaSet: <none>` (glitch do kube-controller-manager, que tinha
> reiniciado 2 dias antes) e o MariaDB ficou fora do ar. Destravado com
> `kubectl -n mariadb scale deploy/mariadb --replicas=0 && ... --replicas=1`.

## Pendências

1. **Sem backup**. Agendar `mariadb-dump` (CronJob) + snapshot do PVC.
2. **Namespace sem Pod Security Admission** e pod sem `securityContext`.
3. **Deployment** (não StatefulSet). Para 1 réplica com PVC RWO funciona, mas
   um StatefulSet seria mais adequado para banco.
4. Imagem por tag flutuante `mariadb:11.8` (não fixada por digest).
