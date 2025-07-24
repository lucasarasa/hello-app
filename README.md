# 🚀 Aplicação Hello App – GitOps com ArgoCD

Este repositório contém a aplicação `hello-app`, uma aplicação simples de exemplo usada para testes de deploy automatizado via GitOps utilizando ArgoCD.

---

## 📌 Objetivo

* Desenvolver uma aplicação containerizada
* Automatizar o build e push da imagem Docker via GitHub Actions
* Preparar a aplicação para deploy em Kubernetes via ArgoCD

---

## 🧱 Estrutura do Repositório

```
hello-app/
├── main.py                 # Código da aplicação (FastAPI, por exemplo)
├── Dockerfile              # Instruções de build da imagem
├── requirements.txt        # Dependências da aplicação
├── .dockerignore           # Arquivos/pastas ignorados no build Docker
├── .gitignore              # Arquivos/pastas ignorados pelo Git
├── .github/
│   └── workflows/
│       └── deploy.yaml     # Pipeline CI/CD com GitHub Actions
└── README.md               # Este documento
```

---

## ⚙️ Tecnologias Utilizadas

* Python / FastAPI
* Docker
* GitHub Actions
* ArgoCD
* Kubernetes

---

## 📝 Código da Aplicação (`main.py`)

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World!"}
```

---

## 🐳 Dockerfile

```dockerfile
FROM python:3.12-alpine

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

---

## 🔐 Configurando as Secrets no GitHub

Para que a pipeline funcione corretamente, é necessário configurar algumas **secrets** no repositório `hello-app`. Acesse o repositório no GitHub, vá até:

> Settings → Secrets → Actions → New repository secret

### ✅ Secrets obrigatórias:

| Nome              | Descrição                                                          |
| ----------------- | ------------------------------------------------------------------ |
| `DOCKER_USERNAME` | Seu nome de usuário no Docker Hub                                  |
| `DOCKER_PASSWORD` | Sua senha ou **token de acesso** do Docker Hub *(necessário se você usa login via Google)*|
| `MANIFEST_REPO`   | Nome do repositório de manifests no formato `usuario/nome-do-repo` |
| `SSH_PRIVATE_KEY` | Chave SSH privada para acessar o repositório de manifests via SSH  |
| `PERSONAL_TOKEN`  | Token pessoal para criar Pull Requests automaticamente             |


