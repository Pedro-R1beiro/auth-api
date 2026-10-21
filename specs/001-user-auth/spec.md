# Feature Specification: Autenticação de usuários pela API

**Feature Branch**: `001-user-auth` (identificador da especificação; nenhuma branch foi criada)

**Created**: 2026-10-21

**Status**: Draft

**Input**: User description: "Permitir cadastro, login com email e senha, emissão de token de acesso, consulta da identidade autenticada e logout com revogação do token atual. Incluir respostas JSON previsíveis, documentação do contrato, testes e controles para impedir exposição de credenciais. Recuperação de senha, verificação de email, refresh token, login social, autenticação multifator, perfis avançados e frontend estão fora do escopo. Esta etapa descreve comportamentos e critérios, sem decidir arquitetura técnica. Também deve haver um limite de tentativas de login em um dispositivo; ao atingir o limite, o usuário fica impedido de efetuar login durante um período determinado."

## Clarifications

### Session 2026-10-21

- Q: As cinco falhas devem ser contadas em uma janela móvel de 15 minutos, considerando sempre as falhas ocorridas nos 15 minutos anteriores? → A: Janela móvel de 15 minutos; cada tentativa considera as falhas dos 15 minutos imediatamente anteriores.
- Q: Como a API deve receber o identificador estável usado para reconhecer o mesmo dispositivo entre tentativas de login? → A: UUID opaco gerado e persistido pelo cliente por instalação, enviado no cabeçalho `X-Device-ID`; o identificador pode ser trocado pelo cliente e não comprova a identidade física do dispositivo.
- Q: Tentativas de login feitas durante o bloqueio de 15 minutos devem reiniciar esse prazo ou deixar o horário original de desbloqueio inalterado? → A: Não prolongar o bloqueio; tentativas durante o período não alteram o horário original de liberação.
- Q: Um login bem-sucedido deve apagar as falhas recentes de todas as contas associadas ao mesmo identificador de dispositivo? → A: Não; falhas não são apagadas por um login bem-sucedido e deixam de contar naturalmente ao sair da janela móvel de 15 minutos.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Criar uma conta (Priority: P1)

Uma pessoa sem conta pode enviar nome, email, senha e confirmação para criar uma
conta e começar a usar a API.

**Why this priority**: O cadastro cria a identidade necessária para os demais fluxos.

**Independent Test**: Enviar dados válidos e verificar que a conta é criada, os dados
públicos e um token são retornados, e nenhuma senha ou hash é exposto.

**Acceptance Scenarios**:

1. **Given** dados válidos e um email ainda não utilizado, **When** a pessoa envia
   `POST /api/register`, **Then** recebe `201`, os dados públicos da conta e um token.
2. **Given** um ou mais campos ausentes ou inválidos, **When** a pessoa envia
   `POST /api/register`, **Then** recebe `422` com erros de validação por campo e
   nenhuma conta ou token é criado.
3. **Given** um email já cadastrado, **When** a pessoa tenta cadastrá-lo novamente,
   **Then** recebe `409` e nenhum novo usuário ou token é criado.

---

### User Story 2 - Entrar na conta (Priority: P1)

Uma pessoa cadastrada pode autenticar-se com email, senha e identificador de
dispositivo para obter um token de acesso.

**Why this priority**: O login permite que uma pessoa retorne à API depois do cadastro.

**Independent Test**: Tentar entrar com credenciais válidas e inválidas; verificar que
somente credenciais válidas emitem token e que falhas não revelam se o email existe.

**Acceptance Scenarios**:

1. **Given** uma conta existente e credenciais corretas, **When** a pessoa envia
   `POST /api/login`, **Then** recebe `200` e um token de acesso.
2. **Given** email inexistente ou senha incorreta, **When** a pessoa envia
   `POST /api/login`, **Then** recebe a mesma resposta genérica `401` em ambos os casos
   e nenhum token é emitido.
3. **Given** campos de login ausentes ou inválidos, **When** a pessoa envia
   `POST /api/login`, **Then** recebe `422` com erros de validação e nenhum token.

