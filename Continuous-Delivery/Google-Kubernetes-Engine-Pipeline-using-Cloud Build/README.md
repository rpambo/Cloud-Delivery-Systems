# Google Kubernetes Engine Pipeline usando Cloud Build

## Sobre o projeto

Este projeto demonstra a implementação de um pipeline CI/CD utilizando
Google Cloud Build, Google Kubernetes Engine (GKE), Artifact Registry,
Kubernetes, Git, Gitea e Secret Manager.

O pipeline automatiza o processo:
``` text
Código → Testes → Build → Container → Artifact Registry
                                      ↓
                              Manifest Kubernetes
                                      ↓
                              Gitea (candidate)
                                      ↓
                              Cloud Build (CD)
                                      ↓
                                   GKE
                                      ↓
                                production
```

O laboratório utiliza dois repositórios Git:

-   `hello-cloudbuild-app`: código da aplicação.
-   `hello-cloudbuild-env`: manifests e configuração de deployment.

## Arquitetura

O repositório da aplicação contém o código, testes, Dockerfile e
configuração do CI.

O repositório de ambiente contém os manifests Kubernetes e a
configuração do CD.

### Branches

-   `main`: branch principal.
-   `candidate`: histórico das tentativas de deployment.
-   `production`: histórico dos deployments bem-sucedidos.

## 1. Configuração inicial

``` bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
export REGION=<SUA_REGIAO>
gcloud config set compute/region $REGION
```

Obter o IP do servidor Gitea:

``` bash
export GIT_SERVER_IP=$(gcloud compute instances describe git-server \
  --zone=<ZONA> \
  --format='get(networkInterfaces[0].accessConfigs[0].natIP)')

echo "Git Server IP: ${GIT_SERVER_IP}"
```

## 2. Ativar APIs

``` bash
gcloud services enable container.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  containeranalysis.googleapis.com
```

## 3. Criar Artifact Registry

``` bash
gcloud artifacts repositories create my-repository \
  --repository-format=docker \
  --location=$REGION
```

## 4. Criar cluster GKE

``` bash
gcloud container clusters create hello-cloudbuild \
  --num-nodes 1 \
  --region $REGION
```

## 5. Configurar Git

``` bash
git config --global user.name "giteaadmin"
git config --global user.email "student@qwiklabs.net"
```

## 6. Repositório da aplicação

``` bash
cd ~
mkdir hello-cloudbuild-app

gcloud storage cp -r \
gs://spls/gsp1077/gke-gitops-tutorial-cloudbuild/* \
hello-cloudbuild-app

cd ~/hello-cloudbuild-app

git init
git remote add origin \
http://${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-app.git

git branch -m main
git add .
git commit -m "initial commit"

git push -u \
http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-app.git \
main
```

## 7. Construção da imagem

``` bash
cd ~/hello-cloudbuild-app

COMMIT_ID="$(git rev-parse --short=7 HEAD)"

gcloud builds submit \
  --tag="${REGION}-docker.pkg.dev/${PROJECT_ID}/my-repository/hello-cloudbuild:${COMMIT_ID}" .
```

## 8. Pipeline de Continuous Integration

O `cloudbuild.yaml` executa os testes, constrói a imagem e publica a
imagem no Artifact Registry.

``` bash
gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=SHORT_SHA=$(git rev-parse --short=7 HEAD) .
```

## 9. Secret Manager

Criar uma chave SSH:

``` bash
mkdir -p ~/workingdir
cd ~/workingdir

ssh-keygen \
  -t rsa \
  -b 4096 \
  -N '' \
  -f id_rsa \
  -C "student@qwiklabs.net"
```

Guardar a chave:

``` bash
gcloud secrets create ssh_key_secret \
  --data-file=$HOME/workingdir/id_rsa
```

## 10. Repositório de ambiente Kubernetes

``` bash
mkdir ~/hello-cloudbuild-env

gcloud storage cp -r \
gs://spls/gsp1077/gke-gitops-tutorial-cloudbuild/* \
~/hello-cloudbuild-env

cd ~/hello-cloudbuild-env

git init
git remote add origin \
http://${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-env.git

git branch -m main
git add .
git commit -m "initial commit"

git push -u \
http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-env.git \
main
```

