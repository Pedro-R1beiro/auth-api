# Contrato HTTP: autenticação de usuários

**Base**: `/api`  
**Formato**: JSON UTF-8, exceto o cabeçalho `X-Device-ID`.  
**Fonte**: [especificação da feature](../spec.md).

## Convenções

- Sucesso usa a propriedade superior `data`.
- Validação retorna HTTP `422` com `message` e `errors` agrupados por campo.
- Credenciais inválidas retornam HTTP `401` com mensagem genérica, igual para email
  inexistente e senha incorreta.
- Rotas privadas exigem `Authorization: Bearer <token>`. Token ausente, inválido,
  expirado ou revogado retorna HTTP `401`.
- Nenhuma resposta deve incluir senha, hash de senha, token armazenado ou segredo.
  O token em claro só é retornado na emissão bem-sucedida.
- Erros não devem expor detalhes internos nem confirmar existência de conta durante
  login ou bloqueio.

## `POST /api/register`

**Autenticação**: não exigida.

### Requisição

```json
{
  "name": "Ana Silva",
  "email": "ana@example.com",
  "password": "senha-segura",
  "password_confirmation": "senha-segura"
}
```

Campos obrigatórios: nome não vazio; email válido; senha com pelo menos oito
caracteres; confirmação idêntica. Espaços externos do email são removidos e o email é
normalizado para minúsculas.

### Sucesso — `201 Created`

```json
{
  "data": {
    "user": {
      "id": 1,
      "name": "Ana Silva",
      "email": "ana@example.com"
    },
    "token": "<credencial de acesso emitida uma única vez>"
  }
}
```

### Falhas

| Status | Condição | Corpo |
|--------|----------|-------|
| `422` | Campo ausente ou inválido | `{"message":"Os dados informados são inválidos.","errors":{"campo":["..."]}}` |
| `409` | Email já utilizado | `{"message":"Não foi possível criar a conta com os dados informados."}` |

Falha de validação ou conflito não cria usuário nem token.

## `POST /api/login`

**Autenticação**: não exigida. O cabeçalho de identificação do dispositivo é
obrigatório.

### Cabeçalho

```http
X-Device-ID: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json
```

O valor é um UUID válido, opaco e persistido pelo cliente por instalação. É controlado
pelo cliente, pode ser trocado e não comprova identidade física do dispositivo.

### Requisição

```json
{
  "email": "ana@example.com",
  "password": "senha-segura"
}
```

### Sucesso — `200 OK`

Mesmo envelope de sucesso do cadastro: `data.user` contém `id`, `name`, `email`;
`data.token` contém a credencial de acesso emitida. O prazo é de 60 minutos desde a
emissão, sem renovação por atividade.

### Falhas

| Status | Condição | Corpo |
|--------|----------|-------|
| `422` | Email, senha ou `X-Device-ID` ausente/inválido | `{"message":"Os dados informados são inválidos.","errors":{"campo":["..."]}}` |
| `401` | Email inexistente ou senha incorreta | `{"message":"Não foi possível autenticar com as credenciais informadas."}` |
| `429` | Identificador está bloqueado | `{"message":"Muitas tentativas de login. Tente novamente mais tarde.","retry_after_seconds":900}` |

Cinco falhas de credenciais dentro de qualquer janela móvel de 15 minutos iniciam
bloqueio de 15 minutos. A quinta falha ainda recebe `401`; as próximas tentativas
recebem `429`. O valor de `retry_after_seconds` é o tempo restante arredondado para
cima. Durante o bloqueio, credenciais certas e erradas recebem o mesmo `429`, e
nenhum token é criado. Tentativas durante o bloqueio não estendem o prazo original.

Uma solicitação com payload/header inválido retorna `422` e não conta como falha de
credenciais. Uma autenticação bem-sucedida não apaga falhas recentes do identificador;
elas deixam de contar ao sair da janela móvel.

## `GET /api/me`

**Autenticação**: `Authorization: Bearer <token>`.

### Sucesso — `200 OK`

```json
{
  "data": {
    "id": 1,
    "name": "Ana Silva",
    "email": "ana@example.com"
  }
}
```

### Falha — `401 Unauthorized`

```json
{
  "message": "Não autenticado."
}
```

## `POST /api/logout`

**Autenticação**: `Authorization: Bearer <token>`.

### Sucesso — `200 OK`

```json
{
  "data": {
    "message": "Sessão encerrada."
  }
}
```

Revoga somente o token apresentado. Chamadas posteriores com esse token recebem
`401`; outros tokens do mesmo usuário permanecem válidos.

### Falha — `401 Unauthorized`

Mesmo corpo JSON de `GET /api/me`; token ausente, inválido, expirado ou revogado não
encerra outra sessão.

## Formato de validação

```json
{
  "message": "Os dados informados são inválidos.",
  "errors": {
    "email": ["Informe um endereço de email válido."]
  }
}
```

Os nomes dos campos seguem os nomes do payload e o cabeçalho obrigatório é indicado
como `X-Device-ID`. Mensagens podem ser localizadas, mas a estrutura, o status e a
regra de não exposição de dados permanecem consistentes.
