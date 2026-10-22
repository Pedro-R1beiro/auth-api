# Modelo de dados: autenticação de usuários pela API

**Fonte**: [especificação](./spec.md)  
**Decisões de implementação**: [pesquisa técnica](./research.md)

## Usuário (`users`)

Tabela Laravel existente, mantida e estendida somente por comportamento/modelo.

| Campo | Tipo lógico | Regras |
|-------|-------------|--------|
| `id` | Identificador | Chave primária existente. |
| `name` | Texto | Obrigatório e não vazio no cadastro. |
| `email` | Email normalizado | Remover espaços externos e converter para minúsculas antes de persistir; único após normalização. |
| `password` | Hash de senha | Obrigatório; gerar com o cast/mecanismo de hashing do Laravel; nunca expor. |
| `email_verified_at` | Data/hora anulável | Mantido pelo schema existente; verificação de email fora do escopo. |
| `remember_token` | Token interno anulável | Mantido pelo schema; não expor pela API. |
| `created_at`, `updated_at` | Data/hora | Timestamps existentes. |

O modelo atual já oculta `password` e `remember_token` e aplica o cast `hashed` à
senha. Acrescentar `HasApiTokens` do Sanctum sem remover essas proteções.

## Token pessoal (`personal_access_tokens`)

Criado pela migration oficial publicada pelo Sanctum.

| Campo | Tipo lógico | Regras |
|-------|-------------|--------|
| `id` | Identificador | Chave primária do pacote. |
| `tokenable_type`, `tokenable_id` | Relação polimórfica | Associa o token ao usuário; indexada pelo pacote. |
| `name` | Texto | Rótulo não sensível do token; não guardar credenciais. |
| `token` | Hash | Hash do token, nunca valor Bearer em claro. |
| `abilities` | JSON | Escopos do token; primeira versão sem permissões avançadas. |
| `last_used_at` | Data/hora anulável | Uso mais recente, conforme o pacote. |
| `expires_at` | Data/hora anulável | Quando aplicável; além disso, a política global do Sanctum recusa tokens com mais de 60 minutos desde a emissão. |
| `created_at`, `updated_at` | Data/hora | Timestamps do pacote. |

**Ciclo de vida**: o token é emitido apenas após cadastro/login válidos; expira em 60
minutos, não tem renovação por atividade e é revogado individualmente no logout. A
revogação remove/invalida o token atual sem alterar outros tokens do usuário.

## Dispositivo de login (`login_devices`)

Uma linha por UUID canonicalizado, representada internamente por seu hash SHA-256.

| Campo | Tipo lógico | Regras |
|-------|-------------|--------|
| `device_hash` | String hexadecimal de 64 caracteres | Chave primária; hash SHA-256 do UUID canônico do cabeçalho `X-Device-ID`; não armazenar UUID em claro. |
| `blocked_until` | Data/hora anulável | Fim do bloqueio ativo; definido uma única vez ao alcançar cinco falhas e não prorrogado por tentativas subsequentes. |
| `created_at`, `updated_at` | Data/hora | Timestamps de auditoria não sensíveis. |

**Regras de estado**:

- Sem bloqueio: solicitações válidas podem autenticar e as falhas na janela móvel são
  avaliadas.
- Bloqueado: se `blocked_until` estiver no futuro, todos os logins com o identificador
  recebem `429`, sem autenticar nem criar token.
- Liberado: ao atingir `blocked_until`, remover o estado expirado e reiniciar a
  contagem aplicável.
- A linha é a unidade de sincronização para serializar logins concorrentes com o mesmo
  identificador; não serializa identificadores diferentes.

## Falha de login (`login_attempts`)

Registra exclusivamente falhas de credenciais válidas para a janela móvel.

| Campo | Tipo lógico | Regras |
|-------|-------------|--------|
| `id` | Inteiro grande | Chave primária. |
| `device_hash` | String hexadecimal de 64 caracteres | Chave estrangeira para `login_devices.device_hash`; índice composto com `failed_at`. |
| `failed_at` | Data/hora com fuso | Instante da falha de credenciais válidas; timestamp usado para selecionar as falhas dos 15 minutos anteriores. |
| `created_at`, `updated_at` | Data/hora | Timestamps convencionais, se mantidos pelo padrão das migrations. |

Não registrar email, senha, token, endereço IP ou UUID bruto nessa tabela.

**Regra de contagem**: consultar primeiro se já existe bloqueio ativo para o
dispositivo; nesse caso, responder `429` sem verificar credenciais nem alterar estado.
Caso contrário, verificar as credenciais e então, dentro de uma transação curta,
bloquear a linha do dispositivo e revalidar `blocked_until`. Se outra requisição tiver
ativado o bloqueio nesse intervalo, responder `429`; caso contrário, emitir o token
válido ou registrar a falha. Para credenciais inválidas, contar eventos dentro da
janela móvel de 15 minutos; a quinta falha inicia bloqueio de 15 minutos e responde com
o erro genérico de credenciais (`401`). As tentativas seguintes recebem `429` até
`blocked_until`. Tentativas bloqueadas não alteram o horário. Um login bem-sucedido
não apaga eventos recentes; cada evento deixa de contar naturalmente ao sair da janela
móvel.

## Integridade e concorrência

- Normalizar o email no limite HTTP antes de consultar ou gravar; manter a restrição
  `UNIQUE` existente como defesa final contra duplicidade concorrente.
- Verificar as credenciais fora de uma transação longa e, antes de registrar a falha
  ou emitir token, iniciar uma transação curta, obter/bloquear a linha do dispositivo
  e revalidar se há bloqueio ativo. Isso evita manter locks durante o hashing lento da
  senha sem permitir que logins concorrentes ultrapassem o limite.
- Fazer gravação da quinta falha e `blocked_until` na mesma transação para que outra
  requisição não passe pelo limite entre verificação e gravação.
- A criação concorrente inicial da linha deve tratar conflito da chave única por
  operação idempotente/upsert e, em seguida, adquirir o bloqueio da linha existente.
- Limpeza de eventos fora da janela e dispositivos sem atividade/bloqueio pode ser
  feita durante a operação ou em manutenção; não executar limpeza destrutiva dos
  dados de desenvolvimento.

## Relações

- Um usuário pode ter vários tokens pessoais ativos, cada um revogável separadamente.
- Um UUID de dispositivo é agrupado sem associação exclusiva a um usuário; as falhas
  e o bloqueio se aplicam a qualquer conta que use esse UUID.
- Um dispositivo tem zero ou mais eventos de falha e no máximo um bloqueio ativo.
