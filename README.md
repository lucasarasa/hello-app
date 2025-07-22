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
    return {"message": "Hello World! Successful test."}
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
| `DOCKER_PASSWORD` | Sua senha ou **token de acesso** do Docker Hub *(necessário se você usa login via Google)*                        |
| `MANIFEST_REPO`   | Nome do repositório de manifests no formato `usuario/nome-do-repo` |
| `SSH_PRIVATE_KEY` | Chave SSH privada para acessar o repositório de manifests via SSH  |

### 📌 Observação:

Se sua conta do Docker Hub está vinculada ao Google, **você precisa criar um token manualmente**:

1. Vá até sua conta Docker: [https://hub.docker.com/settings/security](https://hub.docker.com/settings/security)
2. Clique em **New Access Token**
3. Dê um nome e clique em **Generate**
4. Use esse token no campo `DOCKER_PASSWORD`
---

## 🔧 Build da Imagem com GitHub Actions

A pipeline automatiza o processo de:

1. Geração automática da versão com base em número de commits no `main.py`
2. Build da imagem Docker
3. Push da imagem para o Docker Hub
4. Atualização do manifest de deployment no repositório `hello-manifest`

### 🚦 Disparo

A pipeline é disparada sempre que há push no arquivo `main.py` ou manualmente via **workflow\_dispatch**.

### 📄 Exemplo de workflow (`.github/workflows/deploy.yaml`):

```yaml
name: Build, Push e Deploy com Versão Automática
run-name: Build e Push da Imagem Docker e Deploy Automático
description: Build e Push da Imagem Docker e Deploy Automático com base na mudança do arquivo

on:
  push:
    paths:
      - 'main.py'
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout do código
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Definir versão (v<numero de commits>) baseado na mudança do main.py
        run: |
          VERSION="v$(git log --pretty=oneline -- main.py | wc -l)"
          echo "IMAGE_TAG=$VERSION" >> $GITHUB_ENV
          echo "Versão da imagem gerada (main.py): $VERSION"

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
          sed -i "s|image: .*|image: ${{ secrets.DOCKER_USERNAME }}/hello-app:${{ env.IMAGE_TAG }}|" manifests/manifests/deployment.yaml
          echo "Arquivo de deployment atualizado com a nova imagem."

      - name: Commit e push das mudanças no repo de manifests
        run: |
          cd manifests
          git config user.name "GitHub-Actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add manifests/deployment.yaml
          git commit -m "Atualizando deployment para nova imagem ${IMAGE_TAG}"
          git push origin main
```

---

## 🧪 Testando Localmente

Você pode executar o container localmente com:

```bash
docker build -t hello-app:tag .
docker run -p 8080:80 hello-app:tag
```

Abra no navegador: [http://localhost:8080](http://localhost:8080)

---

## 🤖 Deploy via ArgoCD

Após o push e atualização no repositório `hello-manifest`, o ArgoCD detecta a nova imagem e sincroniza automaticamente o deploy.

---

## 📂 Relacionamento com o Repositório de Manifests

O repositório `hello-manifest` contém os YAMLs de deployment da aplicação. Esse repositório é monitorado pelo ArgoCD.

Link para o repositório de manifests: [https://github.com/lucasarasa/hello-manifests](https://github.com/lucasarasa/hello-manifests)

---

## 🔐 Acesso via SSH no ArgoCD

### ▶️ Usando repositório público (mais simples)

Se o repositório `hello-manifest` for **público**, o ArgoCD pode acessá-lo diretamente. Para criar a aplicação, use o comando:

```bash
argocd app create hello-app \
  --repo https://github.com/SEU-USUARIO/hello-manifest.git \
  --path manifests \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

Em seguida, sincronize:

```bash
argocd app sync hello-app
```

### 🔐 Usando repositório **privado** com chave SSH

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

## 👨‍💻 Autor

**Lucas Sarasa**\
🔗 [LinkedIn](https://www.linkedin.com/in/lucassarasa)
