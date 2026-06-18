# Cross Account RealCloud - RealGlass Billing

## 1 - Processo de Cross Account: 

 Após a migração do cliente para o serviço de RealGlass Billing, inicia-se a etapa de configuração técnica. Para viabilizar a gestão financeira e aplicar as otimizações contratadas, a RealCloud requer acessos específicos à conta do cliente. Estas permissões são estritamente não invasivas e baseadas no princípio do menor privilégio, garantindo que apenas operações de billing sejam realizadas. A conexão será estabelecida via Cross-Account, conforme detalhado nas seções a seguir.

   1.1. Objetivo Cross Account:
   
   O objetivo desta arquitetura é viabilizar um modelo seguro, automatizado e padronizado de acesso cross-account às informações de billing e cost management das contas AWS dos clientes. A solução permite que a RealCloud acesse os dados de forma segura e controlada, utilizando policies de confiança e Roles IAM, fazendo com que não seja mais necessário o compartilhamento de credenciais e garantindo aderência às melhores práticas de segurança e ao least privilege principle
   
   Adicionalmente, a arquitetura foi concebida para simplificar e escalar o processo de integração de contas, utilizando uma stack do CloudFormation, para assegurar consistência na criação dos recursos necessários, uma função Lambda, para orquestração e uma tabela armazenada no DynamoDB, para registro e rastreabilidade das configurações realizadas. Esse modelo de arquitetura garante governança e auditabilidade, viabilizando análise de custos, geração de insights e otimização financeira de forma eficiente e sustentável.

  1.2. Arquitetura:
  
  <img width="644" height="288" alt="image" src="https://github.com/user-attachments/assets/a76aa10a-8ffc-496f-9faa-5e1b39e5c0ed" />
 
  A arquitetura acima descreve como a RealCloud habilita, de forma segura e automatizada, o acesso cross-account às informações de billing a conta do cliente, utilizando serviços nativos AWS. A solução é baseada em CloudFormation, IAM Roles, Lambda, DynamoDB e S3. 

  1.3. Componentes da Arquitetura:
  - Conta RealCloud:
      Amazon S3: Hospeda o template CloudFormation (YAML/JSON) que o cliente executa para realizar o provisionamento.
    
      AWS Lambda:  Atua como um Custom Resource do CloudFormation. Ela processa as notificações de criação/exclusão de stacks, validando a conectividade cross-account e informações necessárias do cliente.
    
      Amazon DynamoDB: Banco de dados NoSQL que armazena os metadados das contas integradas, incluindo IDs, nomes, roles, external id e link pré-populado do cross account.
    
  - Conta do Cliente:
      AWS CloudFormation: Executa o template fornecido pela RealCloud.
    
      RealCloudCrossAccountRole: Cria o acesso de cross-account seguro que permite à RealCloud 'entrar' na conta do cliente de forma restrita, usando apenas as permissões autorizadas.
    
      Trusted Policy: É a regra de segurança que autoriza exclusivamente a conta da RealCloud a utilizar o acesso criado
    
      Billing-Access-RealCloud: Conjunto de permissões focado em leitura de custos e inventário.
    
      Custom Resource (Gatilho de Onboarding): Um componente dentro do template que, ao ser criado, envia uma notificação automática para a RealCloud. Isso elimina a necessidade de configuração manual após a execução do       template, validando a conexão e sincronizando os dados de acesso instantaneamente.

  1.4. Fluxo de Funcionamento da Arquitetura:
  
 - Execução do Template Cloudformation:
  O cliente acessa o link do template CloudFormation que está armazenado no bucket público da  conta Services da RealCloud. Ao executar o template em sua conta, é criada uma role IAM cross-account.
   
 - Notifcação Cross-Account:
   Ao final da execução do CloudFormation na conta do cliente, um gatilho é disparado para invocar a função Lambda centralizada na conta da RealCloud. Esta função recebe um payload contendo os metadados da nova Stack, incluindo o ID da conta do cliente, o nome da Role criada, o nome da Stack, Link pré-populado de Cross-Account; e External ID (parâmetro opcional de segurança).
  
 - Registro:
    A lambda grava essas informações no DynamoDB, permitindo registro de quais contas estão com role, auditoria e rastreabilidade e automação de acessos futuros


