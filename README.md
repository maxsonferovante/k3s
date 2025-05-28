# k3s Project - Cluster Kubernetes Leve

Este repositório contém tutoriais, configurações e exemplos para configurar e usar um cluster k3s (Kubernetes leve) com diversas aplicações.

## 📁 Estrutura do Projeto

```
k3s/
├── init/           # Configuração inicial do k3s e Dashboard
├── postgres/       # PostgreSQL com pgbench para testes de performance
└── README.md       # Este arquivo
```

## 🚀 Início Rápido

### 1. Configuração Inicial do k3s
📂 **Pasta:** `init/`

- **📘 Tutorial de instalação do k3s.md** - Guia completo de instalação do k3s
- **🔧 script.md** - Scripts para configuração do Dashboard do Kubernetes
- **⚙️ dashboard-adminuser.yaml** - Configuração do usuário admin para o Dashboard
- **🔑 admin-user-token-secret.yaml** - Secret para token de acesso ao Dashboard

### 2. PostgreSQL com pgbench
📂 **Pasta:** `postgres/`

- **📗 PostgreSQL_pgbench.md** - Tutorial completo para configurar PostgreSQL com pgbench
- **🗄️ postgres-deployment.yaml** - Deployment do PostgreSQL
- **🌐 postgres-service.yaml** - Service para exposição do PostgreSQL
- **💾 postgres-pvc.yaml** - PersistentVolumeClaim para armazenamento persistente
- **⚡ pgbench-pod.yaml** - Pod para execução de benchmarks com pgbench

## 🛠️ Pré-requisitos

- **Hardware:** Raspberry Pi (3, 4, 5, ou Zero 2 W) com 512MB+ de RAM ou qualquer máquina Linux
- **SO:** Sistema operacional baseado em Linux (Raspberry Pi OS Lite recomendado)
- **Rede:** Conexão com a internet para download dos componentes
- **Permissões:** Acesso root ou sudo

## 🎯 Casos de Uso

### Instalação e Configuração do k3s
1. **Instalação básica:** Siga o tutorial em `init/Tutorial de instalação do k3s.md`
2. **Dashboard do Kubernetes:** Use os arquivos YAML em `init/` para configurar acesso web
3. **Acesso remoto:** Configure kubectl para gerenciar o cluster remotamente

### PostgreSQL para Testes de Performance
1. **Deploy PostgreSQL:** Use os manifests em `postgres/` para implantar uma instância PostgreSQL
2. **Testes de benchmark:** Execute pgbench para avaliar performance do banco de dados
3. **Armazenamento persistente:** Os dados persistem entre reinicializações dos pods

## 📋 Comandos Principais

### Instalação do k3s
```bash
# Instalação automática
curl -sfL https://get.k3s.io | sh -

# Verificar status
sudo systemctl status k3s
sudo kubectl get nodes
```

### Dashboard do Kubernetes
```bash
# Aplicar configurações
kubectl apply -f init/dashboard-adminuser.yaml
kubectl apply -f init/admin-user-token-secret.yaml

# Obter token de acesso
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa admin-user -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

### PostgreSQL + pgbench
```bash
# Deploy PostgreSQL
kubectl apply -f postgres/postgres-pvc.yaml
kubectl apply -f postgres/postgres-deployment.yaml
kubectl apply -f postgres/postgres-service.yaml

# Deploy pgbench client
kubectl apply -f postgres/pgbench-pod.yaml

# Inicializar benchmark
kubectl exec -it pgbench-client -- pgbench --host=postgres-service --port=5432 --username=pgbench_user --initialize pgbench_db

# Executar benchmark
kubectl exec -it pgbench-client -- pgbench --host=postgres-service --port=5432 --username=pgbench_user --time=60 --client=4 --jobs=2 pgbench_db
```

## 🔧 Configurações Importantes

### Para Raspberry Pi
- **cgroups:** Adicione ao `/boot/firmware/cmdline.txt`:
  ```
  cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
  ```
- **Recursos limitados:** Monitore uso de RAM e CPU durante operação

### Armazenamento
- **StorageClass padrão:** k3s usa `local-path` para armazenamento persistente
- **Localização dos dados:** `/var/lib/rancher/k3s/storage`

## 🧹 Limpeza

### Remover recursos PostgreSQL
```bash
kubectl delete -f postgres/pgbench-pod.yaml
kubectl delete -f postgres/postgres-deployment.yaml
kubectl delete -f postgres/postgres-service.yaml
kubectl delete -f postgres/postgres-pvc.yaml
```

### Desinstalar k3s
```bash
sudo /usr/local/bin/k3s-uninstall.sh
sudo reboot
```

## 📚 Recursos Adicionais

- [Documentação oficial do k3s](https://docs.k3s.io/)
- [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
- [PostgreSQL no Kubernetes](https://kubernetes.io/docs/tutorials/stateful-application/postgresql/)
- [pgbench Documentation](https://www.postgresql.org/docs/current/pgbench.html)

## 🚨 Notas de Segurança

⚠️ **Importante:** Este projeto é voltado para desenvolvimento e testes. Para produção:
- Use Kubernetes Secrets para credenciais
- Configure TLS/SSL adequadamente
- Implemente RBAC (Role-Based Access Control)
- Use operadores especializados para bancos de dados

## 🤝 Contribuições

Sinta-se à vontade para contribuir com melhorias, correções ou novos exemplos!

---

**Desenvolvido para aprendizado e experimentação com Kubernetes leve (k3s)** 🚀 