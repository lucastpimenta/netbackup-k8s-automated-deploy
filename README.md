# netbackup-k8s-automated-deploy

Este guia fornece um passo a passo completo para a instalação e configuração automática do NetBackup Operator para Kubernetes. Todas as informações e comandos são baseados exclusivamente no documento oficial **NetBackup105\_AdminGuide\_Kubernetes.pdf**.

### **Fase 1: Preparação e Pré-requisitos do Ambiente**

Esta fase é dedicada exclusivamente à preparação do seu ambiente para garantir que a instalação ocorra sem problemas.

#### **1.1. Pré-requisitos Essenciais**

  * **Acesso Administrativo:** Você precisa de privilégios de administrador no cluster Kubernetes.
  * **Instalação do Helm:** O Helm v3 ou superior deve estar instalado em sua estação de trabalho.
  * **Obtenção dos Pacotes:** Baixe o pacote do **NetBackup Kubernetes Operator** (ex: `netbackupkops-10.5.x.x.tar.gz`) e a imagem do **NetBackup Data Mover** (ex: `veritasnetbackup-datamover-10.5.x.x.tar`) do portal de suporte da Veritas. Extraia o pacote do operador em seu diretório de trabalho.

#### **1.2. Preparação das Imagens de Contêiner**

As imagens do NetBackup devem ser carregadas em um repositório de contêineres que seja acessível pelo seu cluster Kubernetes. Os comandos abaixo usam o Docker como exemplo.

1.  **Faça Login no seu Repositório (se necessário):**
    ```bash
    docker login <URL_DO_SEU_REPOSITORIO>
    ```
2.  **Carregue, Marque (Tag) e Envie as Imagens:** Repita o processo para o arquivo `.tar` do **Operador** e do **Data Mover**.
    ```bash
    # Carrega a imagem do arquivo .tar para o cache local do Docker
    docker load -i <NOME_DO_ARQUIVO.tar>

    # Marque a imagem com a URL do seu repositório
    # Ex: docker tag veritas/netbackup-kops-operator:10.5 meu-repo.com/netbackup/operator:10.5
    docker tag <IMAGEM_CARREGADA:TAG_ORIGINAL> <URL_DO_SEU_REPOSITORIO/NOME_DA_IMAGEM:TAG_NOVA>

    # Envie a imagem marcada para o seu repositório
    docker push <URL_DO_SEU_REPOSITORIO/NOME_DA_IMAGEM:TAG_NOVA>
    ```

#### **1.3. Configuração do Firewall**

Garanta que as seguintes regras de firewall estejam em vigor para permitir a comunicação entre os componentes, conforme detalhado nas páginas 16 e 17 do Guia de Administração:

| Origem | Destino | Porta | Protocolo | Finalidade |
| :--- | :--- | :--- | :--- | :--- |
| Servidor Primário | Cluster Kubernetes | 443 | TCP | Comunicações HTTPS |
| Servidores de Mídia | Cluster Kubernetes | 443 | TCP | Comunicações HTTPS |
| Cluster Kubernetes | Servidor Primário | 1556 | TCP (Saída) | Implantação de certificados e comunicação PBX |
| Cluster Kubernetes | Servidores de Mídia | 1556 | TCP (Saída) | Implantação de certificados |
| Cluster Kubernetes | Servidor Primário e de Mídia | 13724 | TCP (Bidirecional) | VNETD para movimentação de dados |

### **Fase 2: Coleta de Dados para Configuração**

Nesta fase, vamos coletar e anotar todas as informações necessárias antes de começar a editar os arquivos de configuração.

#### **2.1. Checklist de Coleta de Dados**

**a. Chave de API do NetBackup:**

> **Como obter:** Na UI Web do NetBackup, vá para **Segurança \> Chaves de acesso \> Chaves de API** e clique em **Adicionar**.

> **Anote aqui:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**b. Namespace da Instalação:**

> **Como obter:** Defina um nome padrão para sua organização (ex: `netbackup`).

> **Anote aqui:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**c. Nome do Release (Implantação):**

> **Como obter:** Defina um nome único para esta instalação com o Helm (ex: `nbkops-prod`).

> **Anote aqui:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**d. Informações do Servidor Primário NetBackup:**

> **Como obter:**
>
>   * **FQDN:** Nome de domínio completo do servidor (ex: `nbmaster.suaempresa.com`).
>   * **Impressão Digital:** Na UI Web do NetBackup, vá para **Segurança \> Certificados \> Autoridade Certificadora**.

> **Anote o FQDN:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_
>
> **Anote a Impressão Digital:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**e. Informações do Cluster Kubernetes:**

> **Como obter:**
>
> ```bash
> kubectl cluster-info
> ```
>

> **Anote o FQDN do Cluster:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_
>
> **Anote a Porta do Cluster:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**f. URLs das Imagens de Contêiner:**

> **Como obter:** São as URLs completas que você definiu no Passo 1.2.

> **Anote a URL do Operador:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_
>
> **Anote a URL do Data Mover:** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**g. Nomes das Classes de Armazenamento e Snapshot:**

> **Como obter:**
>
> ```bash
> kubectl get storageclasses
> kubectl get volumesnapshotclasses
> ```
>

