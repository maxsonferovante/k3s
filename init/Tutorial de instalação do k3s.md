Com certeza! O k3s é uma excelente escolha para quem busca um cluster Kubernetes leve e fácil de instalar, ideal para desenvolvimento, edge computing ou Raspberry Pi.

Aqui está um tutorial passo a passo para instalar o k3s.

---

## **Tutorial de Instalação do k3s**

Este tutorial cobrirá a instalação de um cluster k3s de nó único (server/agent no mesmo dispositivo), que é o cenário mais comum para testes e ambientes de baixa escala, especialmente em um Raspberry Pi.

### **Pré-requisitos:**

1.  **Máquina Linux:**
    * Um Raspberry Pi (3, 4, 5, ou Zero 2 W com 512MB+ de RAM) com um sistema operacional baseado em Debian (como o Raspberry Pi OS Lite, que é o mais recomendado para k3s, pois não tem interface gráfica consumindo recursos).
    * Qualquer outra máquina Linux (Ubuntu, Debian, CentOS, etc.) com 512MB de RAM ou mais.
    * **Importante para Raspberry Pi:** Certifique-se de que o sistema operacional esteja atualizado:
        ```bash
        sudo apt update
        sudo apt upgrade -y
        ```
    * **Conexão com a Internet:** A máquina de destino precisa de acesso à internet para baixar os pacotes do k3s.

2.  **Acesso Root ou `sudo`:** Você precisará de permissões de superusuário para executar os comandos de instalação.

3.  **Kernel Linux com `cgroups` habilitados (Crucial para Raspberry Pi):**
    Este é um problema comum em Raspberry Pi. Se você estiver usando um, verifique e/ou adicione as seguintes opções ao arquivo `cmdline.txt` no seu cartão SD (geralmente em `/boot/firmware/cmdline.txt` ou `/boot/cmdline.txt`).
    * **Edite o arquivo:**
        ```bash
        sudo nano /boot/firmware/cmdline.txt
        # Ou, em sistemas mais antigos, pode ser: sudo nano /boot/cmdline.txt
        ```
    * **Adicione (ou certifique-se de que estejam lá) as opções:**
        **IMPORTANTE:** Mantenha TUDO em uma única linha, separando as opções por espaços.
        ```
        cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
        ```
        Exemplo: Se sua linha original for `console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait`, ela se tornará:
        `console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1`
    * **Salve (Ctrl+O, Enter) e Saia (Ctrl+X).**
    * **Reinicie a máquina:**
        ```bash
        sudo reboot
        ```
    * **Verifique se os cgroups estão habilitados após o reboot:**
        ```bash
        grep -i cgroup /proc/cmdline
        ```
        Você deve ver as opções que adicionou na saída.

---

### **Passo 1: Instalação do k3s**

O método mais simples e recomendado é usar o script de instalação oficial do k3s. Este script baixa a versão mais recente, instala os binários, configura o serviço `systemd` e o `kubectl` automaticamente.

Execute o seguinte comando na sua máquina Linux (Raspberry Pi ou outra):

```bash
curl -sfL https://get.k3s.io | sh -
```

**Explicação do comando:**
* `curl -sfL https://get.k3s.io`: Baixa o script de instalação do URL do k3s.
    * `-s`: Modo silencioso (não mostra o progresso do download).
    * `-f`: Falha silenciosamente em caso de erro HTTP.
    * `-L`: Segue redirecionamentos.
* `| sh -`: Executa o script baixado com o interpretador de shell.

**O que acontece durante a instalação:**
* O script detecta a arquitetura do seu sistema.
* Baixa o binário do k3s apropriado.
* Instala o `k3s` como um serviço `systemd`.
* Configura automaticamente o `kubectl`, `crictl`, `ctr` e `k3s-uninstall.sh` no `/usr/local/bin`.
* Inicia o serviço `k3s`.

---

### **Passo 2: Verificando a Instalação**

Após a instalação, é crucial verificar se o k3s está rodando corretamente.

