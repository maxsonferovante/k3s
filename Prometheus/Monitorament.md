Excelente ideia! Ter o Grafana para monitorar o seu cluster k3s é fundamental para entender o desempenho e a saúde dos seus projetos.

Para ter um monitoramento completo, geralmente configuramos o seguinte stack:

1.  **Prometheus:** Para coletar métricas do cluster (nós, pods, containers) e de aplicações (Node Exporter, cAdvisor, Kube-state-metrics).
2.  **Grafana:** Para visualizar essas métricas de forma gráfica, utilizando o Prometheus como fonte de dados.
3.  **Traefik (Ingress):** Para expor o Grafana com uma URL pública na rede do Raspberry Pi.

O k3s já vem com o Traefik e o metrics-server. Vamos precisar adicionar o Prometheus e o Grafana. A maneira mais fácil de instalar um stack de monitoramento completo no Kubernetes é usando o **kube-prometheus-stack** (anteriormente conhecido como `prometheus-operator`), via Helm.

### **Passo a Passo: Monitoramento com kube-prometheus-stack e Grafana Exposto**

#### **Pré-requisitos:**

* **Helm:** Uma ferramenta para gerenciar pacotes Kubernetes. Se você ainda não tem, instale no seu PC/Laptop (onde você executa `kubectl`):
    ```bash
    # Para Linux (Manjaro, etc.)
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

    # Para macOS
    # brew install helm

    # Para Windows (com Chocolatey)
    # choco install kubernetes-helm
    ```
    Verifique a instalação: `helm version`

* **`kubectl` configurado e funcional:** Conectado ao seu cluster k3s no Raspberry Pi.

#### **Passo 1: Instalar o kube-prometheus-stack com Helm**

O `kube-prometheus-stack` instala o Prometheus, Grafana, Alertmanager, Node Exporter, kube-state-metrics e outros componentes essenciais. Ele cria um novo namespace `monitoring` para esses recursos.

1.  **Adicione o repositório Helm do Prometheus Community:**
    ```bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    ```

2.  **Instale o `kube-prometheus-stack`:**
    Vamos instalar o stack no namespace `monitoring`. Para um Raspberry Pi, pode ser útil ajustar alguns valores padrão para reduzir o consumo de recursos, embora o stack seja grande.

    ```bash
    helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.hosts={grafana.k3s.local} \
  --set grafana.ingress.annotations."ingress\.kubernetes\.io/ssl-redirect"="false" \
  --set grafana.ingress.annotations."traefik\.ingress\.kubernetes\.io/router\.entrypoints"="web" \
  --set grafana.service.type=ClusterIP \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=local-path \
  --set grafana.persistence.size=1Gi \
  --set alertmanager.enabled=false \
  --set kube-state-metrics.enabled=false \
  --set prometheus-node-exporter.enabled=false \
  --timeout=10m  # <--- AUMENTE O TEMPO LIMITE PARA 10 MINUTOS
    ```
    **Explicação dos parâmetros importantes:**
    * `--namespace monitoring --create-namespace`: Instala no namespace `monitoring`.
    * `--set grafana.ingress.enabled=true`: Habilita a criação de um Ingress para o Grafana.
    * `--set grafana.ingress.hosts={grafana.k3s.local}`: Define o hostname para acessar o Grafana via Ingress.
    * `--set grafana.ingress.annotations."ingress\.kubernetes\.io/ssl-redirect"="false"`: Desabilita o redirecionamento HTTPS para facilitar o acesso HTTP inicial.
    * `--set grafana.ingress.annotations."traefik\.ingress\.kubernetes\.io/router\.entrypoints"="web"`: Garante que o Traefik use a porta HTTP (80).
    * `--set grafana.service.type=ClusterIP`: Mantém o serviço do Grafana acessível apenas internamente, pois o Ingress cuidará da exposição.
    * `--set grafana.persistence.enabled=true`: Habilita o armazenamento persistente para os dados do Grafana (dashboards, configurações, etc.).
    * `--set grafana.persistence.storageClassName=local-path`: Usa o StorageClass padrão do k3s.
    * `--set grafana.persistence.size=2Gi`: Tamanho do volume persistente para o Grafana.

    A instalação pode levar alguns minutos.

    #### Atenção
    Em um Raspberry Pi 3, com seu processador mais modesto e apenas 1GB de RAM, essa operação de instalação em larga escala pode sobrecarregar o sistema. O servidor da API do k3s no Pi simplesmente não consegue processar todas as requisições de criação de CRDs e outros recursos dentro do tempo limite que o Helm está esperando (mesmo com os 10 minutos que você definiu). Ele fica "preso" ou extremamente lento, causando o timeout.

    O Que Fazer Agora?
    Infelizmente, existem algumas limitações com o Raspberry Pi 3 para hospedar um stack tão robusto quanto o kube-prometheus-stack.

    Aqui estão suas opções:

    Tentar uma Versão Mais Leve do Monitoramento:
    Em vez do stack completo, você pode tentar instalar o Prometheus e o Grafana separadamente e manualmente, com configurações mais minimalistas, ou usar uma alternativa mais leve. Isso exige um pouco mais de conhecimento de Kubernetes, mas pode ser mais viável no Pi 3.

    Instalar apenas Prometheus e Grafana (chart mais simples): Você pode instalar apenas os charts de Prometheus e Grafana sem o "stack" completo, que inclui o operator, alertmanager, etc.
    Bash

    # Instalar Prometheus (chart standalone)
    helm install prometheus-server prometheus-community/prometheus \
    --namespace monitoring --create-namespace \
    --set persistence.enabled=true \
    --set persistence.storageClassName=local-path \
    --set persistence.size=1Gi \
    --timeout=5m

    # Instalar Grafana (chart standalone)
    helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.storageClassName=local-path \
  --set persistence.size=1Gi \
  --set service.type=ClusterIP \
  --set ingress.enabled=true \
  --set ingress.annotations."ingress\.kubernetes\.io/ssl-redirect"="\"false\"" \ # <--- MUDANÇA AQUI!
  --set grafana.ingress.annotations."traefik\.ingress\.kubernetes\.io/router\.entrypoints"="web" \
  --set ingress.hosts={grafana.k3s.local} \
  --timeout=5m
    
    Observação: Você precisará configurar manualmente o Prometheus como fonte de dados no Grafana após a instalação, e alguns dashboards podem não funcionar sem o kube-state-metrics e node-exporter.
    Atualizar o Hardware:

    A solução mais garantida para rodar o kube-prometheus-stack confortavelmente seria usar um Raspberry Pi 4 (com 4GB ou 8GB de RAM). O salto de desempenho e RAM entre o Pi 3 e o Pi 4 é significativo e faria uma enorme diferença para cargas de trabalho como esta.

    Monitoramento Externo (Para o Cluster no Pi):
    Uma alternativa é não rodar o monitoramento dentro do Raspberry Pi, mas sim monitorar o Pi de fora. Você poderia ter uma instância de Prometheus e Grafana em outra máquina (um laptop, um PC desktop, ou uma VM na nuvem) e configurar o Node Exporter no Raspberry Pi para enviar métricas para esse Prometheus externo. Isso alivia a carga do Pi.