---

### User Story 3 - Bloquear temporariamente novas tentativas de login (Priority: P1)

Uma pessoa que excede o limite de tentativas inválidas de login em um dispositivo fica
temporariamente impedida de tentar entrar novamente, reduzindo tentativas repetidas de
adivinhação de credenciais.

**Why this priority**: O limite reduz o risco de acesso indevido por tentativas
repetidas de credenciais.

**Independent Test**: Fazer tentativas inválidas até alcançar cinco falhas dentro de
uma janela móvel de quinze minutos e verificar que nenhuma conta pode autenticar-se
por esse identificador de dispositivo durante os quinze minutos seguintes.

**Acceptance Scenarios**:

1. **Given** um identificador de dispositivo sem bloqueio ativo, **When** ocorrem
   cinco falhas de autenticação nos quinze minutos imediatamente anteriores, **Then**
   o dispositivo fica
   bloqueado por quinze minutos e nenhuma tentativa emite token.
2. **Given** um identificador de dispositivo bloqueado, **When** qualquer conta tenta
   autenticar-se antes do fim dos quinze minutos, **Then** o login é recusado mesmo
   com credenciais corretas e a resposta não revela se a conta existe.
3. **Given** um identificador bloqueado cujo prazo de quinze minutos terminou,
   **When** a pessoa envia credenciais válidas, **Then** o login é permitido.
4. **Given** identificadores de dispositivo distintos, **When** um deles atinge o
   limite, **Then** tentativas do outro identificador não são bloqueadas por essa
   contagem.
5. **Given** tentativas inválidas recentes e um login válido de qualquer conta no
   mesmo dispositivo, **When** outra falha acontece antes de as anteriores saírem da
   janela móvel, **Then** as falhas anteriores continuam contando.

---

### User Story 4 - Consultar a própria identidade (Priority: P2)

Uma pessoa autenticada pode consultar os dados públicos da própria conta.

**Why this priority**: Clientes precisam confirmar qual identidade está associada ao
token apresentado.

**Independent Test**: Consultar `GET /api/me` com e sem token; verificar que apenas
uma credencial válida retorna os dados públicos da pessoa correspondente.

**Acceptance Scenarios**:

1. **Given** um token válido no cabeçalho `Authorization: Bearer`, **When** a pessoa
   envia `GET /api/me`, **Then** recebe `200` e somente os dados públicos da própria
   conta.
2. **Given** token ausente, inválido, expirado ou revogado, **When** a pessoa envia
   `GET /api/me`, **Then** recebe `401` sem dados de conta ou credenciais.

---

### User Story 5 - Encerrar a sessão atual (Priority: P2)

Uma pessoa autenticada pode encerrar a sessão associada ao token apresentado sem
encerrar outras sessões da mesma conta.

**Why this priority**: A pessoa precisa poder invalidar uma credencial que não deseja
mais usar, sem afetar outras sessões.

**Independent Test**: Emitir dois tokens para a mesma conta, encerrar a sessão usando
um deles e verificar que somente esse token deixa de autenticar requisições.

**Acceptance Scenarios**:

1. **Given** um token válido no cabeçalho `Authorization: Bearer`, **When** a pessoa
   envia `POST /api/logout`, **Then** recebe `200` e somente o token usado na chamada
   deixa de ser aceito.
2. **Given** um token já revogado, **When** a pessoa tenta reutilizá-lo em
   `GET /api/me` ou `POST /api/logout`, **Then** recebe `401`.
3. **Given** outro token ainda válido para a mesma conta, **When** é usado após o
   logout do primeiro token, **Then** continua autenticando requisições protegidas.
4. **Given** token ausente ou inválido, **When** a pessoa envia `POST /api/logout`,
   **Then** recebe `401` e nenhuma sessão adicional é alterada.

### Edge Cases

- Emails com espaços nas extremidades ou diferenças entre maiúsculas e minúsculas
  representam o mesmo endereço após normalização; o sistema não permite duplicidade.
