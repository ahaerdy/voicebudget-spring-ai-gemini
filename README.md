# VoiceBudget - Assistente Financeiro por Voz com IA

API inteligente em **Spring Boot + Spring AI** que transforma comandos de voz em transações financeiras, usando o **Google Gemini** (chat, tool calling, transcrição multimodal e síntese de voz via SDK nativo) e persistência real em **MySQL**.

O usuário fala algo como *"Passei na farmácia e deixei 80 reais em três itens"*, a aplicação transcreve o áudio, decide via *tool calling* se deve **registrar** um novo gasto ou **consultar** gastos já existentes, executa a operação real contra o banco de dados, e responde **também em áudio**, com uma frase gerada a partir do resultado real da operação.

Pipeline:

```
🎙️ Áudio  →  📝 Texto (STT)  →  🧠 Decisão + execução real (Tool Calling)  →  📝 Texto  →  🔊 Áudio (TTS)
```

---

## Sumário

- [VoiceBudget - Assistente Financeiro por Voz com IA](#voicebudget---assistente-financeiro-por-voz-com-ia)
  - [Sumário](#sumário)
  - [Visão geral](#visão-geral)
  - [Arquitetura](#arquitetura)
    - [Tool Calling como ponte entre linguagem natural e código real](#tool-calling-como-ponte-entre-linguagem-natural-e-código-real)
  - [Stack técnica](#stack-técnica)
  - [Endpoints da API](#endpoints-da-api)
    - [Fluxo de voz e IA](#fluxo-de-voz-e-ia)
    - [REST tradicional (sem IA)](#rest-tradicional-sem-ia)
  - [Estrutura do projeto](#estrutura-do-projeto)
  - [Como executar](#como-executar)
    - [Pré-requisitos](#pré-requisitos)
    - [1. Configurar a chave de API](#1-configurar-a-chave-de-api)
    - [2. Subir a aplicação](#2-subir-a-aplicação)
    - [3. Testar o fluxo completo de voz](#3-testar-o-fluxo-completo-de-voz)
  - [Configuração](#configuração)
  - [Testes](#testes)
  - [Limitações conhecidas e possíveis evoluções](#limitações-conhecidas-e-possíveis-evoluções)
  - [Referências técnicas](#referências-técnicas)
  - [Autor](#autor)

---

## Visão geral

O projeto nasceu como um exercício de **substituição de provedor de IA**: em vez do stack OpenAI usado como referência, toda a integração é feita nativamente com o **Google Gemini**, via **Spring AI** (para chat e tool calling) e via o **SDK Java nativo do Google GenAI** (para os dois pontos sem abstração equivalente no Spring AI — transcrição de áudio e síntese de voz).

O domínio é propositalmente simples — controle de gastos por categoria —, para manter o foco no que realmente importa: uma aplicação Spring Boot **funcionalmente completa**, com IA generativa e multimodal ligada a **efeitos colaterais reais** (escrita e leitura em banco de dados), não apenas a respostas de texto.

**Capacidades principais:**

- 💬 Conversação com o Gemini via `ChatModel` (baixo nível) e `ChatClient` (API fluente).
- 🛠️ **Tool Calling**: o modelo decide, a partir da linguagem natural, qual operação de negócio executar — e a executa de verdade, não apenas simula.
- 🎙️ **Transcrição de áudio para texto** (Speech-to-Text), multimodal, via `Media` do Spring AI.
- 🔊 **Síntese de texto em áudio** (Text-to-Speech), via SDK nativo do Google GenAI, com conversão de PCM cru para WAV.
- 💾 **Persistência real** das transações em MySQL, orquestrado automaticamente via Docker Compose (sem `docker compose up` manual).
- 🌐 API REST convencional para criação e consulta de transações, independente da camada de voz.
- 🔁 **Fluxo de ponta a ponta**: um único endpoint recebe um áudio e devolve outro áudio, com uma ação de negócio real acontecendo no meio.

---

## Arquitetura

O domínio segue os princípios de **Domain-Driven Design** e **Clean Architecture**, em três camadas com dependência apontando sempre para dentro:

```
┌─────────────────────────────────────────────────────────────────┐
│  infrastructure                                                 │
│  (HTTP, persistência JPA, configuração, SDKs externos)          │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  application                                            │   │
│   │  (casos de uso, DTOs de entrada/saída)                  │   │
│   │                                                         │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │  domain                                         │   │   │
│   │   │  (Transaction, Category, TransactionRepository) │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

- **`domain`** não conhece Spring, JPA, HTTP ou o Gemini. Define apenas o modelo de negócio (`Transaction`, `Category`, `TransactionId`) e o **contrato** de persistência (`TransactionRepository`, uma interface).
- **`application`** contém os **casos de uso** (`PersistTransactionUseCase`, `ListTransactionsByCategoryUseCase`), que dependem apenas da interface `TransactionRepository` — nunca de sua implementação. Cada caso de uso já nasce anotado com `@Tool`, tornando-se, ao mesmo tempo, uma unidade de negócio testável isoladamente **e** uma ferramenta que o Gemini pode decidir invocar.
- **`infrastructure`** é onde os detalhes técnicos vivem: a implementação JPA de `TransactionRepository` (`JpaTransactionRepository` + `TransactionEntity`), os controllers REST, e a configuração de *beans*.

Essa separação é o que permite que **o mesmo caso de uso** (`PersistTransactionUseCase`) seja acionado por dois caminhos completamente diferentes — uma requisição HTTP tradicional (`POST /transactions`) e uma instrução de voz interpretada pelo modelo (`POST /api/ai`) — sem duplicar nenhuma regra de negócio.

### Tool Calling como ponte entre linguagem natural e código real

O mecanismo central do projeto é o **Tool Calling** do Spring AI: métodos Java anotados com `@Tool` são descritos automaticamente ao modelo (nome, parâmetros, tipos), que decide, a partir do prompt do usuário, se e quando invocá-los. A aplicação então executa o método real — banco de dados incluído — e devolve o resultado ao modelo, que o usa para formular a resposta final.

```java
@Tool(name = "persistTransaction", description = "Registra uma nova transação financeira")
public TransactionOutput execute(PersistTransactionInput input) { ... }

@Tool(name = "listTransactionsByCategory", description = "Lista as transações de uma categoria")
public List<TransactionOutput> execute(Category category) { ... }
```

No endpoint de voz (`POST /api/ai`), essas duas *tools* são registradas em um `ChatClient` guiado por um prompt de sistema dedicado, que orienta o modelo a inferir a categoria correta a partir do contexto da fala (ex.: "farmácia" → `PHARMA`) e a escolher a operação certa (registrar vs. consultar) sem que o usuário precise usar nenhum vocabulário técnico.

---

## Stack técnica

| Camada | Tecnologia |
| --- | --- |
| Linguagem / JDK | Java 21 |
| Framework | Spring Boot 4.1.0 |
| Build | Gradle (Groovy DSL) |
| Abstração de IA | Spring AI 2.0.0 |
| Provedor de IA | Google Gemini — `spring-ai-starter-model-google-genai` (chat/tool calling) + SDK nativo `com.google.genai` (STT/TTS) |
| Persistência | Spring Data JPA + Hibernate |
| Banco de dados | MySQL 9.6, orquestrado via Docker Compose (`spring-boot-docker-compose`) |
| Pool de conexões | HikariCP |
| Boilerplate | Lombok (`io.freefair.lombok`) |
| Testes | JUnit 5 (Jupiter), AssertJ, testes parametrizados (`@ParameterizedTest`/`@CsvSource`) |

---

## Endpoints da API

### Fluxo de voz e IA

| Método | Endpoint | Entrada | Saída | Descrição |
| --- | --- | --- | --- | --- |
| `POST` | `/api/ai` | Arquivo de áudio (`multipart/form-data`) | Arquivo de áudio (`audio/wav`) | **Fluxo completo de ponta a ponta.** Transcreve o áudio, decide via Tool Calling se deve registrar ou consultar uma transação, executa a operação real e responde em áudio. |
| `POST` | `/api/transcribe` | Arquivo de áudio (`multipart/form-data`) | Texto puro | Apenas transcrição (Speech-to-Text), sem Tool Calling. |
| `POST` | `/api/synthesize` | JSON `{"text": "..."}` | Arquivo de áudio (`audio/wav`) | Apenas síntese de voz (Text-to-Speech) a partir de um texto arbitrário. |
| `GET` | `/api/{category}` | Path variable (`Category`) | JSON | Consulta de transações por categoria, exposta também sob o prefixo `/api`. |
| `GET` | `/api/chat-model?prompt=...` | Query param | Texto | Chamada crua ao modelo via `ChatModel`, sem contexto persistente. Endpoint de referência/estudo. |
| `GET` | `/api/chat?prompt=...` | Query param | Texto | Chamada ao modelo via `ChatClient` (API fluente). Endpoint de referência/estudo. |

### REST tradicional (sem IA)

| Método | Endpoint | Entrada | Saída | Descrição |
| --- | --- | --- | --- | --- |
| `POST` | `/transactions` | JSON `{"description", "category", "amount"}` | `201 Created` + JSON | Cria uma transação diretamente via HTTP, sem envolvimento do modelo. |
| `GET` | `/transactions/{category}` | Path variable (`Category`) | JSON (lista) | Lista as transações de uma categoria diretamente via HTTP. |

**Categorias suportadas:** `GROCERIES`, `PHARMA`, `AUTO`.

---

## Estrutura do projeto

```
budgeting/
└── src/main/java/dio/budgeting/
    ├── BudgetingApplication.java
    │
    ├── domain/                              # Regras de negócio puras, sem dependência de framework
    │   ├── Transaction.java
    │   ├── TransactionId.java
    │   ├── Category.java
    │   └── TransactionRepository.java        # Interface — o "contrato" de persistência
    │
    ├── application/                          # Casos de uso (também expostos como Tools de IA)
    │   ├── PersistTransactionUseCase.java
    │   ├── ListTransactionsByCategoryUseCase.java
    │   ├── input/PersistTransactionInput.java
    │   └── output/TransactionOutput.java
    │
    ├── infrastructure/
    │   ├── persistence/
    │   │   ├── entity/TransactionEntity.java
    │   │   └── repository/
    │   │       ├── TransactionEntityRepository.java   # Spring Data JPA
    │   │       └── JpaTransactionRepository.java       # implementa TransactionRepository
    │   ├── http/
    │   │   ├── TransactionController.java
    │   │   ├── request/TransactionRequest.java
    │   │   └── response/TransactionResponse.java
    │   └── config/UseCaseConfig.java
    │
    ├── ChatModelController.java              # GET /api/chat-model
    ├── ChatClientController.java             # GET /api/chat
    ├── TranscriptionController.java          # /api/transcribe, /api/{category}, /api/ai
    ├── TextToSpeechController.java           # POST /api/synthesize
    └── TextToSpeechService.java              # encapsula o SDK nativo do Gemini para TTS

budgeting/
├── compose.yml                               # Serviço MySQL, subido automaticamente pelo Spring Boot
└── build.gradle
```

---

## Como executar

### Pré-requisitos

- **JDK 21**
- **Docker** em execução (o Spring Boot sobe o container do MySQL automaticamente via `spring-boot-docker-compose` — não é necessário rodar `docker compose up` manualmente)
- Uma **API key do Google Gemini** (gerada no Google AI Studio)

### 1. Configurar a chave de API

A aplicação lê a chave a partir da variável de ambiente `GEMINI_API_KEY`:

```bash
export GEMINI_API_KEY="sua-chave-aqui"
```

> Se estiver rodando pela IDE, configure a variável na *Run Configuration* correspondente — cada configuração de execução (aplicação, testes) mantém seu próprio conjunto de variáveis de ambiente, de forma independente.

### 2. Subir a aplicação

```bash
cd budgeting
./gradlew bootRun
```

Na primeira subida, o Spring Boot detecta o `compose.yml`, inicia o container do MySQL, aguarda o `healthcheck` responder saudável e só então conclui a inicialização do contexto Spring. O schema do banco é criado/atualizado automaticamente pelo Hibernate (`ddl-auto=update`).

### 3. Testar o fluxo completo de voz

```bash
curl -X POST "http://localhost:8080/api/ai" \
  -F "file=@caminho/para/audio.mp3;type=audio/mpeg" \
  --output resposta.wav
```

Ou, sem áudio, testar diretamente a API REST:

```bash
curl -X POST "http://localhost:8080/transactions" \
  -H "Content-Type: application/json" \
  -d '{"description": "Compras do mês", "category": "GROCERIES", "amount": 125.33}'

curl -X GET "http://localhost:8080/transactions/GROCERIES"
```

---

## Configuração

Principais propriedades em `application.properties`:

```properties
spring.application.name=budgeting

# Provedor de IA — Google Gemini
spring.ai.google.genai.api-key=${GEMINI_API_KEY}
spring.ai.google.genai.chat.options.model=gemini-3-flash-preview
spring.ai.google.genai.chat.options.temperature=0.0

# Persistência
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

logging.level.org.springframework.ai=DEBUG
```

`compose.yml` define o serviço de banco de dados usado pela aplicação:

```yaml
services:
  database:
    image: mysql:9.6
    environment:
      MYSQL_DATABASE: transaction
      MYSQL_USER: app
      MYSQL_PASSWORD: app
      MYSQL_ROOT_PASSWORD: root
    ports:
      - "3307:3306"
    volumes:
      - transaction_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  transaction_data:
```

> A porta local `3307` (em vez da `3306` padrão) evita conflito com uma eventual instalação local do MySQL na máquina de desenvolvimento.

---

## Testes

O projeto usa testes de integração reais contra a API do Gemini (não *mocks*) para validar os pontos de contato com o provedor de IA:

| Classe de teste | Cobre |
| --- | --- |
| `GeminiChatModelIT` | Chamada crua ao `ChatModel`. |
| `GeminiChatClientIT` | `ChatClient` com prompt de sistema e contexto persistente. |
| `ToolCallingIT` | Mecanismo de Tool Calling, isolado, com ferramentas matemáticas didáticas (`sum`, `diff`). |
| `GeminiTranscriptionModelIT` | Transcrição de áudio, parametrizado sobre um conjunto de gravações reais, com asserções tolerantes a variação de formato (ex.: números por extenso vs. algarismos). |
| `GeminiSpeechModelIT` | Síntese de voz, validada por asserção automática (tamanho mínimo do arquivo gerado) e por audição manual. |

```bash
./gradlew test
```

> Testes anotados com `@EnabledIfEnvironmentVariable(named = "GEMINI_API_KEY", ...)` são **pulados silenciosamente** (não falham) se a variável não estiver definida no ambiente usado para rodar o Gradle — o que pode gerar um falso `BUILD SUCCESSFUL` sem que o teste tenha de fato executado. Para confirmar a execução real, use `--tests` com o nome da classe e inspecione a saída de console, ou consulte o relatório em `build/reports/tests/test/index.html`.

Os casos de uso e a camada de persistência/HTTP (a partir da introdução de `TransactionRepository` real) foram validados manualmente, com a aplicação em execução, e não possuem testes de integração automatizados dedicados.

---

## Limitações conhecidas e possíveis evoluções

- `PersistTransactionInput.category` ainda não possui a anotação `@ToolParam` presente nos demais parâmetros de entrada das *tools*.
- O conjunto de categorias (`GROCERIES`, `PHARMA`, `AUTO`) é fixo e reduzido; ampliar o `enum Category` é direto, mas não há endpoint de gerenciamento de categorias.
- Não há validação de entrada (Bean Validation) nos DTOs da camada HTTP.
- As colunas `category` e `description` da entidade JPA não estão marcadas como `NOT NULL` no nível de anotação, embora o comportamento observado na prática seja consistente.
- Não existe um endpoint de consulta que liste **todas** as transações, independente de categoria — hoje, uma consulta genérica ("liste meus gastos") é resolvida pelo modelo chamando a *tool* de listagem uma vez por categoria conhecida.
- `UseCaseConfig` registra `PersistTransactionUseCase` como *bean* explícito, redundante com sua própria anotação `@Service` — mantido por não causar conflito, mas candidato natural a remoção.
- Ausência de testes de integração automatizados para a camada de persistência, os controllers REST e o fluxo completo de voz — hoje validados apenas manualmente.

---

## Referências técnicas

- [Spring Boot — Documentação oficial](https://docs.spring.io/spring-boot/index.html)
- [Spring AI — Documentação oficial](https://docs.spring.io/spring-ai/reference/index.html)
- [Spring AI — Google GenAI Chat Integration](https://docs.spring.io/spring-ai/reference/api/chat/google-genai-chat.html)
- [Spring AI — Tool Calling](https://docs.spring.io/spring-ai/reference/api/tools.html)
- [Spring AI — Multimodalidade](https://docs.spring.io/spring-ai/reference/api/multimodality.html)
- [Google Gemini API — Documentação oficial](https://ai.google.dev/gemini-api/docs)
- [Google Gen AI SDK para Java](https://github.com/googleapis/java-genai)
- [Google Gemini API — Speech Generation (Text-to-Speech)](https://ai.google.dev/gemini-api/docs/speech-generation)
- [Spring Data JPA — Documentação oficial](https://docs.spring.io/spring-data/jpa/reference/index.html)
- [Spring Boot — Docker Compose Support](https://docs.spring.io/spring-boot/reference/features/dev-services.html)
- [MySQL 9.6 — Documentação oficial](https://dev.mysql.com/doc/refman/9.6/en/)
- [Project Lombok](https://projectlombok.org/)
- [JUnit 5 — Documentação oficial](https://junit.org/junit5/docs/current/user-guide/)
- [AssertJ — Documentação oficial](https://assertj.github.io/doc/)
- [Clean Architecture (Robert C. Martin)](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Domain-Driven Design Reference (Eric Evans)](https://www.domainlanguage.com/ddd/reference/)

---

## Autor

- LinkedIn: [linkedin.com/in/arthur-haerdy-jr](https://www.linkedin.com/in/arthur-haerdy-jr/)
- GitHub: [github.com/ahaerdy](https://github.com/ahaerdy)
