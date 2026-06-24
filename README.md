# Cross Account RealCloud - RealGlass Billing



## 1 - Processo de Cross Account: 



 Após a migração do cliente para o serviço de RealGlass Billing, inicia-se a etapa de configuração técnica. Para viabilizar a gestão financeira e aplicar as otimizações contratadas, a RealCloud requer acessos específicos à conta do cliente. Estas permissões são estritamente não invasivas e baseadas no princípio do menor privilégio, garantindo que apenas operações de billing sejam realizadas. A conexão será estabelecida via Cross-Account, conforme detalhado nas seções a seguir.
 
---

### 1.1. Objetivo Cross Account:
 O objetivo desta arquitetura é viabilizar um modelo seguro, automatizado e padronizado de acesso cross-account às informações de billing e cost management das contas AWS dos clientes. A solução permite que a RealCloud acesse os dados de forma segura e controlada, utilizando policies de confiança e Roles IAM, fazendo com que não seja mais necessário o compartilhamento de credenciais e garantindo aderência às melhores práticas de segurança e ao least privilege principle
 
 Adicionalmente, a arquitetura foi concebida para simplificar e escalar o processo de integração de contas, utilizando uma stack do CloudFormation, para assegurar consistência na criação dos recursos necessários, uma função Lambda, para orquestração e uma tabela armazenada no DynamoDB, para registro e rastreabilidade das configurações realizadas. Esse modelo de arquitetura garante governança e auditabilidade, viabilizando análise de custos, geração de insights e otimização financeira de forma eficiente e sustentável.
1.2. Fluxo:


![RealCloud Architecture](./realcloud_org_architecture.png)


 O fluxo acima descreve como a RealCloud habilita, de forma segura e automatizada, o acesso cross-account às informações de billing a conta do cliente, utilizando serviços nativos AWS. A solução é baseada em CloudFormation, IAM Roles, Lambda, DynamoDB e S3. 
 
### 1.3. Componentes:
1.3. Componentes:
# - Conta RealCloud
 - Amazon S3:
 Hospeda o template CloudFormation (YAML) disponibilizado ao cliente para realizar a integração.
 - AWS Lambda:
 Atua como Custom Resource do CloudFormation.
 Responsabilidades:
 Assume o role criado na conta do cliente.
 Valida que a conta pertence a uma AWS Organization.
 Garante que a execução está sendo realizada na Management Account, falhando caso seja uma Member Account.
 Habilita o Trusted Access do CloudFormation StackSets por meio de organizations:EnableAWSServiceAccess.
 Ativa o Organizations Access do CloudFormation via ActivateOrganizationsAccess.
 Obtém o Root ID da Organization através de organizations:ListRoots.
 Registra apenas a Management Account no DynamoDB.
 Processa eventos de exclusão (offboarding), removendo o registro correspondente do DynamoDB.

 - Amazon DynamoDB:
 Armazena os metadados das organizações integradas.
 Campos armazenados:
 account_id
 client_name
 role_name
 external_id
 stack_name
 cross_account_link
 is_management_account
 org_id
 root_id
 status
 updated_at

 A tabela armazena exclusivamente informações da Management Account. Nenhuma Member Account é registrada.
# - Conta do Cliente
 - AWS CloudFormation:
 Executa o template fornecido pela RealCloud.
 - RealCloudCrossAccountRole:
 Role IAM criada na Management Account.
 Permite que a RealCloud assuma acesso cross-account e consulte informações relacionadas a:
 Billing e Cost Explorer;
 Organizations;
 Compute;
 Storage;
 Databases;
 Monitoramento;
 Serviços de otimização;
 Recursos utilizados pelo ambiente.
 - Custom Resource EnableTrustedAccess:
 Invoca a Lambda da RealCloud para:
 Assumir o role recém-criado;
 Validar que a conta é a Management Account;
 Garantir que a conta faz parte de uma AWS Organization;
 Habilitar o Trusted Access do CloudFormation StackSets;
 Ativar o Organizations Access do CloudFormation;
 Resolver o Root ID da Organization.
 - Custom Resource NotifyOrgOnboarding:
 Executado ao final do deploy.
 Invoca a Lambda da RealCloud para registrar no DynamoDB os metadados da Management Account, incluindo:
 Account ID;
 Nome do cliente;
 External ID;
 Organization ID;
 Root ID;
 Link de acesso cross-account;
 Status da integração.
 

### 1.4. Fluxo de Funcionamento da Arquitetura:
1. Execução do Template:
 O cliente acessa o template CloudFormation hospedado no bucket S3 da RealCloud e realiza o deploy na Management Account da AWS Organization.
2. Criação do IAM Role:
 O CloudFormation cria o recurso RealCloudCrossAccountRole, que permitirá à RealCloud assumir acesso cross-account na conta do cliente.
