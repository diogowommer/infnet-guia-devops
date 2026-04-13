# Guia de Estudos Infnet - DevOps e Kubernetes

Este repositório foi desenvolvido como projeto da disciplina **Integração Contínua, DevOps e Computação em Nuvem**.

O projeto usa uma aplicação web em Next.js como base para demonstrar práticas de empacotamento com Docker, deploy em Kubernetes, health checks, observabilidade com Prometheus/Grafana, persistência com PVC e pipeline de entrega com GitHub Actions.

## Objetivos Atendidos

- Aplicação containerizada com Docker.
- Imagem publicada no Docker Hub.
- Deploy Kubernetes com `Deployment`.
- Aplicação com 4 réplicas.
- Exposição externa da aplicação via `NodePort`.
- Probes de `readiness` e `liveness`.
- Prometheus como servidor de métricas.
- Grafana como ferramenta de dashboards.
- Prometheus com dados persistidos em PVC.
- Dashboard com métricas de CPU e memória da aplicação.
- Stress test para gerar carga e observar alteração no dashboard.
- Pipeline de entrega com GitHub Actions.

## Tecnologias

- Next.js
- React
- Docker
- Docker Hub
- Kubernetes
- Minikube
- Prometheus
- Grafana
- GitHub Actions

## Estrutura Principal

```text
.
├── app/
│   └── api/health/
│       ├── live/route.ts
│       └── ready/route.ts
├── k8s/
│   ├── deployment.yaml
│   └── prometheus.yaml
├── scripts/
│   ├── apply-all.sh
│   └── stress-test.sh
├── .github/workflows/
│   └── kubernetes-deploy.yml
├── Dockerfile
├── .dockerignore
└── README.md
```

## Aplicação

A aplicação é um guia de estudos web desenvolvido em Next.js.

Scripts principais:

```bash
npm install
npm run dev
npm run build
npm run start
```

## Health Checks

A aplicação possui rotas específicas para probes Kubernetes:

```text
/api/health/live
/api/health/ready
```

Exemplo de resposta:

```json
{
  "status": "ok",
  "check": "liveness",
  "timestamp": "2026-04-13T01:35:02.576Z"
}
```

No Kubernetes, os probes estão configurados no container `app`:

```yaml
readinessProbe:
  httpGet:
    path: /api/health/ready
    port: http

livenessProbe:
  httpGet:
    path: /api/health/live
    port: http
```

## Docker

A imagem da aplicação é construída a partir do `Dockerfile`.

Build local:

```bash
docker build -t diogowommer/infnet-guia-devops:latest .
```

Teste local:

```bash
docker run --rm -p 3000:3000 diogowommer/infnet-guia-devops:latest
```

Teste dos health checks:

```bash
curl http://localhost:3000/api/health/live
curl http://localhost:3000/api/health/ready
```

Push para o Docker Hub:

```bash
docker login
docker push diogowommer/infnet-guia-devops:latest
```

Imagem usada no Kubernetes:

```text
diogowommer/infnet-guia-devops
```

## Kubernetes

O manifesto principal da aplicação fica em:

```text
k8s/deployment.yaml
```

Ele contém:

- `Deployment` da aplicação `guia-app`;
- 4 réplicas;
- `Service` do tipo `NodePort`;
- probes de readiness e liveness;
- Grafana;
- ConfigMaps de datasource e dashboard do Grafana.

Aplicar a aplicação:

```bash
kubectl apply -f k8s/deployment.yaml
```

Verificar:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

Resultado esperado para a aplicação:

```text
guia-app   4/4   Running
```

## Exposição via NodePort

A aplicação é exposta fora do cluster com `NodePort`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: guia-app
spec:
  type: NodePort
  ports:
    - name: http
      port: 3000
      targetPort: http
```

Para descobrir a porta externa:

```bash
kubectl get svc guia-app
```

Com Minikube:

```bash
minikube service guia-app --url
```

## Prometheus

O manifesto do Prometheus fica em:

```text
k8s/prometheus.yaml
```

Ele contém:

- `ServiceAccount`;
- `ClusterRole`;
- `ClusterRoleBinding`;
- `ConfigMap` com a configuração do Prometheus;
- `PersistentVolumeClaim` para persistir dados;
- `Deployment` do Prometheus;
- sidecar `node-exporter`;
- `Service` interno `ClusterIP`.

Aplicar:

```bash
kubectl apply -f k8s/prometheus.yaml
```

Verificar:

```bash
kubectl get pods -l app=prometheus
kubectl get svc prometheus
kubectl get pvc
```

O Prometheus fica acessível apenas dentro do cluster:

```yaml
type: ClusterIP
```

## PVC do Prometheus

O Prometheus usa um PVC para persistir os dados do TSDB:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

O volume é montado em:

```text
/prometheus
```

E o Prometheus grava os dados neste caminho:

```yaml
--storage.tsdb.path=/prometheus
```

Verificar o PVC:

```bash
kubectl get pvc prometheus-data
```

Resultado esperado:

```text
prometheus-data   Bound   ...   2Gi   RWO
```

## Grafana

O Grafana está definido em:

```text
k8s/deployment.yaml
```

Ele é a única ferramenta de monitoramento exposta fora do cluster, usando `NodePort`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  type: NodePort
```

