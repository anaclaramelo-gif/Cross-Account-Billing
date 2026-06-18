# Cross Account RealCloud - RealGlass Billing

## 1 - Processo de Cross Account:

Após a migração do cliente para o serviço de RealGlass Billing, inicia-se a etapa de configuração técnica. Para viabilizar a gestão financeira e aplicar as otimizações contratadas, a RealCloud requer acessos específicos à conta do cliente. Estas permissões são estritamente não invasivas e baseadas no princípio do menor privilégio, garantindo que apenas operações de billing sejam realizadas. A conexão será estabelecida via Cross-Account, conforme detalhado nas seções a seguir.

### 1.1. Objetivo Cross Account:

O objetivo desta arquitetura é viabilizar um modelo seguro, automatizado e padronizado de acesso cross-account às informações de billing e cost management das contas AWS dos clientes. A solução permite que a RealCloud acesse os dados de forma segura e controlada, utilizando policies de confiança e Roles IAM, fazendo com que não seja mais necessário o compartilhamento de credenciais e garantindo aderência às melhores práticas de segurança e ao least privilege principle.

Adicionalmente, a arquitetura foi concebida para simplificar e escalar o processo de integração de contas, utilizando uma stack do CloudFormation, para assegurar consistência na criação dos recursos necessários, uma função Lambda, para orquestração e uma tabela armazenada no DynamoDB, para registro e rastreabilidade das configurações realizadas. Esse modelo de arquitetura garante governança e auditabilidade, viabilizando análise de custos, geração de insights e otimização financeira de forma eficiente e sustentável.

### 1.2. Fluxo:

O fluxo acima descreve como a RealCloud habilita, de forma segura e automatizada, o acesso cross-account às informações de billing a conta do cliente, utilizando serviços nativos AWS. A solução é baseada em CloudFormation, IAM Roles, Lambda, DynamoDB e S3.

### 1.3. Componentes:

Conta RealCloud:
Amazon S3: Hospeda o template CloudFormation (YAML/JSON) que o cliente executa para realizar o provisionamento.

AWS Lambda: Atua como Custom Resource do CloudFormation.  
Valida que a conta é a Management Account da Organization — falha imediatamente se for standalone ou member account.  
Habilita o Trusted Access para StackSets via organizations:EnableAWSServiceAccess.  
Resolve o Root ID real da Organization via organizations:ListRoots.  
Registra Management Account e Member Accounts no DynamoDB.  
Processa notificações de exclusão de stacks (offboarding).

Amazon DynamoDB: Armazena metadados de todas as contas integradas: Management Account e Member Accounts.  
Campos: account_id, client_name, role_name, external_id, org_id, root_id, member_count, source, cross_account_link, status, updated_at.

Conta do Cliente:
AWS CloudFormation: Executa o template fornecido pela RealCloud.

RealCloudCrossAccountRole: IAM Role criada na Management Account que autoriza o acesso cross-account da RealCloud.  
Inclui permissão organizations:EnableAWSServiceAccess para que a Lambda possa ativar o Trusted Access.

Custom Resource EnableTrustedAccess:  
Invoca a Lambda para validar que é Management Account, ativar Trusted Access e resolver o Root ID da Organization.  
Retorna o Root Id como output para o StackSet usar automaticamente.

StackSet SERVICE_MANAGED (RealCloud-CrossAccount-MemberAccounts):  
Propaga o RealCloudCrossAccountRole para todas as member accounts da Organization.  
AutoDeployment habilitado: novas contas que entrem na Org recebem o role automaticamente.  
RetainStacksOnAccountRemoval: false — ao sair da Org, o role é removido da member account.

Custom Resource NotifyOrgOnboarding:  
Invoca a Lambda ao final do deploy para registrar a Management Account e listar todas as member accounts no DynamoDB.

Member Accounts (via StackSet)

RealCloudCrossAccountRole:  
Mesma role de acesso, porém sem permissões de organizations:EnableAWSServiceAccess (desnecessárias em member accounts).

Custom Resource NotifyMemberOnboarding:  
Notifica a Lambda com IsOrgMember: true para registrar a member account no DynamoDB com source: stackset.

### 1.4. Fluxo de Funcionamento da Arquitetura:

EXECUÇÃO DO TEMPLATE: O cliente acessa o link do template CloudFormation que está armazenado no bucket S3 da RealCloud. Ao executar o template em sua conta, é criada uma role IAM cross-account.

NOTIFICAÇÃO CROSS-ACCOUNT: Ao final da execução do CloudFormation na conta do cliente, um gatilho é disparado para invocar a função Lambda centralizada na conta da RealCloud. Esta função recebe um payload contendo os metadados da nova Stack, incluindo o ID da conta do cliente, o nome da Role criada, o nome da Stack, Link pré-populado de Cross-Account; e External ID (parâmetro opcional de segurança).

REGISTRO: A lambda grava essas informações no DynamoDB, permitindo registro de quais contas estão com role, auditoria e rastreabilidade e automação de acessos futuros.

---

## 2 - Políticas de Acesso Cross-Account

Para habilitar a integração cross-account, é criada na conta do cliente a IAM Role **RealCloudCrossAccount**, que pode ser assumida exclusivamente pela conta de Services da RealCloud por meio do **AWS Security Token Service (STS)** utilizando **AssumeRole**.

A role concede acesso **somente leitura (read-only)** e controlado a serviços relacionados a custos, billing, consumo, otimização, governança, segurança e inventário da infraestrutura, sem permissões de alteração, exclusão ou provisionamento de recursos.

