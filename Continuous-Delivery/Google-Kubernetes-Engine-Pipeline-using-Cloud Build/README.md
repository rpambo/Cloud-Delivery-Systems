# Pipeline CI/CD: Google Kubernetes Engine + Cloud Build

[![Status](https://img.shields.io/badge/status-active-brightgreen)]()
[![License](https://img.shields.io/badge/license-Apache%202.0-blue)]()
[![Google Cloud](https://img.shields.io/badge/platform-Google%20Cloud-red)]()

Implementação completa de um pipeline **Integração Contínua e Entrega Contínua (CI/CD)** automatizado usando Google Cloud Build e Google Kubernetes Engine.

## 📋 Sumário

- [Overview](#overview)
- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Instalação Rápida](#instalação-rápida)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Fluxo de Trabalho](#fluxo-de-trabalho)
- [Configuração Detalhada](#configuração-detalhada)
- [Operações Comuns](#operações-comuns)
- [Troubleshooting](#troubleshooting)
- [Documentação](#documentação)
- [Contribuindo](#contribuindo)
- [Licença](#licença)

## 🎯 Overview

Este projeto demonstra como configurar um pipeline CI/CD profissional onde:

1. **Código** é automaticamente testado quando há push
2. **Imagens Docker** são construídas e armazenadas no Artifact Registry
3. **Manifests Kubernetes** são atualizados automaticamente
4. **Aplicação** é implantada automaticamente no GKE
5. **Histórico completo** é mantido para auditoria e rollback

### ✨ Características

- ✅ **Automação Completa**: Desde o commit até produção
- ✅ **Testes Automatizados**: Cada build passa por testes unitários
- ✅ **Versionamento de Imagens**: Cada versão é rastreável
- ✅ **Histórico de Deploys**: Veja todos os builds em Cloud Build
- ✅ **Rollback Fácil**: Reverta para versão anterior com 1 clique
- ✅ **Separação de Responsabilidades**: Código e config em repos diferentes
- ✅ **Segurança**: Secrets gerenciados no Secret Manager
- ✅ **GitOps**: Tudo declarativo em Git

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                        GOOGLE CLOUD                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Git Repositories (Gitea Server)                               │
│  ┌──────────────────────┐   ┌──────────────────────┐           │
│  │ hello-cloudbuild-app │   │ hello-cloudbuild-env │           │
│  │  (código fonte)      │   │ (configuração K8s)   │           │
│  └──────────┬───────────┘   └──────────┬───────────┘           │
│             │                          │                       │
│             │ commit                   │ candidate branch      │
│             │                          │                       │
│  ┌──────────▼──────────────────────────▼────────┐              │
│  │        Cloud Build - Pipeline CI             │              │
│  │  1. Execute unit tests                       │              │
│  │  2. Build Docker image                       │              │
│  │  3. Push to Artifact Registry                │              │
│  │  4. Update Kubernetes manifest               │              │
│  │  5. Push to env repo (candidate branch)      │              │
│  └──────────┬─────────────────────────────────┬─┘              │
│             │                                 │                │
│             │ Docker image                    │ manifest        │
│             │                                 │                │
│  ┌──────────▼──────────┐  ┌──────────────────▼─────────┐       │
│  │  Artifact Registry   │  │  Cloud Build - Pipeline CD │       │
│  │  (Image storage)     │  │  1. Apply manifest to GKE  │       │
│  └──────────┬───────────┘  │  2. Copy to production     │       │
│             │              └──────────────┬──────────────┘      │
│             │                             │                    │
│             └─────────────────┬───────────┘                    │
│                               │                                │
│              ┌────────────────▼──────────────┐                 │
│              │   GKE Cluster                 │                 │
│              │  (Kubernetes Pods running)   │                 │
│              │   hello-cloudbuild-app       │                 │
│              └────────────────┬──────────────┘                 │
│                               │                                │
│              ┌────────────────▼──────────────┐                 │
│              │   Service + Load Balancer    │                 │
│              │   (Acessível publicamente)   │                 │
│              └─────────────────────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 📋 Pré-requisitos

### Conta e Credenciais
- [Google Cloud Account](https://cloud.google.com) com billing ativado
- `gcloud` CLI instalado ([instruções](https://cloud.google.com/sdk/docs/install))
- `kubectl` instalado (`gcloud components install kubectl`)
- `git` instalado

### Permissões
- Proprietário (Owner) ou Editor no projeto Google Cloud
- Acesso ao servidor Git (Gitea) com credenciais

### Recursos
- Limite de cota para GKE (verificar em "Quotas" no Console)
- Limite de cota para Cloud Build
- Limite de cota para Artifact Registry

## 🚀 Instalação Rápida

### 1. Clone este repositório
```bash
git clone https://github.com/seu-usuario/hello-cloudbuild.git
cd hello-cloudbuild
```

### 2. Configure as variáveis
```bash
export PROJECT_ID=seu-projeto-id
export REGION=us-central1
export GIT_SERVER_IP=seu-servidor-git-ip

gcloud config set project $PROJECT_ID
gcloud config set compute/region $REGION
```

### 3. Execute o script de setup
```bash
bash setup.sh
```

Ou execute manualmente cada tarefa (veja a seção [Configuração Detalhada](#configuração-detalhada)).

### 4. Verifique o status
```bash
# Ver clusters GKE
gcloud container clusters list

# Ver builds recentes
gcloud builds list --limit=10

# Ver repositórios Artifact Registry
gcloud artifacts repositories list --location=$REGION
```

## 📁 Estrutura do Projeto

```
hello-cloudbuild/
├── hello-cloudbuild-app/          # Repositório de código
│   ├── app.py                      # Aplicação Python (Flask)
│   ├── test_app.py                 # Testes unitários
│   ├── Dockerfile                  # Definição da imagem Docker
│   ├── cloudbuild.yaml             # Configuração Pipeline CI
│   ├── kubernetes.yaml.tpl         # Template do manifest Kubernetes
│   └── requirements.txt            # Dependências Python (opcional)
│
├── hello-cloudbuild-env/           # Repositório de configuração
│   ├── kubernetes.yaml             # Manifest Kubernetes (gerado)
│   ├── cloudbuild.yaml             # Configuração Pipeline CD
│   ├── cloudbuild-delivery.yaml    # Pipeline de delivery (backup)
│   └── cloudbuild-trigger-cd.yaml  # Trigger do CD pipeline
│
├── docs/                           # Documentação
│   ├── GUIDE_PT.md                 # Guia em Português (detalhado)
│   ├── ARCHITECTURE.md             # Detalhes de arquitetura
│   ├── TROUBLESHOOTING.md          # Solução de problemas
│   └── OPERATIONS.md               # Operações comuns
│
├── scripts/                        # Scripts auxiliares
│   ├── setup.sh                    # Setup automático
│   ├── cleanup.sh                  # Limpeza de recursos
│   ├── rollback.sh                 # Script de rollback
│   └── monitor.sh                  # Monitoramento
│
├── .gitignore                      # Arquivos a ignorar no Git
├── README.md                       # Este arquivo
└── LICENSE                         # Apache 2.0 License
```

## 🔄 Fluxo de Trabalho

### Fluxo Padrão

```
1. Desenvolvedor faz mudança no código
   └─> git add . && git commit -m "mudança"

2. Push para repo hello-cloudbuild-app
   └─> git push origin main

3. Cloud Build dispara automaticamente (Pipeline CI)
   ├─> Executa testes (test_app.py)
   ├─> Constrói imagem Docker
   ├─> Push para Artifact Registry
   ├─> Atualiza manifest Kubernetes
   └─> Push para hello-cloudbuild-env/candidate

4. Cloud Build dispara Pipeline CD
   ├─> Aplica manifest ao cluster GKE
   ├─> Se sucesso:
   │   └─> Copia manifest para production
   └─> Se falha:
       └─> Notifica desenvolvedor

5. Aplicação ativa em produção
   └─> Usuários veem nova versão
```

### Fluxo de Rollback

```
1. Acesse Cloud Build History
2. Encontre o build anterior que funcionava
3. Clique em "Rebuild"
4. O Kubernetes automaticamente reverão a versão anterior
5. Nenhum dado é perdido (banco de dados continua)
```

## 🔧 Configuração Detalhada

### Tarefa 1: Inicializar Laboratório

```bash
# Definir variáveis
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
export REGION=us-central1
gcloud config set compute/region $REGION
export GIT_SERVER_IP=$(gcloud compute instances describe git-server --zone=us-central1-a --format='get(networkInterfaces[0].accessConfigs[0].natIP)')

# Ativar APIs
gcloud services enable container.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  containeranalysis.googleapis.com

# Criar repositório Artifact Registry
gcloud artifacts repositories create my-repository \
  --repository-format=docker \
  --location=$REGION

# Criar cluster GKE
gcloud container clusters create hello-cloudbuild \
  --num-nodes 1 \
  --region $REGION

# Configurar Git
git config --global user.name "seu-nome"
git config --global user.email "seu-email@example.com"
```

### Tarefa 2: Conectar ao Git

```bash
# Baixar código de exemplo
mkdir hello-cloudbuild-app
gcloud storage cp -r gs://spls/gsp1077/gke-gitops-tutorial-cloudbuild/* hello-cloudbuild-app

# Personalizar para sua região
cd hello-cloudbuild-app
sed -i "s/us-central1/$REGION/g" cloudbuild.yaml
sed -i "s/us-central1/$REGION/g" kubernetes.yaml.tpl

# Inicializar repositório Git
git init
git remote add origin http://${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-app.git
git branch -m main
git add .
git commit -m "initial commit"
git push -u http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-app.git main
```

### Tarefa 3-6: Configurar Pipelines

Veja [Configuração Detalhada](docs/GUIDE_PT.md) para instruções passo a passo de cada tarefa.

## 🛠️ Operações Comuns

### Ver Status do Build
```bash
# Listar builds recentes
gcloud builds list --limit=20

# Ver detalhes de um build específico
gcloud builds log <BUILD_ID>

# Watch build em tempo real
gcloud builds log <BUILD_ID> --stream
```

### Ver Aplicação em Execução
```bash
# Listar serviços
kubectl get services --namespace default

# Ver logs dos pods
kubectl logs -f deployment/hello-cloudbuild

# Descrever pod
kubectl describe pod <POD_NAME>
```

### Fazer Mudanças no Código
```bash
cd hello-cloudbuild-app

# Fazer mudanças
vim app.py

# Commit e push
git add app.py
git commit -m "mudança descritiva"
git push origin main

# Cloud Build automaticamente testa e deploya
```

### Rollback para Versão Anterior
```bash
# Opção 1: Via Console
# 1. Cloud Build > History
# 2. Clique no build anterior
# 3. Clique "Rebuild"

# Opção 2: Via CLI
gcloud builds submit --config=cloudbuild.yaml \
  --substitutions=_SHORT_SHA=abc123def,_COMMIT_SHA=abc123... .
```

### Escalar Aplicação
```bash
# Aumentar número de réplicas
kubectl scale deployment hello-cloudbuild --replicas=5

# Ver replicas
kubectl get deployment hello-cloudbuild
```

### Ver Histórico de Deployments
```bash
# Repositório env contém histórico
cd hello-cloudbuild-env

# Ver production branch
git log production --oneline

# Ver changes específicas
git show <COMMIT_SHA>
```

## 🐛 Troubleshooting

### Build Falha nos Testes
```bash
# Verificar logs detalhados
gcloud builds log <BUILD_ID>

# Testar localmente
cd hello-cloudbuild-app
pip install flask
python test_app.py -v
```

### Imagem Não Aparece no Artifact Registry
```bash
# Verificar repositório
gcloud artifacts repositories list --location=$REGION

# Verificar API ativa
gcloud services list | grep artifact

# Verificar permissões
gcloud projects get-iam-policy $PROJECT_ID | grep Cloud
```

### Kubernetes Não Consegue Puxar Imagem
```bash
# Verificar manifest
kubectl get deployment hello-cloudbuild -o yaml

# Verificar URL da imagem
kubectl describe pod <POD_NAME> | grep Image

# Reconstruir imagem
gcloud builds submit --config=cloudbuild.yaml .
```

### Cloud Build Não Consegue Acessar Secrets
```bash
# Verificar secret existe
gcloud secrets list

# Verificar permissões
gcloud secrets get-iam-policy ssh_key_secret
```

Para mais ajuda, veja [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md).

## 📚 Documentação

| Documento | Descrição |
|-----------|-----------|
| [GUIDE_PT.md](docs/GUIDE_PT.md) | Guia completo em Português (detalhado) |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Detalhes técnicos da arquitetura |
| [OPERATIONS.md](docs/OPERATIONS.md) | Operações diárias e manutenção |
| [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Solução de problemas comuns |

## 🤝 Contribuindo

Contributions são bem-vindas! Para contribuir:

1. Fork este repositório
2. Crie uma branch para sua feature (`git checkout -b feature/AmazingFeature`)
3. Commit suas mudanças (`git commit -m 'Add AmazingFeature'`)
4. Push para a branch (`git push origin feature/AmazingFeature`)
5. Abra um Pull Request

## 📋 Checklist de Pré-Produção

- [ ] Testes unitários cobrem 80%+ do código
- [ ] Testes de integração configurados
- [ ] Alertas de falha de build configurados
- [ ] Monitoramento da aplicação ativo
- [ ] Backup de dados configurado
- [ ] Plano de disaster recovery definido
- [ ] Documentação atualizada
- [ ] Equipe treinada no rollback
- [ ] Secrets migrados para Secret Manager
- [ ] Versões de imagens tagueadas com semver

## 📈 Métricas e Monitoramento

### Cloud Build Metrics
- Tempo de build médio
- Taxa de sucesso/falha
- Frequência de deploys

### Application Metrics
- Latência de resposta
- Taxa de erro
- CPU/Memory usage

Configurar alertas em Cloud Monitoring:
```bash
gcloud monitoring policies list
```

## 🔒 Segurança

### Best Practices Implementadas
- ✅ Secrets em Secret Manager (não em Git)
- ✅ IAM roles específicas (principle of least privilege)
- ✅ Imagens assinadas no Artifact Registry
- ✅ Network policies no Kubernetes
- ✅ RBAC configurado

### Melhorias Recomendadas
- [ ] Adicionar Container Image Scanning
- [ ] Implementar Policy Controller
- [ ] Usar Workload Identity
- [ ] Configurar VPC-native networking
- [ ] Adicionar DDoS protection

## 💰 Custo

### Estimativa Mensal (US$)
| Serviço | Estimativa |
|---------|-----------|
| GKE (1 nó) | $25-40 |
| Cloud Build (300 min/mês) | $0 (free tier) |
| Artifact Registry | $0.10 GB |
| Secret Manager | $0.06 secret |
| Load Balancer | $16 |
| **Total** | **$41-60** |

*Valores aproximados. Veja [Google Cloud Pricing](https://cloud.google.com/pricing) para detalhes.*

## 📞 Suporte

- 📖 [Google Cloud Documentation](https://cloud.google.com/docs)
- 🐛 [Issues neste repositório](https://github.com/seu-usuario/hello-cloudbuild/issues)
- 💬 [Google Cloud Community](https://www.googlecloudcommunity.com/)

## 📄 Licença

Este projeto está licenciado sob Apache License 2.0 - veja [LICENSE](LICENSE) para detalhes.

## 🙏 Agradecimentos

Baseado no tutorial oficial do Google Cloud:
- [GKE CI/CD with Cloud Build](https://cloud.google.com/build/docs/building/build-containers)
- [GitOps Best Practices](https://kubernetes.io/docs/concepts/gitops/)

## ✨ Roadmap

- [ ] Adicionar Helm charts
- [ ] Integrar com ArgoCD
- [ ] Adicionar SonarQube para análise de código
- [ ] Implementar Feature flags
- [ ] Adicionar canary deployments
- [ ] Integrar com Slack notifications
- [ ] Adicionar terraform para IaC
- [ ] Documentação em outros idiomas

---

**Versão**: 1.0.0  
**Última atualização**: 2024  
**Mantido por**: Seu Nome / Organização

⭐ Se este projeto foi útil, deixe uma star!