Acessar com Minikube:

```bash
minikube service grafana --url
```

Credenciais:

```text
Usuário: admin
Senha: admin123
```

Dashboard provisionado:

```text
Guia Infnet - Aplicação + Kubernetes
```

O dashboard mostra métricas como:

- pods da aplicação monitorados;
- CPU da aplicação;
- memória da aplicação;
- CPU por pod;
- memória por pod;
- métricas do nó;
- métricas do próprio Prometheus.

## Aplicar Tudo

Existe um script para aplicar os manifests principais:

```bash
chmod +x scripts/apply-all.sh
./scripts/apply-all.sh
```

Também é possível aplicar manualmente:

```bash
kubectl apply -f k8s/prometheus.yaml
kubectl apply -f k8s/deployment.yaml
```

Verificar rollouts:

```bash
kubectl rollout status deployment/guia-app
kubectl rollout status deployment/prometheus
kubectl rollout status deployment/grafana
```

## Stress Test

O script de stress test fica em:

```text
scripts/stress-test.sh
```

Ele gera carga HTTP na aplicação para provocar variação nos gráficos do Grafana.

Com Minikube, abra a URL da aplicação:

```bash
minikube service guia-app -n default --url
```

Em alguns ambientes com Minikube e driver Docker, esse comando precisa ficar aberto. Copie a URL exibida e, em outro terminal, rode:

```bash
chmod +x scripts/stress-test.sh
./scripts/stress-test.sh http://127.0.0.1:<porta>
```

Também é possível deixar o script tentar detectar a URL:

```bash
./scripts/stress-test.sh
```

Durante o teste, abra o Grafana e observe os painéis:

```text
CPU guia-app
Memória guia-app
CPU guia-app por pod
Memória guia-app por pod
```

Para a entrega, recomenda-se capturar prints:

- dashboard antes do stress test;
- terminal executando o stress test;
- dashboard durante ou após o stress test com os gráficos alterados.

## Pipeline GitHub Actions

A pipeline fica em:

```text
.github/workflows/kubernetes-deploy.yml
```

Ela executa:

1. checkout do repositório;
2. login no Docker Hub;
3. build da imagem Docker;
4. push para o Docker Hub;
5. configuração do `kubectl`;
6. aplicação dos manifests Kubernetes;
7. atualização do `Deployment` para a imagem do commit;
8. espera dos rollouts;
9. listagem de pods, services e PVCs.

Secrets necessários no GitHub:

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
KUBE_CONFIG
```

Configuração:

```text
GitHub > Settings > Secrets and variables > Actions
```

Observação: se o cluster usado for Minikube local na WSL, o runner hospedado do GitHub Actions normalmente não conseguirá acessá-lo. Nesse caso, use um cluster acessível externamente ou um runner self-hosted na máquina local.

## Comandos Úteis

Ver recursos:

```bash
kubectl get pods,svc,pvc
```

Ver detalhes da aplicação:

```bash
kubectl describe deployment guia-app
```

Ver probes:

```bash
kubectl describe pod -l app=guia-app
```

Ver logs da aplicação:

```bash
kubectl logs -l app=guia-app
```

Ver targets do Prometheus:

```bash
kubectl port-forward svc/prometheus 9090:9090
```

Depois acesse:

```text
http://localhost:9090/targets
```

Abrir Grafana:

```bash
minikube service grafana --url
```

Abrir aplicação:

```bash
minikube service guia-app --url
```

## Resumo da Entrega

Este projeto demonstra um fluxo DevOps completo:

- desenvolvimento de aplicação web;
- empacotamento em container;
- publicação de imagem;
- deploy em Kubernetes;
- health checks;
- monitoramento;
- dashboard;
- persistência de métricas;
- stress test;
- automação com pipeline CI/CD.

Projeto desenvolvido para a disciplina **Integração Contínua, DevOps e Computação em Nuvem**.