3. Validação da Management Account e Habilitação do Trusted Access:
O recurso EnableTrustedAccess invoca a Lambda da RealCloud.
A função:
Assume o role recém-criado;
Verifica se a conta pertence a uma AWS Organization;
Garante que a conta é a Management Account;
Habilita o Trusted Access do serviço CloudFormation StackSets;
Ativa o Organizations Access do CloudFormation;
Obtém o Root ID da Organization.
 Caso a execução seja realizada em uma Member Account ou em uma conta standalone, o processo é interrompido.
4. Registro da Organização:
Após a validação, o recurso NotifyOrgOnboarding invoca novamente a Lambda.
A função registra no DynamoDB os metadados da Management Account:
Account ID;
Client Name;
Role Name;
External ID;
Stack Name;
Organization ID;
Root ID;
Link de acesso cross-account;
Timestamp da última atualização;
Status da integração.


## 2 - Políticas de Acesso Cross-Account:

---

Para habilitar a integração cross-account, é criada na conta do cliente a IAM Role RealCloudCrossAccount, que pode ser assumida exclusivamente pela conta de Services da RealCloud por meio do AWS Security Token Service (STS) utilizando AssumeRole.

A role concede acesso somente leitura (read-only) e controlado a serviços relacionados a custos, billing, consumo, otimização, governança, segurança e inventário da infraestrutura, sem permissões de alteração, exclusão ou provisionamento de recursos.


2.1. Faturamento e Custos (Billing & Cost Management):

- account:GetAccountInformation, account:ListRegions — Informações gerais da conta e regiões habilitadas.
- billing:Get*, billing:List* — Dados de faturamento e contratos.
- budgets:ViewBudget, budgets:Describe* — Orçamentos configurados e status de gastos.
- cur:Get*, cur:Describe* — Cost and Usage Reports (CUR).
- ce:Get*, ce:Describe*, ce:List* — Cost Explorer, análises históricas e previsões de custos.
- invoicing:Get*, invoicing:List* — Faturas emitidas e histórico.
- payments:Get*, payments:List* — Histórico de pagamentos.
- purchase-orders:GetPurchaseOrder, purchase-orders:ViewPurchaseOrders — Ordens de compra.
- pricing:DescribeServices — Consulta à tabela oficial de preços da AWS.
- freetier:GetFreeTier* — Monitoramento do Free Tier.
- consolidatedbilling:GetAccountBillingRole, consolidatedbilling:ListLinkedAccounts — Faturamento consolidado em ambientes AWS Organizations.
- mapcredits:List* — Créditos do programa AWS Migration Acceleration Program (MAP).

2.2. FinOps, Planejamento e Recomendações:

- savingsplans:Describe*, savingsplans:List* — Savings Plans ativos.
- bcm-pricing-calculator:Get*, bcm-pricing-calculator:List* — Calculadora de preços e projeções financeiras.
- bcm-recommended-actions:List* — Recomendações financeiras.
- cost-optimization-hub:List* — Hub centralizado de otimização de custos.
- compute-optimizer:Get*, compute-optimizer:Describe*, compute-optimizer:Export* — Recomendações de rightsizing.
- trustedadvisor:Describe*, trustedadvisor:List*, trustedadvisor:View* — Recomendações de desempenho, disponibilidade e economia.

2.3. Cost Allocation, Categorias e Anomaly Detection:

- ce:ListCostAllocationTags, ce:GetTags — Tags utilizadas para alocação de custos.
- ce:ListCostCategoryDefinitions — Categorias de custo definidas na conta.
- ce:GetAnomalies — Detecção de anomalias de custos.
- ce:GetAnomalyMonitors — Monitores de anomalias configurados.
- ce:GetAnomalySubscriptions — Configurações de notificações de anomalias.

2.4. Reserved Instances e Savings:

- ec2:DescribeReservedInstances
- ec2:DescribeReservedInstancesOfferings
- rds:DescribeReservedDBInstances
- elasticache:DescribeReservedCacheNodes
- elasticache:DescribeReservedCacheNodesOfferings
Permitem identificar reservas existentes e oportunidades de otimização financeira.

2.5. Computação, Containers e Escalabilidade:

- ec2:Describe* — Instâncias EC2.
- autoscaling:Describe*, DescribePolicies, DescribeScalingActivities — Auto Scaling Groups.
- eks:Describe*, eks:List* — Clusters Amazon EKS.
- ecs:Describe*, ecs:List* — Clusters Amazon ECS.
- lambda:Get*, lambda:List* — Funções AWS Lambda.

2.6. Armazenamento e Bancos de Dados: 