- A confirmação de senha ausente ou diferente da senha é erro de validação.
- Uma senha com menos de oito caracteres é rejeitada no cadastro.
- Email inexistente e senha incorreta produzem o mesmo status e mensagem no login.
- Falhas de validação, autenticação ou conflito não emitem tokens.
- Cinco falhas de autenticação dentro de qualquer janela móvel de quinze minutos
  bloqueiam durante quinze minutos todas as tentativas de login que usem o mesmo
  identificador estável de dispositivo, independentemente da conta informada.
- Uma autenticação bem-sucedida não apaga as falhas recentes do identificador; cada
  falha deixa de contar naturalmente ao completar quinze minutos.
- O fim do bloqueio reinicia a contagem de falhas do identificador.
- Tentativas recusadas durante o bloqueio não prolongam nem reiniciam o período; o
  dispositivo é liberado quinze minutos após o início original do bloqueio.
- Solicitações com campos ausentes ou inválidos continuam sendo erros de validação e
  não contam como falhas de autenticação de credenciais.
- O identificador estável do dispositivo é obrigatório em toda tentativa de login;
  deve ser um UUID opaco gerado e persistido pelo cliente por instalação e enviado no
  cabeçalho `X-Device-ID`. Quando ausente ou inválido, a solicitação recebe erro de
  validação e não altera a contagem.
- O cliente pode trocar ou apagar o identificador, por exemplo ao reinstalar o
  aplicativo; essa troca cria um identificador diferente e pode contornar o bloqueio
  anterior. O valor não comprova a identidade física do dispositivo.
- Durante o bloqueio, credenciais corretas e incorretas recebem a mesma resposta de
  bloqueio, sem indicar se a conta existe e sem emitir token.
- Cabeçalho de autenticação ausente ou malformado, ou associado a token inválido,
  expirado ou revogado
  não autentica uma rota protegida.
- Encerrar um token não invalida outros tokens ativos da mesma conta.
- Respostas e erros não incluem senha, hash de senha ou segredos; o token só é incluído
  nas respostas bem-sucedidas de cadastro e login.
- Cada token de acesso expira uma hora após sua emissão; o uso do token não prolonga
  esse prazo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema MUST disponibilizar `POST /api/register` para criar uma conta
  com nome, email, senha e confirmação de senha.
- **FR-002**: O sistema MUST exigir nome não vazio, email em formato válido, senha com
  pelo menos oito caracteres e confirmação exatamente igual à senha.
- **FR-003**: O sistema MUST remover espaços externos e normalizar o email para
  minúsculas antes de verificar unicidade, cadastrar ou autenticar a conta.
- **FR-004**: O sistema MUST impedir cadastro de email já utilizado e responder com
  `409`, sem criar conta ou token.
- **FR-005**: O sistema MUST armazenar senhas pelo mecanismo de hashing do Laravel e
  MUST NOT expor senha ou hash em qualquer resposta da API.
- **FR-006**: Cadastro bem-sucedido MUST responder com `201` e JSON contendo um token
  de acesso e os dados públicos do usuário (`id`, `name` e `email`), sem outros dados
  de credenciais, no formato
  `{"data":{"user":{"id":1,"name":"...","email":"..."},"token":"..."}}`.
- **FR-007**: O sistema MUST disponibilizar `POST /api/login`, exigindo email e senha,
  e responder com `200`, token de acesso e dados públicos do usuário quando as
  credenciais forem válidas, no mesmo formato de sucesso do cadastro.
- **FR-008**: Email inexistente e senha incorreta MUST produzir o mesmo status `401`
  e uma mensagem genérica, sem indicar qual credencial falhou e sem emitir token.
- **FR-009**: O sistema MUST disponibilizar `GET /api/me` e `POST /api/logout` somente
  para chamadas com token de acesso válido apresentado como Bearer.
- **FR-010**: Chamadas a rotas protegidas sem token válido MUST responder com `401`
  em JSON e MUST NOT retornar dados do usuário ou credenciais.
