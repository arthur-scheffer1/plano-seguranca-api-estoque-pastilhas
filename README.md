# 🔐 Matriz e Plano de Segurança da API — Estoque de Pastilhas

**API de Controle de Estoque de Pastilhas — DDA Metalúrgica**

| | |
|---|---|
| **Unidade Curricular** | Desenvolvimento de Sistemas Web — UniSenai |
| **Aluno** | Scheffer |
| **Tecnologias** | Java 21, Spring Boot (Spring Web, Spring Data JPA, Spring Security), PostgreSQL, Flyway |
| **Data** | Outubro de 2026 |

> 📄 Versão em Word: [`Plano_Seguranca_API_Estoque_Pastilhas.docx`](Plano_Seguranca_API_Estoque_Pastilhas.docx)

## Sumário

- [Contexto da API](#contexto-da-api)
- [1. Matriz de Segurança](#1-matriz-de-segurança)
- [2. Plano de Segurança da API](#2-plano-de-segurança-da-api)
- [3. Perfis e Permissões de Acesso](#3-perfis-e-permissões-de-acesso)
- [4. Relação com o conteúdo das aulas](#4-relação-com-o-conteúdo-das-aulas)

---

## Contexto da API

A API controla o estoque de pastilhas de usinagem da DDA Metalúrgica: cadastro de pastilhas e fornecedores, registro de entradas e saídas e consulta de itens com estoque baixo. É um monólito Spring Boot organizado em camadas (**Controller → Service → Repository → Domain**), com dados persistidos no PostgreSQL por meio do Spring Data JPA e esquema versionado com Flyway.

Como os dados são internos da empresa e cada movimentação altera o saldo real de ferramentas, a API exige autenticação em quase todos os recursos e registra quem fez cada operação.

### Contrato HTTP considerado

| Método e caminho | Operação | Status esperados |
|---|---|---|
| `POST /auth/login` | Autenticar e obter token | 200 / 401 |
| `GET /pastilhas?page=&size=&sort=` | Listar pastilhas (paginado) | 200 / 401 |
| `GET /pastilhas/{id}` | Consultar pastilha | 200 / 404 |
| `GET /pastilhas/pesquisa?codigo=` | Pesquisar por código/descrição | 200 |
| `GET /pastilhas/estoque-baixo` | Pastilhas abaixo do estoque mínimo | 200 |
| `POST /pastilhas` | Cadastrar pastilha | 201 / 400 / 403 |
| `PUT /pastilhas/{id}` | Atualizar pastilha | 200 / 403 / 404 |
| `DELETE /pastilhas/{id}` | Remover pastilha | 204 / 403 / 404 |
| `GET, POST, PUT, DELETE /fornecedores[/{id}]` | CRUD de fornecedores | 200 / 201 / 204 / 403 / 404 |
| `GET /movimentacoes` | Histórico de entradas e saídas | 200 |
| `POST /movimentacoes` | Registrar entrada ou saída | 201 / 400 / 422 |
| `POST /movimentacoes/{id}/estorno` | Estornar movimentação | 201 / 403 / 404 |
| `GET, POST, PUT, DELETE /usuarios[/{id}]` | Gerenciar usuários | 200 / 201 / 204 / 403 / 404 |
| `GET, PUT /usuarios/me` | Ver dados e trocar a própria senha | 200 / 401 |
| `GET /actuator/health` | Verificar se a API está no ar | 200 |

---

## 1. Matriz de Segurança

A matriz relaciona cada recurso ou operação da API às ameaças pertinentes, ao impacto possível e ao controle preventivo que será implementado.

| # | Recurso / Operação | Ameaça ou vulnerabilidade | Possível impacto | Controle preventivo |
|---|---|---|---|---|
| 1 | POST /auth/login | Tentativas repetidas de senha (força bruta) | Acesso indevido a uma conta e ao estoque | Senhas com hash BCrypt; bloqueio temporário após 5 falhas seguidas (contador na entidade Usuario); resposta 401 sem detalhes |
| 2 | POST /auth/login | Descobrir quais usuários existem pela mensagem de erro | Facilita ataques direcionados | Mesma resposta (401 "Usuário ou senha inválidos") para usuário inexistente ou senha errada |
| 3 | Header Authorization (token JWT) | Token falsificado, roubado ou usado depois de vencido | Alguém age com a identidade de outro usuário | Token assinado com chave secreta e expiração de 1 h; filtro do Spring Security valida assinatura e validade em toda requisição; envio apenas pelo header Authorization: Bearer |
| 4 | Tabela usuario (PostgreSQL) | Vazamento do banco de dados | Senhas expostas e reutilizadas em outros sistemas | Coluna senha guarda só o hash BCrypt; a entidade Usuario nunca é devolvida no JSON (resposta usa UsuarioResponse, sem senha) |
| 5 | POST/PUT /pastilhas e /fornecedores | SQL Injection pelo corpo da requisição | Leitura, alteração ou exclusão indevida de dados | Persistência pelo JpaRepository (save, findById); Hibernate gera SQL parametrizado, igual ao PreparedStatement com "?"; Bean Validation (@Valid) nos records de request |
| 6 | GET /pastilhas/pesquisa?codigo= | SQL Injection pelo parâmetro de pesquisa | Vazamento de dados de outras tabelas | Consultas derivadas (findByCodigoContainingIgnoreCase) ou @Query em JPQL com @Param; proibido montar SQL concatenando Strings, inclusive em nativeQuery |
| 7 | Campos de texto (descrição, observação, nome do fornecedor) | XSS armazenado (script salvo e exibido depois no front-end) | Script executado no navegador de outro usuário, podendo roubar o token | Validação com @Size e @Pattern nos records; API responde sempre Content-Type: application/json; front-end exibe os dados como texto; header X-Content-Type-Options: nosniff |
| 8 | Todos os endpoints, exceto login e health | Acesso sem autenticação | Exposição ou alteração do estoque por qualquer pessoa | Spring Security exige token em toda rota não pública; sem token a resposta é 401 Unauthorized |
| 9 | POST/PUT/DELETE de pastilhas, fornecedores e usuários; estorno | Usuário comum tentando operação de Administrador | Alteração do catálogo, exclusão de registros ou criação de contas | Autorização por perfil (hasRole) nas rotas e @PreAuthorize no Service; perfil vem do banco/token, nunca do JSON; resposta 403 Forbidden |
| 10 | GET /usuarios/{id} | Trocar o id na URL para ver outro usuário (IDOR) | Exposição de dados de colegas | Usuário comum só acessa /usuarios/me; /usuarios/{id} é exclusivo do Administrador |
| 11 | POST /movimentacoes | Dados manipulados: quantidade negativa, saída maior que o saldo, envio de campos como usuarioId ou data | Estoque inconsistente e retiradas sem rastreio | Record MovimentacaoRequest só com tipo, pastilhaId, quantidade e observação; @Positive na quantidade; Service valida o saldo; @Transactional faz rollback se algo falhar; usuário e data definidos pelo servidor |
| 12 | Requisições vindas do navegador | CSRF (site malicioso dispara requisição usando a sessão do usuário) | Operações feitas sem o usuário saber | API sem sessão e sem cookie: o token vai no header Authorization, que o navegador não envia sozinho; por isso o CSRF do Spring Security é desabilitado de forma justificada |
| 13 | Acesso de outros sites | CORS liberado para qualquer origem ("*") | Sites de terceiros consumindo a API | CORS liberado só para a origem do front-end do projeto, com métodos e headers definidos |
| 14 | Respostas de erro | Stack trace, SQL ou nomes de classes no corpo da resposta | Informações que ajudam um atacante | @RestControllerAdvice converte exceções (ex.: PastilhaNaoEncontradaException → 404) em JSON padronizado; server.error.include-stacktrace=never |
| 15 | application.properties | Senha do banco e chave do token escritas no código e enviadas ao GitHub | Comprometimento do banco e falsificação de tokens | Configuração externa do Spring Boot: spring.datasource.password=${DB_PASSWORD} e chave do token por variável de ambiente; .env no .gitignore |
| 16 | Conexão com o PostgreSQL | Usuário do banco com poder demais (ex.: postgres) | Um ataque bem-sucedido poderia apagar o banco inteiro | Usuário próprio (ex.: estoque_user) só com permissão no banco da aplicação; esquema criado pelo Flyway; ddl-auto=validate e show-sql=false fora do ambiente de desenvolvimento |
| 17 | Esquema do banco e usuário inicial | Alterações manuais e sem histórico; admin criado com senha padrão | Ambientes diferentes entre si; conta administrativa fácil de adivinhar | Migrações Flyway versionadas (V1__..., V2__...); o admin inicial é inserido por migração já com hash BCrypt e deve trocar a senha no primeiro acesso |
| 18 | GET /pastilhas e /movimentacoes | Pedir milhares de registros de uma vez | Lentidão ou queda da API | Paginação com Pageable e limite de 50 itens por página (spring.data.web.pageable.max-page-size=50) |
| 19 | /actuator e documentação | Exposição de informações internas (beans, variáveis, configurações) | Mapeamento facilitado da aplicação | Apenas /actuator/health exposto (management.endpoints.web.exposure.include=health) |
| 20 | Histórico de movimentações | Usuário nega ter feito uma retirada; registro apagado ou editado | Perdas de estoque sem explicação | Movimentação não tem PUT nem DELETE, só estorno pelo Administrador; cada registro guarda usuário e data/hora |

---

## 2. Plano de Segurança da API

### 2.1 Como será realizada a autenticação

- Será usado o **Spring Security** com autenticação **por token JWT**, sem sessão no servidor (`SessionCreationPolicy.STATELESS`).
- O cliente envia login e senha em **`POST /auth/login`**. O Spring Security busca o usuário no PostgreSQL (`UsuarioRepository.findByLogin`) e compara a senha com o hash BCrypt.
- Se estiver correto, a API responde **200 OK** com um token que contém o login, o perfil e a validade (**1 hora**). Se não estiver, responde **401**.
- Nas demais requisições o cliente envia o header **`Authorization: Bearer <token>`**. Um filtro valida o token antes de a requisição chegar ao Controller.
- Usuário inativo não autentica. Após 5 tentativas erradas seguidas, a conta fica bloqueada por 15 minutos.

### 2.2 Recursos acessíveis sem autenticação

- **`POST /auth/login`**: necessário para obter o token.
- **`GET /actuator/health`**: apenas informa se a API está no ar.

Não haverá cadastro público de usuários: contas são criadas somente pelo Administrador.

### 2.3 Recursos protegidos

Todos os outros: **`/pastilhas`**, **`/fornecedores`**, **`/movimentacoes`** e **`/usuarios`**. Sem token válido a API responde **401 Unauthorized**; com token, mas sem o perfil necessário, responde **403 Forbidden**. Rotas não previstas são negadas (`denyAll`).

### 2.4 Perfis de acesso

- **Público**: quem ainda não fez login. Só acessa login e health.
- **Usuário (`ROLE_USUARIO`)**: operador ou almoxarife. Consulta pastilhas, fornecedores e estoque baixo, e registra entradas e saídas.
- **Administrador (`ROLE_ADMIN`)**: responsável pelo estoque. Pode tudo o que o Usuário pode, além de manter o cadastro de pastilhas e fornecedores, estornar movimentações e gerenciar usuários.

O perfil fica em uma coluna da tabela `usuario`, mapeada como `enum Perfil` na entidade `Usuario`. As operações de cada perfil estão detalhadas na [seção 3](#3-perfis-e-permissões-de-acesso).

### 2.5 Como será feito o controle de autorização

- **Por rota**, no `SecurityFilterChain`, combinando método HTTP e caminho (tabela abaixo).
- **Por método**, com `@PreAuthorize("hasRole('ADMIN')")` nos métodos críticos do Service (estorno e gestão de usuários), como segunda barreira.
- O perfil vem do banco no login e é lido do token assinado; **nunca** é aceito vindo do corpo JSON.
- O usuário que fez a movimentação é obtido do `SecurityContext`, e não de um campo enviado pelo cliente.

| Rota | Regra no SecurityFilterChain |
|---|---|
| POST /auth/login, GET /actuator/health | `permitAll()` |
| GET /pastilhas/**, GET /fornecedores/** | `hasAnyRole("USUARIO","ADMIN")` |
| GET e POST /movimentacoes | `hasAnyRole("USUARIO","ADMIN")` |
| GET e PUT /usuarios/me | `authenticated()` |
| POST, PUT, DELETE /pastilhas/** e /fornecedores/** | `hasRole("ADMIN")` |
| POST /movimentacoes/{id}/estorno | `hasRole("ADMIN")` |
| /usuarios/** (demais rotas) | `hasRole("ADMIN")` |
| Qualquer outra rota | `denyAll()` |

### 2.6 Armazenamento e proteção das credenciais

- Senhas armazenadas com **`BCryptPasswordEncoder`**, um bean injetado pelo Spring no Service. Nunca são salvas em texto puro.
- A entidade `Usuario` não é devolvida diretamente: a resposta usa um record `UsuarioResponse` **sem o campo senha**. A senha também não aparece em logs.
- Senha mínima de 8 caracteres, com letras e números, validada com `@Size` e `@Pattern` no record de request.
- `spring.datasource.password` e a chave secreta do token ficam em **variáveis de ambiente**, nunca escritas no `application.properties` versionado.
- O banco usa um usuário próprio (ex.: `estoque_user`) em vez do superusuário `postgres`.

```properties
# application.properties (sem segredos versionados)
spring.datasource.url=jdbc:postgresql://localhost:5432/estoque_db
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
server.error.include-stacktrace=never
management.endpoints.web.exposure.include=health
spring.data.web.pageable.max-page-size=50
jwt.secret=${JWT_SECRET}
```

### 2.7 Configuração do CORS

- Configurado no Spring Security com `CorsConfigurationSource`.
- **Origem permitida**: apenas a do front-end do projeto (ex.: `http://localhost:5173` no desenvolvimento). Nenhum `"*"`.
- **Métodos**: `GET`, `POST`, `PUT`, `DELETE` e `OPTIONS` (preflight).
- **Headers**: `Authorization` e `Content-Type`.
- `allowCredentials = false`, pois o token vai no header e não em cookie.

### 2.8 Outros mecanismos contra as vulnerabilidades identificadas

- **CSRF**: desabilitado de forma justificada, porque a API não usa sessão nem cookie. Se o token passar a ser guardado em cookie, a proteção será reativada.
- **SQL Injection**: acesso a dados somente pelo `JpaRepository`, por consultas derivadas ou por `@Query` com `@Param`. Nunca concatenar Strings em SQL, nem em `nativeQuery`.
- **XSS**: validação dos campos de texto (`@NotBlank`, `@Size`, `@Pattern`), respostas sempre em `application/json` e exibição como texto no front-end.
- **Regras de negócio**: `@Positive` na quantidade; o Service impede saldo negativo; `@Transactional` garante rollback em caso de falha.
- **Records de request/response** separados das entidades, para o cliente não conseguir alterar campos como id, perfil, usuário ou data.
- **Erros padronizados** com `@RestControllerAdvice`, sem stack trace na resposta.
- **Paginação** com `Pageable` e limite de 50 itens por página.
- **Flyway**: esquema e usuário admin inicial criados por migrações versionadas, com `ddl-auto=validate` e `show-sql=false` em produção.
- **Actuator**: apenas o endpoint `health` exposto.
- **HTTPS** em produção, para que senha e token não trafeguem em texto aberto.

### 2.9 Onde cada controle fica na arquitetura

| Camada | Responsabilidade de segurança |
|---|---|
| **Filtro do Spring Security (antes do Controller)** | Lê o header Authorization, valida o token, aplica as regras por rota, CORS e responde 401/403. |
| **Controller (@RestController)** | Recebe records de request com @Valid (retorna 400 se inválido); devolve DTOs de resposta, nunca a entidade; usa o status code correto (201, 204, 404). |
| **Service (@Service)** | Regras de negócio (saldo, estorno), @Transactional, @PreAuthorize nas operações críticas e o usuário logado obtido do SecurityContext. |
| **Repository (JpaRepository)** | Acesso ao banco só por métodos do Spring Data, consultas derivadas ou @Query com @Param; nada de SQL concatenado. |
| **Domain (@Entity)** | Usuario com senha em hash e perfil (enum ADMIN/USUARIO); Movimentacao com usuário e data obrigatórios (@Column(nullable = false)). |
| **Banco e configuração** | PostgreSQL com usuário próprio; Flyway versionando o esquema; segredos em variáveis de ambiente; Actuator restrito. |

---

## 3. Perfis e Permissões de Acesso

| Recurso / Operação | Público | Usuário | Administrador |
|---|---|---|---|
| Realizar login (POST /auth/login) | ✅ Sim | ✅ Sim | ✅ Sim |
| Verificar status (GET /actuator/health) | ✅ Sim | ✅ Sim | ✅ Sim |
| Listar, consultar e pesquisar pastilhas | ❌ Não | ✅ Sim | ✅ Sim |
| Ver pastilhas com estoque baixo | ❌ Não | ✅ Sim | ✅ Sim |
| Cadastrar pastilha | ❌ Não | ❌ Não | ✅ Sim |
| Alterar pastilha | ❌ Não | ❌ Não | ✅ Sim |
| Excluir pastilha | ❌ Não | ❌ Não | ✅ Sim |
| Consultar fornecedores | ❌ Não | ✅ Sim | ✅ Sim |
| Cadastrar, alterar ou excluir fornecedor | ❌ Não | ❌ Não | ✅ Sim |
| Registrar movimentação (entrada/saída) | ❌ Não | ✅ Sim | ✅ Sim |
| Consultar histórico de movimentações | ❌ Não | ✅ Sim | ✅ Sim |
| Estornar movimentação | ❌ Não | ❌ Não | ✅ Sim |
| Ver os próprios dados e trocar a senha (/usuarios/me) | ❌ Não | ✅ Sim | ✅ Sim |
| Gerenciar usuários (/usuarios) | ❌ Não | ❌ Não | ✅ Sim |

---

## 4. Relação com o conteúdo das aulas

| Aula | Como foi aplicada no plano |
|---|---|
| **Aula 2 — HTTP na prática** | Token enviado no header Authorization; status codes coerentes (401 sem login, 403 sem permissão, 400 dados inválidos, 404 não encontrado); Content-Type: application/json. |
| **Aula 3 — Arquitetura e System Design** | Cada controle foi colocado em uma camada (Controller, Service, Repository, Domain), sem misturar segurança com regra de negócio. |
| **Aula 4 — Spring e Spring Boot** | Uso do Spring Security (projeto do ecossistema para autenticação e autorização); injeção de dependência do PasswordEncoder; configuração externa e Actuator restrito a health. |
| **Aula 5 — ORM, JPA e PostgreSQL** | Comparação com o PreparedStatement (parâmetros "?") para evitar SQL Injection; @Transactional com rollback nas movimentações; usuário próprio no PostgreSQL; ddl-auto e show-sql ajustados por ambiente. |
| **Aula 6 — Spring Data JPA** | JpaRepository, consultas derivadas e @Query com @Param em vez de SQL montado; cuidado com nativeQuery; Pageable para limitar listagens; findById + exceção para responder 404. |
| **Aula 7 — Versionamento de banco (Flyway)** | Tabelas usuario e perfil criadas por migrações versionadas; admin inicial inserido por migração com senha já em hash BCrypt. |

---

## Próximos passos

Essas decisões orientarão a próxima etapa, a implementação com **Spring Security**: `SecurityFilterChain`, filtro do token, `@PreAuthorize`, `BCryptPasswordEncoder`, configuração de CORS e as migrações Flyway das tabelas de usuário.
