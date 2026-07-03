# cllone — GitOps

Conteúdo pronto para o **repositório GitOps** separado (ex.: `cllone-gitops`).
Baseado no tutorial: <https://eduardo-da-silva.github.io/fundamentos-devops/cd/>.

## Estrutura

```
cllone-gitops/
  k8s/
    namespace.yaml
    configmap.yaml
    secret.yaml.example   # modelo — copie para secret.yaml local
    .gitignore            # ignora secret.yaml real
    postgres-pvc.yaml
    postgres-deployment.yaml
    postgres-service.yaml
    backend-deployment.yaml
    backend-service.yaml
    frontend-deployment.yaml
    frontend-service.yaml
    adminer-deployment.yaml   # visualizador do banco (Adminer)
    adminer-service.yaml      # NodePort 30800
    ingress.yaml              # Traefik: / -> frontend, /api -> backend
    kustomization.yaml    # GitHub Actions atualiza as tags aqui
  argocd/
    application.yaml      # manifesto ArgoCD Application
```

## Como subir o repositório GitOps

```bash
# A partir da raiz deste workspace
cp -r gitops/ /tmp/cllone-gitops
cd /tmp/cllone-gitops
git init -b main
git remote add origin git@github.com:henrique-furtado47/cllone-gitops.git
git add .
git commit -m "chore: bootstrap gitops repo"
git push -u origin main
```

## Criar o secret no cluster (uma vez, fora do Git)

```bash
cp k8s/secret.yaml.example k8s/secret.yaml
# edite os valores reais
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.yaml
```

## Validar manifests localmente

```bash
kubectl apply -k k8s
kubectl get pods -n cllone
```

## Registrar a aplicação no ArgoCD

Pela CLI/`kubectl` (depois de ajustar `repoURL` em `argocd/application.yaml`):

```bash
kubectl apply -f argocd/application.yaml
```

Ou pela UI do ArgoCD com:
- Repository URL: URL do repo GitOps
- Path: `k8s`
- Namespace: `cllone`
- Sync Policy: Automatic + Auto-Prune + Self-Heal

## Acesso à aplicação (Traefik) e ao banco (Adminer)

O k3s já traz o **Traefik** como Ingress Controller. O `ingress.yaml` faz o
roteamento por caminho, como sugerido na avaliação:

| Caminho   | Serviço          | Descrição                     |
|-----------|------------------|-------------------------------|
| `/`       | frontend-service | SPA Vue                       |
| `/api`    | backend          | API REST Django               |
| `/admin`  | backend          | Django admin                  |
| `/static` | backend          | estáticos do Django           |

Acesso na porta 80 do IP público do control plane: `http://<IP>/` e `http://<IP>/api/`.

O **Adminer** (visualização do banco) fica no NodePort **30800**:
`http://<IP>:30800` — sistema `PostgreSQL`, servidor `postgres-service`,
usuário/senha/base conforme o `Secret`.

## Fluxo CD ponta-a-ponta

1. Push na `main` deste repo (cllone) → GitHub Actions roda testes
2. Workflow builda e empurra `cllone-backend:<sha>` e `cllone-frontend:<sha>` no Docker Hub
3. Workflow faz commit no repo GitOps atualizando `newTag` em `kustomization.yaml`
4. ArgoCD detecta o commit e sincroniza no cluster
