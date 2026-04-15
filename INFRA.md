# Documentação de Infraestrutura — Prova Técnica DevOps Sênior Philips

## Visão Geral

A infraestrutura implementada é composta por um servidor Jenkins (controller) e dois agentes de execução (nodes), todos rodando como contêineres Docker em uma rede bridge isolada. O pipeline de CI/CD realiza checagem de código, build, testes unitários, empacotamento e arquivamento de artefatos de um projeto C++17.

---

## Arquitetura

```
┌─────────────────────────────────────────────────────┐
│                  Host Ubuntu 24.04                  │
│                                                     │
│  ┌─────────────┐        Docker Network: jenkins-net │
│  │   Jenkins   │◄──────────────────────────────┐   │
│  │ (controller)│                               │   │
│  │  porta 8080 │        ┌──────────────────┐   │   │
│  └─────────────┘        │  agent1          │   │   │
│                          │  label: build    │───┘   │
│                          │  (Dockerfile.build)│      │
│                          └──────────────────┘       │
│                          ┌──────────────────┐       │
│                          │  agent2          │       │
│                          │  label: test     │───────┘
│                          │  (Dockerfile.test)│
│                          └──────────────────┘
└─────────────────────────────────────────────────────┘
```

---

## Componentes

### Jenkins Controller

| Item | Valor |
|---|---|
| Imagem base | `jenkins/jenkins:lts` |
| Porta exposta | `8080` |
| Rede Docker | `jenkins-net` |
| Acesso | Via SSH tunnel (`localhost:8080`) |

### Agent 1 — Build

| Item | Valor |
|---|---|
| Imagem | `jenkins-agent-build` |
| Dockerfile | `docker/Dockerfile.build` |
| Label Jenkins | `build` |
| Ferramentas | `g++`, `make`, `clang-tidy`, `clang-format`, `git` |
| Responsabilidade | Checkout, lint, format-check, compilação e empacotamento |

### Agent 2 — Test

| Item | Valor |
|---|---|
| Imagem | `jenkins-agent-test` |
| Dockerfile | `docker/Dockerfile.test` |
| Label Jenkins | `test` |
| Ferramentas | `g++`, `make`, `cmake`, `libgtest` (compilado em `/usr/local`) |
| Responsabilidade | Execução dos testes unitários com Google Test |

---

## Dockerfiles

### `docker/Dockerfile.build`

```dockerfile
FROM jenkins/inbound-agent

USER root

RUN apt-get update && \
    apt-get install -y \
        build-essential \
        git \
        clang-tidy \
        clang-format && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

USER jenkins
```

### `docker/Dockerfile.test`

```dockerfile
FROM jenkins/inbound-agent

USER root

RUN apt-get update && \
    apt-get install -y \
        build-essential \
        git \
        cmake \
        libgtest-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Google Test compilado e instalado em /usr/local
# (caminho obrigatório — hardcoded em calculator/tests/Makefile)
RUN cd /usr/src/googletest && \
    cmake -DCMAKE_INSTALL_PREFIX=/usr/local . && \
    make && \
    make install && \
    ldconfig

USER jenkins
```

> **Por que compilar o GTest manualmente?**
> O `apt` instala os headers em `/usr/include/gtest`, mas o `tests/Makefile` do projeto referencia explicitamente `/usr/local/include/gtest` e `/usr/local/lib`. A compilação a partir do source e instalação em `/usr/local` é obrigatória para o link funcionar.

---

## Rede Docker

```bash
docker network create jenkins-net
```

Todos os contêineres (controller + agentes) são conectados à mesma rede bridge `jenkins-net`, permitindo que os agentes se comuniquem com o controller pelo hostname `jenkins` (nome do contêiner).

---

## Como subir a infraestrutura

### 1. Criar a rede

```bash
docker network create jenkins-net
```

### 2. Subir o Jenkins controller

```bash
docker run -d \
  --name jenkins \
  --network jenkins-net \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

### 3. Build das imagens dos agentes

```bash
git clone https://github.com/dehbelicegit/Test_DevOps_Philips.git
cd Test_DevOps_Philips

