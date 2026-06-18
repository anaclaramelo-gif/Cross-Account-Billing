# Cross Account RealCloud - RealGlass Billing

---

## 1 - Processo de Cross Account

Após a migração do cliente para o serviço de RealGlass Billing, inicia-se a etapa de configuração técnica. Para viabilizar a gestão financeira e aplicar as otimizações contratadas, a RealCloud requer acessos específicos à conta do cliente. Estas permissões são estritamente não invasivas e baseadas no princípio do menor privilégio, garantindo que apenas operações de billing sejam realizadas. A conexão será estabelecida via Cross-Account, conforme detalhado nas seções a seguir.

---

### 1.1 - Objetivo Cross Account

O objetivo desta arquitetura é viabilizar um modelo seguro, automatizado e padronizado de acesso cross-account às informações de billing e cost management das contas AWS dos clientes. A solução permite que a RealCloud acesse os dados de forma segura e controlada, utilizando policies de confiança e Roles IAM, eliminando a necessidade de compartilhamento de credenciais e garantindo aderência às melhores práticas de segurança e ao least privilege principle.

Adicionalmente, a arquitetura foi concebida para simplificar e escalar o processo de integração de contas, utilizando:
- AWS CloudFormation (provisionamento)
- AWS Lambda (orquestração)
- Amazon DynamoDB (registro e rastreabilidade)

Esse modelo garante governança, auditabilidade e escalabilidade para análise de custos e otimização financeira.

---

### 1.2 - Fluxo

O fluxo abaixo descreve como a RealCloud habilita, de forma segura e automatizada, o acesso cross-account às informações de billing da conta do cliente, utilizando serviços nativos AWS como CloudFormation, IAM, Lambda, DynamoDB e S3.

<img width="644" height="288" alt="image" src="https://github.com/user-attachments/assets/a76aa10a-8ffc-496f-9faa-5e1b39e5c0ed" />

---

### 1.3 - Componentes

#### Conta RealCloud

- **Amazon S3**: Hospeda o template CloudFormation utilizado pelo cliente para provisionamento.
- **AWS Lambda**: Atua como Custom Resource do CloudFormation e:
  - Valida se a conta é Management Account
  - Habilita Trusted Access para StackSets (`organizations:EnableAWSServiceAccess`)
  - Resolve o Root ID da Organization (`organizations:ListRoots`)
  - Registra Management e Member Accounts no DynamoDB
  - Processa eventos de delete (offboarding)

- **Amazon DynamoDB**: Armazena metadados das contas integradas:
  - account_id
  - client_name
  - role_name
  - external_id
  - org_id
  - root_id
  - member_count
  - source
  - cross_account_link
  - status
  - updated_at

---

#### Conta do Cliente

- **AWS CloudFormation**: Executa o template fornecido pela RealCloud.
- **RealCloudCrossAccountRole**: Role IAM de acesso cross-account (read-only).
- **Custom Resource EnableTrustedAccess**:
  - Invoca Lambda para validação da Management Account
  - Ativa Trusted Access
  - Retorna Root ID para StackSets
- **StackSet (SERVICE_MANAGED)**:
  - Propaga role para todas as member accounts
  - AutoDeployment habilitado
  - Remove stacks automaticamente quando contas saem da Organization
- **Custom Resource NotifyOrgOnboarding**:
  - Registra Management Account
  - Lista member accounts no DynamoDB

---

#### Member Accounts (StackSet)

- **RealCloudCrossAccountRole**
- **Custom Resource NotifyMemberOnboarding**
  - Registra conta com `source: stackset`

---

### 1.4 - Fluxo de Funcionamento da Arquitetura

- Execução do template CloudFormation pelo cliente
- Criação da IAM Role na Management Account
- Lambda valida Management Account e habilita Trusted Access
- StackSet propaga role para member accounts
- Lambda registra todas as contas no DynamoDB
- Offboarding automático via delete stack

---

## 2 - Políticas de Acesso Cross-Account

A role **RealCloudCrossAccount** concede acesso somente leitura via AWS STS AssumeRole, permitindo análise de custos, billing, otimização e inventário.

---

### 2.1 - Faturamento e Custos

- account:GetAccountInformation — Informações da conta AWS
- billing:Get*, billing:List* — Dados de faturamento
- budgets:* — Orçamentos
- cur:* — Cost and Usage Reports
- ce:* — Cost Explorer
- invoicing:* — Faturas e histórico
- payments:* — Pagamentos
- purchase-orders:* — Ordens de compra
- pricing:DescribeServices — Tabela de preços AWS
- freetier:* — Free Tier
- consolidatedbilling:* — Billing consolidado
- mapcredits:* — Créditos MAP

---

### 2.2 - Otimização Financeira (FinOps)

- savingsplans:* — Savings Plans
- compute-optimizer:* — Rightsizing
- cost-optimization-hub:* — Recomendações
- trustedadvisor:* — Recomendações AWS
- bcm-* — Calculadora e recomendações

---

### 2.3 - Infraestrutura e Utilização

- ec2:* — Instâncias
- autoscaling:* — Auto Scaling
- eks:* — Kubernetes
- lambda:* — Serverless
- s3:* — Storage
- rds:* — Bancos de dados
- dynamodb:* — NoSQL

---

### 2.4 - Monitoramento

- cloudwatch:* — Métricas e logs
- logs:* — Logs e retenção

---

### 2.5 - Governança e Segurança

- organizations:* — AWS Organizations
- config:* — Compliance
- securityhub:* — Segurança
- inspector2:* — Vulnerabilidades
- tag:* — Tags de custos
- servicequotas:* — Limites
- support:* — Suporte AWS
- servicecatalog:* — Catálogo

---

### 2.6 - Segurança do Acesso

- Acesso via STS AssumeRole
- Somente leitura (read-only)
- Protegido por External ID (opcional)
- Sem permissões de criação, alteração ou exclusão

---

## 3 - Como usar

Use a URL abaixo para iniciar o onboarding:

https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=https%3A%2F%2Frealcloudcrossaccount.s3.us-east-2.amazonaws.com%2FRealCloudCrossAccountClient.yml&stackName=RealCloud-CrossAccount-Client&param_ClientName=NOME_DO_CLIENTE


---

### 3.1 - Execução da pilha

Clique em **Create Stack** no CloudFormation.

---

### 3.2 - Pós execução

O sistema registra automaticamente no DynamoDB:

- account_id
- client_name
- role_name
- external_id
- org_id
- root_id
- member_count
- cross_account_link
- status = active
- is_management_account = true

---

### 3.3 - Acesso

Use o campo `cross_account_link` para realizar switch role.

---

### 3.4 - Offboarding

- Remoção da role
- Remoção via StackSet
- Limpeza automática no DynamoDB

---

### 3.5 - Limpeza via Lambda

(Imagem do código aqui)

---

## 4 - Atualizações

- Atualizar stack via CloudFormation
- StackSet atualiza automaticamente member accounts
- Sem impacto operacional

---

## 5 - Logs

Logs disponíveis em:

    
  
  



