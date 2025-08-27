## \# netbackup-k8s-automated-deploy

Este guia fornece um passo a passo completo para a instalação e configuração **automatizada** do NetBackup Operator para Kubernetes. Todas as informações e comandos são baseados exclusivamente no documento oficial **NetBackup105\_AdminGuide\_Kubernetes.pdf**.

## 📦 O que este Guia Faz
* Detalha todos os pré-requisitos de ambiente, firewall e imagens de contêiner.
* Organiza a coleta de todos os dados necessários antes da configuração.
* Fornece os arquivos YAML (Secret e values.yaml) prontos para preenchimento.
* Instrui sobre a execução da implantação via Helm e como verificar o sucesso da operação.
* Resulta em um cluster Kubernetes totalmente integrado e pronto para ser gerenciado pelo Veritas NetBackup.

## 🛠️ Tecnologias Envolvidas
* Veritas NetBackup 10.5
* Kubernetes (com suporte a APIs de Snapshot v1)
* Helm v3
* Docker (ou outra ferramenta de linha de comando para contêineres)

## 🚀 Como Usar
O processo é dividido em quatro fases: preparar o ambiente, coletar os dados, criar os arquivos e executar a implantação.

### 1️⃣ **Fase 1: Preparação do Ambiente**

Esta fase é dedicada exclusivamente à preparação do seu ambiente para garantir que a instalação ocorra sem problemas.

#### **1.1. Pré-requisitos Essenciais**

  * **Acesso Administrativo:** Você precisa de privilégios de administrador no cluster Kubernetes.
  * **Instalação do Helm:** O Helm v3 ou superior deve estar instalado em sua estação de trabalho.
  * **Obtenção dos Pacotes:** Baixe e extraia o pacote do **NetBackup Kubernetes Operator**.
  * **Preparo das Imagens de Contêiner:** Faça o upload das imagens do **NetBackup Operator** e do **Data Mover** para um repositório de contêineres que seja acessível pelo seu cluster Kubernetes.

#### **1.2. Configuração do Firewall**

Garanta que as seguintes regras de firewall estejam em vigor para permitir a comunicação entre os componentes:

| Origem                | Destino                       | Porta | Protocolo        | Finalidade                          |
| :-------------------- | :---------------------------- | :---- | :--------------- | :---------------------------------- |
| Servidor Primário     | Cluster Kubernetes            | 443   | TCP              | Comunicações HTTPS                  |
| Servidores de Mídia   | Cluster Kubernetes            | 443   | TCP              | Comunicações HTTPS                  |
| Cluster Kubernetes    | Servidor Primário             | 1556  | TCP (Saída)      | Comunicação PBX e Certificados      |
| Cluster Kubernetes    | Servidores de Mídia           | 1556  | TCP (Saída)      | Certificados                        |
| Cluster Kubernetes    | Servidor Primário e de Mídia  | 13724 | TCP (Bidirecional) | VNETD para movimentação de dados    |

### 2️⃣ **Fase 2: Coleta de Dados para Configuração**

Nesta fase, vamos coletar e anotar todas as informações que serão usadas como variáveis nas fases de configuração e execução.

#### **2.1. Checklist de Coleta de Dados**

**a. Chave de API do NetBackup:**

Na UI Web do NetBackup, vá para **Segurança \> Chaves de acesso \> Chaves de API** e crie uma nova chave.

**`<CHAVE_DE_API_DO_NETBACKUP>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**b. Namespace da Instalação:**

Defina um nome padrão para sua organização.

**`<NAMESPACE_DA_INSTALACAO>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**c. Nome do Release:**

Defina um nome único para esta instalação com o Helm.

**`<NOME_DO_RELEASE>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**d. URLs das Imagens de Contêiner:**

URLs completas que você definiu ao fazer `docker push`.

**`<URL_DA_IMAGEM_DO_OPERADOR>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<URL_DA_IMAGEM_DO_DATA_MOVER>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**e. Informações do Servidor Primário:**

**FQDN:** Nome de domínio completo do seu servidor.
**Impressão Digital:** Na UI Web do NetBackup, vá para **Segurança \> Certificados \> Autoridade Certificadora**.

**`<FQDN_DO_SERVIDOR_PRIMARIO>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<IMPRESSAO_DIGITAL_SHA256>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**f. Informações do Cluster Kubernetes:**

> ```bash
> kubectl cluster-info
> ```

**`<FQDN_DO_CLUSTER_K8S>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<PORTA_DO_CLUSTER_K8S>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**g. Nomes das Classes de Armazenamento e Snapshot:**

> ```bash
> kubectl get storageclasses
> kubectl get volumesnapshotclasses
> ```

