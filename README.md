# 🛠️ Configurações Java + Spring Boot + PostgreSQL com OpenAPI — Aplicação Conteinerizada com Docker e Script de Auto-Restart Ready! 💥

Este repositório guarda arquivos de dependência e configuração de aplicação java com persistência em
um banco de dados postgres via Dockerfile e docker-compose.yml

- Configuração do PgAdmin(server.json)
- Configuração do application.properties
- Configuração da imagem com Dockerfile
- Configuração do container com o docker-compose
- Configuração das dependências no arquivo pom.yml

---

## 📁 Estrutura

- Script: `restar-docker.sh`
- Local: `~/bin/restart-docker.sh`

---

## 📜 Conteúdo do Script

```bash
#!/bin/bash

# Cores
GREEN="\e[32m"
YELLOW="\e[33m"
BLUE="\e[34m"
RED="\e[31m"
RESET="\e[0m"

echo -e "${RED}🧨 Parando containers existentes...${RESET}"
docker compose down

echo -e "${BLUE}🚀 Rebuild e inicialização em background...${RESET}"
docker compose up -d --build

echo -e "${GREEN}✅ Containers atualizados e rodando!${RESET}"

echo -e "${YELLOW}📜 Exibindo logs em tempo real da aplicação:${RESET}"
echo -e "${BLUE}(Pressione Ctrl+C para parar de visualizar os logs)\n${RESET}"

sleep 2

docker exec -it todolist-api tail -f /logs/app.log
```

---

## ✅ Como Instalar e Usar

### 1. 📥 Criar o script

- Crie um diretório mkdir bin

```bash
nano ~/restart-docker.sh
```

