# Bairro Vivo — DevOps Tools & Cloud Computing (Global Solution)

Repositório da entrega de **DevOps Tools & Cloud Computing** da Global Solution: **pipelines de CI/CD no Azure DevOps** com deploy automático da API Java em um **Azure Web App**, usando **Oracle em nuvem** e **RabbitMQ em container (Azure Container Instances)**, com a infraestrutura criada por **script Azure CLI**.

---

## 1. Solução proposta

O **Bairro Vivo** é uma plataforma de zeladoria colaborativa, no tema **CitizenTech**: o cidadão registra problemas do seu bairro (ocorrências) e acompanha o que acontece com eles. O problema que a solução ataca é prático: muitas vezes a pessoa registra a ocorrência, mas **não sabe a quem recorrer** nem qual é o canal oficial para aquele tipo de problema.

Esta entrega trata da **API Java** do ecossistema, responsável por:

- cadastro, autenticação e **JWT** (RS256);
- preferências e **locais favoritos** do cidadão;
- **orientação cidadã com IA generativa (RAG)**: a partir de uma ocorrência, indica o canal oficial adequado, usando somente fontes aprovadas.

As ocorrências, votos, fotos, histórico e relatórios pertencem à **API .NET** do grupo, publicada à parte. As duas APIs compartilham o mesmo banco Oracle.

### Stack de tecnologias

| Camada | Tecnologia |
|---|---|
| Linguagem | Java 17 |
| Framework | Spring Boot 3.4.4 (Web, Data JPA, Security, OAuth2 Resource Server, AMQP, Actuator) |
| IA generativa | Spring AI com modelo OpenAI (`gpt-4o-mini`), com RAG sobre fontes oficiais |
| Documentação da API | springdoc-openapi (Swagger UI) |
| Mensageria | RabbitMQ (evento `usuario.cadastrado`) |
| Banco de dados em nuvem | Oracle (instância da FIAP), acesso via JDBC |
| Build e testes | Maven (JUnit, H2 nos testes) |
| Código-fonte | Azure Repos |
| Planejamento | Azure Boards |
| CI/CD | Azure Pipelines (YAML) |
| Agente de build | Self-hosted (pool `Default`) |
| Hospedagem da API | Azure Web App (App Service, Linux B1) |
| Hospedagem do RabbitMQ | Azure Container Instances (imagem `rabbitmq:3-management`) |
| Infraestrutura como código | Script Azure CLI (`scripts/`) |

## 2. Conceito: como a orientação com IA funciona (RAG)

**RAG** (*Retrieval-Augmented Generation*) é a técnica em que o modelo **consulta uma base de documentos confiáveis antes de responder**, em vez de responder só com o que aprendeu no treinamento. Isso reduz alucinações, permite atualizar o conhecimento sem retreinar o modelo e deixa a resposta rastreável até uma fonte. Para um cidadão que vai ligar para um órgão público, um telefone inventado seria um problema real, e é isso que o RAG evita.

Fluxo de uma orientação (`POST /api/v1/ia/orientacoes`):

1. O cliente envia só o `ocorrenciaId`.
2. A API lê a ocorrência, a categoria e o município no Oracle (sem dados pessoais do autor).
3. **Recuperação:** busca até 5 trechos em uma view de fontes oficiais aprovadas e os canais verificados.
4. **Portão de elegibilidade:** o modelo só é chamado se município e categoria tiverem fontes aprovadas.
5. **Geração:** o Spring AI recebe a ocorrência e os trechos. A instrução trata a descrição como dado não confiável, bloqueia *prompt injection* e proíbe inventar contatos e horários.
6. **Validação da saída:** a resposta é rejeitada se contiver contato, horário ou URL que não esteja nas fontes aprovadas.
7. A orientação e as fontes são gravadas no Oracle, e somente orientações publicadas ficam visíveis ao público.

