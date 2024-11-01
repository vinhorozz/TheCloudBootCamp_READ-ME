# MultiCloud, DevOps & IA Challenge - Desafio 2 - Parte 1 - Docker

# Guia de Configuração do CloudMart

## Passo 1: Criar as tabelas do CloudMart usando Terraform

Logue na instancia EC2

Remova o arquivo [`main.tf`](http://main.tf) usado no Desafio 1

```bash
cd terraform-project
rm main.tf
```

Crie um novo arquivo terraform com o conteúdo abaixo

```bash
nano main.tf
```

Copy e cole o conteúdo do codigo terraform abaixo dentro do arquivo e salve-o.

```
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

```

Inicialize o Terraform:

```
terraform init

```

Revise o plano:

```
terraform plan

```

Aplique a configuração:

```
terraform apply

```

1. Digite "yes" quando solicitado para criar os recursos.

## Passo 2: Instalar o Docker na EC2

Execute os seguintes comandos:

```bash
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo docker run hello-world
sudo systemctl enable docker
docker --version
sudo usermod -a -G docker $(whoami)
newgrp docker

```

## Passo 3: Criar a imagem Docker do CloudMart

### Backend

Criar pasta e baixar o código-fonte:

```bash
mkdir -p challenge-day2/backend && cd challenge-day2/backend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-backend.zip
unzip cloudmart-backend.zip

```

Criar arquivo `.env`:

```bash
nano .env

```

Conteúdo do `.env`:

```
PORT=5000
AWS_REGION=us-east-1
BEDROCK_AGENT_ID=<seu-bedrock-agent-id>
BEDROCK_AGENT_ALIAS_ID=<seu-bedrock-agent-alias-id>
OPENAI_API_KEY=<sua-chave-api-openai>
OPENAI_ASSISTANT_ID=<seu-id-assistente-openai>

```

Criar Dockerfile:

```bash
nano Dockerfile

```

Conteúdo do Dockerfile:

```
FROM node:18
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]

```

Construir e executar a imagem Docker:

```bash
docker build -t cloudmart-backend .
docker run -d -p 5000:5000 --env-file .env cloudmart-backend

```

### Frontend

Criar pasta e baixar o código-fonte:

```bash
cd ..
mkdir frontend && cd frontend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/day2/cloudmart-frontend.zip
unzip cloudmart-frontend.zip

```

Criar arquivo `.env`:

```bash
nano .env

```

Conteúdo do `.env`:

```
VITE_API_BASE_URL=http://<seu-ip-ec2>:5000/api

```

Criar Dockerfile:

```bash
nano Dockerfile

```

Conteúdo do Dockerfile:

```
FROM node:16-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:16-alpine
WORKDIR /app
RUN npm install -g serve
COPY --from=build /app/dist /app
ENV PORT=5001
ENV NODE_ENV=production
EXPOSE 5001
CMD ["serve", "-s", ".", "-l", "5001"]

```

Construir e executar a imagem Docker:

```bash
docker build -t cloudmart-frontend .
docker run -d -p 5001:5001 cloudmart-frontend

```

## Passo 4: Configurar regras de firewall

Abra as seguintes portas no grupo de segurança da EC2:

1. Porta 5000 (para o backend)
2. Porta 5001 (para o frontend)

Agora você pode acessar o CloudMart remotamente usando o IP público da sua instância EC2.