- **FR-011**: `GET /api/me` MUST responder com `200` no formato
  `{"data":{"id":1,"name":"...","email":"..."}}`, sem incluir senha, hash ou token.
- **FR-012**: `POST /api/logout` MUST responder com `200` e revogar somente o token
  usado na requisição; esse token MUST deixar de autenticar chamadas posteriores. A
  resposta MUST seguir o formato `{"data":{"message":"..."}}`.
- **FR-013**: A revogação de um token MUST NOT invalidar outros tokens ativos da mesma
  conta.
- **FR-014**: Erros de validação MUST responder com `422` e JSON no formato
  `{"message":"...","errors":{"campo":["..."]}}`, com os erros agrupados pelo campo.
- **FR-015**: Respostas de conflito e autenticação MUST usar JSON previsível com uma
  propriedade `message`; respostas de sucesso MUST usar a propriedade `data`. Falhas
  de autenticação MUST usar a mesma mensagem genérica para credenciais inválidas e
  uma mensagem de não autenticação para tokens ausentes ou inválidos.
- **FR-016**: Cada token MUST expirar uma hora após sua emissão, sem prorrogação por
  atividade. Tokens expirados MUST ser recusados como não autenticados.
- **FR-017**: O contrato de cada endpoint, incluindo campos, códigos de status e erros,
  MUST ser documentado em português brasileiro.
- **FR-018**: Os fluxos de cadastro, login, consulta de identidade e logout MUST ter
  testes automatizados para sucesso, validação e falha, incluindo emissão e revogação
  de tokens e proteção de dados sensíveis.
- **FR-019**: O sistema MUST contar falhas de autenticação de credenciais por
  identificador estável de dispositivo; solicitações com campos ausentes ou inválidos
  MUST NOT aumentar essa contagem.
- **FR-020**: Cinco falhas de autenticação dentro de qualquer janela móvel de quinze
  minutos MUST bloquear novas tentativas de login com o mesmo identificador por quinze
  minutos, independentemente da conta informada. Cada tentativa MUST considerar as
  falhas ocorridas nos quinze minutos imediatamente anteriores.
- **FR-021**: Durante o bloqueio, o sistema MUST recusar qualquer tentativa de login
  com aquele identificador, inclusive com credenciais corretas, MUST NOT emitir token
  e MUST NOT revelar se a conta informada existe.
- **FR-022**: Uma autenticação bem-sucedida MUST NOT apagar falhas recentes associadas
  ao identificador do dispositivo; cada falha MUST deixar de contar ao sair da janela
  móvel de quinze minutos. Ao terminar o bloqueio, a contagem MUST ser reiniciada.
- **FR-023**: O bloqueio associado a um identificador MUST NOT afetar tentativas
  associadas a identificadores distintos.
- **FR-024**: Tentativas recusadas por bloqueio MUST responder com HTTP `429` e JSON
  previsível, incluindo uma mensagem genérica e `retry_after_seconds` com o número de
  segundos restantes. A tentativa inválida que completa o limite MUST receber a
  resposta genérica `401` definida para credenciais inválidas; tentativas seguintes
  recebem `429`. Credenciais corretas ou incorretas MUST receber a mesma resposta
  durante o bloqueio. Tentativas recusadas MUST NOT prolongar nem reiniciar o prazo
  original de liberação.
- **FR-025**: O contrato do login MUST documentar o identificador exigido para
  distinguir dispositivos e a resposta emitida quando o limite é atingido.
- **FR-026**: Testes automatizados MUST cobrir o limite de cinco falhas na janela
  móvel de quinze minutos, início e fim do bloqueio, manutenção de falhas após login bem-sucedido,
  recusa de credenciais corretas durante o bloqueio, não emissão de tokens e
  isolamento entre identificadores.
- **FR-027**: `POST /api/login` MUST exigir um UUID opaco e estável por instalação,
  enviado no cabeçalho `X-Device-ID`. Sua ausência ou formato inválido MUST produzir
  `422` e MUST NOT alterar a contagem de falhas.