- s3:Get*, s3:List* — Buckets e configurações.
- rds:Describe* — Amazon RDS.
- dynamodb:Describe*, dynamodb:List* — Amazon DynamoDB.
- elasticache:Describe*, elasticache:List* — ElastiCache.
- redshift:Describe*, redshift:List*, redshift:View* — Amazon Redshift.
- elasticfilesystem:Describe*, elasticfilesystem:List* — Amazon EFS.
- glacier:Describe*, glacier:List*, glacier:GetVaultNotifications — Armazenamento de longo prazo.

2.7. Redes, Balanceamento e Distribuição:
- elasticloadbalancing:Describe* — Load Balancers.
- cloudfront:Get*, cloudfront:List* — Distribuições CloudFront.
- globalaccelerator:Describe*, globalaccelerator:List* — AWS Global Accelerator.
- directconnect:Describe* — AWS Direct Connect.

2.8. Mensageria, Streaming e Integração: 

- sqs:Get*, sqs:List* — Amazon SQS.
- sns:Get*, sns:List* — Amazon SNS.
- events:Describe*, events:List* — Amazon EventBridge.
- kinesis:Describe*, kinesis:List* — Amazon Kinesis.
- firehose:Describe*, firehose:List* — Amazon Kinesis Data Firehose.
- states:Describe*, states:List* — AWS Step Functions.
- glue:Get*, glue:List* — AWS Glue.
- transfer:Describe*, transfer:List* — AWS Transfer Family.

2.9. Machine Learning e Inteligência Artificial:

- sagemaker:Describe*
- sagemaker:List*
Permite identificar recursos de Machine Learning e potenciais oportunidades de otimização.

2.10. End User Computing:

- workspaces:Describe* — Amazon WorkSpaces.
- appstream:Describe*, appstream:List* — Amazon AppStream 2.0.

2.11. Backup e Recuperação: 

- backup:Describe*
- backup:List*
- backup:Get*
Permite visibilidade sobre planos, vaults e jobs do AWS Backup.

2.12. Marketplace:

- aws-marketplace:GetEntitlements
- aws-marketplace:ViewSubscriptions
Permite identificar produtos e assinaturas adquiridos pelo AWS Marketplace.

2.13. Monitoramento e Observabilidade:
- cloudwatch:Get*, cloudwatch:List* — Métricas operacionais.
- logs:Describe*, logs:Get* — Logs e políticas de retenção.

2.14. Governança, Segurança e Conformidade:
- organizations:Describe*, organizations:List* — Estrutura da AWS Organization.-
- organizations:EnableAWSServiceAccess — Habilita o Trusted Access do CloudFormation StackSets (somente na Management Account).
- organizations:DisableAWSServiceAccess
- organizations:ListAWSServiceAccessForOrganization
- config:Describe*, config:Get* — AWS Config.
- securityhub:Get*, securityhub:Describe* — AWS Security Hub.
- inspector2:List*, inspector2:Get* — Amazon Inspector.
- tag:Get* — Tags de recursos.
- servicequotas:Get*, servicequotas:List*, servicequotas:Describe* — Service Quotas.
- support:Describe* — Casos de suporte.
- servicecatalog:ListApplications — Aplicações registradas.
- wafv2:Get*, wafv2:List* — AWS WAF.
- shield:Describe*, shield:List* — AWS Shield.


## 3 - Como usar:

---
Forneça ao cliente a URL do template CloudFormation hospedado no S3
A URL Executa o onboarding na Management Account e propaga automaticamente o acesso para todas as contas-membro da organização por meio de AWS CloudFormation StackSets.
URL:
https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=https%3A%2F%2Frealcloudcrossaccount.s3.us-east-2.amazonaws.com%2FRealCloudCrossAccountClient.yml&stackName=RealCloud-CrossAccount-Management&param_ClientName=NOME_DO_CLIENTE



### 3.1. Execução da pilha
 O cliente será direcionado diretamente para a tela do CloudFormation com todos os parâmetros necessários já preenchidos. Será necessário apenas clicar em “Create Stack”.

### 3.2. Após a execução do CloudFormation:
 Assim que a pilha for criada com sucesso na conta do cliente, um processo automático (Custom Resource) enviará os metadados da integração para a conta de Services da RealCloud. A tabela DynamoDB será preenchida automaticamente com as seguintes informações:
 
- account_id: ID numérico da Management Account do cliente.
- client_name: Nome identificador fornecido no parâmetro ClientName.
- role_name: Nome da role criada para acesso cross-account (por exemplo, RealCloudCrossAccount).
- external_id: External ID utilizado, caso tenha sido fornecido.
- stack_name: Nome da stack do CloudFormation utilizada durante a implantação.
- org_id: Identificador da AWS Organization (por exemplo, o-abc123).
- root_id: Root ID da AWS Organization (por exemplo, r-ab12), obtido automaticamente.
- cross_account_link: URL para realizar o Switch Role na Management Account.
- status: Estado atual da integração, definido como active.
- is_management_account: Indica que o registro corresponde à Management Account, com valor true.
- updated_at: Data e hora da última atualização do registro.

  
### 3.3. Acesso à Conta do Cliente
 Com o registro criado no DynamoDB, você já poderá realizar o "switch role" (salto) para a conta do cliente utilizando o link gerado na coluna cross_account_link.
 