### 📌 Observação:
#### 🐳 Docker Hub com login via Google
#### Se sua conta do Docker Hub está vinculada ao Google, você precisa criar um token manualmente:
1. Vá até sua conta Docker: [https://hub.docker.com/settings/security](https://hub.docker.com/settings/security)
2. Clique em **New Access Token**
3. Dê um nome e clique em **Generate**
4. Use esse token no campo `DOCKER_PASSWORD`
---
#### 🔧 Criando PERSONAL_TOKEN para criar Pull Requests automaticamente:

1. Acesse: [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. Clique em **Generate new token (classic)**
3. Selecione permissões:
   - `repo` → para permitir leitura e escrita nos repositórios
   - `workflow` → para autorizar execuções de workflows automatizados
4. Salve o token gerado e adicione como `PERSONAL_TOKEN` em **Secrets** no repositório `hello-app`.

## 🔧 Build, Push e Deploy com GitHub Actions

O workflow principal (`01-build-deploy-exec.yaml`) automatiza:

1. Detecta o push de uma nova tag no formato `v*` (ex: `v1.0.0`)
2. Define a versão da imagem com base nessa tag (extraída de `${{ github.ref_name }}`)
3. Builda a imagem Docker e envia para o Docker Hub
4. Clona o repositório `hello-manifests`
5. Atualiza o `deployment.yaml` com a nova tag da imagem
6. Cria uma nova branch baseada na `main` (ex: `update-image-v1.0.0`)
7. Abre um Pull Request automático com a alteração

---

### 📄 Exemplo de workflow (`.github/workflows/deploy.yaml`):

```yaml
name: Build, Push e Deploy com Versão Automática
run-name: Build, Push e Deploy - Imagem ${{ github.ref_name }}
description: >
  Workflow para buildar e enviar automaticamente a imagem Docker ao Docker Hub
  sempre que uma nova tag for criada no formato "v*". Após o push da imagem,
  o deployment da aplicação é atualizado automaticamente para usar a nova versão.

on:
  push:
    tags:
      - "v*"
  workflow_dispatch:

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.set-version.outputs.image_tag }}
    steps:
      - name: Checkout do código
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Definir versão a partir da tag
        id: set-version
        run: |
          TAG_NAME="${GITHUB_REF#refs/tags/}"
          echo "IMAGE_TAG=$TAG_NAME" >> $GITHUB_ENV
          echo "image_tag=$TAG_NAME" >> $GITHUB_OUTPUT
          echo "Versão detectada: $TAG_NAME"

      - name: Login no Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build da imagem Docker
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/hello-app:${{ env.IMAGE_TAG }} .

      - name: Push da imagem para o Docker Hub
        run: |
          docker push ${{ secrets.DOCKER_USERNAME }}/hello-app:${{ env.IMAGE_TAG }}

  update-manifest:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - name: Clonar repo de manifests via SSH
        uses: actions/checkout@v3
        with:
          repository: ${{ secrets.MANIFEST_REPO }}
          path: manifests
          ssh-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Listar arquivos do diretório manifests
        run: ls -l manifests

      - name: Atualizar arquivo de deployment com nova imagem
        run: |
          sed -i "s|image: .*|image: ${{ secrets.DOCKER_USERNAME }}/hello-app:${{ needs.build-and-push.outputs.image_tag }}|" manifests/manifests/deployment.yaml
          echo "Arquivo de deployment atualizado com a nova imagem."

      - name: Commit e criar Pull Request com nova imagem
        uses: peter-evans/create-pull-request@v5
        with:
          token: ${{ secrets.PERSONAL_TOKEN }}
          title: "Atualiza deployment para imagem ${{ env.IMAGE_TAG }}"
          commit-message: "Atualiza imagem para ${{ env.IMAGE_TAG }}"
          base: main
          branch: update-image-${{ needs.build-and-push.outputs.image_tag }}
          path: manifests
```

---

## 📊 Como usar Git Tags para versionar corretamente

1. Faça commit normalmente no seu projeto:

```bash
git add .
git commit -m "feat: adiciona nova funcionalidade"
git push
```

2. Crie a tag:

```bash
git tag v1.0.0
```

3. Envie a tag para o GitHub:

```bash
git push origin v1.0.0
```

> Isso vai disparar o workflow, que builda a imagem, envia para o Docker Hub e cria o Pull Request automaticamente no repositório de manifests.

---

## 🤖 Deploy via ArgoCD

Após o push e atualização no repositório `hello-manifest`, o ArgoCD detecta a nova imagem e sincroniza automaticamente o deploy.

---

## 🔐 Acesso via SSH no ArgoCD

### 🛠️ Pré-requisitos: instalação e configuração do ArgoCD já concluídas

---

### 🌐 1. Expor o ArgoCD localmente (acesso via navegador)

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Acesse via browser: [https://localhost:8081](https://localhost:8081)

---

### 🔑 2. Obter senha inicial do usuário `admin`

#### 👉 No **Linux**:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

#### 👉 No **Windows** (PowerShell):

```powershell
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

---

### 🔐 3. Fazer login no ArgoCD via CLI

```bash
argocd login localhost:8081
```

Usuário: `admin` Senha: a senha obtida no passo anterior

---

### ▶️ 4. Criar a aplicação (repositório público)

Se o repositório `hello-manifest` for **público**, o ArgoCD pode acessá-lo diretamente. Para criar a aplicação, use o comando:

```bash
argocd app create hello-app \
  --repo https://github.com/SEU-USUARIO/SEU-REPOSITORIO.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

### 🔁 5. Sincronizar a aplicação manualmente

```bash
argocd app sync hello-app
```

> Ou faça isso pela interface gráfica

---

### 🔐 4.1. Criar a aplicação (repositório privado)

1. Gere um par de chaves SSH com:

```bash
ssh-keygen -t rsa -b 4096 -C "argocd@acesso" -f argocd-ssh-key
```

> Pressione Enter nas perguntas para manter os padrões. Isso criará dois arquivos: `argocd-ssh-key` (privada) e `argocd-ssh-key.pub` (pública).

2. Adicione a chave pública (`argocd-ssh-key.pub`) no GitHub:

   * Acesse o repositório `hello-app` ou `hello-manifest`
   * Vá em **Settings > Deploy Keys > Add deploy key**
   * Cole a chave pública e marque **Allow read access**

3. Crie uma `Secret` no cluster com a chave privada:

Crie um arquivo `repo-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-ssh-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  url: git@github.com:SEU-USUARIO/SEU-REPOSITORIO.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    SUA_CHAVE_PRIVADA_AQUI
    -----END OPENSSH PRIVATE KEY-----
```

Aplique com:

```bash
kubectl apply -f repo-secret.yaml
```

4. O ArgoCD poderá acessar o repositório via SSH.

Veja os detalhes no [README do projeto de manifests](https://github.com/lucasarasa/hello-manifests/blob/main/README.md)

---

## 📂 Relacionamento com o Repositório de Manifests

O repositório `hello-manifest` contém os YAMLs de deployment da aplicação. Esse repositório é monitorado pelo ArgoCD.

Link para o repositório de manifests: [https://github.com/lucasarasa/hello-manifests](https://github.com/lucasarasa/hello-manifests)

---

## 🔮 Evidências de Execução

### As imagens abaixo documentam a execução bem-sucedida da pipeline:

### - ✅ **Build e push da imagem no Docker Hub:**
<br>

Workflow: ![alt text](assets/WORKFLOW-HELLO-APP.png)
Push da Imagem: ![alt text](assets/WORKFLOW-HELLO-APP-PUSH-IMAGEM.png)
Docker Hub: ![alt text](assets/DOCKERHUB.png)
### - ✅ **Atualização automática dos manifests com a nova tag:**
<br>

Commit Nova Tag: ![alt text](assets/WORKFLOW-HELLO-APP-ATT-DEPLOYMENT.png)
PR criado com nova Tag: ![alt text](assets/PR-HELLO-MANIFESTS.png)
### - ✅ **Print do ArgoCD com a aplicação sincronizada:**
<br>

ArgoCD com Tag antiga (desatualizado): ![alt text](assets/ENVIDENCIA-ARGOCD-IMAGE.png)
ArgoCD Sync Policy Enable: ![alt text](assets/ARGO-CD-ENABLE.png)
ArgoCD Sincronizado (atualizado): ![alt text](assets/ENVIDENCIA-ARGOCD-IMAGE-DEPOIS-SYNC.png)
ArgoCD aplicação sincronizada: ![alt text](assets/ARGO-CD-INTERFACE-HEALTHY-SYNCED.png)
### - ✅ **Print do **kubectl get pods** com a aplicação em execução:**
<br>

Pods em execução: ![alt text](assets/PODS.png)
### - ✅ **Resposta da aplicação via navegador:**
<br>

Resposta via navegador: ![alt text](assets/URL-NAVEGADOR.png)

---

## 👨‍💻 Autor

**Lucas Sarasa**\
🔗 [LinkedIn](https://www.linkedin.com/in/lucassarasa)