**`<NOME_DA_STORAGECLASS_FILESYSTEM>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<NOME_DA_STORAGECLASS_BLOCK>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<NOME_DA_SNAPSHOTCLASS_FILESYSTEM>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

**`<NOME_DA_SNAPSHOTCLASS_BLOCK>`** = \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_

### 3️⃣ **Fase 3: Criação dos Arquivos de Configuração**

Com os dados da Fase 2 em mãos, vamos agora criar os arquivos de configuração.

#### **3.1. Passo Condicional: Acesso a Repositórios Privados**

**Execute este passo apenas se suas imagens estiverem em um repositório privado que exige autenticação.**

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
  name: <NAMESPACE_DA_INSTALACAO>-nb-config-deploy-secret
  namespace: <NAMESPACE_DA_INSTALACAO>
type: Opaque
stringData:
  apikey: <CHAVE_DE_API_DO_NETBACKUP>
```

#### **3.3. Arquivo `values.yaml`**

Edite o arquivo `netbackupkops-helm-chart/values.yaml` com os valores que você coletou.

```yaml
# Arquivo: netbackupkops-helm-chart/values.yaml
netbackupkops:
  containers:
    manager:
      image: <URL_DA_IMAGEM_DO_OPERADOR>

  # Se você executou o Passo 3.1, descomente a seção abaixo.
  # O nome 'netbackupkops-docker-cred' corresponde ao secret criado.
  # imagePullSecrets:
  #   - name: netbackupkops-docker-cred

nbsetup:
  replicas: 1
  containers:
    netbackup_config_pod:
      nbprimaryserver: <FQDN_DO_SERVIDOR_PRIMARIO>
      nbsha256fingerprint: <IMPRESSAO_DIGITAL_SHA256>
      k8sCluster: <FQDN_DO_CLUSTER_K8S>
      k8sPort: <PORTA_DO_CLUSTER_K8S>
      datamoverimage: <URL_DA_IMAGEM_DO_DATA_MOVER>
      storageclassblock: <NOME_DA_STORAGECLASS_BLOCK>
      storageclassfilesystem: <NOME_DA_STORAGECLASS_FILESYSTEM>
      volumesnapshotclassblock: <NOME_DA_SNAPSHOTCLASS_BLOCK>
      volumesnapshotclassfilesystem: <NOME_DA_SNAPSHOTCLASS_FILESYSTEM>

      storageMap:
        <NOME_DA_STORAGECLASS_FILESYSTEM>:
          snapshotClass: <NOME_DA_SNAPSHOTCLASS_FILESYSTEM>
          storageClassForBackupDataMovement: <NOME_DA_STORAGECLASS_FILESYSTEM>
          storageClassForRestoreFromBackup: <NOME_DA_STORAGECLASS_FILESYSTEM>
        
        <NOME_DA_STORAGECLASS_BLOCK>:
          snapshotClass: <NOME_DA_SNAPSHOTCLASS_BLOCK>
          storageClassForBackupDataMovement: <NOME_DA_STORAGECLASS_BLOCK>
          storageClassForRestoreFromBackup: <NOME_DA_STORAGECLASS_BLOCK>
```

### 4️⃣ **Fase 4: Execução e Verificação**

#### **4.1. Aplique os Arquivos**

```bash
# Use o namespace anotado no Passo 2.1.b
kubectl create namespace <NAMESPACE_DA_INSTALACAO>
kubectl apply -f nb-config-deploy-secret.yaml -n <NAMESPACE_DA_INSTALACAO>
```

#### **4.2. Execute a Instalação com Helm**

```bash
# Use o nome do release (Passo 2.1.c) e o namespace (Passo 2.1.b)
helm install <NOME_DO_RELEASE> ./netbackupkops-helm-chart -n <NAMESPACE_DA_INSTALACAO>
```

#### **4.3. Verifique a Implantação**

  * **No Cluster:** Monitore os pods. O pod de configuração (`...-config-deploy...`) deve ser criado e completar sua tarefa.
    ```bash
    kubectl get pods -n <NAMESPACE_DA_INSTALACAO> --watch
    ```
  * **Na UI do NetBackup:** Navegue até **Cargas de Trabalho \> Kubernetes**. Seu cluster deve aparecer na lista com o status "Sucesso".

## 🆘 Suporte

Para suporte, comece verificando se você seguiu todas as instruções corretamente. Se o problema persistir, considere consultar a documentação do Oracle e do NetBackup para configurações adicionais.

## 🌟 Contribuições

Contribuições são sempre bem-vindas! Se você tem uma sugestão para melhorar este script, sinta-se à vontade para criar um pull request.

## ✒️ Autor

[Lucas Pimenta](https://github.com/lucastpimenta) - Trabalho Inicial