### 3.4. Offboarding
 Caso a stack seja removida pelo cliente, o CloudFormation envia um evento de exclusão para a Lambda.
A função remove do DynamoDB o registro correspondente à Management Account, concluindo o processo de offboarding.

## 4 - Atualizações:

---

  Quando uma nova versão do template estiver disponível no bucket S3 da RealCloud, o processo de atualização é simples: 
Abra o link do template no CloudFormation. 

1. No CloudFormation, selecione a stack existente e clique em Update. 

2. Confirme a atualização — o CloudFormation aplicará apenas as mudanças necessárias. 

3. O IAM Role e as permissões serão ajustados automaticamente, sem impacto na operação. 



## 5 - Logs:

---

 A função Lambda também gera logs no Amazon CloudWatch, na conta de Services da RealCloud, permitindo o acompanhamento da execução e facilitando o troubleshooting em caso de erros ou falhas no processo de integração cross-account.
 Todos os logs dessa automação são gerados automaticamente no Amazon CloudWatch Logs, no seguinte local:
/aws/lambda/cross-account

Nesse Log Group são registrados os principais eventos da execução, incluindo:
Recebimento do evento do CloudFormation


- Execução do acesso cross-account via IAM Role


- Identificação da conta do cliente (Account Alias ou Account ID)


- Criação, atualização ou remoção do registro da integração cross-account


- Envio do status de sucesso ou falha para o CloudFormation


 Cada execução da Lambda gera um Log Stream, que pode ser utilizado para troubleshooting.
 Em caso de falha na integração cross-account, o próprio CloudFormation refere o Log Stream da execução, facilitando a análise.
 Os logs são mantidos na conta de Services e podem ser utilizados para auditoria, rastreabilidade e diagnóstico de problemas, não sendo necessária nenhuma ação por parte do cliente.


## 6 - Dúvidas:

---

### 6.1. O cliente pode remover o acesso quando quiser?
Sim.
 O cliente pode remover o acesso a qualquer momento, bastando excluir a stack do   CloudFormation ou a IAM Role RealCloudCrossAccount criada para a integração.
 Após a remoção, a RealCloud perde imediatamente o acesso à conta do cliente.

### 6.2. Há impacto em custos?
Não.
 A criação da IAM Role e das políticas não gera custo adicional para o cliente.
 O acesso é somente para leitura e análise de dados já existentes na conta.

### 6.3. A RealCloud consegue criar, alterar ou excluir recursos?
Não.
 A role possui permissões restritas, focadas apenas em billing, custos, consumo e otimização, sem permitir ações de criação, modificação ou exclusão de recursos.

### 6.4. A RealCloud tem acesso a dados de aplicações?
Não.
 A integração não concede acesso a dados de aplicação, bancos de dados, código ou informações sensíveis.
 O escopo é limitado a metadados e informações financeiras.

### 6.5. O acesso é monitorado?
Sim.
 Todas as ações realizadas via cross-account podem ser auditadas pelo cliente através do AWS CloudTrail, garantindo total transparência.

### 6.6. O acesso é permanente?
Não.
 O acesso permanece ativo apenas enquanto a role existir na conta do cliente.
 Caso a role seja removida ou a stack excluída, o acesso é automaticamente revogado.

### 6.7. O que acontece se a conta estiver em AWS Organizations?
A role funciona normalmente tanto em contas standalone quanto em contas membros de uma Organization.
 As permissões concedidas respeitam as políticas da Organization (SCPs) existentes.

### 6.8. É possível limitar ainda mais as permissões?
Sim.
 Caso o cliente tenha alguma restrição específica, as permissões podem ser avaliadas e ajustadas, desde que não comprometam as funcionalidades necessárias para análise de custos e otimização.

### 6.9. A RealCloud acessa a conta o tempo todo?
Apenas sob demanda.
 O acesso é realizado via Cross-Account Role sempre que for necessário coletar dados atualizados (ex: relatórios de billing, métricas de consumo, etc.). 


## 7 - Conclusão:
 A integração Cross Account RealCloud – RealGlass Billing foi projetada para oferecer um modelo seguro, automatizado e transparente de acesso às informações de billing e cost management das contas AWS dos clientes.
 Por meio do uso de serviços nativos da AWS, a solução garante controle total ao cliente, aderência ao princípio do menor privilégio, auditabilidade e facilidade de operação, permitindo que a RealCloud realize análises financeiras, gere insights e proponha otimizações sem impactar a operação ou a segurança do ambiente do cliente.
