### Download Updated Frontend and Backend Code

Faça backup das pastas existentes

cp -R challenge-day2/ challenge-day2_bkp
cp -R terraform-project/ terraform-project_bkp

Limpe os arquivos de aplicacao existentes, exceto o Docker e o arquivo YAML (Backend)

cd challenge-day2/backend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name " .* " -o -name Dockerfile -o -name=3.yaml" \))

Limpe os arquivos de aplicacao existentes, exceto o Docker e o arquivo YAML (Frontend)

cd challenge-day2/frontend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name " .* " -o -name Dockerfile -o -name"%.yaml" \))

Baixe o codigo-fonte atualizado e descompacte-o (Backend)

cd challenge-day2/backend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/final/cloudmart-backend-
final.zip
unzip cloudmart-backend-final.zip

Baixe o codigo-fonte atualizado e descompacte-o (Frontend)

cd challenge-day2/frontend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/final/cloudmart-fronten
d-final.zip
unzip cloudmart-frontend-final.zip
git add -A
git commit -m "final code"
git push


###Configuração do Google Cloud BigQuery###

Siga estes passos para configurar o Google Cloud BigQuery para o CloudMart:
1. Criar um Projeto Google Cloud:
. Acesse o Google Cloud Console (https://console.cloud.google.com/).
. Clique no menu suspenso de projetos e selecione "Novo Projeto".
. Nomeie o projeto como "CloudMart" e crie-o.
2. Ativar a API do BigQuery:
. No Google Cloud Console, va para "APIs & Serviços" > "Painel".
· Clique em "+ ATIVAR APIS E SERVIÇOS".
. Pesquise por "BigQuery API" e ative-a.
3. Criar um Dataset do BigQuery:
. No Google Cloud Console, va para "BigQuery".
. No painel Explorer, clique no nome do seu projeto.
. Clique em "CRIAR DATASET".

. Defina o ID do Dataset como "cloudmart".

. Escolha a localizacao dos dados e clique em "CRIAR DATASET".
4. Criar uma Tabela do BigQuery:
. No dataset que voce acabou de criar, clique em "CRIAR TABELA".
. Defina o nome da Tabela como "cloudmart-orders".

. Defina o schema de acordo com a estrutura do seu pedido. Por exemplo:
o id: STRING

o items: JSON

o userEmail: STRING

o total: FLOAT

o status: STRING

o createdAt: TIMESTAMP

. Clique em "CRIAR TABELA".
5. Criar Conta de Serviço e Chave:
. No Google Cloud Console, va para "IAM & Admin" > "Contas de Serviço".
· Clique em "CRIAR CONTA DE SERVIÇO".
. Nomeie-a como "cloudmart-bigquery-sa" e conceda a ela a funcao "Editor de Dados do
BigQuery".
. Apos criar, clique na conta de serviço, vá para a aba "Chaves" e clique em "ADICIONAR CHAVE" > "Criar nova chave".

. Escolha JSON como o tipo de chave e crie.

. Salve o arquivo JSON baixado como google_credentials. json ,

6. Configurar Função Lambda:
· Navegue até o diretório raiz do seu projeto.
· Entre no diretório da funçao Lambda:

cd src/lambda/addToBigQuery

. Instale as dependências necessárias:

sudo yum install npm
npm install

. Edite o arquivo google_credentials. json neste diretorio e coloque o conteudo da sua
chave.

· Crie um arquivo zip de todo o diretório:

zip -r dynamodb_to_bigquery.zip .

· Este arquivo zip será usado ao criar ou atualizar a função Lambda.

· Retorne ao diretório raiz do seu projeto:

cd .. / .. / ..

7. Atualizar Variaveis de Ambiente da Funçao Lambda:

Lembre-se de nunca fazer commit do arquivo
google_credentials. json para o controle de
versao. Ele deve ser adicionado ao seu arquivo

gitignore .


#### Passos no Terraform
Remova o arquivo main.tf e crie um novo vazio
'''
rm main.tf
nano main.tf
'''

Adicione estas linhas de terraform ao final do arquivo main.tf para criar o Lambda para a inserção no BigQuery. Atualize os valores em vermelho primeiro.


'''
provider "aws" {
  region = "us-east-1"  # Altere para sua região preferida
}

# Tabelas DynamoDB
resource "aws_dynamodb_table" "cloudmart_products" {
  name           = "cloudmart-products"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
    name = "id"
    type = "S"
  }
}

resource "aws_dynamodb_table" "cloudmart_orders" {
  name           = "cloudmart-orders"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
    name = "id"
    type = "S"
  }
  
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
}

resource "aws_dynamodb_table" "cloudmart_tickets" {
  name           = "cloudmart-tickets"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
    name = "id"
    type = "S"
  }
}

# IAM Role for Lambda function
resource "aws_iam_role" "lambda_role" {
  name = "cloudmart_lambda_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

# IAM Policy for Lambda function
resource "aws_iam_role_policy" "lambda_policy" {
  name = "cloudmart_lambda_policy"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:Scan",
          "dynamodb:GetRecords",
          "dynamodb:GetShardIterator",
          "dynamodb:DescribeStream",
          "dynamodb:ListStreams",
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = [
          aws_dynamodb_table.cloudmart_products.arn,
          aws_dynamodb_table.cloudmart_orders.arn,
          "${aws_dynamodb_table.cloudmart_orders.arn}/stream/*",
          aws_dynamodb_table.cloudmart_tickets.arn,
          "arn:aws:logs:*:*:*"
        ]
      }
    ]
  })
}

# Lambda function for listing products
resource "aws_lambda_function" "list_products" {
  filename         = "list_products.zip"
  function_name    = "cloudmart-list-products"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("list_products.zip")

  environment {
    variables = {
      PRODUCTS_TABLE = aws_dynamodb_table.cloudmart_products.name
    }
  }
}

# Lambda permission for Bedrock
resource "aws_lambda_permission" "allow_bedrock" {
  statement_id  = "AllowBedrockInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.list_products.function_name
  principal     = "bedrock.amazonaws.com"
}

# Output the ARN of the Lambda function
output "list_products_function_arn" {
  value = aws_lambda_function.list_products.arn
}

# Lambda function for DynamoDB to BigQuery
resource "aws_lambda_function" "dynamodb_to_bigquery" {
  filename         = "../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip"
  function_name    = "cloudmart-dynamodb-to-bigquery"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip")

  environment {
    variables = {
      GOOGLE_CLOUD_PROJECT_ID        = "lustrous-bounty-436219-f1"
      BIGQUERY_DATASET_ID            = "cloudmart"
      BIGQUERY_TABLE_ID              = "cloudmart-orders"
      GOOGLE_APPLICATION_CREDENTIALS = "/var/task/google_credentials.json"
    }
  }
}

# Lambda event source mapping for DynamoDB stream
resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  event_source_arn  = aws_dynamodb_table.cloudmart_orders.stream_arn
  function_name     = aws_lambda_function.dynamodb_to_bigquery.arn
  starting_position = "LATEST"
}

'''
Execute terraform apply now

#### **Configuração do Azure Text Analytics**

Siga estes passos para configurar o Azure Text Analytics para análise de sentimento:
1. Criar uma Conta Azure:
. Acesse o portal Azure (https://portal.azure.com/).

· Faça login ou crie uma nova conta se nao tiver uma.
2. Criar um Recurso:

. No portal Azure, clique em "Create a resource".
. Pesquise por "Text Analytics" e selecione.
. Clique em "Create".
3. Configurar o Recurso:
. Escolha sua assinatura e grupo de recursos (crie um novo se necessário).

. Nomeie o recurso (ex: "cloudmart-text-analytics").

· Escolha sua regiao e nível de preço.
. Clique em "Review + create", depois "Create".
4. Obter o Endpoint e a Chave:
· Quando o recurso for criado, vá para a página de visao geral.
. No menu à esquerda, em "Resource Management", clique em "Keys and Endpoint".
. Copie a URL do endpoint e uma das chaves.

Implante as alterações no Backend
Abra o arquivo cloudmart-backend.yaml :

nano cloudmart-backend.yaml

Conteudo do

cloudmart-backend.yaml :

'''
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-backend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-backend-app
  template:
    metadata:
      labels:
        app: cloudmart-backend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-backend-app
        image: public.ecr.aws/l4c0j8h9/cloudmaster-backend:latest
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_AGENT_ID
          value: "xxxx"
        - name: BEDROCK_AGENT_ALIAS_ID
          value: "xxxx"
        - name: OPENAI_API_KEY
          value: "xxxx"
        - name: OPENAI_ASSISTANT_ID
          value: "xxxx"
        - name: AZURE_ENDPOINT
          value: "xxxx"
        - name: AZURE_API_KEY
          value: "xxxx"
        
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-backend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-backend-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
'''

Construa uma nova imagem

'''
<Siga passos do ECR>
'''

# MultiCloud, DevOps & IA Challenge - Desafio 5 - MultiCloud (GCP & Azure)

# Download Updated Frontend and Backend Code

Faça backup das pastas existentes

```yaml
cp -R challenge-day2/ challenge-day2_bkp
cp -R terraform-project/ terraform-project_bkp
```

Limpe os arquivos de aplicação existentes, exceto o Docker e o arquivo YAML (Backend)

```bash
cd challenge-day2/backend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name ".*" -o -name Dockerfile -o -name "*.yaml" \))
```

Limpe os arquivos de aplicação existentes, exceto o Docker e o arquivo YAML (Frontend)

```bash
cd challenge-day2/frontend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name ".*" -o -name Dockerfile -o -name "*.yaml" \))
```

Baixe o código-fonte atualizado e descompacte-o (Backend)

```yaml
cd challenge-day2/backend
wget  https://tcb-public-events.s3.amazonaws.com/mdac/resources/final/cloudmart-backend-final.zip
unzip cloudmart-backend-final.zip
```

Baixe o código-fonte atualizado e descompacte-o (Frontend)

```yaml
cd challenge-day2/frontend
wget  https://tcb-public-events.s3.amazonaws.com/mdac/resources/final/cloudmart-frontend-final.zip
unzip cloudmart-frontend-final.zip
git add -A
git commit -m "final code"
git push 

```

# **Configuração do Google Cloud BigQuery**

Siga estes passos para configurar o Google Cloud BigQuery para o CloudMart:

1. Criar um Projeto Google Cloud:
    - Acesse o Google Cloud Console (https://console.cloud.google.com/).
    - Clique no menu suspenso de projetos e selecione "Novo Projeto".
    - Nomeie o projeto como "CloudMart" e crie-o.
2. Ativar a API do BigQuery:
    - No Google Cloud Console, vá para "APIs & Serviços" > "Painel".
    - Clique em "+ ATIVAR APIS E SERVIÇOS".
    - Pesquise por "BigQuery API" e ative-a.
3. Criar um Dataset do BigQuery:
    - No Google Cloud Console, vá para "BigQuery".
    - No painel Explorer, clique no nome do seu projeto.
    - Clique em "CRIAR DATASET".
    - Defina o ID do Dataset como "cloudmart".
    - Escolha a localização dos dados e clique em "CRIAR DATASET".
4. Criar uma Tabela do BigQuery:
    - No dataset que você acabou de criar, clique em "CRIAR TABELA".
    - Defina o nome da Tabela como "cloudmart-orders".
    - Defina o schema de acordo com a estrutura do seu pedido. Por exemplo:
        - id: STRING
        - items: JSON
        - userEmail: STRING
        - total: FLOAT
        - status: STRING
        - createdAt: TIMESTAMP
    - Clique em "CRIAR TABELA".
5. Criar Conta de Serviço e Chave:
    - No Google Cloud Console, vá para "IAM & Admin" > "Contas de Serviço".
    - Clique em "CRIAR CONTA DE SERVIÇO".
    - Nomeie-a como "cloudmart-bigquery-sa" e conceda a ela a função "Editor de Dados do BigQuery".
    - Após criar, clique na conta de serviço, vá para a aba "Chaves" e clique em "ADICIONAR CHAVE" > "Criar nova chave".
    - Escolha JSON como o tipo de chave e crie.
    - Salve o arquivo JSON baixado como `google_credentials.json`.
6. Configurar Função Lambda:
    - Navegue até o diretório raiz do seu projeto.
    - Entre no diretório da função Lambda:
        
        ```
        cd src/lambda/addToBigQuery
        ```
        
    - Instale as dependências necessárias:
        
        ```
        sudo yum install npm
        npm install
        ```
        
    - Edite o arquivo `google_credentials.json` neste diretório e coloque o conteúdo da sua chave.
    - Crie um arquivo zip de todo o diretório:
        
        ```
        zip -r dynamodb_to_bigquery.zip .
        ```
        
    - Este arquivo zip será usado ao criar ou atualizar a função Lambda.
    - Retorne ao diretório raiz do seu projeto:
        
        ```
        cd ../../..
        ```
        
7. Atualizar Variáveis de Ambiente da Função Lambda:

Lembre-se de nunca fazer commit do arquivo `google_credentials.json` para o controle de versão. Ele deve ser adicionado ao seu arquivo `.gitignore`.

# Passos no Terraform

Remova o arquivo [`main.tf`](http://main.tf) e crie um novo vazio

```bash
rm main.tf
nano main.tf
```

Adicione estas linhas de terraform ao final do arquivo [`main.tf`](http://main.tf) para criar o Lambda para a inserção no BigQuery. Atualize os valores em vermelho primeiro.

```bash
provider "aws" {
  region = "us-east-1"  # Altere para sua região preferida
}

# Tabelas DynamoDB
resource "aws_dynamodb_table" "cloudmart_products" {
  name           = "cloudmart-products"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
    name = "id"
    type = "S"
  }
}

resource "aws_dynamodb_table" "cloudmart_orders" {
  name           = "cloudmart-orders"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
    name = "id"
    type = "S"
  }
  
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
}

resource "aws_dynamodb_table" "cloudmart_tickets" {
  name           = "cloudmart-tickets"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "id"

  attribute {
    name = "id"
    type = "S"
  }
}

# IAM Role for Lambda function
resource "aws_iam_role" "lambda_role" {
  name = "cloudmart_lambda_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

# IAM Policy for Lambda function
resource "aws_iam_role_policy" "lambda_policy" {
  name = "cloudmart_lambda_policy"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:Scan",
          "dynamodb:GetRecords",
          "dynamodb:GetShardIterator",
          "dynamodb:DescribeStream",
          "dynamodb:ListStreams",
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = [
          aws_dynamodb_table.cloudmart_products.arn,
          aws_dynamodb_table.cloudmart_orders.arn,
          "${aws_dynamodb_table.cloudmart_orders.arn}/stream/*",
          aws_dynamodb_table.cloudmart_tickets.arn,
          "arn:aws:logs:*:*:*"
        ]
      }
    ]
  })
}

# Lambda function for listing products
resource "aws_lambda_function" "list_products" {
  filename         = "list_products.zip"
  function_name    = "cloudmart-list-products"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("list_products.zip")

  environment {
    variables = {
      PRODUCTS_TABLE = aws_dynamodb_table.cloudmart_products.name
    }
  }
}

# Lambda permission for Bedrock
resource "aws_lambda_permission" "allow_bedrock" {
  statement_id  = "AllowBedrockInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.list_products.function_name
  principal     = "bedrock.amazonaws.com"
}

# Output the ARN of the Lambda function
output "list_products_function_arn" {
  value = aws_lambda_function.list_products.arn
}

# Lambda function for DynamoDB to BigQuery
resource "aws_lambda_function" "dynamodb_to_bigquery" {
  filename         = "../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip"
  function_name    = "cloudmart-dynamodb-to-bigquery"
  role             = aws_iam_role.lambda_role.arn
  handler          = "index.handler"
  runtime          = "nodejs20.x"
  source_code_hash = filebase64sha256("../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip")

  environment {
    variables = {
      GOOGLE_CLOUD_PROJECT_ID        = "lustrous-bounty-436219-f1"
      BIGQUERY_DATASET_ID            = "cloudmart"
      BIGQUERY_TABLE_ID              = "cloudmart-orders"
      GOOGLE_APPLICATION_CREDENTIALS = "/var/task/google_credentials.json"
    }
  }
}

# Lambda event source mapping for DynamoDB stream
resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  event_source_arn  = aws_dynamodb_table.cloudmart_orders.stream_arn
  function_name     = aws_lambda_function.dynamodb_to_bigquery.arn
  starting_position = "LATEST"
}

```

Execute `terraform apply` now

# **Configuração do Azure Text Analytics**

Siga estes passos para configurar o Azure Text Analytics para análise de sentimento:

1. Criar uma Conta Azure:
    - Acesse o portal Azure (https://portal.azure.com/).
    - Faça login ou crie uma nova conta se não tiver uma.
2. Criar um Recurso:
    - No portal Azure, clique em "Create a resource".
    - Pesquise por "Text Analytics" e selecione.
    - Clique em "Create".
3. Configurar o Recurso:
    - Escolha sua assinatura e grupo de recursos (crie um novo se necessário).
    - Nomeie o recurso (ex: "cloudmart-text-analytics").
    - Escolha sua região e nível de preço.
    - Clique em "Review + create", depois "Create".
4. Obter o Endpoint e a Chave:
    - Quando o recurso for criado, vá para a página de visão geral.
    - No menu à esquerda, em "Resource Management", clique em "Keys and Endpoint".
    - Copie a URL do endpoint e uma das chaves.

# Implante as alterações no Backend

Abra o arquivo `cloudmart-backend.yaml`:

```bash
nano cloudmart-backend.yaml

```

Conteúdo do `cloudmart-backend.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-backend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-backend-app
  template:
    metadata:
      labels:
        app: cloudmart-backend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-backend-app
        image: public.ecr.aws/l4c0j8h9/cloudmaster-backend:latest
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_AGENT_ID
          value: "xxxx"
        - name: BEDROCK_AGENT_ALIAS_ID
          value: "xxxx"
        - name: OPENAI_API_KEY
          value: "xxxx"
        - name: OPENAI_ASSISTANT_ID
          value: "xxxx"
        - name: AZURE_ENDPOINT
          value: "xxxx"
        - name: AZURE_API_KEY
          value: "xxxx"
        
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-backend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-backend-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```

Construa uma nova imagem

```yaml
<Siga passos do ECR>
```

## Atualize o deployment no Kubernetes

```json
kubectl apply -f cloudmart-backend.yaml
```