## 2 - Políticas de Acesso Cross-Account

Para habilitar a integração cross-account, é criada na conta do cliente a IAM Role **RealCloudCrossAccount**, que pode ser assumida exclusivamente pela conta de Services da RealCloud por meio do **AWS Security Token Service (STS)** utilizando **AssumeRole**.

A role concede acesso **somente leitura (read-only)** e controlado a serviços relacionados a **custos, billing, consumo, otimização, governança, segurança e inventário da infraestrutura**, sem permissões de alteração, exclusão ou provisionamento de recursos.

As permissões são utilizadas exclusivamente para descoberta do ambiente, análise de consumo, recomendações de otimização e geração de insights operacionais, conforme detalhado abaixo.

### 2.1. Faturamento e Custos (Billing & Cost Management)

**Account/Billing:** Visualiza informações da conta, contratos, dados de faturamento e configurações relacionadas ao billing.

**Free Tier:** Monitora o consumo da camada gratuita da AWS para identificação de limites e potenciais cobranças futuras.

**Budgets:** Permite visualizar orçamentos configurados e acompanhar o status de gastos em relação aos valores planejados.

**Consolidated Billing:** Em ambientes com AWS Organizations, possibilita listar contas vinculadas e compreender a estrutura de faturamento consolidado.

**CUR (Cost and Usage Report):** Acessa definições dos relatórios detalhados de custos e consumo enviados ao Amazon S3.

**Invoicing & Payments:** Permite visualizar faturas emitidas, histórico de pagamentos e ordens de compra (Purchase Orders).

**Cost Explorer (CE):** Permissão central para geração de análises históricas, tendências, previsões de gastos e segmentação de custos.

**Pricing & Cost Modeling:** Consulta a tabela oficial de preços da AWS e recursos de estimativa para projeções financeiras e simulações de cenários.

---

### 2.2. Otimização Financeira e Planejamento (FinOps & Savings)

**Cost Optimization Hub:** Acessa recomendações centralizadas de redução de custos fornecidas pela AWS.

**Savings Plans & Reserved Capacity:** Lista e analisa Savings Plans e reservas ativas para identificar cobertura, utilização e oportunidades de otimização financeira.

**Compute Optimizer:** Avalia recomendações técnicas de right sizing para recursos computacionais com base em métricas reais de utilização.

**MAP Credits (Migration Acceleration Program):** Monitora créditos de migração e aderência aos objetivos financeiros associados.

**Trusted Advisor:** Consolida recomendações relacionadas a performance, resiliência, segurança e oportunidades de economia.

---

### 2.3. Infraestrutura, Performance e Utilização (Compute, Containers & Storage)

**EC2 & Auto Scaling:** Analisa instâncias, famílias computacionais, padrões de utilização e grupos de Auto Scaling para identificação de subutilização, superdimensionamento ou oportunidades de rightsizing.

**Containers (EKS & ECS):** Identifica clusters Kubernetes e workloads em containers para correlação entre custos, consumo computacional e arquitetura de execução.

**Elastic Load Balancing (ALB/NLB):** Analisa balanceadores de carga utilizados e sua relação com arquitetura, disponibilidade e custos operacionais.

**Serverless (Lambda):** Lista funções e configurações para análise de custos de execução, memória provisionada e frequência de uso.

**Storage & Databases (S3, RDS, DynamoDB, ElastiCache, Redshift e EFS):** Identifica buckets, bancos de dados, caches distribuídos, data warehouses e sistemas de arquivos compartilhados para mapear crescimento de armazenamento, utilização e oportunidades de otimização.

**OpenSearch / Elasticsearch:** Permite analisar clusters de busca e observabilidade para avaliação de utilização e custos associados.

**Infraestrutura de Rede (EC2 Networking):** Avalia componentes de rede, conectividade e topologia da infraestrutura para apoiar análises de arquitetura e eficiência operacional.

---

### 2.4. Monitoramento e Utilização Real (CloudWatch & Logs)

**CloudWatch Metrics:** Acessa métricas operacionais (CPU, memória, throughput, I/O, latência e utilização) para validar tecnicamente recomendações de rightsizing e otimização.