Limitações assumidas: escopo do município de São Paulo, dependência da qualidade das fontes aprovadas e limite de 5 solicitações por minuto por usuário.

## 3. Benefícios para o negócio

- O cidadão recebe o **canal oficial correto** para o seu problema, o que aumenta a chance de a ocorrência ser resolvida.
- A resposta é baseada em **fontes aprovadas e validada**, reduzindo o risco de informação errada em serviços públicos.
- O cadastro publica eventos no RabbitMQ, deixando a plataforma pronta para integrações assíncronas (notificações, auditoria).
- A esteira de CI/CD elimina o deploy manual: todo código aprovado é compilado, testado e publicado de forma padronizada e repetível.
- Segredos e credenciais ficam fora do código-fonte.

## 4. Arquitetura e fluxo CI/CD

![Diagrama da arquitetura e do fluxo CI/CD do Bairro Vivo](./images/diagrama-arquitetura.png)

### Fluxo numerado

| # | Etapa | O que acontece |
|---|---|---|
| 1 | **Azure Boards** | Cada mudança começa numa Tarefa. A branch, os commits (`#ID`) e o Pull Request são vinculados a ela. |
| 2 | **Azure Repos** | O desenvolvedor cria a branch, faz o commit e abre o PR para a `master`. A branch é protegida por *branch policy* (revisor e work item obrigatórios). |
| 3 | **Azure Pipelines (CI)** | Após o merge na `master`, a pipeline compila com Maven, executa os testes, publica os resultados JUnit e gera o artefato `drop` (`.jar`). |
| 4 | **Azure Pipelines (CD)** | Disparada após o CI gerar o artefato, faz o deploy automático no Web App `bairro-vivo-java-566385`. |
| 5 | **Execução em produção** | O Web App recebe as variáveis de ambiente, conecta ao Oracle e ao RabbitMQ e expõe a API (Swagger em `/swagger-ui.html`). |

### Onde cada ferramenta atua

| Ferramenta | Papel na solução |
|---|---|
| Azure Boards | Planejamento e rastreabilidade das tarefas. |
| Azure Repos | Hospeda o código e controla branches e Pull Requests. |
| Azure Pipelines | Executa o CI (build e testes) e o CD (deploy). |
| Agente self-hosted (pool `Default`) | Máquina que executa os jobs das pipelines. |
| Azure Web App (Linux B1) | Executa a API Java em produção. |
| Azure Container Instances | Executa o RabbitMQ em container. |
| Oracle (FIAP) | Persiste os dados, compartilhado com a API .NET. |
| API .NET (Render) | Gerencia ocorrências, votos, fotos e relatórios. |
| OpenAI (Spring AI) | Gera o texto das orientações, a partir do contexto recuperado. |

### Personas

- **Desenvolvedor:** cria a Tarefa, a branch e o PR, o que aciona o fluxo.
- **Cidadão:** acessa a API publicada no Web App e solicita orientações.

## 5. Banco de dados em nuvem

- Banco: **Oracle (instância da FIAP)**, acessado por JDBC.
- O esquema é fornecido pelo grupo e **não é criado pela aplicação** (`ddl-auto=none`). O DDL está em `scripts/script-bd.sql`.
- URL, usuário e senha **não ficam no código**: são variáveis de ambiente protegidas (`ORACLE_URL`, `ORACLE_USER`, `ORACLE_PASSWORD`).

## 6. Infraestrutura via Azure CLI

Os recursos são criados pelo script em `scripts/`, sem uso do portal:

| Recurso | Nome |
|---|---|
| Resource Group | `rg-bairro-vivo` (região `brazilsouth`) |
| App Service Plan | `plan-bairro-vivo` (Linux, B1) |
| Web App | `bairro-vivo-java-566385` (runtime Java 17) |
| Container Instance (RabbitMQ) | `rabbitmq-bairro-vivo` |

