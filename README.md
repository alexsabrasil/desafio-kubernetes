# Desafio Kubernetes | Curso DevOps FAP - Aponti Academy

Projeto prático para aplicação de conceitos de conteinerização e orquestração com Kubernetes.

A solução contém três APIs Node.js, cada uma integrada ao seu próprio PostgreSQL, executadas em cluster Kubernetes local com Minikube e Docker como driver.

## Arquitetura

```text
API 1 -> Service api1 -> Pod API 1 -> Service postgres-api1 -> PostgreSQL 1
API 2 -> Service api2 -> Pod API 2 -> Service postgres-api2 -> PostgreSQL 2
API 3 -> Service api3 -> Pod API 3 -> Service postgres-api3 -> PostgreSQL 3
```

Cada aplicação e banco é gerenciado por um Deployment. Os Services fornecem comunicação estável e os ReplicaSets são gerenciados pelos Deployments.

## Tecnologias

- Node.js e Express
- PostgreSQL
- Docker
- Kubernetes
- Minikube
- kubectl
- Git e GitHub

## Estrutura

```text
desafio-kubernetes/
├── desafio-1/
├── desafio-2/
├── desafio-3/
├── kubernetes/
│   ├── api1/
│   │   ├── api.yaml
│   │   └── postgres.yaml
│   ├── api2/
│   │   ├── api.yaml
│   │   ├── postgres.yaml
│   │   └── initdb-configmap.yaml
│   └── api3/
│       ├── api.yaml
│       ├── postgres.yaml
│       └── initdb-configmap.yaml
├── .gitignore
└── README.md
```

Os arquivos `.env` são locais e não devem ser versionados.

## Ambiente Kubernetes

Após dificuldades de inicialização do Kubernetes integrado ao Docker Desktop, o laboratório foi concluído com Minikube utilizando Docker como driver.

```powershell
minikube start --driver=docker
kubectl get nodes
kubectl get pods -A
```

O nó `minikube` foi validado no estado `Ready`.

## Construção das imagens

```powershell
minikube image build -t desafio-api1:1.0 ./desafio-1
minikube image build -t desafio-api2:1.0 ./desafio-2
minikube image build -t desafio-api3:1.0 ./desafio-3
minikube image ls
```

Imagens utilizadas:

```text
desafio-api1:1.0
desafio-api2:1.0
desafio-api3:1.0
```

Os Deployments utilizam `imagePullPolicy: Never` para consumir as imagens locais do Minikube.

## PostgreSQL

Cada API possui um banco independente:

| API | Service PostgreSQL | Banco |
|---|---|---|
| API 1 | `postgres-api1` | `api_database` |
| API 2 | `postgres-api2` | `desafio2_db` |
| API 3 | `postgres-api3` | `mydatabase` |

Todos utilizam a porta interna `5432`.

A API 1 inicializa a tabela pela aplicação. As APIs 2 e 3 utilizam ConfigMaps com `init.sql`, montados em `/docker-entrypoint-initdb.d/init.sql`, para criar automaticamente a tabela `users`.

## ConfigMaps e Secrets

ConfigMaps:

```text
api2-initdb
api3-initdb
```

Secrets utilizados:

```text
api1-db-secret
api2-db-secret
api3-db-secret
```

As senhas não estão nos manifests versionados. Exemplo de criação segura a partir do `.env` local:

```powershell
$dbPassword1 = (Get-Content .\desafio-1\.env | Where-Object { $_ -match '^DB_PASSWORD=' }) -replace '^DB_PASSWORD=', ''

kubectl create secret generic api1-db-secret `
  --from-literal=POSTGRES_PASSWORD="$dbPassword1"
```

O mesmo procedimento pode ser repetido para API 2 e API 3.

## Deploy

Depois de criar os Secrets e construir as imagens:

```powershell
kubectl apply -f .\kubernetes\api1\postgres.yaml
kubectl apply -f .\kubernetes\api1\api.yaml

kubectl apply -f .\kubernetes\api2\initdb-configmap.yaml
kubectl apply -f .\kubernetes\api2\postgres.yaml
kubectl apply -f .\kubernetes\api2\api.yaml