**CloudWatch Logs:** Descreve grupos de logs e configurações de retenção para identificar retenções excessivas e potenciais custos ocultos de armazenamento.

---

### 2.5. Governança, Segurança e Conformidade

**Tagging:** Lê etiquetas (tags) aplicadas aos recursos, fundamentais para alocação de custos por centro de custo, projeto, ambiente, squad ou unidade de negócio.

**AWS Config:** Verifica conformidade dos recursos e auxilia na identificação de ativos não padronizados ou fora de políticas corporativas.

**Security Hub:** Consolida achados de segurança e postura de conformidade do ambiente.

**Amazon Inspector:** Permite avaliar vulnerabilidades e exposição de workloads.

**GuardDuty:** Auxilia na identificação de riscos de segurança, anomalias e possíveis comportamentos suspeitos detectados pela AWS.

**Service Quotas:** Monitora limites de serviço para prever necessidades de expansão e evitar impactos operacionais.

**AWS Organizations:** Permite mapear a estrutura hierárquica organizacional (OUs e contas vinculadas), facilitando análises centralizadas e segmentação financeira.

**KMS, Secrets Manager e Systems Manager (SSM):** Utilizados para descoberta e inventário operacional de componentes gerenciados da infraestrutura.

**SNS, SQS e AWS Backup:** Permitem identificar componentes de mensageria, automação e retenção/backup que possam impactar custos e arquitetura operacional.

---

### 2.6. Modelo de Segurança do Acesso

O acesso ocorre exclusivamente via **AWS STS AssumeRole**, restrito à conta de Services da RealCloud previamente autorizada pelo cliente.

A role possui permissões de **somente leitura (read-only)**, não incluindo privilégios de criação, alteração, exclusão ou interrupção de recursos do ambiente AWS do cliente.

Adicionalmente, o acesso pode ser protegido por **External ID**, garantindo maior segurança no modelo cross-account e mitigando riscos de acesso indevido entre contas AWS.



## 3 - Como usar:

Forneça ao cliente a URL do template CloudFormation hospedado no S3:

# Conta AWS Individual (Standalone)

Executa o onboarding apenas na conta em que a stack for criada.

URL:
https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=https%3A%2F%2Frealcloudcrossaccount.s3.us-east-2.amazonaws.com%2FRealCloudCrossAccountClient.yml&stackName=RealCloud-CrossAccount-Client

# Conta Management da AWS Organization

Executa o onboarding na Management Account e propaga automaticamente o acesso para todas as contas-membro da organização por meio de AWS CloudFormation StackSets.

URL:
https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=https%3A%2F%2Frealcloudcrossaccount.s3.us-east-2.amazonaws.com%2FRealCloudCrossAccountClient.yml&stackName=RealCloud-CrossAccount-Client&param_ClientName=NOME_DO_CLIENTE


  3.1. Execução da pilha:
  
   O cliente será direcionado diretamente para a tela do CloudFormation, com todos os parâmetros necessários já preenchidos. Será necessário apenas clicar em “Create Stack”.

  3.2. Após a execução do CloudFormation:
  
   Assim que a pilha for criada com sucesso na conta do cliente, um processo automático (Custom Resource) enviará os dados para a nossa Conta de Services. A tabela DynamoDB será preenchida automaticamente com as seguintes informações:

  - client_name: Nome identificador do cliente (preenchido manualmente pelo usuário durante o setup da Stack).
  
  - account_id: ID numérico da conta AWS do cliente.
  
  - role_name: Nome da Role de acesso criada pela Stack.
  
  - external_id: Identificador externo utilizado para reforçar a segurança no acesso à Role (Opcional).
  
  - stack_name: Nome da pilha do CloudFormation gerada.
  
  - cross_account_link: URL direta para efetuar o Switch Role e acessar a conta do cliente via console.

  3.3. Acesso à Conta do Cliente:
  
   Com o registro criado no DynamoDB, você já poderá realizar o "switch role" (salto) para a conta do cliente utilizando o link gerado na coluna cross_account_link.

  3.4. Offboarding:
  
   Se o cliente desejar revogar o acesso, basta ele deletar a Stack no CloudFormation. Esse processo garante segurança total para ambas as partes:

  1 - Remoção de Acesso: A Role de Cross-Account é excluída da conta do cliente, impedindo qualquer acesso futuro.
  
  2-  Limpeza de Dados: Uma função Lambda de custódia detecta a exclusão e remove automaticamente os dados do cliente da nossa tabela DynamoDB.

  3.5. trecho de Código que Garante a Limpeza
  
   Abaixo está o trecho da função Lambda que assegura a remoção dos dados após a exclusão da stack:

   <img width="635" height="122" alt="image" src="https://github.com/user-attachments/assets/1a8c3291-aa6b-4231-b108-a1bdd3a91988" />


