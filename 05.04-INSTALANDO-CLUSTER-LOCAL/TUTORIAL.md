# Tutorial — Cluster Kind do Módulo 5 (`algadelivery`)

> Arquivo de apoio real da aula `5.04` (`../roteiro-5.04.md`). Usado sem alteração até o fim do módulo (5.05–5.09 e além): nunca recriar o cluster com outro nome ou outra topologia no meio do módulo — os nomes de node (`algadelivery-worker`, `algadelivery-worker2`) são assumidos ao vivo na 5.07 (`kubectl drain`).

## Versões fixadas (nada de `latest`)

| Item | Versão |
|---|---|
| Kind (CLI) | **v0.32.0** |
| Node image | **`kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5`** (o node image padrão do Kind v0.32.0) |
| Kubernetes (dentro do node) | **v1.36.1** |

Confirmado na página de release oficial do Kind: <https://github.com/kubernetes-sigs/kind/releases/tag/v0.32.0>. Se o instrutor gravar as aulas meses depois de uma nova versão do Kind sair, **não** rode `kind create cluster` sem `--config` (pega o node image "latest" da versão instalada) — revalide a combinação Kind↔node-image na página de releases e atualize `kind-config.yaml` antes de gravar.

## Topologia

`kind-config.yaml` (neste diretório) declara **1 control-plane + 2 workers** — a topologia que a `5.04` mostra no desenho ("materializando a 5.03") e que a `5.07` precisa para o `kubectl drain` não esvaziar o cluster inteiro:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5
  - role: worker
    image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5
  - role: worker
    image: kindest/node:v1.36.1@sha256:3489c7674813ba5d8b1a9977baea8a6e553784dab7b84759d1014dbd78f7ebd5
```

Com o nome de cluster `algadelivery` (fixado em todo o módulo — é o mesmo `--name` usado em `kind load docker-image ... --name algadelivery` nas aulas 5.05–5.09), o Kind nomeia os nodes automaticamente:

| Node | Papel |
|---|---|
| `algadelivery-control-plane` | Control Plane |
| `algadelivery-worker` | Worker |
| `algadelivery-worker2` | Worker |

**Importante para a 5.07:** quando o roteiro disser `kubectl drain <node>`, o `<node>` real é `algadelivery-worker` **ou** `algadelivery-worker2` (o que o `kubectl get pod hello -o wide` mostrar na coluna `NODE`) — nunca `kind-worker` nem `control-plane`.

## Passo a passo

**1. Pré-requisitos:** Docker rodando + `kubectl`/`kind` instalados (bloco 3 da `5.04`).

**2. Criar o cluster** (da raiz do repositório):
```bash
kind create cluster --name algadelivery --config kind-config.yaml
```
Isso também configura o contexto `kind-algadelivery` no seu `kubeconfig` automaticamente.

**3. Verificar que os 3 nodes subiram e ficaram `Ready`:**
```bash
kubectl get nodes
```
Saída esperada — 1 `control-plane` + 2 `worker`, todos `STATUS Ready`:
```
NAME                          STATUS   ROLES           AGE   VERSION
algadelivery-control-plane    Ready    control-plane   1m    v1.36.1
algadelivery-worker           Ready    <none>          1m    v1.36.1
algadelivery-worker2          Ready    <none>          1m    v1.36.1
```

**4. Confirmar os componentes do Control Plane (callback à 5.03):**
```bash
kubectl get pods -n kube-system
```

**5. Destruir o cluster** (fim do módulo, ou para recriar do zero):
```bash
kind delete cluster --name algadelivery
```

## Troubleshooting

- **`kubeadm init` trava/falha com `context deadline exceeded` bootstrapando o admin (`ClusterRoleBinding`)** — sintoma clássico de rodar o Kind dentro de um ambiente com **cgroup v1** ou fortemente virtualizado/sandboxed (containers dentro de containers sem cgroup v2 real, algumas VMs de CI). Em uma máquina normal com Docker Desktop (macOS/Windows) ou Docker Engine nativo (Linux com cgroup v2, o padrão em distros atuais), isso não acontece. Se acontecer na sua máquina: confirme `docker info | grep -i cgroup` — se aparecer `Cgroup Version: 1`, é preciso migrar o host para cgroup v2 antes de gravar (fora do escopo deste tutorial; é configuração do sistema operacional/WSL, não do Kind).
- **Nodes ficam `NotReady` por mais de 1–2 minutos** — normalmente o CNI (`kindnet`) ainda está subindo; espere e rode `kubectl get pods -n kube-system` de novo antes de investigar.
- **Esqueceu de destruir um cluster de teste anterior com o mesmo nome** — `kind get clusters` lista, `kind delete cluster --name algadelivery` remove antes de recriar.