O script também grava as variáveis de ambiente no Web App. Credenciais são lidas do ambiente, nunca escritas no arquivo.

## 7. Projeto no Azure DevOps

| Configuração | Valor |
|---|---|
| Organização / Projeto | `RM566385` / `Bairro Vivo` |
| Visibility | Private |
| Version control | Git |
| Acesso do professor | Convidado com nível **Basic** na organização e **Project Administrators** no projeto |

**Boards:** uma Tarefa inicial, com branch, commits e Pull Request vinculados.

**Branch policy na `master`:** mínimo de 1 revisor, vínculo obrigatório com work item e revisor incluído automaticamente. Não é possível commitar direto na branch principal.

## 8. Pipelines CI/CD

O arquivo `azure-pipeline.yml` (raiz do repositório) define duas etapas:

| Requisito | Como é atendido |
|---|---|
| Pipeline conectada ao repositório | Azure Repos, branch `master`. |
| CI disparado por alteração | Trigger na `master`; com a branch protegida, só o merge de PR dispara o build. |
| CD após novo artefato | A etapa `Release` depende do sucesso do `Build` e roda automaticamente. |
| Artefato publicado | O `.jar` é publicado como artefato `drop`. |
| Testes no CI | `mvn clean package` executa os testes e publica os resultados JUnit. |
| Deploy na nuvem | Task `AzureWebApp@1` no Web App Linux. |
| Variáveis sensíveis protegidas | Senhas, tokens e chaves em variáveis secretas da pipeline (nunca no código). |

### Variáveis de ambiente da aplicação

`ORACLE_URL`, `ORACLE_USER`, `ORACLE_PASSWORD`, `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_TTL`, `JWT_PRIVATE_KEY_PATH`, `JWT_PUBLIC_KEY_PATH`, `RABBITMQ_HOST`, `RABBITMQ_PORT`, `RABBITMQ_USER`, `RABBITMQ_PASSWORD`, `DOTNET_API_BASE_URL`, `DOTNET_OCCURRENCE_PATH`, `DOTNET_SERVICE_TOKEN`, `BAIRROVIVO_AI_ENABLED`, `OPENAI_API_KEY`, `OPENAI_MODEL`.

Somente os **nomes** ficam documentados; os valores existem apenas nas variáveis da pipeline e nos *App settings* do Web App.

## 9. Como executar localmente

```bash
docker compose up -d rabbitmq
./mvnw spring-boot:run
./mvnw test
```

Pré-requisitos: JDK 17, Docker, Oracle com o esquema do projeto e chaves RSA em `.secrets/` (fora do Git). Swagger UI em `http://localhost:8080/swagger-ui.html`.

## 10. Roteiro demonstrado no vídeo

1. Ferramentas: IDE, Azure DevOps, banco de dados e Portal do Azure.
2. Configuração das pipelines (jobs e tasks).
3. Recursos criados pelo script Azure CLI no Portal do Azure.
4. Nova Tarefa no Boards e nova branch.
5. Alteração visível no código da aplicação.
6. Commit e PR vinculados à Tarefa, e merge na `master`.
7. Início automático da pipeline e explicação de cada etapa do CI/CD.
8. Artefato criado e resultado dos testes.
9. Recursos atualizados pelo deploy e aplicação no ar com a alteração.
10. CRUD completo em duas tabelas, com `SELECT` direto no banco após cada operação.
11. Tarefa concluída, com os links de branch, commits e PR.

## 11. Integrantes

| RM | Nome |
|---|---|
| 566385 | Diego Andrade dos Santos |
| _RM_ | _Nome_ |
| _RM_ | _Nome_ |

## Links da entrega

- Projeto no Azure DevOps: https://dev.azure.com/RM566385/Bairro%20Vivo
- API em produção (Swagger): https://bairro-vivo-java-566385.azurewebsites.net/swagger-ui.html
- Vídeo demonstrativo (YouTube): _link_
