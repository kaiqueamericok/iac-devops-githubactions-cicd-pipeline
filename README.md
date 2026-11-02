# Infra as Code: Deploy de Infraestrutura com Terraform e GitHub Actions

Pipeline de CI/CD que provisiona e destrói infraestrutura na AWS (EC2, ECR e S3) de forma automatizada, usando **Terraform** e **GitHub Actions**, com autenticação segura via*OIDC (sem access keys armazenadas).

---

## 📌 Contexto e problema

Em projetos reais, o código muda o tempo todo (cor de um botão, nomes, tabelas...). Fazer builds e deploys manuais:

- exige muito trabalho repetitivo;
- se torna inviável em larga escala, com vários clientes;
- não dá para repetir várias vezes no mesmo dia sem perder produtividade e aumentar o risco de erros.

**Solução:** automatizar o processo com pipelines **CI/CD**.

- **CI (Integração Contínua):** testar, validar e detectar erros no código de forma contínua.
- **CD (Entrega/Deploy Contínuo):** o deploy acontece automaticamente, sem intervenção humana, levando o código à produção de forma rápida e segura.

---

## Arquitetura

```
┌──────────────┐   workflow_dispatch   ┌────────────────────┐
│  Você (UI do │ ───────────────────▶  │   GitHub Actions   │
│   GitHub)    │   inputs: apply /     │  Runner (ubuntu)   │
└──────────────┘   plan_destroy /      └─────────┬──────────┘
                   destroy                       │ OIDC (AssumeRole)
                                                 ▼
                                      ┌────────────────────┐
                                      │  AWS (us-east-1)   │
                                      │  EC2 · ECR · S3    │
                                      └────────────────────┘
```

### Conceitos do GitHub Actions

| Conceito | Descrição |
|----------|-----------|
| **Evento** | Dispara a pipeline (aqui, manualmente via `workflow_dispatch`) |
| **Workflow** | Conjunto de etapas executadas em ordem |
| **Job** | Unidade de trabalho executada em um runner |
| **Runner** | Máquina virtual que executa o job (`ubuntu-latest`) |
| **Step** | Script ou action executado dentro do job |

### Por que separar o repositório de infra do da aplicação?

O código Terraform fica em um repositório **separado** do código da aplicação. Uma mesma infraestrutura (por exemplo, um cluster EKS) pode servir a centenas de repositórios com microsserviços diferentes, todos apontando para a mesma base.

---

## 🔐 Autenticação segura com OIDC

Em vez de salvar *access key* e *secret key* no GitHub (prática **não recomendada**), o runner assume uma **IAM Role** na AWS via OpenID Connect.

### 1. Criar o Identity Provider na AWS (IAM)

- **Tipo:** OpenID Connect
- **Provider URL:** `https://token.actions.githubusercontent.com`
- **Audience:** `sts.amazonaws.com`

### 2. Criar a IAM Role

- **Trusted entity:** Web identity (GitHub)
- **Audience:** `sts.amazonaws.com`
- **GitHub organization:** usuário/organização no GitHub
- **Repositório:** `infra-as-code`
- **Nome da role:** `GitHubInfraRole`

### 3. Políticas de permissão

- `AmazonEC2ContainerRegistryFullAccess` (ECR)
- `AmazonEC2FullAccess` (EC2)
- `AmazonS3FullAccess` (S3, para ler/gravar o state do Terraform)

> ⚠️ **Atenção:** essas permissões (*Full Access*) **não são ideais para produção**. Foram usadas apenas para fins de demonstração/laboratório. Em ambientes reais, devemos seguir o princípio do menor privilégio.

---

## ⚙️ O Workflow

Arquivo: `.github/workflows/terraform.yaml`

```yaml
name: Terraform CI/CD

on:
  workflow_dispatch:
    inputs:
      apply:
        description: 'Executar o Terraform apply?'
        required: true
        default: 'false'
        type: choice
        options:
          - true
          - false

      destroy:
        description: 'Destruir ambiente?'
        required: true
        default: 'false'
        type: choice
        options:
          - true
          - false

      plan_destroy:
        description: 'Planejar o destroy do ambiente?'
        required: true
        default: 'false'
        type: choice
        options:
          - true
          - false

permissions:
  id-token: write   # necessário para OIDC
  contents: read

jobs:
  job1:
    name: Terraform
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v5.0.0

      - name: "Configure AWS Credentials"
        uses: aws-actions/configure-aws-credentials@v6.3.0
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/GitHubInfraRole
          aws-region: us-east-1

      - name: HashiCorp - Setup Terraform
        uses: hashicorp/setup-terraform@v4.0.1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -out=tfplan.tfout

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan.tfout
        if: ${{ github.event.inputs.apply == 'true' }}

      - name: Plan Destroy
        run: terraform plan -destroy -out=tfplan.tfout
        if: ${{ github.event.inputs.plan_destroy == 'true' }}

      - name: Terraform Destroy
        run: terraform apply -destroy -auto-approve tfplan.tfout
        if: ${{ github.event.inputs.destroy == 'true' }}
```

### Explicação das etapas

| Etapa | O que faz |
|-------|-----------|
| **Checkout** | Clona o repositório na máquina Ubuntu do runner |
| **Configure AWS Credentials** | Assume a role `GitHubInfraRole` via OIDC na região `us-east-1` |
| **Setup Terraform** | Instala o Terraform no runner |
| **Terraform Init** | Inicializa o projeto e o backend |
| **Terraform Validate** | Valida a sintaxe e a consistência dos arquivos |
| **Terraform Plan** | Gera o plano e o salva em `tfplan.tfout` |
| **Terraform Apply** | Aplica o plano salvo (apenas se `apply == true`) |
| **Plan Destroy** | Gera um plano de destruição (apenas se `plan_destroy == true`) |
| **Terraform Destroy** | Executa o plano de destruição (apenas se `destroy == true`) |

> 💡 O `-auto-approve` é necessário porque a pipeline roda em segundo plano e não há como digitar `yes`. Usar um arquivo de plano (`-out`) garante que apenas o que foi planejado será aplicado/destruído.

---

## ▶️ Como usar

1. Faça push dos arquivos Terraform e do workflow:
   ```bash
   git add .
   git commit -m "feat: add tf files"
   git add .github/
   git commit -m "feat: add workflow"
   git push
   ```
2. No GitHub, vá em **Actions → Terraform CI/CD → Run workflow**.
3. Escolha os inputs:
   - `apply = true` para provisionar a infraestrutura;
   - `plan_destroy = true` para visualizar o que seria destruído;
   - `destroy = true` para destruir os recursos.

## 🛠️ Tecnologias

- [Terraform](https://www.terraform.io/)
- [GitHub Actions](https://docs.github.com/actions)
- AWS: EC2, ECR, S3, IAM (OIDC)
- Docker

---

## ✅ O que foi demonstrado

- Containerizar uma aplicação com Docker (Dockerfile, build, tag e push para o ECR)
- Fazer o deploy de um site estático na AWS EC2
- Entender os problemas de escala e a necessidade de automação
- Provisionar infraestrutura como código (IaC) com Terraform
- Automatizar tudo com CI/CD usando GitHub Actions e autenticação via OIDC

---

## 📚 Referências

- [configure-aws-credentials (GitHub Marketplace)](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions)
- [OIDC na AWS (GitHub Docs)](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [Sintaxe de workflows do GitHub Actions](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)
- [setup-terraform (HashiCorp)](https://github.com/hashicorp/setup-terraform)
