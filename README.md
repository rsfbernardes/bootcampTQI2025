☁️ Arquitetura de Processamento Assíncrono: S3, Lambda, EC2 e EBS

Esta documentação descreve uma arquitetura de referência na AWS (Amazon Web Services) projetada para lidar com cargas de trabalho de processamento assíncrono e de longa duração, garantindo a persistência dos dados através do EBS.

🎯 Objetivo da Arquitetura

O objetivo principal é desacoplar a ingestão de dados da sua etapa de processamento, garantindo a durabilidade e a capacidade de backup do ambiente de execução.

    O upload de um arquivo inicie o processamento automaticamente (Lambda).

    O processamento seja executado em um recurso dedicado e redimensionável (EC2).

    O armazenamento do sistema operacional e de quaisquer dados temporários ou logs importantes do EC2 seja persistente e recuperável (EBS).

📊 Componentes e Fluxo de Trabalho

1. 📂 Amazon S3 (Simple Storage Service)

Componente	Função	Detalhes
Bucket de Input	Repositório central para dados brutos/entrada.	Configurado como a Fonte de Evento para a função Lambda.
Bucket de Output	Repositório para resultados finais do processamento.	Ideal para o armazenamento de resultados finais duráveis.

2. ⚡️ AWS Lambda Function (Orquestração)

Componente	Função	Ação
Função Orquestradora	Atua como a "cola" para iniciar o processo. Não processa o dado em si.	Invocada pelo evento s3:ObjectCreated e envia um comando de execução para a instância EC2.

3. 🖥️ Amazon EC2 (Elastic Compute Cloud)

Componente	Função	Detalhes
Instância de Processamento	Executa o script de processamento de longa duração.	Deve ter o Agente SSM instalado para receber comandos da Lambda.

4. 💾 Amazon EBS (Elastic Block Store) [NOVO]

O EBS é um serviço de armazenamento em bloco persistente, ideal para ser usado como o disco rígido principal (Volume de Boot) e/ou como um disco de dados adicional para a instância EC2.
Componente	Função	Benefício Principal
Volume EBS	Fornece armazenamento persistente para a instância EC2.	Garante que dados do sistema operacional, aplicações, configurações e logs persistam mesmo se a instância EC2 for encerrada ou falhar.
Snapshots EBS	Backups pontuais do Volume EBS.	Permite criar backups (snapshots) facilmente, que podem ser usados para restaurar a instância EC2 ou para iniciar novas instâncias idênticas.

🔄 Fluxo de Processamento

    UPLOAD: Um arquivo é carregado no S3 Bucket (Input).

    GATILHO: O S3 dispara a Lambda Function com os detalhes do arquivo.

    ORQUESTRAÇÃO: A Lambda envia um comando via AWS SSM para a EC2 Instance.

    PROCESSAMENTO: A EC2 Instance lê o input do S3, usa seu Volume EBS para armazenar logs e arquivos temporários de trabalho, executa o processamento.

    OUTPUT: A EC2 Instance salva o resultado final de volta no S3 Bucket (Output).

    BACKUP: Periodicamente, um Snapshot EBS é criado para garantir o backup do ambiente EC2.

🔒 Considerações de Segurança (IAM)

A segurança é crucial para garantir que cada componente tenha apenas as permissões necessárias (Princípio do Mínimo Privilégio).

1. IAM Role para a Lambda Function

    ssm:SendCommand: Permissão para enviar comandos para a instância EC2.

    s3:GetObject (Opcional): Se o Lambda precisar ler metadados do arquivo de input.

2. IAM Role para a Instância EC2

    s3:GetObject e s3:PutObject: Permissões para ler e gravar dados nos Buckets S3.

    Permissões SSM: Necessárias para que o Agente SSM funcione e execute o comando enviado pela Lambda.

🚀 Como Fazer o Backup (Snapshots EBS)

Para aproveitar o EBS para backup, você pode configurar o Amazon Data Lifecycle Manager (DLM).

    Marcar o Volume EBS: Use tags (ex: Backup: True) no Volume EBS.

    Configurar o DLM: Crie uma política no DLM para automaticamente tirar Snapshots dos Volumes com essa tag em intervalos regulares (ex: diário), gerenciando a retenção dos backups.
