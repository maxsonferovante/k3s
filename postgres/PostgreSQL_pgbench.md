Okay\! Configurar um "pgbench cluster" no Kubernetes localmente com k3s é um excelente caso de uso para testar cargas de trabalho de banco de dados.

Primeiro, é importante esclarecer que **`pgbench` é uma ferramenta de benchmark de desempenho do PostgreSQL, e não um "cluster" de PostgreSQL em si**. Para ter um cluster PostgreSQL (alta disponibilidade, replicação, etc.), você geralmente usa:

1.  **Operadores de Kubernetes:** Como o [CloudNativePG](https://cloudnative-pg.io/), [CrunchyData PostgreSQL Operator](https://www.crunchydata.com/products/crunchy-postgresql-for-kubernetes/) ou [Zalando PostgreSQL Operator](https://github.com/zalando/postgres-operator). Estes são a forma **recomendada e robusta** de gerenciar clusters PostgreSQL no Kubernetes.
2.  **StatefulSets e Volumes Persistentes:** Uma abordagem mais manual, onde você configura a replicação e a alta disponibilidade do PostgreSQL diretamente com recursos do Kubernetes.

Dado que você mencionou "pgbench cluster para meus projetos", presumo que você queira:

  * **Implantar uma instância PostgreSQL (ou um cluster simples).**
  * **Implantar um Pod separado com o `pgbench` para executar benchmarks contra essa instância PostgreSQL.**

Vamos focar na abordagem de implantar um PostgreSQL simples e depois o `pgbench` para interagir com ele. Para um "cluster" de PostgreSQL de verdade (com replicação e alta disponibilidade), um operador seria a melhor escolha, mas é mais complexo para um primeiro passo.

-----

### **Objetivo:**

1.  **Implantar um Pod PostgreSQL simples.**
2.  **Configurar um `PersistentVolume` (PV) e `PersistentVolumeClaim` (PVC)** para garantir que os dados do PostgreSQL persistam mesmo se o Pod for reiniciado ou movido.
3.  **Implantar um Pod separado com `pgbench`** para executar benchmarks.

-----

### **Pré-requisitos:**

  * Cluster k3s funcionando no seu Raspberry Pi.
  * `kubectl` configurado e funcionando no seu PC/Laptop ou diretamente no Pi.

-----

### **Passo a Passo: PostgreSQL + pgbench**

#### **Passo 1: Configurar Persistent Storage (Armazenamento Persistente)**

Para que os dados do seu PostgreSQL não se percam, precisamos de armazenamento persistente. Como o k3s vem com um StorageClass padrão chamado `local-path`, que usa o armazenamento local do nó, vamos utilizá-lo.

1.  **Verifique o StorageClass padrão:**

    ```bash
    kubectl get storageclass
    ```

    Você deve ver `local-path` listado, e provavelmente marcado como `(default)`.

2.  **Crie um `PersistentVolumeClaim` (PVC) para o PostgreSQL:**
    Este PVC solicitará espaço de armazenamento persistente.

    Crie um arquivo `postgres-pvc.yaml`:

    ```yaml
    # postgres-pvc.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: postgres-data-pvc
      labels:
        app: postgres
    spec:
      accessModes:
        - ReadWriteOnce # O PVC pode ser montado como leitura/escrita por um único nó
      resources:
        requests:
          storage: 5Gi # Solicita 5 GB de armazenamento (ajuste conforme a necessidade do seu Pi)
      storageClassName: local-path # Usa o StorageClass padrão do k3s
    ```

    Aplique o PVC:

    ```bash
    kubectl apply -f postgres-pvc.yaml
    ```

    Verifique se o PVC foi criado e está `Bound`:

    ```bash
    kubectl get pvc
    ```

#### **Passo 2: Implantar o PostgreSQL**

Vamos criar um Deployment para o PostgreSQL e um Service para expô-lo internamente no cluster.

1.  **Crie um arquivo `postgres-deployment.yaml`:**

    ```yaml
    # postgres-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: postgres-deployment
      labels:
        app: postgres
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: postgres
      template:
        metadata:
          labels:
            app: postgres
        spec:
          containers:
            - name: postgres
              image: postgres:15 # Versão estável do PostgreSQL
              env:
                - name: POSTGRES_DB
                  value: pgbench_db # Nome do banco de dados para pgbench
                - name: POSTGRES_USER
                  value: pgbench_user # Usuário para pgbench
                - name: POSTGRES_PASSWORD
                  value: mysecretpassword # Senha do usuário (ATENÇÃO: Não use senhas em texto puro em produção!)
              ports:
                - containerPort: 5432 # Porta padrão do PostgreSQL
              volumeMounts:
                - name: postgres-persistent-storage
                  mountPath: /var/lib/postgresql/data # Onde o PostgreSQL armazena os dados
          volumes:
            - name: postgres-persistent-storage
              persistentVolumeClaim:
                claimName: postgres-data-pvc # Referencia o PVC criado anteriormente
    ```

2.  **Crie um arquivo `postgres-service.yaml`:**

    ```yaml
    # postgres-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: postgres-service
    spec:
      selector:
        app: postgres
      ports:
        - protocol: TCP
          port: 5432 # Porta que o serviço expõe
          targetPort: 5432 # Porta que o container PostgreSQL escuta
      type: ClusterIP # Apenas acessível dentro do cluster
    ```

3.  **Aplique os manifestos:**

    ```bash
    kubectl apply -f postgres-deployment.yaml
    kubectl apply -f postgres-service.yaml
    ```

4.  **Verifique os recursos do PostgreSQL:**

    ```bash
    kubectl get deployment postgres-deployment
    kubectl get service postgres-service
    kubectl get pods -l app=postgres
    ```

    Aguarde até que o Pod do PostgreSQL esteja `Running`.

#### **Passo 3: Implantar o Pod `pgbench` e Executar o Benchmark**

Vamos criar um Pod temporário que terá a ferramenta `pgbench` e o usaremos para executar os benchmarks.

1.  **Crie um arquivo `pgbench-pod.yaml`:**
    Este Pod não precisa ser um Deployment, pois será usado para tarefas pontuais de benchmark.

    ```yaml
    # pgbench-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pgbench-client
      labels:
        app: pgbench
    spec:
      containers:
        - name: pgbench
          image: postgres:15 # A imagem do PostgreSQL geralmente já inclui o pgbench
          command: ["/bin/bash", "-c", "sleep infinity"] # Mantém o container rodando para que possamos executar comandos
          env:
            - name: PGPASSWORD
              value: mysecretpassword # Senha para o pgbench se conectar
      restartPolicy: Never # Não reinicia o pod automaticamente após a conclusão (se usássemos um comando direto)
    ```

    Aplique o Pod:

    ```bash
    kubectl apply -f pgbench-pod.yaml
    ```

2.  **Aguarde o Pod `pgbench-client` estar `Running`:**

    ```bash
    kubectl get pod pgbench-client
    ```

3.  **Inicialize o banco de dados `pgbench_db`:**
    Agora que o `pgbench-client` está rodando, vamos inicializar o banco de dados no PostgreSQL com dados de teste.

    ```bash
    kubectl exec -it pgbench-client -- pgbench --host=postgres-service --port=5432 --username=pgbench_user --initialize pgbench_db
    ```

      * `kubectl exec -it pgbench-client --`: Executa um comando interativamente dentro do container `pgbench-client`.
      * `pgbench --host=postgres-service ... --initialize pgbench_db`: Comando do `pgbench` para inicializar o banco de dados.
          * `host`: Usa o nome do `Service` do PostgreSQL para comunicação interna no cluster.
          * `port`: Porta do PostgreSQL.
          * `username`: Usuário do PostgreSQL.
          * `initialize`: Comando para criar as tabelas e dados de teste.

    Você verá uma saída como `creating tables...`, `generating data...`, `vacuuming...`, `done`.

4.  **Execute o Benchmark com `pgbench`:**
    Agora você pode executar o benchmark real.

    ```bash
    kubectl exec -it pgbench-client -- pgbench --host=postgres-service --port=5432 --username=pgbench_user --time=60 --client=4 --jobs=2 pgbench_db
    ```

      * `time=60`: Executa o benchmark por 60 segundos.
      * `client=4`: Simula 4 clientes paralelos.
      * `jobs=2`: Executa 2 threads de trabalho em paralelo (para aproveitar múltiplos cores se disponíveis).

    Você verá uma saída detalhada do benchmark, incluindo TPS (Transações por Segundo), latência, etc.

-----

### **Considerações Finais e Próximos Passos:**

  * **Cluster Real de PostgreSQL:** Para um cluster PostgreSQL com alta disponibilidade e replicação, é **fortemente recomendado** explorar [Operadores de PostgreSQL para Kubernetes](https://www.google.com/search?q=https://operatorhub.io/operators%3Fcategory%3DDatabase%26keyword%3Dpostgresql). Eles automatizam grande parte da complexidade de gerenciar um banco de dados stateful em Kubernetes.
  * **Senhas e Credenciais:** No exemplo, usamos senhas em texto puro (`mysecretpassword`). Em um ambiente de produção, você DEVE usar [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) para gerenciar credenciais de forma segura.
  * **Recursos do Raspberry Pi 3:** Lembre-se que o RPi 3 tem recursos limitados (1GB de RAM). Um PostgreSQL rodando e um `pgbench` podem consumir bastante RAM e CPU. Monitore o uso de recursos com `htop` ou `free -h` no seu Pi.
  * **Limpeza:** Para remover os recursos criados:
    ```bash
    kubectl delete -f pgbench-pod.yaml
    kubectl delete -f postgres-deployment.yaml
    kubectl delete -f postgres-service.yaml
    kubectl delete -f postgres-pvc.yaml
    ```
    Se você quiser remover os dados persistidos pelo `local-path` (que ficam no `/var/lib/rancher/k3s/storage` no nó), você precisará fazer isso manualmente após excluir o PVC.

Com estes passos, você terá um ambiente para testar o desempenho do PostgreSQL no seu cluster k3s.