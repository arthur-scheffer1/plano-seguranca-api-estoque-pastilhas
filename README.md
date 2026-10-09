# Matriz e Plano de Segurança da API — Estoque de Pastilhas

**API de Controle de Estoque de Pastilhas — DDA Metalúrgica**

| | |
|---|---|
| **Unidade Curricular** | Desenvolvimento de Sistemas Web — UniSenai |
| **Aluno** | Arthur Scheffer |
| **Tecnologias** | Java, Spring Boot, Spring Security, Spring Data JPA, PostgreSQL, Flyway |
| **Data** | Outubro de 2026 |

> 📄 Versão em Word: [Plano_Seguranca_API_Estoque_Pastilhas.docx](Plano_Seguranca_API_Estoque_Pastilhas.docx)

## Sumário

- [Contexto da API](#contexto-da-api)
- [1. Matriz de Segurança](#1-matriz-de-segurança)
- [2. Plano de Segurança da API](#2-plano-de-segurança-da-api)
- [3. Perfis e Permissões de Acesso](#3-perfis-e-permissões-de-acesso)
- [4. Relação com o conteúdo das aulas](#4-relação-com-o-conteúdo-das-aulas)

---

## Contexto da API

A API controla o estoque de pastilhas de usinagem da DDA Metalúrgica: cadastro de pastilhas e fornecedores, registro de entradas e saídas e consulta de itens com estoque baixo. É uma aplicação Spring Boot organizada em camadas (**Controller → Service → Repository → Domain**), com dados no PostgreSQL via Spring Data JPA e esquema versionado com Flyway.

Os dados são internos da empresa e cada movimentação altera o saldo real de ferramentas. Por isso a API exige autenticação em todos os recursos, exceto cadastro e login, e registra quem fez cada movimentação.

### Endpoints considerados

| Método e caminho | Operação | Status esperados |
|---|---|---|
| `POST /auth/register` | Cadastrar usuário (perfil USUARIO) | 201 / 400 |
| `POST /auth/login` | Validar credenciais e devolver o token JWT | 200 / 401 |
| `GET /pastilhas` | Listar pastilhas (paginado) | 200 / 401 |
| `GET /pastilhas/{id}` | Consultar pastilha | 200 / 404 |
| `GET /pastilhas/pesquisa?codigo=` | Pesquisar pastilha por código | 200 |
| `GET /pastilhas/estoque-baixo` | Pastilhas abaixo do estoque mínimo | 200 |
| `POST /pastilhas` | Cadastrar pastilha | 201 / 400 / 403 |
| `PUT /pastilhas/{id}` | Atualizar pastilha | 200 / 403 / 404 |
| `DELETE /pastilhas/{id}` | Remover pastilha | 204 / 403 / 404 |
| `GET /fornecedores, GET /fornecedores/{id}` | Consultar fornecedores | 200 / 404 |
| `POST, PUT, DELETE /fornecedores` | Manter fornecedores | 201 / 200 / 204 / 403 |
| `GET /movimentacoes` | Histórico de entradas e saídas | 200 |
| `POST /movimentacoes` | Registrar entrada ou saída de estoque | 201 / 400 |
| `GET, PUT, DELETE /usuarios` | Gerenciar usuários | 200 / 204 / 403 |

---

## 1. Matriz de Segurança

A matriz relaciona cada recurso ou operação da API às ameaças estudadas, ao impacto possível e ao controle preventivo que será implementado.

| # | Recurso / Operação | Ameaça ou vulnerabilidade | Possível impacto | Controle preventivo |
|---|---|---|---|---|
| 1 | POST /auth/login | Tentativas de acesso indevido (senha errada ou de outra pessoa) | Acesso não autorizado ao estoque | Login validado pelo Spring Security: AuthenticationManager, UserDetailsService (busca por e-mail) e PasswordEncoder (compara a senha com o hash). Credencial inválida responde 401. Usuário com ativo = false não autentica |
| 2 | Tabela usuarios (PostgreSQL) | Vazamento dos dados do banco | Senhas expostas e reutilizadas em outros sistemas | Senha guardada só como hash BCrypt (salt + custo). O DTO de resposta nunca devolve a senha |
| 3 | POST /auth/register | Cadastro tentando se dar o perfil ADMIN, ou com dados inválidos/e-mail repetido | Escalonamento de privilégio e dados inconsistentes | DTO de entrada só com nome, e-mail e senha; o perfil USUARIO é definido no Service, nunca vem do JSON. Bean Validation (@NotBlank, @Email, @Size) e e-mail UNIQUE no banco |
| 4 | Header Authorization: Bearer <token> | Token falsificado, alterado ou vencido | Alguém age com a identidade de outro usuário | JWT assinado: a assinatura (signature) detecta alterações; claims sub, iat e exp; o token é validado na cadeia de filtros antes do Controller |
| 5 | Todos os endpoints, exceto /auth/\*\* | Acesso sem autenticação | Exposição ou alteração indevida das informações do estoque | anyRequest().authenticated() no SecurityFilterChain: sem token válido a resposta é 401 |
| 6 | POST, PUT, DELETE de pastilhas e fornecedores; /usuarios | Acesso sem autorização (Usuário comum tentando operação de Administrador) | Alteração do cadastro, exclusão de registros ou de contas | Controle de acesso por perfil (ADMIN / USUARIO) nas regras do SecurityFilterChain; resposta 403 |
| 7 | POST e PUT /pastilhas e /fornecedores | SQL Injection pelo corpo da requisição | Manipulação ou acesso indevido aos dados | Persistência pelo JpaRepository (save, findById): o Hibernate gera SQL com parâmetros, como o PreparedStatement com "?". Validação das entradas com Bean Validation nos DTOs |
| 8 | GET /pastilhas/pesquisa | SQL Injection pelo parâmetro de busca | Leitura de dados de outras tabelas | Consulta derivada (findByCodigoContainingIgnoreCase) ou @Query em JPQL com @Param. Nunca concatenar texto do usuário em SQL, nem em nativeQuery |
| 9 | Campos de texto (descrição da pastilha, nome do fornecedor, observação) | XSS (script salvo e exibido depois no front-end) | Execução de script no navegador de outro usuário, podendo roubar o token | Validação das entradas (@NotBlank, @Size) nos DTOs; API responde sempre JSON (Content-Type: application/json); o front-end exibe os dados como texto |
| 10 | POST /movimentacoes | Dados inválidos: quantidade negativa ou saída maior que o saldo | Estoque inconsistente | @Positive na quantidade; o Service confere o saldo antes de salvar; @Transactional faz rollback se algo falhar. O usuário responsável vem do token (SecurityContext), não do JSON |
| 11 | Requisições vindas do navegador | CSRF (site malicioso aproveita a sessão do usuário) | Operações feitas sem o usuário saber | API stateless: sem sessão e sem cookie, o token vai no header Authorization. Por isso csrf.disable(), como na Aula 10 |
| 12 | Acesso de outros sites (navegador) | CORS liberado para qualquer origem | Sites de terceiros consumindo a API | CORS liberado só para a origem do front-end do projeto, com os métodos e headers usados |
| 13 | Páginas que exibem a API | Clickjacking (página colocada dentro de um iframe de outro site) | Usuário clica em algo sem perceber | Mantidos os cabeçalhos de segurança padrão do Spring Security, que bloqueiam o uso em iframe (X-Frame-Options) |
| 14 | Comunicação cliente–servidor | Interceptação do tráfego | Captura da senha no login e do token | HTTPS (SSL/TLS) em produção: o HTTPS protege o transporte |
| 15 | application.properties | Senha do banco e chave do token escritas no código e enviadas ao GitHub | Acesso ao banco e falsificação de tokens | Configuração externa do Spring Boot: valores sensíveis lidos de variáveis de ambiente |
| 16 | Banco de dados e usuário inicial | Alterações manuais no esquema; Administrador criado sem controle | Ambientes diferentes entre si; conta admin insegura | Tabela usuarios e Administrador inicial criados por migrações Flyway versionadas, com a senha já em hash BCrypt. Usuário próprio no PostgreSQL para a aplicação |
| 17 | GET /pastilhas e /movimentacoes | Pedir todos os registros de uma vez | Lentidão da API | Listagens paginadas com Pageable |

---

## 2. Plano de Segurança da API

### 2.1 Como será realizada a autenticação

- Com **Spring Security** e **token JWT**, no modelo da Aula 10. A API é **stateless**: não guarda sessão, o cliente envia o token em cada requisição.
- **POST /auth/register** cadastra o usuário e guarda o **hash** da senha.
- **POST /auth/login** recebe e-mail e senha. O AuthenticationManager delega ao UserDetailsService, que busca o usuário no PostgreSQL (repository.findByEmail), e ao PasswordEncoder, que compara a senha com o hash.
- Se estiver certo, o tokenService gera o **JWT** com sub (e-mail), iat, exp e o perfil, e a API devolve o token. Se estiver errado, responde **401**.
- Nas outras requisições o cliente envia **Authorization: Bearer <token>**. O token é validado na cadeia de filtros, antes do Controller.
- Usuário com ativo = false não consegue fazer login.

### 2.2 Recursos acessíveis sem autenticação

- **POST /auth/register**: cadastro de usuário (sempre com perfil USUARIO).
- **POST /auth/login**: obter o token.

No SecurityFilterChain: `requestMatchers("/auth/**").permitAll()`.

### 2.3 Recursos protegidos

Todos os outros: **/pastilhas**, **/fornecedores**, **/movimentacoes** e **/usuarios**, com anyRequest().authenticated(). Sem token válido: **401 Unauthorized**. Com token, mas sem o perfil necessário: **403 Forbidden**.

### 2.4 Perfis de acesso

- **Público**: quem ainda não fez login. Só pode se cadastrar e fazer login.
- **Usuário (USUARIO)**: operador do estoque. Consulta pastilhas, fornecedores e estoque baixo, e registra entradas e saídas.
- **Administrador (ADMIN)**: responsável pelo estoque. Faz tudo o que o Usuário faz e também cadastra, altera e exclui pastilhas e fornecedores e gerencia usuários.

O perfil fica em uma coluna da tabela usuarios. Todo cadastro feito por /auth/register recebe USUARIO; o primeiro Administrador é criado por migração Flyway. As operações de cada perfil estão na seção 3.

### 2.5 Como será feito o controle de autorização

- A autorização depende de uma autenticação válida: primeiro o filtro valida o token, depois verifica o perfil.
- As regras ficam no **SecurityFilterChain**, combinando método HTTP e caminho com requestMatchers, conforme a tabela abaixo.
- O perfil vem do banco no login e viaja dentro do token assinado. Ele **nunca** é aceito vindo do JSON.
- O usuário que registrou uma movimentação é obtido do SecurityContext (token), e não de um campo enviado pelo cliente.

| Rota | Quem pode acessar |
|---|---|
| `/auth/** (register e login)` | permitAll() |
| `GET /pastilhas/**, GET /fornecedores/**, GET /movimentacoes` | USUARIO ou ADMIN |
| `POST /movimentacoes` | USUARIO ou ADMIN |
| `POST, PUT, DELETE /pastilhas/** e /fornecedores/**` | ADMIN |
| `/usuarios/**` | ADMIN |
| `Qualquer outra rota` | authenticated() |

### 2.6 Armazenamento e proteção das credenciais

- Senhas salvas com **BCryptPasswordEncoder** (declarado como @Bean PasswordEncoder). O banco guarda só o hash, nunca a senha original.
- Tabela usuarios criada por migração Flyway: id BIGSERIAL, nome, email VARCHAR(255) NOT NULL UNIQUE, senha VARCHAR(255) NOT NULL, ativo BOOLEAN NOT NULL DEFAULT TRUE e perfil.
- Os DTOs de resposta não têm o campo senha.
- Senha do banco e chave de assinatura do token ficam em variáveis de ambiente, fora do application.properties versionado no GitHub.
- A aplicação usa um usuário próprio no PostgreSQL.

### 2.7 Configuração do CORS

- **Origem permitida**: somente a do front-end do projeto (ex.: http://localhost:5173 no desenvolvimento). Nada de liberar todas as origens ("*").
- **Métodos**: GET, POST, PUT, DELETE e OPTIONS.
- **Headers**: Authorization e Content-Type.

### 2.8 Outros mecanismos contra as vulnerabilidades

- **CSRF**: csrf.disable(), porque a API é stateless e o token vai no header, não em cookie (como na Aula 10).
- **SQL Injection**: acesso ao banco só pelo JpaRepository, consultas derivadas ou @Query com @Param.
- **XSS**: Bean Validation nos DTOs e respostas sempre em JSON.
- **Clickjacking**: manter os cabeçalhos padrão do Spring Security.
- **HTTPS** em produção, para proteger senha e token no transporte.
- **Regras de negócio**: @Positive na quantidade, Service confere o saldo, @Transactional com rollback.
- **Paginação** com Pageable nas listagens.

### 2.9 Onde cada controle fica na arquitetura

| Camada | Responsabilidade de segurança |
|---|---|
| **Cadeia de filtros (SecurityFilterChain)** | Antes do Controller: lê o Bearer token, valida o JWT, aplica as regras de rota e perfil. Falhas param o fluxo com 401 ou 403. |
| **Controller (@RestController)** | Recebe DTOs com @Valid (400 se inválido), devolve DTOs de resposta (nunca a entidade Usuario com senha) e o status code correto. |
| **Service (@Service)** | Regras de negócio (saldo, perfil USUARIO no cadastro), hash da senha com PasswordEncoder, @Transactional. |
| **Repository (JpaRepository)** | Consultas pelo Spring Data: métodos prontos, consultas derivadas ou @Query com @Param. Nada de SQL concatenado. |
| **Domain (@Entity)** | Usuario (nome, email único, senha em hash, ativo, perfil) e Movimentacao com usuário e data obrigatórios. |
| **Banco e configuração** | PostgreSQL com usuário próprio, Flyway versionando o esquema, segredos em variáveis de ambiente. |

### 2.10 Configuração prevista 

```java
http
  .csrf(csrf -> csrf.disable())
  .sessionManagement(session -> session
      .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
  .authorizeHttpRequests(auth -> auth
      .requestMatchers("/auth/**").permitAll()
      // regras por perfil da tabela 2.5
      .anyRequest().authenticated())
  .oauth2ResourceServer(oauth2 -> oauth2.jwt());
```

---

## 3. Perfis e Permissões de Acesso

| Recurso / Operação | Público | Usuário | Administrador |
|---|---|---|---|
| Cadastrar-se (POST /auth/register) | ✅ Sim | — | — |
| Realizar login (POST /auth/login) | ✅ Sim | ✅ Sim | ✅ Sim |
| Consultar e pesquisar pastilhas | ❌ Não | ✅ Sim | ✅ Sim |
| Ver pastilhas com estoque baixo | ❌ Não | ✅ Sim | ✅ Sim |
| Cadastrar pastilha | ❌ Não | ❌ Não | ✅ Sim |
| Alterar pastilha | ❌ Não | ❌ Não | ✅ Sim |
| Excluir pastilha | ❌ Não | ❌ Não | ✅ Sim |
| Consultar fornecedores | ❌ Não | ✅ Sim | ✅ Sim |
| Cadastrar, alterar ou excluir fornecedor | ❌ Não | ❌ Não | ✅ Sim |
| Registrar movimentação (entrada/saída) | ❌ Não | ✅ Sim | ✅ Sim |
| Consultar histórico de movimentações | ❌ Não | ✅ Sim | ✅ Sim |
| Gerenciar usuários (/usuarios) | ❌ Não | ❌ Não | ✅ Sim |