## 4 - Atualizações:

 Caso seja necessário, por parte da RealCloud, realizar alguma atualização no template de Cross Account disponibilizado no bucket S3, o cliente precisará apenas baixar e aplicar novamente o template atualizado. 

 Esse processo é simples e rápido, feito com apenas um clique, conforme descrito na seção 4.

  - Quando uma nova versão do template estiver disponível:
  O cliente fará o download do template atualizado
  
  
  - O CloudFormation aplicará a atualização na stack existente
  
  
  - A IAM Role e as permissões serão ajustadas automaticamente, sem impacto na operação da conta

   
## 5 - Logs:

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

6.1. O cliente pode remover o acesso quando quiser?

Sim.

 O cliente pode remover o acesso a qualquer momento, bastando excluir a stack do   CloudFormation ou a IAM Role RealCloudCrossAccount criada para a integração.
 Após a remoção, a RealCloud perde imediatamente o acesso à conta do cliente.

6.2. Há impacto em custos?

Não.

 A criação da IAM Role e das políticas não gera custo adicional para o cliente.
 O acesso é somente para leitura e análise de dados já existentes na conta.

6.3. A RealCloud consegue criar, alterar ou excluir recursos?

Não.

 A role possui permissões restritas, focadas apenas em billing, custos, consumo e otimização, sem permitir ações de criação, modificação ou exclusão de recursos.

6.4. A RealCloud tem acesso a dados de aplicações?

Não.

 A integração não concede acesso a dados de aplicação, bancos de dados, código ou informações sensíveis.
 O escopo é limitado a metadados e informações financeiras.

6.5. O acesso é monitorado?

Sim.

 Todas as ações realizadas via cross-account podem ser auditadas pelo cliente através do AWS CloudTrail, garantindo total transparência.

6.6. O acesso é permanente?

Não.

 O acesso permanece ativo apenas enquanto a role existir na conta do cliente.
 Caso a role seja removida ou a stack excluída, o acesso é automaticamente revogado.

6.7. O que acontece se a conta estiver em AWS Organizations?

A role funciona normalmente tanto em contas standalone quanto em contas membros de uma Organization.
 As permissões concedidas respeitam as políticas da Organization (SCPs) existentes.

6.8. É possível limitar ainda mais as permissões?

Sim.

 Caso o cliente tenha alguma restrição específica, as permissões podem ser avaliadas e ajustadas, desde que não comprometam as funcionalidades necessárias para análise de custos e otimização.

6.9. A RealCloud acessa a conta o tempo todo?

Apenas sob demanda.

 O acesso é realizado via Cross-Account Role sempre que for necessário coletar dados atualizados (ex: relatórios de billing, métricas de consumo, etc). 


7.1. Existe algum repositório no github mostrando o yaml utilizado ? 

Sim.

https://github.com/anaclaramelo-gif/Cross-Account-Billing


## 7 - Conclusão:

 A integração Cross Account RealCloud – RealGlass Billing foi projetada para oferecer um modelo seguro, automatizado e transparente de acesso às informações de billing e cost management das contas AWS dos clientes.
 Por meio do uso de serviços nativos da AWS, a solução garante controle total ao cliente, aderência ao princípio do menor privilégio, auditabilidade e facilidade de operação, permitindo que a RealCloud realize análises financeiras, gere insights e proponha otimizações sem impactar a operação ou a segurança do ambiente do cliente.

    
  
  



