# MultiCloud, DevOps & IA Challenge - Desafio 1

# Passo 1: Use o Claude para Gerar o Código Terraform

1. Inicie uma conversa com o Claude.
2. Peça ao Claude para criar um código Terraform para um bucket S3. Use um prompt como:
"Por favor, forneça um código Terraform para criar um bucket S3 na AWS com um nome único na região us-east-1."
3. O Claude deve gerar um código semelhante a este:
    
    ```
    provider "aws" {
      region = "us-east-1"  # Substitua pela região desejada
    }
    
    resource "random_id" "bucket_suffix" {
      byte_length = 8
    }
    
    resource "aws_s3_bucket" "my_bucket" {
      bucket = "my-unique-bucket-name-${random_id.bucket_suffix.hex}"
    
      tags = {
        Name        = "My bucket"
        Environment = "Dev"
      }
    }
    ```
    
4. Salve este código para uso no Passo 5.

## Passo 2: Criar uma IAM Role para o EC2

1. Faça login no AWS Management Console.
2. Navegue até o painel do IAM.
3. Clique em "Roles" na barra lateral esquerda e depois em "Create role".
4. Escolha "AWS service" como tipo de entidade confiável e "EC2" como caso de uso.
5. Procure e anexe a política "AdministratorAccess".
Nota: Em um ambiente de produção, use uma política mais restrita.
6. Nomeie a role como "EC2Admin" e forneça uma descrição.
7. Revise e crie a role.

## Passo 3: Lançar uma Instância EC2

1. Vá para o painel EC2 no AWS Management Console.
2. Clique em "Launch Instance".
3. Escolha uma AMI Amazon Linux 2.
4. Selecione um tipo de instância t2.micro.
5. Configure os detalhes da instância:
    - Network: VPC padrão
    - Subnet: Qualquer disponível
    - Auto-assign Public IP: Enable
    - IAM role: Selecione "EC2Admin"
6. Mantenha as configurações de armazenamento padrão.
7. Adicione uma tag: Key="Name", Value="workstation".
8. Crie um security group permitindo acesso SSH do seu IP.
9. Revise e lance, selecionando ou criando um key pair.

## Passo 4: Conectar à Instância EC2 e Instalar o Terraform

1. No painel EC2, selecione sua instância "workstation".
2. Clique em "Connect" e use o método "EC2 Instance Connect".
3. Na sessão SSH baseada no navegador, atualize os pacotes do sistema:
    
    ```
    sudo yum update -y
    
    ```
    
4. Instale o yum-utils:
    
    ```
    sudo yum install -y yum-utils
    
    ```
    
5. Adicione o repositório HashiCorp:
    
    ```
    sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
    
    ```
    
6. Instale o Terraform:
    
    ```
    sudo yum -y install terraform
    
    ```
    
7. Verifique a instalação:
    
    ```
    terraform version
    
    ```
    

## Passo 5: Aplicar a Configuração Terraform

1. Crie um novo diretório e navegue até ele:
    
    ```
    mkdir terraform-project && cd terraform-project
    
    ```
    
2. Crie e abra o arquivo [main.tf](http://main.tf/):
    
    ```
    nano main.tf
    
    ```
    
3. Cole o código Terraform gerado pelo Claude no Passo 1.
4. Salve e saia do editor (no nano, pressione Ctrl+X, depois Y, depois Enter).
5. Inicialize o Terraform:
    
    ```
    terraform init
    
    ```
    
6. Revise o plano:
    
    ```
    terraform plan
    
    ```
    
7. Aplique a configuração:
    
    ```
    terraform apply
    
    ```
    
8. Digite "yes" quando solicitado para criar os recursos.

## Passo 6: Verificar a Criação do Bucket S3

1. Use o AWS CLI para listar os buckets:
    
    ```
    aws s3 ls
    
    ```
    
2. Verifique se o seu novo bucket está na lista.

## Passo 7: Limpeza

1. Para excluir os recursos criados:
    
    ```
    terraform destroy
    
    ```
    
2. Digite "yes" quando solicitado para destruir os recursos.

Parabéns! Você usou com sucesso o Claude para gerar código Terraform, configurou uma workstation EC2, instalou o Terraform e criou um bucket S3. Isso completa o Dia 1 do Desafio MultiCloud DevOps & AI.