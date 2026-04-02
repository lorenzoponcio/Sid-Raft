# Raft Cluster – Web UI

Simulação de cluster Raft com 3 nós, painel web em tempo real e suporte a
eleição de líder, heartbeat e falha de nó.

## Rodar com Docker Compose

```bash
docker compose up --build
```

Acesse qualquer nó:
- http://localhost:5001  (node1)
- http://localhost:5002  (node2)
- http://localhost:5003  (node3)

## Como testar

1. **Identificar líder e followers** – abra qualquer painel; o card com borda
   verde e badge `Leader` é o líder atual.

2. **Derrubar o líder** – no painel do nó líder, clique em
   **Simular falha / reativar este nó**. Aguarde 3–6 s e recarregue: um novo
   líder terá sido eleito.

3. **Mudar o intervalo de timeout** – na seção "Ajustar Timeouts" do painel,
   altere `Election Min/Max` e `Heartbeat Interval` e clique em **Aplicar**.
   - Valores baixos (ex: 1–2 s / 0.5 s) → eleições acontecem rapidamente.
   - Valores altos (ex: 10–15 s / 3 s) → cluster demora mais para reagir a falhas.

## Consultas úteis

```bash
curl http://localhost:5001/status   # estado do node1
curl http://localhost:5001/cluster  # visão agregada de todos os nós
```

## Kubernetes

### Gerar imagem

```bash
docker build -t raft-project-webui:latest .
```

### Aplicar manifests

```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/statefulset.yaml
```

### Verificar

```bash
kubectl get pods
kubectl logs raft-0
kubectl port-forward raft-0 5001:5000
```

> O StatefulSet usa DNS headless (`raft-0.raft`, `raft-1.raft`, `raft-2.raft`),
> então os nós se comunicam automaticamente dentro do cluster Kubernetes
> sem precisar de IPs fixos.

## Correções aplicadas vs versão original

| # | Problema | Correção |
|---|----------|----------|
| 1 | `/cluster` fazia HTTP para `localhost:PORT` para obter o status local | Lê direto do `state` interno |
| 2 | `election_timer` recalculava `timeout` a cada 0.5 s, nunca acumulando tempo real | Dorme o `timeout` completo antes de verificar |
| 3 | `docker-compose.yml` sem rede compartilhada | Adicionado `raft-net` bridge network |
| 4 | Sem endpoint `/config` para ajuste de timeouts em tempo real | Adicionado `POST /config` |
| 5 | Painel não exibia configuração de timeouts nem permitia alterá-la | Seção "Ajustar Timeouts" no painel |
| 6 | Dockerfile sem `HEALTHCHECK` | Adicionado `HEALTHCHECK` |
| 7 | k8s `statefulset.yaml` sem `readinessProbe` / `livenessProbe` | Adicionadas as probes |
| 8 | k8s: pods não tinham como descobrir peers via DNS | `PEERS` configurado com DNS headless |
