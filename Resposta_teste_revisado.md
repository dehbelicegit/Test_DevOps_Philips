# Prova Técnica — DevOps Sênior Philips
**Candidato:** André Belice
**Data:** Abril de 2025

---

## 1. Preparação do Ambiente

### 1.1 Acesso ao Servidor

A chave `.pem` fornecida foi utilizada para conexão SSH ao servidor Ubuntu 24.04 disponibilizado (IP: `54.94.13.81`).

> **[INSERIR SCREENSHOT: terminal com comando ssh e conexão estabelecida]**

```bash
ssh -i chave.pem ubuntu@54.94.13.81
```

Após a conexão, o sistema foi atualizado:

```bash
sudo apt update && sudo apt upgrade -y
```

> **[INSERIR SCREENSHOT: saída do apt update/upgrade]**

---

### 1.2 Instalação do Docker

O Docker foi instalado, habilitado para iniciar automaticamente com o sistema e configurado para uso sem `sudo`:

```bash
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu
```

> **[INSERIR SCREENSHOT: docker --version e status do serviço]**

---

## 2. Infraestrutura Jenkins

### 2.1 Problema de Conectividade — Security Group AWS

Ao tentar acessar o Jenkins na porta `8080`, foi identificado que as portas estavam bloqueadas pelo Security Group da instância AWS. A solução adotada foi criar um **tunnel SSH** para mapear a porta remota localmente:

```bash
ssh -i chave.pem -L 8080:localhost:8080 ubuntu@54.94.13.81
```

Com isso, o Jenkins passou a ser acessível em `http://localhost:8080` na máquina local.

