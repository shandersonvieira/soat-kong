# SOAT Kong API Gateway

O objetivo deste projeto é centralizar a entrada de tráfego, gerenciar o roteamento para os microsserviços de backend e aplicar políticas de segurança (como autenticação JWT) de forma unificada, utilizando o **Kong Gateway** em modo *DB-less* (declarativo) rodando sobre **Kubernetes**.

## 🚀 Tecnologias Utilizadas

* **Kong Gateway 3.4**: API Gateway utilizado para orquestração de tráfego.
* **Kubernetes (K8s)**: Orquestração de containers (Deployment, Service, HPA, ConfigMap, Namespace).
* **Kustomize**: Gerenciamento de manifestos Kubernetes.
* **AWS EKS**: Plataforma de nuvem alvo para o deploy (conforme pipeline de CI).
* **GitHub Actions**: Automação de CI/CD.

## ⚙️ Arquitetura e Serviços

O Kong está configurado para rotear requisições para os seguintes microsserviços internos do cluster:

| Serviço (Upstream) | Rota Externa | Métodos Permitidos | Autenticação (JWT) | Descrição |
| :--- | :--- | :--- | :---: | :--- |
| **Auth Service** | `/tokens` | `GET`, `POST` | ❌ Não | Serviço de autenticação para emissão de tokens. |
| **Customer Service** | `/customers` | `GET`, `POST` | ✅ Sim | Gestão de clientes. |
| **Product Service** | `/products` | `GET`, `POST`, `PUT`, `DELETE` | ✅ Sim | Catálogo e gestão de produtos. |

### Configuração de Segurança (JWT)

O projeto utiliza o plugin **JWT** do Kong para proteger rotas sensíveis.
* As rotas `/customers` e `/products` exigem um cabeçalho `Authorization` válido.
* O consumidor configurado é o `python-auth`.
* O segredo (secret) do JWT é injetado dinamicamente via variável de ambiente `SOAT_JWT_SECRET` durante o deploy.

## 📂 Estrutura do Projeto

A infraestrutura está organizada dentro da pasta `infra/` utilizando Kustomize:

* **`infra/namespace.yml`**: Cria o namespace `kong-service` onde os recursos residem.
* **`infra/configmaps/soat-backend.yml`**: Contém o arquivo `kong.yml` (Configuração Declarativa) que define serviços, rotas e plugins.
* **`infra/deployments/soat-backend.yml`**: Define o Deployment do Kong, utilizando a imagem `kong:3.4`, portas de administração e volumes de configuração.
* **`infra/services/soat-backend.yml`**: Expõe o Kong via `LoadBalancer`, mapeando portas HTTP (80) e HTTPS (443) para o proxy do Kong, além de portas administrativas.
* **`infra/hpas/soat-backend.yml`**: Configura o *Horizontal Pod Autoscaler* para escalar o Kong entre 1 e 2 réplicas baseando-se em 75% de uso de CPU.

## 🔄 CI/CD (Integração Contínua)

O deploy é automatizado via **GitHub Actions** no arquivo `.github/workflows/ci.yml`.

**Fluxo do Pipeline:**
1.  **Gatilhos**: Push na branch `develop` ou Pull Request na `main`.
2.  **Configuração AWS**: Autentica na AWS e atualiza o `kubeconfig` para o cluster EKS.
3.  **Injeção de Secrets**:
    * Utiliza `envsubst` para substituir a variável `${SOAT_JWT_SECRET}` dentro do ConfigMap `infra/configmaps/soat-backend.yml` pelo valor armazenado nos *Secrets* do GitHub.
4.  **Deploy**:
    * Aplica os manifestos usando `kubectl apply -k infra/`.
    * Aguarda o sucesso do rollout do deployment.

## 🛠️ Como Executar Localmente ou Manualmente

Para aplicar essa infraestrutura manualmente em um cluster Kubernetes, você precisará ter o `kubectl` configurado e a variável `SOAT_JWT_SECRET` definida.

1.  **Pré-requisitos**:
    * Cluster Kubernetes ativo.
    * `kubectl` instalado.
    * Ferramenta `gettext` (para o comando `envsubst`) ou substituição manual do secret.

2.  **Passos de Deploy**:

    ```bash
    # 1. Definir o segredo JWT (exemplo)
    export SOAT_JWT_SECRET="seu-segredo-aqui"

    # 2. Substituir a variável no ConfigMap (similar ao CI)
    # Nota: Isso altera o arquivo localmente. Em produção, o CI faz isso em tempo de execução.
    envsubst < infra/configmaps/soat-backend.yml > infra/configmaps/soat-backend-processed.yml
    mv infra/configmaps/soat-backend-processed.yml infra/configmaps/soat-backend.yml

    # 3. Aplicar os manifestos com Kustomize
    kubectl apply -k infra/

    # 4. Verificar o status
    kubectl get pods -n kong-service
    ```