Cole o conteúdo acima e salve (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### 2. 🔓 Dar permissão de execução

```bash
chmod +x ~/restart-docker.sh
```

---

### 3. ⚙️ Tornar o script global

Edite o arquivo `.zshrc` ou `.bashrc`:

```bash
nano ~/.zshrc
```

Adicione esta linha ao final:

```bash
alias restart-docker="~/bin/restart-docker.sh"
```

Salve e recarregue o terminal:

```bash
source ~/.zshrc
```

---

### 4. 🚀 Usar o comando

Navegue até o diretório do seu projeto e rode:

```bash
restart-docker
```

---

## 🧠 Dica

Você pode personalizar este script para incluir validações, log de histórico, commits convencionais, entre outros.

---

## 📌 Requisitos

- Ambiente Linux (Ubuntu, Kali, Arch, etc...)
- Docker e Docker compose instalado

---

## Configuração do PgAdmin(server.json)

```bash
{
  "Servers": {
    "1": {
      "Name": "Local PostgreSQL",
      "Group": "Servers",
      "Host": "db",
      "Port": 5432,
      "Username": "docker",
      "SSLMode": "prefer",
      "MaintenanceDB": "goal_track_db"
    }
  }
}
```

## Configuração do application.properties

```bash
# Datasource
spring.datasource.url=${SPRING_DATASOURCE_URL}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}

# Hibernate (ajuste se desejar gerar automaticamente a estrutura do banco)
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Swagger (opcional, caso use springdoc-openapi)
springdoc.api-docs.enabled=true
springdoc.swagger-ui.enabled=true
```

## Configuração da imagem com Dockerfile

```bash
# Etapa 1: Build da aplicação com cache de dependências
FROM maven:3.9.6-eclipse-temurin-17 AS builder

# Define diretório de trabalho
WORKDIR /app

# Copia apenas os arquivos de configuração do Maven (para cache de dependências)
COPY pom.xml .
COPY mvnw .
COPY .mvn .mvn

# Faz o download das dependências antes de copiar o código (melhor cache)
RUN ./mvnw dependency:go-offline -B

# Agora copia o restante do projeto
COPY . .

# Compila o projeto e gera o JAR (sem rodar os testes)
RUN ./mvnw clean package -DskipTests

# Etapa 2: Imagem final com o JAR
FROM eclipse-temurin:17-jdk

WORKDIR /app

# Copia o JAR gerado da etapa anterior
COPY --from=builder /app/target/*.jar app.jar

# Exponha a porta padrão do Spring Boot
EXPOSE 8080

# Comando de inicialização
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Configuração do container com o docker-compose

```bash
version: "3.8"

services:
  app:
    build: .
    container_name: spring-app
    env_file:
      - .env
    depends_on:
      - db
    ports:
      - "8080:8080"
    networks:
      - backend
    environment:
      SPRING_DATASOURCE_URL: ${SPRING_DATASOURCE_URL}
      SPRING_DATASOURCE_USERNAME: ${SPRING_DATASOURCE_USERNAME}
      SPRING_DATASOURCE_PASSWORD: ${SPRING_DATASOURCE_PASSWORD}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  db:
    image: postgres:15
    container_name: postgres-db
    restart: always
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    env_file:
      - .env
    environment:
      POSTGRES_USER: ${SPRING_DATASOURCE_USERNAME}
      POSTGRES_PASSWORD: ${SPRING_DATASOURCE_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  swagger-ui:
    image: swaggerapi/swagger-ui
    container_name: swagger-ui
    restart: always
    ports:
      - "9000:8080"
    environment:
      SWAGGER_JSON: /app/swagger.yaml
    volumes:
      - ./docs/swagger.yaml:/app/swagger.yaml
    networks:
      - backend

  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin
    restart: always
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_DEFAULT_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_DEFAULT_PASSWORD}
    depends_on:
      - db
    volumes:
      - pgadmin-data:/var/lib/pgadmin
      - ./pgadmin/servers.json:/pgadmin4/servers.json
    networks:
      - backend

volumes:
  pgdata:
  pgadmin-data:

networks:
  backend:

```

## Configuração das dependências no arquivo pom.yml

```bash
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.0</version>
        <relativePath/> <!-- Busca no repositório -->
    </parent>

    <groupId>br.com.cardosofiles</groupId>
    <artifactId>vacancy-management-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>vacancy-management-server</name>
    <description>Spring Boot REST API com PostgreSQL, JWT e Swagger</description>

    <properties>
        <java.version>17</java.version>
        <jjwt.version>0.11.5</jjwt.version>
    </properties>

    <dependencies>

        <!-- REST e JSON -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- JPA + PostgreSQL -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>

        <!-- Validação (JSR-380) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Swagger / OpenAPI -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.5.0</version>
        </dependency>

        <!-- Segurança com Spring Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- JWT Auth -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId> <!-- usa Jackson para serializar -->
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>

        <!-- Lombok (boas práticas, menos boilerplate) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.38</version>
            <scope>provided</scope>
        </dependency>

        <!-- Spring Boot DevTools (dev only) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <!-- Monitoramento (Actuator) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Testes -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>br.com.cardosofiles.Application</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>

```

## 💻 Observação

- Será necessário criar o arquivo `.env` com as variáveis ambiente.

---

# Annotations Fundamentais para Projetos Java Escaláveis

## 1. 📦 Spring Boot e Spring Framework

🔹 `@SpringBootApplication`

- Descrição: Combina três anotações: `@Configuration`, `@EnableAutoConfiguration` e `@ComponentScan`.
- Uso: Classe principal da aplicação Spring Boot.
- Exemplo:

```bash
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

🔹 `@Configuration`

- Descrição: Define uma classe como fonte de configurações Spring Beans.
- Boa prática: Use para agrupar configurações complexas de beans (ex: Beans de segurança ou integração).

---

🔹 `@ComponentScan`

- Descrição: Especifica os pacotes a serem escaneados por componentes (beans).
- Exemplo:

```bash
@ComponentScan(basePackages = "com.example.myapp")
```

---

🔹 `@Bean`

- Descrição: Define um método que instancia, configura e inicializa um objeto a ser gerenciado pelo Spring.
- Contexto comum: Configuração de beans externos (ex: ModelMapper, RestTemplate).
- Exemplo:

```bash
@Bean
public ModelMapper modelMapper() {
    return new ModelMapper();
}
```

## 2. 👤 Spring Core / DI

🔹 `@Component`

- Descrição: Marca uma classe como um bean genérico.
- Nota: Classes anotadas com @Component são detectadas automaticamente via `@ComponentScan`.

---

🔹 `@Service`

- Descrição: Específico para classes que representam a lógica de negócio (serviços).
- Semântica: Mesmo comportamento que `@Component`, mas semanticamente distinto.

---

🔹 `@Repository`

- Descrição: Marca classes de persistência de dados (DAO).
- Funcionalidade extra: Traduz exceções de persistência para `DataAccessException`.

---

🔹 `@Controller`

- Descrição: Define um controlador Web MVC.
- Nota: Retorna views (não JSON por padrão).

---

🔹 `@RestController`

- Descrição: Combina `@Controller` + `@ResponseBody`. Ideal para APIs REST.
- Boa prática: Use em aplicações que expõem JSON/REST.

---

🔹 @Autowired

- Descrição: Injeção automática de dependências.
- Importante: A partir do Spring 4.3+, pode-se usar construtores com `@Autowired` opcionalmente omitido.
- Recomendação: Prefira constructor injection.

---

🔹 `@Qualifier`

- Descrição: Diferencia beans do mesmo tipo.
- Exemplo:

```bash
@Autowired
@Qualifier("emailService")
private NotificationService service;
```

## 🛠️ 3. Spring Web / REST

🔹 `@RequestMapping`

- Descrição: Mapeia requisições HTTP em classes ou métodos.
- Exemplo:

```bash
@RequestMapping("/api/v1/path")
```

---

🔹 `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`

- Descrição: Atalhos para `@RequestMapping(method = ...)`.

---

🔹 `@PathVariable`

- Descrição: Captura parâmetros da URL.

```bash
@GetMapping("/{id}")
public Candidate findById(@PathVariable Long id) {}
```

---

🔹 `@RequestParam`

- Descrição: Captura parâmetros de query string.

```bash
@GetMapping("/search")
public List<Candidate> search(@RequestParam String name) {}
```

---

🔹 `@RequestBody`

- Descrição: Converte JSON do corpo da requisição para objeto Java.

---

🔹 `@ResponseBody`

- Descrição: Converte objetos Java em JSON na resposta.

---

🔹 `@ExceptionHandler`

- Descrição: Trata exceções de forma customizada no controller.

## 🧩 4. JPA / Hibernate (Persistência)

🔹 `@Entity`

- Descrição: Marca uma classe como entidade persistente.

---

🔹 `@Table(name = "nome_tabela")`

- Descrição: Define a tabela no banco de dados.

---

🔹 @Id

- Descrição: Define o campo primário da entidade.

---

🔹 `@GeneratedValue`

- Descrição: Gera valores automaticamente para a @Id.

  - GenerationType.IDENTITY

  - GenerationType.SEQUENCE

  - GenerationType.AUTO

  - GenerationType.TABLE

---

🔹 `@Column`

- Descrição: Configura propriedades da coluna.
- Exemplo:

```bash
@Column(name = "email", nullable = false, unique = true)
```

---

🔹 `@OneToMany, @ManyToOne, @OneToOne, @ManyToMany`

- Descrição: Define relacionamentos entre entidades.

- #### `@ManyToOne`

  - 📌 Significado:
  - Muitos objetos de uma entidade se relacionam com um único objeto de outra entidade.
  - É o lado "muitos" de um relacionamento N:1.
  - 📚 Exemplo Clássico:

  ```bash
  @Entity

    public class Candidate {

    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    @JoinColumn(name = "vacancy_id") // nome da FK na tabela Candidate
    private Vacancy vacancy;

    }
  ```

- #### `@ManyToMany`

  - 📌 Significado:
  - Muitos para muitos: N entidades estão relacionadas a N outras.
  - 📚 Exemplo Clássico:

  ```bash
  @Entity
  public class Candidate {

    @Id @GeneratedValue
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "candidate_skill",
        joinColumns = @JoinColumn(name = "candidate_id"),
        inverseJoinColumns = @JoinColumn(name = "skill_id")
    )
    private Set<Skill> skills = new HashSet<>();

  }
  ```

- #### `@OneToMany`

  - 📌 Significado:
  - Um objeto se relaciona com muitos de outra entidade.
  - Inverso do `@ManyToOne`.
  - 📚 Exemplo Clássico:

  ```bash
  @Entity
  public class Vacancy {

    @Id @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "vacancy", cascade = CascadeType.ALL)
    private List<Candidate> candidates = new ArrayList<>();

  }
  ```

---

🔹 `@JoinColumn`

- Descrição: Customiza a coluna de junção em relacionamentos.

---

🔹 `@Embedded, @Embeddable`

- Descrição: Utilizadas para componentes reutilizáveis dentro da entidade.

## 💡 5. Lombok (Redução de Boilerplate)

🔹 `@Data`

- Descrição: Gera getters, setters, toString, equals e hashCode.
- Atenção: Não recomendado para entidades JPA com relacionamentos bidirecionais.

---

🔹 `@Getter, @Setter`

- Descrição: Gera os respectivos métodos de acesso.

---

🔹 `@ToString, @EqualsAndHashCode`

- Descrição: Define como os métodos toString e equals/hashCode são gerados.

---

🔹 `@NoArgsConstructor, @AllArgsConstructor, @RequiredArgsConstructor`

- Descrição: Gera construtores automaticamente.

---

🔹 `@Builder`

- Descrição: Gera um builder pattern para criação fluente de objetos.

```bash
Candidate c = Candidate.builder().name("João").email("joao@example.com").build();
```

## 📋 6. Validação (Spring Validation + Jakarta Bean Validation)

🔹 `@Valid`

- Descrição: Ativa a validação do bean.

```bash
public ResponseEntity<?> save(@Valid @RequestBody UserDTO dto) {}
```

---

🔹 `@NotNull, @NotEmpty, @Size, @Email, @Min, @Max`

- Descrição: Validações declarativas em atributos.

## 📡 7. Miscellaneous

🔹 `@Profile`

- Descrição: Define o ambiente onde o bean será ativado (ex: dev, prod).

```bash
@Profile("dev")
```

---

🔹 `@Transactional`

- Descrição: Gerencia transações (início, commit e rollback).
- Boa prática: Use em métodos de serviço que interagem com o repositório.

## 🧠 Extras e Recomendados

Annotation Categoria Finalidade
`@EnableJpaRepositories` Spring Data Ativa repositórios Spring JPA
`@EntityListeners(AuditingEntityListener.class)` JPA Permite auditoria de campos (createdAt/updatedAt)
`@CreatedDate`, `@LastModifiedDate` JPA Auditar timestamps com Spring Data
`@RestControllerAdvice` Spring Web Trata erros globais de APIs

| Annotation                                       | Categoria   | Finalidade                                                                      |
| ------------------------------------------------ | ----------- | ------------------------------------------------------------------------------- |
| `@EnableJpaRepositories`                         | Spring Data | Ativa repositórios Spring JPA                                                   |
| `@EntityListeners(AuditingEntityListener.class)` | JPA         | Permite auditoria de campos (createdAt/updatedAt)                               |
| `@CreatedDate`, `@LastModifiedDate`              | JPA         | Auditar timestamps com Spring Data                                              |
| `@RestControllerAdvice`                          | Spring Web  | Trata erros globais de APIs                                                     |
| `@Slf4j`                                         | Lombok      | Gera um logger `private static final Logger log = LoggerFactory.getLogger(...)` |

---

# 🏦 Guia Java para Desenvolvimento de Sistemas Bancários

Este documento reúne as **principais classes, anotações e APIs do ecossistema Java** utilizadas em sistemas financeiros robustos, como bancos, fintechs e sistemas de crédito.

> 📘 Ideal para projetos com **Spring Boot, JPA, segurança, precisão financeira, autenticação e persistência de dados**.

---

## 📅 1. Manipulação de Datas e Tempo

| Classe              | Uso Comum                                          |
| ------------------- | -------------------------------------------------- |
| `LocalDate`         | Datas sem hora (ex: nascimento, vencimento)        |
| `LocalDateTime`     | Data e hora locais                                 |
| `ZonedDateTime`     | Datas com fuso horário                             |
| `Instant`           | Ponto fixo na linha do tempo (timestamp em UTC)    |
| `Period`            | Diferença entre datas em **anos/meses/dias**       |
| `Duration`          | Diferença entre horários em **segundos/minutos**   |
| `DateTimeFormatter` | Conversão entre texto e datas (formatação/parsing) |

---

## 💰 2. Tipos Numéricos Precisos

| Classe         | Uso Comum                                             |
| -------------- | ----------------------------------------------------- |
| `BigDecimal`   | Cálculos monetários com precisão exata (juros, saldo) |
| `NumberFormat` | Formatação de valores monetários                      |

> ⚠️ Nunca use `double` ou `float` para dinheiro.

---

## 📦 3. Coleções (Collections API)

| Classe/Interface    | Finalidade                                  |
| ------------------- | ------------------------------------------- |
| `List`, `ArrayList` | Lista ordenada (transações, clientes)       |
| `Map`, `HashMap`    | Dicionários (por CPF, número da conta)      |
| `Set`, `HashSet`    | Conjuntos únicos (ex: CPFs sem duplicidade) |
| `LinkedHashMap`     | Ordem de inserção (extratos cronológicos)   |
| `TreeMap`           | Ordenado por chave                          |

---

## 🔐 4. Segurança e Criptografia

| Classe/API              | Finalidade                                      |
| ----------------------- | ----------------------------------------------- |
| `MessageDigest`         | Geração de hash (ex: SHA-256 para senhas)       |
| `SecureRandom`          | Geração de tokens seguros                       |
| `javax.crypto.*`        | Criptografia simétrica e assimétrica (AES, RSA) |
| `BCryptPasswordEncoder` | Hash seguro de senha com salt                   |
| `JWT`                   | Autenticação via tokens                         |

---

## 🛠️ 5. Utilitários e Validação

| Classe/Anotação               | Uso                                           |
| ----------------------------- | --------------------------------------------- |
| `Objects.requireNonNull()`    | Evita NullPointerException                    |
| `Pattern`, `Matcher`          | Regex para CPF, e-mails                       |
| `@NotNull`, `@Email`, `@Size` | Validações com `javax.validation.constraints` |
| `@CPF`                        | Validação de CPF (via Hibernate Validator)    |

---

## 🧵 6. Concorrência

| Classe/API                 | Finalidade                               |
| -------------------------- | ---------------------------------------- |
| `ExecutorService`          | Execução assíncrona com controle de pool |
| `CompletableFuture`        | Concorrência moderna e reativa           |
| `ScheduledExecutorService` | Agendamento de tarefas                   |
| `ReentrantLock`            | Controle explícito de acesso concorrente |

---

## 🧾 7. Persistência com JPA e Spring Data

| Anotação                        | Finalidade                                |
| ------------------------------- | ----------------------------------------- |
| `@Entity`, `@Table`             | Representa tabela no banco                |
| `@Id`, `@GeneratedValue`        | Identificador e geração automática        |
| `@OneToMany`, `@ManyToOne`, etc | Relacionamentos entre entidades           |
| `JpaRepository<T, ID>`          | Repositórios prontos para acesso ao banco |
| `@Query`                        | Queries customizadas em JPQL              |
| `EntityManager`                 | Acesso manual ao contexto de persistência |

---

## 🔄 8. Serialização e Conversão

| Classe/API               | Uso                                 |
| ------------------------ | ----------------------------------- |
| `ObjectMapper` (Jackson) | Conversão entre JSON e objetos Java |
| `@JsonProperty`          | Personaliza nome de campo JSON      |
| `@JsonIgnore`            | Ignora campos na serialização       |
| `ModelMapper`            | Conversão entre DTOs e Entidades    |

---

## 📊 9. Logging e Observabilidade

| Ferramenta/Classe           | Finalidade                              |
| --------------------------- | --------------------------------------- |
| `@Slf4j` (Lombok)           | Geração automática de logger            |
| `LoggerFactory.getLogger()` | Logger manual via SLF4J                 |
| `Logback`, `Log4j2`         | Frameworks de logging configuráveis     |
| `Micrometer` + `Prometheus` | Métricas para monitoramento em produção |

---

## 🧪 10. Testes Automatizados

| Framework/Classe  | Finalidade                                |
| ----------------- | ----------------------------------------- |
| `JUnit 5`         | Testes unitários                          |
| `Mockito`         | Mock de dependências                      |
| `Testcontainers`  | Integração com bancos em containers reais |
| `@SpringBootTest` | Testes com contexto Spring                |
| `@DataJpaTest`    | Testes isolados de repositórios JPA       |

---

## ✅ Recomendação Final

Use esse guia como **referência técnica para arquitetura, testes, segurança e persistência** em sistemas bancários modernos baseados em Java + Spring Boot.

> Para projetos reais, utilize também:
>
> - Docker + Docker Compose
> - Spring Boot Actuator
> - Spring Security com JWT
> - Auditoria com Envers ou listeners

---

## 📚 Créditos

Elaborado por um estudante de Java e engenharia de software com foco em sistemas bancários, back-end e alta escalabilidade.

Se você gostou deste guia ou quer contribuir com mais conteúdo voltado a bancos, pagamentos, crédito ou microserviços Java, fique à vontade para abrir uma issue ou pull request.

---

📌 **Contato e Contribuição**

- ✉️ Email: cardosofiles@outlook.com
- 💻 GitHub: [Cardosofiles](https://github.com/Cardosofiles)
- 🌐 Site pessoal [MinimalistPortfolio](https://minimalist-portfolio-joao-batista-snowy.vercel.app/)
- ℹ️ LinkedIn [Perfil](https://www.linkedin.com/in/joaobatista-dev/)

  > “Software bancário exige precisão, segurança e desempenho. A base disso é um domínio sólido da linguagem, arquitetura e boas práticas.”  
  > — Estudante. Cardoso, Eng. de Software

---