> **[INSERIR SCREENSHOT: browser abrindo http://localhost:8080]**

---

### 2.2 Configuração Inicial do Jenkins

O Jenkins foi inicializado via Docker e configurado com os plugins essenciais para o pipeline:

- Pipeline
- Git
- GitHub Integration
- Credentials Binding

Foi criado um usuário administrador para o ambiente de teste:

| Campo | Valor |
|---|---|
| Usuário | `admin` |
| Senha | gerada no primeiro acesso |

> **[INSERIR SCREENSHOT: tela de setup do Jenkins com plugins instalados]**

---

### 2.3 Rede Docker e Agentes

Foi criada uma rede Docker isolada para comunicação entre o controller e os agentes:

```bash
docker network create jenkins-net
```

Dois agentes foram criados no Jenkins (Gerenciar Jenkins → Nodes) e instanciados como contêineres Docker conectados à rede `jenkins-net`:

```bash
# Agent 1 — responsável por build, lint e empacotamento
docker run -d \
  --name agent1 \
  --network jenkins-net \
  -e JENKINS_URL=http://jenkins:8080 \
  -e JENKINS_SECRET=ef8dae552708258fa4368dec2de94d9f3bde461face735c864c6fe8d6cde191d \
  -e JENKINS_AGENT_NAME=agent1 \
  jenkins-agent-build

# Agent 2 — responsável pela execução dos testes unitários
docker run -d \
  --name agent2 \
  --network jenkins-net \
  -e JENKINS_URL=http://jenkins:8080 \
  -e JENKINS_SECRET=fb78a4d985e24b13173a77e461b1316bc0cd4f18387201e5e2a6da5549c37520 \
  -e JENKINS_AGENT_NAME=agent2 \
  jenkins-agent-test
```

> **[INSERIR SCREENSHOT: Jenkins → Nodes com agent1 e agent2 conectados (ícone verde)]**

---

### 2.4 Problema: Servidor Remoto Indisponível

Durante a execução, o servidor de IP `54.94.13.81` parou de responder. Como solução de contorno, a infraestrutura foi reproduzida **localmente em uma instância Ubuntu rodando dentro do Windows (WSL2 ou VM)**, garantindo a continuidade da prova sem dependência do servidor remoto.

> **[INSERIR SCREENSHOT: Ubuntu local com Jenkins e agentes rodando]**

---

## 3. Dockerfiles dos Agentes

Para ter controle preciso sobre as ferramentas disponíveis em cada agente, foram criados Dockerfiles customizados a partir da imagem oficial `jenkins/inbound-agent`.

### Agent Build (`docker/Dockerfile.build`)

Contém as ferramentas necessárias para compilação C++17 e análise estática de código:
- `build-essential` (g++, make)
- `clang-tidy` — análise estática (linter)
- `clang-format` — verificação de formatação

### Agent Test (`docker/Dockerfile.test`)

Contém as ferramentas para execução dos testes unitários com Google Test:
- `build-essential` (g++, make)
- `libgtest-dev` + compilação manual para `/usr/local`

> O Google Test precisa ser **compilado a partir do source** e instalado em `/usr/local` porque o `tests/Makefile` do projeto referencia explicitamente esse caminho.

Os Dockerfiles estão disponíveis em: https://github.com/dehbelicegit/Test_DevOps_Philips/tree/main/docker

> **[INSERIR SCREENSHOT: docker images mostrando jenkins-agent-build e jenkins-agent-test]**

---

## 4. Fork e Configuração do Repositório

O repositório original foi forkado para a conta pessoal do candidato, permitindo commits e configuração de webhooks:

**Fork:** https://github.com/dehbelicegit/Test_DevOps_Philips

O Jenkins foi configurado para usar o repositório forkado como SCM, com o `Jenkinsfile` na raiz do repositório definindo todos os estágios do pipeline.

> **[INSERIR SCREENSHOT: configuração do job no Jenkins apontando para o fork]**

---

## 5. Pipeline CI/CD

### 5.1 Estrutura do Jenkinsfile

O pipeline declarativo implementado cobre todos os estágios exigidos pela prova:

```
Checkout → Check → Build → Test → Package
```

**Gatilhos configurados:**
- **Manual:** qualquer execução via interface do Jenkins
- **Agendado:** diariamente às ~2h (`cron('H 2 * * *')`)

O `Jenkinsfile` completo está disponível em:
https://github.com/dehbelicegit/Test_DevOps_Philips/blob/main/Jenkinsfile

> **[INSERIR SCREENSHOT: visualização do pipeline no Jenkins (Stage View) com todos os estágios verdes]**

---

### 5.2 Debugging e Correções Aplicadas

Durante a execução do pipeline, foram identificados e corrigidos os seguintes problemas:

#### Problema 1 — Sistema de build incorreto (cmake vs make)

O `Jenkinsfile` original chamava `cmake` e `ctest`, mas o projeto utiliza **GNU Make** (sem `CMakeLists.txt`). Os comandos foram corrigidos para `make` e `make unittest`.

#### Problema 2 — Falha no clang-tidy: variáveis não inicializadas

```
error: variable 'number1' is not initialized [cppcoreguidelines-init-variables]
```

**Correção em `calculator/src/main.cpp`:**
```cpp
// Antes
double number1, number2;

// Depois
double number1 = 0.0, number2 = 0.0;
```

> **[INSERIR SCREENSHOT: estágio Check passando após a correção]**

#### Problema 3 — Divisão por zero causando SIGFPE nos testes

O teste `Calculator<int>(0, 0).divide()` causava **Floating Point Exception** porque o método `divide()` não tratava divisor zero.

**Correção em `calculator/src/calculator.hpp`:**
```cpp
// Antes
inline T divide() { return number1 / number2; }

// Depois
inline T divide() {
    if (number2 == T{}) return T{};
    return number1 / number2;
}
```

> **[INSERIR SCREENSHOT: estágio Test com 2 testes passando (PASSED)]**

#### Problema 4 — Binários obsoletos (stale binaries)

Os agentes mantinham binários compilados de execuções anteriores no workspace. O `make` detectava os arquivos como atualizados e não recompilava, executando código antigo.

**Correção no Jenkinsfile:**
```groovy
sh 'make clean && make'                          // Build
sh 'make -C tests clean && make unittest'        // Test
```

#### Problema 5 — Caminho do artefato incorreto

O `tar` apontava para `calculator/src/bin` mas o Makefile gera o binário em `calculator/bin`.

**Correção:**
```groovy
sh 'tar -czf artifact.tar.gz -C calculator/bin .'
```

---

## 6. Resultados

### Pipeline executando com sucesso

> **[INSERIR SCREENSHOT: pipeline completo com todos os 5 estágios verdes]**

### Testes unitários passando

```
[==========] Running 2 tests from 1 test suite.
[ RUN      ] TestCalculator.Integer
[       OK ] TestCalculator.Integer (0 ms)
[ RUN      ] TestCalculator.Double
[       OK ] TestCalculator.Double (0 ms)
[  PASSED  ] 2 tests.
```

> **[INSERIR SCREENSHOT: log do estágio Test com os 2 testes OK]**

### Artefato arquivado

> **[INSERIR SCREENSHOT: página do build no Jenkins com artifact.tar.gz disponível para download]**

---

## 7. Histórico de Commits

Todas as alterações realizadas durante a prova foram commitadas no fork com mensagens descritivas para rastreabilidade:

> **[INSERIR SCREENSHOT: git log ou página de commits no GitHub]**

---

## 8. Referências

- Repositório fork: https://github.com/dehbelicegit/Test_DevOps_Philips
- Jenkinsfile: https://github.com/dehbelicegit/Test_DevOps_Philips/blob/main/Jenkinsfile
- Dockerfiles: https://github.com/dehbelicegit/Test_DevOps_Philips/tree/main/docker
- Documentação da infraestrutura: [INFRA.md](INFRA.md)