## 11. Criar branches de deployment

Em um repositório novo:

``` bash
git checkout -b production
git checkout -b candidate
```

Depois:

``` bash
git push \
http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-env.git \
production

git push \
http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-env.git \
candidate
```

### Se as branches já existirem

Não use `git checkout -b` novamente.

Use:

``` bash
git checkout production
git push -u origin production

git checkout candidate
git push -u origin candidate
```

## 12. Continuous Delivery

O pipeline de CD é executado a partir da branch `candidate`.

Fluxo:

``` text
candidate
    ↓
Cloud Build
    ↓
kubectl apply
    ↓
GKE
    ↓
Deployment bem-sucedido
    ↓
production
```

## 13. Permissão do Cloud Build no GKE

``` bash
PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} \
  --format='get(projectNumber)')"

gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role=roles/container.developer
```

## 14. Fluxo completo

``` text
Developer
    │
    │ git push
    ▼
hello-cloudbuild-app
    │
    ▼
Cloud Build - CI
    │
    ├── Testes
    ├── Docker Build
    └── Push
          │
          ▼
    Artifact Registry
          │
          ▼
hello-cloudbuild-env
    │
    │ candidate
    ▼
Cloud Build - CD
    │
    ▼
kubectl apply
    │
    ▼
Google Kubernetes Engine
    │
    ├── sucesso
    ▼
production
```

## 15. Testar a aplicação

No Google Cloud Console:

``` text
Kubernetes Engine
    ↓
Gateways, Services & Ingress
    ↓
Services
```

Procure pelo serviço:

``` text
hello-cloudbuild
```

A aplicação inicialmente deverá apresentar:

``` text
Hello World!
```

## 16. Alterar a aplicação

``` bash
cd ~/hello-cloudbuild-app

sed -i 's/Hello World/Hello Cloud Build/g' app.py
sed -i 's/Hello World/Hello Cloud Build/g' test_app.py

git add app.py test_app.py
git commit -m "Hello Cloud Build"

git push \
http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-app.git \
main
```

Executar CI:

``` bash
gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=_SHORT_SHA=$(git rev-parse --short=7 HEAD),_COMMIT_SHA=$(git rev-parse HEAD) .
```

Atualizar o ambiente:

``` bash
cd ~/hello-cloudbuild-env

git pull \
http://giteaadmin:GiteaPassword123@${GIT_SERVER_IP}:3000/giteaadmin/hello-cloudbuild-env.git \
candidate
```

Executar CD:

``` bash
gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=_COMMIT_SHA=$(git rev-parse HEAD) .
```

Após o deployment:

``` text
Hello Cloud Build!
```

## 17. Rollback

No Google Cloud Console:

``` text
Cloud Build
    ↓
History
    ↓
Selecionar um build anterior
    ↓
Rebuild
```

O pipeline permite retornar a uma versão anterior da aplicação.

## Tecnologias utilizadas

  Tecnologia          Função
  ------------------- ---------------------------
  Git/Gitea           Versionamento
  Cloud Build         CI/CD
  Docker              Containerização
  Artifact Registry   Armazenamento das imagens
  Kubernetes          Orquestração
  GKE                 Cluster Kubernetes
  kubectl             Deployment
  Secret Manager      Gestão de segredos
  Cloud Logging       Logs

## Segurança

As credenciais usadas no laboratório são específicas do ambiente de
laboratório. Para um projeto real ou repositório público no GitHub, não
coloque senhas diretamente nos comandos, scripts ou arquivos
versionados.

Use Secrets, Secret Manager, variáveis protegidas ou outro mecanismo
seguro de autenticação.

## Referência

Documentação baseada no laboratório fornecido:

**Google Kubernetes Engine Pipeline using Cloud Build**

O laboratório demonstra um pipeline CI/CD completo com dois repositórios
Gitea, Cloud Build, Artifact Registry e GKE.