1.  **Verificar o status do serviço k3s:**
    ```bash
    sudo systemctl status k3s
    ```
    A saída deve mostrar `Active: active (running)`.

2.  **Verificar os nós do cluster:**
    O `kubectl` é a ferramenta de linha de comando para interagir com clusters Kubernetes. O k3s instala uma versão dela em `/usr/local/bin`.

    ```bash
    sudo kubectl get nodes
    ```
    Você deve ver o nome da sua máquina listado com o status `Ready`. Isso indica que o nó está saudável e pronto para agendar cargas de trabalho.

3.  **Verificar os pods do sistema k3s:**
    O k3s instala vários componentes essenciais em seu próprio namespace. Você pode vê-los com:

    ```bash
    sudo kubectl get pods -A
    ```
    Todos os pods devem estar em status `Running` ou `Completed`. Os pods importantes aqui são `coredns`, `helm-install-traefik`, `metrics-server`, e `traefik`.

---

### **Passo 3: Configurando o Acesso Remoto (Opcional, mas Altamente Recomendado)**

É muito mais conveniente gerenciar seu cluster k3s do seu PC/laptop do que fazer SSH para o Raspberry Pi toda vez.

1.  **No seu Raspberry Pi (ou máquina k3s):**
    O arquivo `kubeconfig` gerado pelo k3s está localizado em `/etc/rancher/k3s/k3s.yaml`.
    Obtenha o endereço IP da sua máquina k3s:
    ```bash
    ip a | grep inet | grep -v 127.0.0.1 | grep -v ::1
    ```
    Anote o endereço IP (ex: `192.168.1.100`).

2.  **No seu PC/Laptop:**
    * **Copie o `kubeconfig`:** Use `scp` para copiar o arquivo do Pi para o seu PC/laptop. Substitua `[seu_usuario_no_pi]` pelo nome de usuário do SSH no Pi (ex: `pi`) e `[ip_do_seu_pi]` pelo IP que você anotou.
        ```bash
        scp [seu_usuario_no_pi]@[ip_do_seu_pi]:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s.yaml
        ```
    * **Instale o `kubectl` (se ainda não tiver):** Siga as instruções oficiais do Kubernetes para instalar o `kubectl` no seu sistema operacional.
        [Guia de instalação do kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
    * **Ajuste o endereço IP no `kubeconfig` copiado:** Abra o arquivo `~/.kube/k3s.yaml` no seu PC/laptop com um editor de texto.
        Localize a linha que começa com `server: https://127.0.0.1:6443` (ou `localhost`).
        **Altere `127.0.0.1` para o endereço IP do seu Raspberry Pi.**
        ```yaml
        # Exemplo de como deve ficar (apenas a seção relevante)
        clusters:
        - cluster:
            certificate-authority-data: ...
            server: https://[ip_do_seu_pi]:6443  # <--- Mude este IP!
          name: default
        ```
    * **Defina a variável de ambiente `KUBECONFIG`:** Isso informará ao `kubectl` qual arquivo de configuração usar.
        ```bash
        export KUBECONFIG=~/.kube/k3s.yaml
        ```
        Para tornar isso permanente, adicione a linha ao seu `~/.bashrc` (para Bash) ou `~/.zshrc` (para Zsh).
    * **Teste o acesso remoto:**
        ```bash
        kubectl get nodes
        ```
        Você deve ver a sua máquina k3s listada como `Ready`.

---

### **Passa 4: Desinstalando o k3s (Caso precise)**

Se você precisar desinstalar o k3s para uma nova instalação ou para liberar recursos, o instalador do k3s também fornece um script de desinstalação.

1.  **No seu Raspberry Pi (ou máquina k3s):**
    ```bash
    sudo /usr/local/bin/k3s-uninstall.sh
    ```
2.  **Reinicie a máquina (opcional, mas recomendado):**
    ```bash
    sudo reboot
    ```

---

Parabéns! Você tem um cluster k3s funcional. Agora você pode começar a implantar suas aplicações, experimentar com manifestos YAML, Helm charts e explorar o mundo do Kubernetes.