kubectl apply -f .\kubernetes\api3\initdb-configmap.yaml
kubectl apply -f .\kubernetes\api3\postgres.yaml
kubectl apply -f .\kubernetes\api3\api.yaml
```

## Recursos validados

Ao final do laboratório estavam em execução:

```text
api1
api2
api3
postgres-api1
postgres-api2
postgres-api3
```

O comando `kubectl get all` confirmou 6 Pods `Running`, 6 Deployments disponíveis, 6 ReplicaSets prontos e Services para as APIs e bancos.

Também foram validados:

```powershell
kubectl get configmaps
kubectl get secrets
```

## Comunicação interna

As aplicações utilizam os nomes DNS dos Services, sem depender dos IPs dos Pods:

```text
API 1 -> postgres-api1:5432
API 2 -> postgres-api2:5432
API 3 -> postgres-api3:5432
```

## Testes

O acesso local foi realizado com `port-forward`.

Exemplo:

```powershell
kubectl port-forward service/api1 3000:3000
```

### API 1

```powershell
curl.exe http://localhost:3000/api/users

curl.exe -X POST http://localhost:3000/api/users `
  -H "Content-Type: application/json" `
  -d '{\"name\":\"Ale\",\"email\":\"teste-api1@example.com\"}'
```

O cadastro e a consulta foram executados com sucesso.

### API 2

```powershell
kubectl port-forward service/api2 3000:3000
curl.exe http://localhost:3000/db-test
curl.exe http://localhost:3000/users
```

O `/db-test` retornou `success: true`.

```powershell
curl.exe -X POST http://localhost:3000/users `
  -H "Content-Type: application/json" `
  -d '{\"name\":\"Ale\",\"email\":\"teste-api2@example.com\"}'
```

O registro foi persistido e recuperado corretamente.

### API 3

```powershell
kubectl port-forward service/api3 3000:3000
curl.exe http://localhost:3000/users

curl.exe -X POST http://localhost:3000/users `
  -H "Content-Type: application/json" `
  -d '{\"name\":\"Ale\",\"email\":\"teste-api3@example.com\"}'
```

Os logs confirmaram:

```text
Servidor rodando na porta 3000
Conectado ao Banco de Dados Postgres
```

O registro também foi persistido e recuperado corretamente.

## Inicialização automática do banco

Nos PostgreSQL das APIs 2 e 3, os logs confirmaram:

```text
running /docker-entrypoint-initdb.d/init.sql
CREATE TABLE
database system is ready to accept connections
```

Assim, a tabela `users` é criada automaticamente na inicialização.

## Segurança

- `.env` não versionado;
- `node_modules` não versionado;
- `.dockerignore` nas aplicações;
- credenciais não incluídas nas imagens;
- credenciais não armazenadas nos manifests versionados;
- Kubernetes Secrets para as senhas;
- `.gitignore` na raiz do projeto.

## Persistência

Para este laboratório foi utilizado `emptyDir` nos bancos PostgreSQL.

Essa configuração atende ao cenário didático, mas não mantém os dados caso o Pod seja removido. Uma evolução seria utilizar PersistentVolume e PersistentVolumeClaim.

## Fluxo praticado

```text
Código Node.js
      |
      v
Dockerfile
      |
      v
Imagem
      |
      v
Deployment
      |
      v
ReplicaSet
      |
      v
Pod
      |
      v
Service
      |
      v
Aplicação <-> PostgreSQL
```

## Comandos úteis

```powershell
kubectl get nodes
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get replicasets
kubectl get configmaps
kubectl get secrets
kubectl get all
```

## Resultado

As três APIs Node.js e seus respectivos bancos PostgreSQL foram implantados e testados com sucesso no Kubernetes.

O desafio permitiu praticar Pods, Deployments, ReplicaSets, Services, ConfigMaps, Secrets, comunicação entre aplicações e bancos, inicialização automatizada de banco, `port-forward` e troubleshooting de ambiente Kubernetes.
