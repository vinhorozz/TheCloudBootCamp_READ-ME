# MultiCloud, DevOps & IA Challenge - Desafio 2 - Parte 2 - Kubernetes

<aside>
💡

Atenção: o serviço de Kubernetes da AWS não é gratuito, portanto ao executar o hands-on abaixo você terá cobrança de alguns centavos de dólar na sua conta da AWS de acordo com a precificação do EKS na AWS. Lembre-se de deletar o cluster ao final do hands-on para evitar cobranças indesejadas. Use a seção de remoção no final do doc.

</aside>

## Setup do Cluster no AWS Elastic Kubernetes Services (EKS)

1. Crie um usuário chamado `eksuser` com privilegio de Admin e se autentique com ele
    
    ```yaml
    aws configure
    ```
    
2. Instale a ferramenta CLI `eksctl` 
    
    ```bash
    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo cp /tmp/eksctl /usr/bin
    eksctl version
    ```
    
3. Instale a ferramenta CLI `kubectl` 
    
    ```bash
    curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
    chmod +x ./kubectl
    mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
    echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
    kubectl version --short --client
    ```
    
4. Crie um EKS Cluster
    
    ```bash
    eksctl create cluster \
      --name cloudmart \
      --region us-east-1 \
      --nodegroup-name standard-workers \
      --node-type t3.medium \
      --nodes 1 \
      --with-oidc \
      --managed
    ```
    
5. Conecte-se ao cluster EKS usando a configuração do `kubectl` 
    
    ```bash
    aws eks update-kubeconfig --name cloudmart
    
    ```
    
6. Verifique a conectividade do Cluster
    
    ```bash
    kubectl get svc
    kubectl get nodes
    ```
    

6. Crie uma Role & Service Account para fornecer aos pods acesso aos serviços usados pela aplicação (DynamoDB, Bedrock, etc).

```bash
eksctl create iamserviceaccount \
  --cluster=cloudmart \
  --name=cloudmart-pod-execution-role \
  --role-name CloudMartPodExecutionRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/AdministratorAccess\
  --region us-east-1 \
  --approve
```

<aside>
💡

OBS: No exemplo acima, foram usados privilégios de Admin para facilitar fins educacionais. Lembre-se sempre de seguir o princípio do mínimo privilégio em ambientes de produção

</aside>

## Deployment do Backend no Kubernetes

### **Crie um Repositório ECR para o Backend e suba a imagem Docker para o mesmo**

```bash
<Siga os passos do ECR>
```

### **Crie um arquivo de deployment do Kubernetes (YAML) para o Backend**

```yaml
cd challenge-day2/backend
nano cloudmart-backend.yaml
```

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
        **image: public.ecr.aws/l4c0j8h9/cloudmart-backend:latest**
        env:
        - name: PORT
          value: "5000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BEDROCK_AGENT_ID
          value: "xxxxxx"
        - name: BEDROCK_AGENT_ALIAS_ID
          value: "xxxx"
        - name: OPENAI_API_KEY
          value: "xxxxxx"
        - name: OPENAI_ASSISTANT_ID
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

### **Realize o deployment do Backend no Kubernetes**

```yaml
kubectl apply -f cloudmart-backend.yaml
```

Acompanhe o status dos objetos sendo criados e obtenha o IP público gerado para a API

```yaml
kubectl get pods
kubectl get deployment
kubectl get service
```

## Deployment do Frontend no Kubernetes

### Preparação

Altere o arquivo .env do Frontend para apontar para a URL da API criada dentro do Kubernetes obtida pela comando `kubectl get service` 

```yaml
cd ../challenge-day2/frontend
nano .env
```

Conteúdo do `.env`:

```
VITE_API_BASE_URL=http://<sua_url_kubernetes_api>:5000/api

```

### **Crie um Repositório ECR para o Frontend e suba a imagem Docker para o mesmo**

```bash
<Siga os passos do ECR>
```

### **Crie um arquivo de deployment do Kubernetes (YAML) para o Frontend**

```yaml
cd challenge-day2/frontend
nano cloudmart-frontend.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudmart-frontend-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudmart-frontend-app
  template:
    metadata:
      labels:
        app: cloudmart-frontend-app
    spec:
      serviceAccountName: cloudmart-pod-execution-role
      containers:
      - name: cloudmart-frontend-app
        **image: public.ecr.aws/l4c0j8h9/cloudmart-frontend:latest**
---

apiVersion: v1
kind: Service
metadata:
  name: cloudmart-frontend-app-service
spec:
  type: LoadBalancer
  selector:
    app: cloudmart-frontend-app
  ports:
    - protocol: TCP
      port: 5001
      targetPort: 5001
```

### **Realize o deployment do Frontend no Kubernetes**

```yaml
kubectl apply -f cloudmart-frontend.yaml
```

Acompanhe o status dos objetos sendo criados e obtenha o IP público gerado para a API

```yaml
kubectl get pods
kubectl get deployment
kubectl get service
```

# Remoção

Ao final do hands-on, delete todos os recursos:

```yaml
kubectl delete service cloudmart-frontend-app-service
kubectl delete deployment cloudmart-frontend-app
kubectl delete service cloudmart-backend-app-service
kubectl delete deployment cloudmart-backend-app

eksctl delete cluster --name cloudmart --region us-east-1

```