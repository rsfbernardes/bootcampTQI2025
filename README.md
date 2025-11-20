# ☁️ Arquitetura de Processamento Assíncrono: S3, Lambda, EC2 e EBS

Esta documentação descreve uma arquitetura de referência na **AWS (Amazon Web Services)** projetada para lidar com cargas de trabalho de **processamento assíncrono e de longa duração**, garantindo a **persistência dos dados** através do EBS.

---

## 🎯 Objetivo da Arquitetura

O objetivo principal é **desacoplar** a ingestão de dados da sua etapa de processamento, garantindo a **durabilidade** e a **capacidade de backup** do ambiente de execução.

* O *upload* de um arquivo inicie o processamento automaticamente (**Lambda**).
* O processamento seja executado em um recurso dedicado e redimensionável (**EC2**).
* O armazenamento do sistema operacional e *logs* importantes do EC2 seja **persistente e recuperável** (**EBS**).

---

## 📊 Componentes Principais

| Componente | Função | Detalhes/Configuração |
| :--- | :--- | :--- |
| **1. 📂 Amazon S3** | **Armazenamento de Objetos Durável** | **Bucket de Input:** Repositório de dados brutos e **Fonte de Evento** para o Lambda (`s3:ObjectCreated`). |
| | **Bucket de Output** | Repositório para resultados finais do processamento. |
| **2. ⚡️ AWS Lambda** | **Orquestração *Serverless*** | Atua como **gatilho** e **controlador**. Não processa os dados, apenas envia o comando de execução para o EC2. |
| **3. 🖥️ Amazon EC2** | **Computação de Longa Duração** | **Instância de Processamento** que executa o *script* pesado. Deve ter o **Agente SSM** instalado. |
| **4. 💾 Amazon EBS** | **Armazenamento em Bloco Persistente** | **Volume EBS** acoplado ao EC2 para garantir que o SO, configurações e *logs* persistam e sejam passíveis de backup. |

### Detalhe do EBS

O **EBS** é crucial para a durabilidade do ambiente de execução da tarefa.

| Tipo | Função | Benefício Principal |
| :--- | :--- | :--- |
| **Volume EBS** | Fornece armazenamento persistente para a instância EC2. | Garante que dados do sistema operacional e aplicações **persistam** mesmo que a instância EC2 falhe ou seja encerrada. |
| **Snapshots EBS** | Backups pontuais do Volume EBS. | Permite criar **backups** que podem ser usados para restaurar ou iniciar novas instâncias idênticas. |

---

## 🔄 Fluxo de Processamento

O fluxo é sequencial e assíncrono:

1.  **UPLOAD:** Um arquivo é carregado no **S3 Bucket (Input)**.
2.  **GATILHO:** O S3 dispara a **Lambda Function** com os detalhes do arquivo.
3.  **ORQUESTRAÇÃO:** A Lambda envia um comando via **AWS SSM** para a **EC2 Instance**.
4.  **PROCESSAMENTO:** A EC2 Instance lê o *input* do S3, utiliza seu **Volume** EBS para logs e arquivos temporários, e executa o processamento.
5.  **OUTPUT**: A EC2 Instance salva o resultado final de volta no **S3 Bucket (Output)**.
6.  **BACKUP**: Periodicamente, um **Snapshot EBS** é criado para garantir o backup do ambiente EC2.

---

## 🔒 Considerações de Segurança (IAM)

A segurança é gerenciada pelo **IAM (Identity and Access Management)**, aplicando o **Princípio do Mínimo Privilégio**.

### 1. IAM Role para a Lambda Function
* `ssm:SendCommand`: Permissão para enviar comandos para a instância EC2.
* `s3:GetObject` (Opcional): Se o Lambda precisar ler metadados do arquivo de *input*.

### 2. IAM Role para a Instância EC2
* `s3:GetObject` e `s3:PutObject`: Permissões para ler e gravar dados nos Buckets S3.
* Permissões SSM: Necessárias para que o Agente SSM funcione e receba o comando da Lambda.

---

## 🚀 Como Fazer o Backup (Snapshots EBS)

Para automatizar a rotina de backup dos Volumes EBS, recomenda-se usar o **Amazon Data Lifecycle Manager (DLM)**:

* **Marcar o Volume EBS:** Use *tags* (ex: `Backup: True`) no Volume EBS.
* **Configurar o DLM:** Crie uma política que automaticamente tira **Snapshots** dos Volumes com essa *tag* em intervalos regulares (ex: diário), gerenciando a retenção dos backups.
