# README — Kind + Kubernetes (Pods)

Guia prático para criar **clusters com Kind** e **Pods no Kubernetes** de quatro formas:

1. Cluster **com arquivo YAML** (config Kind)
2. Cluster **sem arquivo YAML** (padrão)
3. Pod **com arquivo YAML**
4. Pod **sem arquivo YAML** (linha de comando)
5. **Gerar YAML via dry-run** (a partir do comando) e **fazer o deploy**

> Testado com `kind >= 0.20` e `kubectl >= 1.27`. Adapte conforme sua versão.

---

## 🧰 Pré-requisitos

* **Docker** instalado e em execução
* **Kind** instalado

  * Linux/macOS (binário): [https://kind.sigs.k8s.io](https://kind.sigs.k8s.io)
  * Go: `go install sigs.k8s.io/kind@latest`
* **kubectl** instalado

```bash
kubectl version --client
kind version
```

---

## 1) Criar cluster **com arquivo YAML**

Crie um arquivo `kind-config.yaml` (exemplo com 1 control-plane + 2 workers):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Crie o cluster com base no arquivo:

```bash
kind create cluster --name lab --config kind-config.yaml
```

Verifique:

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

> Dica: para mapear portas em localhost, adicione `extraPortMappings` no nó `control-plane`.

Exemplo (mapeando porta 30000):

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30000
        hostPort: 30000
        protocol: TCP
  - role: worker
```

---

## 2) Criar cluster **sem arquivo YAML** (configuração padrão)

Cria um cluster mínimo (apenas control-plane):

```bash
kind create cluster --name dev
```

Verifique o contexto e nós:

```bash
kubectl config get-contexts
kubectl get nodes
```

Remover cluster quando não precisar mais:

```bash
kind delete cluster --name dev
# ou
kind delete clusters dev lab
```

---

## 3) Criar **Pod com arquivo YAML**

Crie `pod-nginx.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

Aplicar e validar:

```bash
kubectl apply -f pod-nginx.yaml
kubectl get pods -o wide
kubectl logs -f pod/nginx-pod
```

Acessar via port-forward (opcional):

```bash
kubectl port-forward pod/nginx-pod 8080:80
# acessar: http://localhost:8080
```

Apagar:

```bash
kubectl delete -f pod-nginx.yaml
```

---

## 4) Criar **Pod sem arquivo YAML** (linha de comando)

Usando `kubectl run` para criar um **Pod** diretamente (garantindo que seja Pod com `--restart=Never`):

```bash
kubectl run nginx-pod \
  --image=nginx:latest \
  --port=80 \
  --restart=Never
```

Verificar:

```bash
kubectl get pod nginx-pod -o yaml | head -n 20
kubectl describe pod nginx-pod
```

Apagar:

```bash
kubectl delete pod nginx-pod
```

> Observação: sem `--restart=Never`, algumas versões do `kubectl run` podem criar outros tipos (ex.: Deployment). Use o parâmetro para garantir `Pod`.

---

## 5) **Gerar YAML via dry-run** e **fazer o deploy**

O fluxo clássico é **gerar o manifesto** com `dry-run=client`, salvar em um arquivo e então aplicar.

### 5.1 Gerar YAML do Pod a partir de um comando

```bash
kubectl run nginx-pod \
  --image=nginx:latest \
  --port=80 \
  --restart=Never \
  --dry-run=client -o yaml > pod-nginx.yaml
```

Inspecione o arquivo:

```bash
sed -n '1,40p' pod-nginx.yaml
```

### 5.2 Fazer o deploy a partir do YAML gerado

```bash
kubectl apply -f pod-nginx.yaml
kubectl get pods
```

### 5.3 (Opcional) Gerar YAML de um **Service** simples e aplicar

```bash
kubectl expose pod nginx-pod \
  --name=svc-nginx \
  --port=80 \
  --target-port=80 \
  --type=NodePort \
  --dry-run=client -o yaml > svc-nginx.yaml

kubectl apply -f svc-nginx.yaml
kubectl get svc svc-nginx -o wide
```

> Com Kind, para acessar NodePort via `localhost`, use `extraPortMappings` no `control-plane` (seção 1) ou `port-forward`.

---

## 🔍 Troubleshooting rápido

* **ImagePullBackOff**: confira conectividade e política de pull (`imagePullPolicy`), tente `nginx:alpine` ou `busybox` para testes.
* **CrashLoopBackOff**: verifique `kubectl logs` e `kubectl describe pod`, ajuste `command/args` do contêiner.
* **Cluster não sobe**: reinicie Docker, valide versões de Kind e kubectl; apague clusters antigos: `kind delete clusters --all`.

---

## 🧹 Limpeza geral

```bash
kubectl delete -f pod-nginx.yaml 2>/dev/null || true
kubectl delete svc svc-nginx 2>/dev/null || true
kind delete clusters dev lab 2>/dev/null || true
```

---

## 📎 Referências rápidas

* Kind docs: [https://kind.sigs.k8s.io](https://kind.sigs.k8s.io)
* kubectl cheatsheet: [https://kubernetes.io/docs/reference/kubectl/cheatsheet/](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