docker build -f docker/Dockerfile.build -t jenkins-agent-build .
docker build -f docker/Dockerfile.test  -t jenkins-agent-test  .
```

### 4. Subir os agentes

Os secrets são gerados pelo Jenkins ao criar cada node em:
**Gerenciar Jenkins → Nodes → (nome do agente) → Status**

```bash
# Agent 1 — label: build
docker run -d \
  --name agent1 \
  --network jenkins-net \
  -e JENKINS_URL=http://jenkins:8080 \
  -e JENKINS_SECRET=<secret_agent1> \
  -e JENKINS_AGENT_NAME=agent1 \
  jenkins-agent-build

# Agent 2 — label: test
docker run -d \
  --name agent2 \
  --network jenkins-net \
  -e JENKINS_URL=http://jenkins:8080 \
  -e JENKINS_SECRET=<secret_agent2> \
  -e JENKINS_AGENT_NAME=agent2 \
  jenkins-agent-test
```

---

## Pipeline CI/CD

O pipeline é definido no `Jenkinsfile` na raiz do repositório e é carregado automaticamente pelo Jenkins via **Pipeline from SCM**.

### Gatilhos

| Tipo | Configuração |
|---|---|
| Manual | Qualquer usuário pode iniciar via interface do Jenkins |
| Agendado | Diariamente às ~2h (`cron('H 2 * * *')`) |
| Webhook | Configurável via GitHub Webhooks apontando para `http://<jenkins>:8080/github-webhook/` |

### Estágios

```
Checkout → Check → Build → Test → Package
```

| Estágio | Agent | Comando | Descrição |
|---|---|---|---|
| **Checkout** | `build` | `checkout scm` | Clona o repositório do GitHub |
| **Check** | `build` | `make check` | Executa `clang-tidy` (lint) + `clang-format` (formatação) |
| **Build** | `build` | `make clean && make` | Compila o binário C++17 em `calculator/bin/calculator` |
| **Test** | `test` | `make -C tests clean && make unittest` | Compila e executa os testes unitários com Google Test |
| **Package** | `build` | `tar -czf artifact.tar.gz` + `archiveArtifacts` | Empacota o binário e arquiva no Jenkins |

### Comportamento em falha

Qualquer erro em qualquer estágio aborta os estágios seguintes (`failFast` implícito do Declarative Pipeline). O bloco `post` registra o status final.

### Artefato gerado

`artifact.tar.gz` — contém o binário `calculator` compilado, arquivado diretamente no Jenkins com fingerprint para rastreabilidade.

---

## Projeto: Calculator C++17

| Item | Detalhe |
|---|---|
| Linguagem | C++17 |
| Build system | GNU Make |
| Testes | Google Test (GTest) |
| Análise estática | clang-tidy (`cppcoreguidelines*`) |
| Formatação | clang-format (Google style) |

### Correções aplicadas ao código fonte

Durante a execução do pipeline, dois bugs foram identificados e corrigidos:

1. **`calculator/src/main.cpp`** — variáveis `number1` e `number2` declaradas sem inicialização, violando a regra `cppcoreguidelines-init-variables` do clang-tidy.
   - Correção: `double number1 = 0.0, number2 = 0.0;`

2. **`calculator/src/calculator.hpp`** — método `divide()` sem proteção contra divisão por zero, causando `SIGFPE` (Floating Point Exception) no teste `Calculator<int>(0, 0).divide()`.
   - Correção: verificação `if (number2 == T{}) return T{};` antes da divisão.

---

## Acesso via SSH Tunnel

Quando as portas do servidor estão bloqueadas por Security Group (AWS), utilizar tunnel SSH para acessar o Jenkins localmente:

```bash
ssh -i chave.pem -L 8080:localhost:8080 ubuntu@54.94.13.81
```

Após isso, acessar: `http://localhost:8080`

---

## Repositório

- **Fork:** https://github.com/dehbelicegit/Test_DevOps_Philips
- **Jenkinsfile:** [Jenkinsfile](Jenkinsfile)
- **Dockerfiles:** [docker/](docker/)