> **Anote a StorageClass (Block):** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_
>
> **Anote a StorageClass (Filesystem):** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_
>
> **Anote a VolumeSnapshotClass (Block):** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_
>
> **Anote a VolumeSnapshotClass (Filesystem):** \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

### **Fase 3: Configuração e Criação dos Arquivos**

Com os dados da Fase 2 em mãos, vamos agora criar os arquivos de configuração.

#### **3.1. Passo Condicional: Acesso a Repositórios Privados**

**Execute este passo apenas se suas imagens estiverem em um repositório privado que exige autenticação.** O `Secret` criado aqui será referenciado no arquivo `values.yaml`.

```bash
# Use o namespace que você anotou no Passo 2.1.b
kubectl create secret generic netbackupkops-docker-cred \
  --from-file=.dockerconfigjson=${HOME}/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson \
  -n <NAMESPACE_DA_INSTALACAO>
```

#### **3.2. Arquivo `nb-config-deploy-secret.yaml`**

Crie este arquivo para armazenar a chave de API de forma segura.

```yaml
# Arquivo: nb-config-deploy-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  # Use o valor anotado no Passo 2.1.b
  name: <NAMESPACE_DA_INSTALACAO>-nb-config-deploy-secret
  namespace: <NAMESPACE_DA_INSTALACAO>
type: Opaque
stringData:
  # Use o valor anotado no Passo 2.1.a
  apikey: <CHAVE_DE_API_DO_NETBACKUP>
```

#### **3.3. Arquivo `values.yaml`**

Edite o arquivo `netbackupkops-helm-chart/values.yaml` com os valores que você coletou.

```yaml
# Arquivo: netbackupkops-helm-chart/values.yaml
netbackupkops:
  containers:
    manager:
      # Use o valor anotado no Passo 2.1.f
      image: <URL_DA_IMAGEM_DO_OPERADOR>

  # Se você executou o Passo 3.1, descomente a seção abaixo.
  # O nome 'netbackupkops-docker-cred' corresponde ao secret criado.
  # imagePullSecrets:
  #   - name: netbackupkops-docker-cred

nbsetup:
  replicas: 1
  containers:
    netbackup_config_pod:
      # Use os valores anotados no Passo 2.1.d
      nbprimaryserver: <FQDN_DO_SERVIDOR_PRIMARIO>
      nbsha256fingerprint: <IMPRESSAO_DIGITAL_SHA256>

      # Use os valores anotados no Passo 2.1.e
      k8sCluster: <FQDN_DO_CLUSTER_K8S>
      k8sPort: <PORTA_DO_CLUSTER_K8S>

      # Use o valor anotado no Passo 2.1.f
      datamoverimage: <URL_DA_IMAGEM_DO_DATA_MOVER>

      # Use os valores anotados no Passo 2.1.g
      storageclassblock: <NOME_DA_STORAGECLASS_BLOCK>
      storageclassfilesystem: <NOME_DA_STORAGECLASS_FILESYSTEM>
      volumesnapshotclassblock: <NOME_DA_SNAPSHOTCLASS_BLOCK>
      volumesnapshotclassfilesystem: <NOME_DA_SNAPSHOTCLASS_FILESYSTEM>

      storageMap:
        # Substitua pelas suas classes do Passo 2.1.g
        <NOME_DA_STORAGECLASS_FILESYSTEM>:
          snapshotClass: <NOME_DA_SNAPSHOTCLASS_FILESYSTEM>
          storageClassForBackupDataMovement: <NOME_DA_STORAGECLASS_FILESYSTEM>
          storageClassForRestoreFromBackup: <NOME_DA_STORAGECLASS_FILESYSTEM>
        
        <NOME_DA_STORAGECLASS_BLOCK>:
          snapshotClass: <NOME_DA_SNAPSHOTCLASS_BLOCK>
          storageClassForBackupDataMovement: <NOME_DA_STORAGECLASS_BLOCK>
          storageClassForRestoreFromBackup: <NOME_DA_STORAGECLASS_BLOCK>
```

### **Fase 4: Execução e Verificação**

1.  **Aplique os Arquivos:**

    ```bash
    # Use o namespace anotado no Passo 2.1.b
    kubectl create namespace <NAMESPACE_DA_INSTALACAO>
    kubectl apply -f nb-config-deploy-secret.yaml -n <NAMESPACE_DA_INSTALACAO>
    ```

2.  **Execute a Instalação com Helm:**

    ```bash
    # Use o nome do release (Passo 2.1.c) e o namespace (Passo 2.1.b)
    helm install <NOME_DO_RELEASE> ./netbackupkops-helm-chart -n <NAMESPACE_DA_INSTALACAO>
    ```

3.  **Verifique a Implantação:**

      * **No Cluster:** Monitore os pods. O pod de configuração (`...-config-deploy...`) deve ser criado e completar sua tarefa.
        ```bash
        kubectl get pods -n <NAMESPACE_DA_INSTALACAO> --watch
        ```
      * **Na UI do NetBackup:** Navegue até **Cargas de Trabalho \> Kubernetes**. Seu cluster deve aparecer na lista com o status "Sucesso".