---

## 2.1. Faturamento e Custos (Billing & Cost Management)

account:GetAccountInformation — Informações gerais da conta AWS  
billing:Get*, billing:List* — Dados de faturamento e contratos  
budgets:ViewBudget, budgets:Describe* — Orçamentos configurados e status de gastos  
cur:Get*, cur:Describe* — Cost and Usage Reports (CUR)  
ce:Get*, ce:Describe*, ce:List* — Cost Explorer — análises históricas e previsões  
invoicing:Get*, invoicing:List* — Faturas emitidas e histórico  
payments:Get*, payments:List* — Histórico de pagamentos  
purchase-orders:* — Ordens de compra  
pricing:DescribeServices — Tabela oficial de preços AWS  
freetier:GetFreeTier* — Monitoramento do Free Tier  
consolidatedbilling:* — Faturamento consolidado (Organizations)  
mapcredits:List* — Créditos do programa MAP  

---

## 2.2. Otimização Financeira e Planejamento (FinOps & Savings)

savingsplans:Describe*, savingsplans:List* — Savings Plans e reservas ativas  
bcm-pricing-calculator:Get*, bcm-pricing-calculator:List* — Calculadora de preços e projeções  
bcm-recommended-actions:List* — Ações recomendadas de otimização  
cost-optimization-hub:List* — Hub centralizado de recomendações de custo  
compute-optimizer:Get*, compute-optimizer:Describe*, compute-optimizer:Export* — Recomendações de rightsizing  
trustedadvisor:Describe*, trustedadvisor:List*, trustedadvisor:View* — Recomendações de performance e economia  

---

## 2.3. Infraestrutura, Performance e Utilização

ec2:Describe*, autoscaling:Describe* — Instâncias, ASGs e padrões de utilização  
eks:Describe* — Clusters Kubernetes  
lambda:Get*, lambda:List* — Funções Lambda  
s3:Get*, s3:List* — Buckets e storage  
rds:Describe* — Bancos de dados RDS  
dynamodb:Describe*, dynamodb:List* — Tabelas DynamoDB  

---

## 2.4. Monitoramento e Utilização Real (CloudWatch & Logs)

cloudwatch:Get*, cloudwatch:List* — Métricas operacionais  
logs:Describe*, logs:Get* — Logs e retenção  

---

## 2.5. Governança, Segurança e Conformidade

organizations:Describe*, organizations:List* — Estrutura AWS Organization  
organizations:EnableAWSServiceAccess — Trusted Access para StackSets  
organizations:DisableAWSServiceAccess, organizations:ListAWSServiceAccessForOrganization — Gerenciar Trusted Access  
config:Describe*, config:Get* — Conformidade  
securityhub:Get*, securityhub:Describe* — Segurança  
inspector2:List*, inspector2:Get* — Vulnerabilidades  
tag:Get* — Tags de custo  
servicequotas:Get*, servicequotas:List*, servicequotas:Describe* — Limites AWS  
support:Describe* — Suporte AWS  
servicecatalog:ListApplications — Service Catalog  

---

## 2.6. Modelo de Segurança do Acesso

O acesso ocorre exclusivamente via **AWS STS AssumeRole**, restrito à conta de Services da RealCloud previamente autorizada pelo cliente.

A role possui permissões de **somente leitura (read-only)**, não incluindo privilégios de criação, alteração, exclusão ou interrupção de recursos.

Adicionalmente, o acesso pode ser protegido por **External ID**.

---

## 3 - Como usar:

Forneça ao cliente a URL do template CloudFormation hospedado no S3:

URL:  
https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=https%3A%2F%2Frealcloudcrossaccount.s3.us-east-2.amazonaws.com%2FRealCloudCrossAccountClient.yml&stackName=RealCloud-CrossAccount-Client&param_ClientName=NOME_DO_CLIENTE  

---

## 3.1. Execução da pilha:

O cliente será direcionado diretamente para a tela do CloudFormation. Será necessário apenas clicar em “Create Stack”.

---

## 3.2. Após a execução do CloudFormation:

account_id: ID numérico da Management Account do cliente  
client_name: Nome identificador do cliente  
role_name: Nome da Role criada  
external_id: External ID (se fornecido)  
org_id: ID da AWS Organization  
root_id: Root ID da Organization — resolvido automaticamente  
member_count: Número de member accounts  
cross_account_link: URL de switch role  
status: active  
is_management_account: true  

Os registros das member accounts possuem o campo source:  
stackset: criado via StackSet  
org_discovery: identificado via Organization  

---

## 3.3. Acesso à Conta do Cliente:

Uso do cross_account_link para switch role.

---

## 3.4. Offboarding:

Remoção da Role  
Remoção via StackSet  
Limpeza no DynamoDB  

---

## 4 - Atualizações:

Atualização via CloudFormation  
StackSet atualiza automaticamente member accounts  

---

## 5 - Logs:

/aws/lambda/cross-account  

Eventos:
- execução
- assume role
- erros
- auditoria  

---

## 6 - Dúvidas:

6.1 Sim  
6.2 Não  
6.3 Não  
6.4 Não  
6.5 Sim  
6.6 Não  
6.7 Sim  
6.8 Sim  
6.9 Sob demanda  

---

## 7 - Repositório:

github

---

## 7 - Conclusão:

A integração Cross Account RealCloud – RealGlass Billing foi projetada para oferecer um modelo seguro, automatizado e transparente de acesso às informações de billing e cost management das contas AWS dos clientes.
  