3.  **Verifique os Pods:**
    ```bash
    kubectl get pods -n monitoring
    ```
    Você deve ver vários pods como `prometheus-`, `grafana-`, `alertmanager-`, `kube-state-metrics-`, `prometheus-node-exporter-` todos rodando.

#### **Passo 2: Obter a Senha do Admin do Grafana**

O Grafana é instalado com uma senha gerada automaticamente para o usuário `admin`.

1.  **Obtenha a senha do Secret:**
    ```bash
    kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
    ```
    Anote essa senha. O usuário é `admin`.

#### **Passo 3: Configurar o Acesso ao Grafana via Traefik (URL Pública na Rede)**

1.  **Configure o arquivo `hosts` no seu PC/Laptop:**
    Como usamos o hostname `grafana.k3s.local` no Ingress, você precisa mapeá-lo para o IP do seu Raspberry Pi no arquivo `hosts` do seu PC/Laptop.

    * **No Linux/macOS:** Edite `/etc/hosts`
        ```bash
        sudo nano /etc/hosts
        ```
    * **No Windows:** Edite `C:\Windows\System32\drivers\etc\hosts` (execute o Bloco de Notas como administrador).

    Adicione a seguinte linha:
    ```
    [IP_DO_SEU_PI]   grafana.k3s.local
    ```
    Substitua `[IP_DO_SEU_PI]` pelo endereço IP real do seu Raspberry Pi (aquele que você usou para configurar o `kubectl`, ex: `192.168.15.10`).

2.  **Acesse o Grafana:**
    Abra seu navegador e navegue para:
    ```
    http://grafana.k3s.local
    ```
    Você será direcionado para a página de login do Grafana. Use o usuário `admin` e a senha que você obteve no Passo 2.

#### **Passo 4: Explorar o Grafana e Dashboards**

O `kube-prometheus-stack` já vem com vários dashboards pré-configurados para monitorar o cluster, nós, pods, etc.

1.  **Login no Grafana:**
    Após logar, você verá a tela inicial.

2.  **Navegar pelos Dashboards:**
    No menu lateral esquerdo, clique no ícone de "Dashboards" (quadrado com linhas).
    Vá em "Browse". Você verá pastas como "Kubernetes / Compute Resources", "Node Exporter", etc.

    * **"Kubernetes / Compute Resources / Cluster"**: Mostra o uso geral de CPU e memória do cluster.
    * **"Kubernetes / Compute Resources / Namespace (Pods)"**: Detalhes por namespace.
    * **"Kubernetes / Compute Resources / Node"**: Uso de recursos por nó (seu Raspberry Pi).
    * **"Node Exporter / Use Method / Node"**: Métricas mais detalhadas do sistema operacional.

    Comece a explorar esses dashboards para ver o desempenho do seu cluster!

---

### **Considerações Importantes para Raspberry Pi:**

* **Uso de Recursos:** O `kube-prometheus-stack` é abrangente e consome uma quantidade significativa de recursos (RAM e CPU). Monitore o Raspberry Pi com `htop` ou `free -h`. Se o desempenho for um problema, você pode precisar otimizar o stack, como reduzir o número de réplicas de alguns componentes, ajustar limites de recursos (resource limits) ou até mesmo desabilitar alguns componentes que não são essenciais para seu caso (ex: Alertmanager, se não for usar alertas).
* **Armazenamento:** O Prometheus e o Grafana requerem armazenamento persistente. O `local-path` StorageClass é bom para testes, mas lembre-se que os dados estão no cartão SD do seu Pi.
* **Segurança:** Para um ambiente de produção ou se o Grafana for acessível de fora da sua rede local, configure HTTPS com certificados reais (Let's Encrypt, por exemplo, via cert-manager e Traefik), e implemente autenticação mais robusta que não seja apenas o login admin padrão.

Com esses passos, você terá um sistema de monitoramento poderoso para seu cluster k3s no Raspberry Pi!