- **FR-028**: O bloqueio MUST terminar quinze minutos após seu início, sem ser
  prolongado por tentativas recusadas durante esse período.

### Key Entities *(include if feature involves data)*

- **Usuário**: representa a pessoa com conta; possui identificador, nome, email e uma
  credencial protegida. A senha não é dado público.
- **Token de acesso**: representa a credencial temporária usada para identificar uma
  sessão; pode ser revogada individualmente e expira uma hora após sua emissão.
- **Identificador de dispositivo**: UUID opaco gerado e persistido pelo cliente por
  instalação e enviado no cabeçalho `X-Device-ID`; tentativas que o compartilham
  participam da mesma contagem e bloqueio, independentemente da conta informada. O
  cliente pode trocá-lo, portanto ele não comprova a identidade física do dispositivo.
- **Tentativas de login**: falhas de autenticação de credenciais associadas a um
  identificador de dispositivo, usadas para aplicar a janela de quinze minutos e o
  bloqueio de quinze minutos.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Em 100% dos cadastros válidos, a pessoa recebe `201`, os dados públicos
  definidos e um token; nenhum segredo de autenticação aparece na resposta.
- **SC-002**: Em 100% das tentativas de login com email inexistente ou senha incorreta,
  a resposta é indistinguível pelo status e mensagem, e nenhum token é emitido.
- **SC-003**: Em 100% das chamadas às rotas protegidas com token ausente, inválido,
  expirado ou revogado, a pessoa recebe `401` sem dados privados.
- **SC-004**: Após logout, 100% das tentativas de reutilização do token revogado são
  recusadas, enquanto outros tokens ativos da mesma conta permanecem válidos.
- **SC-005**: Clientes conseguem distinguir sucesso, validação, conflito e falha de
  autenticação pelos códigos HTTP e pelo formato JSON definidos nesta especificação.
- **SC-006**: Em 100% dos casos em que o limite definido é atingido, tentativas
  adicionais não autenticam nem emitem tokens até terminar o bloqueio.
- **SC-007**: Em 100% dos casos, login válido volta a ser possível após o término do
  bloqueio, e dispositivos fora do escopo bloqueado permanecem sujeitos à regra de
  isolamento definida.
- **SC-008**: Em 100% das tentativas durante o bloqueio, a resposta é HTTP `429`,
  informa o tempo restante de maneira previsível e não revela se a conta existe.

## Assumptions

- O email é comparado sem distinção entre maiúsculas e minúsculas e com espaços
  externos removidos.
- A senha tem no mínimo oito caracteres; espaços internos ou externos fazem parte da
  senha e não são removidos.
- Uma conta pública expõe somente `id`, `name` e `email`.
- Cadastro bem-sucedido retorna `201`; login, consulta de identidade e logout bem
  sucedidos retornam `200`; dados inválidos retornam `422`; email duplicado retorna
  `409`; credenciais ou tokens não autenticados retornam `401`.
- Cada token expira uma hora após sua emissão, sem prorrogação por atividade; não há
  emissão ou uso de refresh token nesta funcionalidade.
- O cliente gera um UUID opaco por instalação, persiste-o e envia o mesmo valor no
  cabeçalho `X-Device-ID` em tentativas subsequentes; a autenticidade do valor não é
  presumida, e sua troca permite iniciar uma contagem distinta.
- Uma tentativa inválida de autenticação é aquela em que o email e a senha foram
  enviados com formato válido, mas não autenticam uma conta.
- Cinco falhas dentro de qualquer janela móvel de quinze minutos bloqueiam todas as
  contas usadas com o mesmo identificador de dispositivo por quinze minutos;
  login bem-sucedido não apaga falhas recentes, que expiram naturalmente da janela.
  O fim do bloqueio reinicia a contagem. Tentativas durante o bloqueio não prolongam
  o prazo de liberação.
- Recuperação de senha, verificação de email, login social, autenticação multifator,
  perfis e permissões avançadas e frontend permanecem fora do escopo.
