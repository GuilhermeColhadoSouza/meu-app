# 🚀 Task 3 - Docker Swarm + CI/CD  
**Autor:** Guilherme Colhado

Este projeto demonstra a criação de um cluster Docker Swarm, deploy de serviços usando Docker Compose, e implementação de um pipeline de CI/CD via GitHub Actions para atualização automática dos serviços.

---

## 📋 Etapas

### 1. Configuração do Docker Swarm

- **Criar rede:**
  ```bash
  docker network create --driver bridge --subnet 10.10.10.0/24 swarm-net
  ```

- **Subir o nó gerenciador:**
  ```bash
  docker run -d --privileged --name manager --hostname manager --network swarm-net --ip 10.10.10.10 docker:dind
  ```

- **Subir o nó trabalhador:**
  ```bash
  docker run -d --privileged --name worker1 --hostname worker1 --network swarm-net --ip 10.10.10.11 docker:dind
  ```

- **Inicializar o Swarm no Manager:**
  ```bash
  docker exec -it manager sh -c "docker swarm init --advertise-addr 10.10.10.10"
  ```

- **Adicionar o Worker ao cluster:**
  ```bash
  docker exec -it worker1 sh -c "docker swarm join --token <SEU_TOKEN> 10.10.10.10:2377"
  ```

- **Verificar os nós:**
  ```bash
  docker exec -it manager docker node ls
  ```

---

### 2. Criando um Serviço com Docker Compose

- **Acesse o Manager:**
  ```bash
  docker exec -it manager sh
  ```

- **Crie o arquivo `docker-compose.yml`:**

```yaml
version: '3.8'
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - app-net
    volumes:
      - html-data:/usr/share/nginx/html

networks:
  app-net:
    driver: overlay

volumes:
  html-data:
```

- **Deploy da stack:**
  ```bash
  docker stack deploy -c docker-compose.yml webstack
  ```

- **Verificar se o serviço está ativo:**
  ```bash
  docker stack services webstack
  ```

---

### 3. Implementação de Pipeline CI/CD

- **Estrutura do projeto:**
  ```
  meu-app/
  ├── app/
  │   └── index.html
  ├── Dockerfile
  └── .github/
      └── workflows/
          └── deploy.yml
  ```

- **Exemplo do `index.html`:**
  ```html
  <!DOCTYPE html>
  <html>
    <head><title>Meu Site com CI/CD</title></head>
    <body><h1>Funcionou!</h1></body>
  </html>
  ```

- **Exemplo do `Dockerfile`:**
  ```Dockerfile
  FROM nginx:alpine
  COPY app/ /usr/share/nginx/html
  ```

- **Pipeline no GitHub Actions (`deploy.yml`):**

```yaml
name: Deploy no Docker Swarm

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout do código
        uses: actions/checkout@v3

      - name: Login no Docker Hub
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      - name: Build da imagem
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/meu-app:latest .

      - name: Push da imagem
        run: docker push ${{ secrets.DOCKER_USERNAME }}/meu-app:latest

      - name: Atualizar serviço no Swarm via SSH
        uses: appleboy/ssh-action@v0.1.6
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            docker service update --image ${{ secrets.DOCKER_USERNAME }}/meu-app:latest webstack_web
```

---

### 4. Estratégia de Implantação

- **Atualizações contínuas:**
  ```bash
  docker service update --update-parallelism 1 --update-delay 10s webstack_web
  ```

- **Simulando Implantação Azul-Verde:**

1. Crie duas versões diferentes do `index.html` e gere duas imagens:
   - `meu-app:latest` (versão azul)
   - `meu-app:v2` (versão verde)

2. Suba dois serviços separados:
   ```bash
   docker service create --name meu-site-azul -p 8887:80 guic170602/meu-app:latest
   docker service create --name meu-site-verde -p 8888:80 guic170602/meu-app:v2
   ```

3. Teste as duas versões:
   - `http://localhost:8887` → Azul
   - `http://localhost:8888` → Verde

4. Para promover a versão verde:
   ```bash
   docker service rm meu-site-azul
   docker service update --publish-add published=8887,target=80 meu-site-verde
   ```

---

## 📄 Conclusão

Este projeto implementa um processo completo de orquestração de containers com Docker Swarm, implantação de serviços com Docker Compose, CI/CD via GitHub Actions e estratégias de atualização contínua e azul-verde para aumentar a resiliência e disponibilidade do sistema.