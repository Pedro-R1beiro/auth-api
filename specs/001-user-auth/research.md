# Pesquisa técnica: autenticação de usuários pela API

**Feature**: [spec.md](./spec.md)  
**Data**: 2026-10-22

## Decisões

### Laravel 13, PHP 8.4 e dependências

- **Decisão**: Manter Laravel 13 e PHP 8.4. Restringir o requisito PHP do Composer a
  `^8.4`, conforme a stack escolhida; o Dockerfile já usa `php:8.4-cli-bookworm`.
- **Racional**: O lockfile contém Laravel 13 e PHPUnit 12, compatíveis com essa versão.
  A imagem já instala `pdo_pgsql`, necessária ao banco selecionado.
- **Alternativas consideradas**: Trocar a imagem ou framework; não há motivo técnico
  nem benefício que justifique essa alteração.
- **Fontes**: [Laravel 13: versões e suporte](https://laravel.com/docs/13.x/releases);
  [versões PHP suportadas](https://www.php.net/supported-versions.php).

### Tokens Bearer e expiração

- **Decisão**: Usar Laravel Sanctum 4 para emitir e revogar personal access tokens,
  protegendo rotas com `auth:sanctum`. Configurar expiração global de 60 minutos,
  sem renovar o prazo quando o token é usado.
- **Racional**: O Sanctum é compatível com Laravel 13/PHP 8.4, armazena o segredo do
  token em hash e fornece revogação do token atual. A expiração global corresponde à
  política única de uma hora aprovada na especificação.
- **Alternativas consideradas**: JWT ou tokens próprios; rejeitados para evitar
  implementar emissão, armazenamento ou revogação manual. Expiração renovável por
  atividade; rejeitada por contradizer a especificação.
- **Fontes**: [requisitos do Sanctum 4.x](https://github.com/laravel/sanctum/blob/4.x/composer.json);
  [emissão de tokens](https://laravel.com/docs/13.x/sanctum#issuing-api-tokens);
  [expiração](https://laravel.com/docs/13.x/sanctum#token-expiration);
  [revogação](https://laravel.com/docs/13.x/sanctum#revoking-tokens).

### Bloqueio por identificador de dispositivo

- **Decisão**: Persistir falhas e prazo de bloqueio no PostgreSQL, indexando o hash do
  UUID do cabeçalho `X-Device-ID`. Usar transação e bloqueio da linha do dispositivo
  para serializar requisições concorrentes que compartilham o identificador.
- **Racional**: A regra exige janela móvel exata de 15 minutos, bloqueio global entre
  contas no mesmo dispositivo e horário de desbloqueio não prorrogável. Um contador
  simples com TTL não representa necessariamente o histórico de falhas dessa janela.
  Estado em memória local não coordena várias instâncias. O PostgreSQL já é a
  infraestrutura obrigatória e oferece transações e bloqueio pessimista.
- **Alternativas consideradas**: RateLimiter/cache padrão; insuficiente sozinho para
  modelar janela móvel exata e bloqueio imutável. Redis; tecnicamente possível, mas
  adiciona serviço não aprovado nem presente no Compose.
- **Fontes**: [transações Laravel](https://laravel.com/docs/13.x/database#database-transactions);
  [bloqueio pessimista](https://laravel.com/docs/13.x/queries#pessimistic-locking);
  [RateLimiter Laravel](https://laravel.com/docs/13.x/rate-limiting).

### Identificador e privacidade

- **Decisão**: Validar e canonicalizar o UUID enviado em `X-Device-ID`; armazenar e
  indexar seu SHA-256, não o UUID em claro. Nunca incluir UUID nem hash em logs ou
  respostas.
- **Racional**: UUID tem entropia suficiente para que o hash simples preserve a
  contagem sem persistir o valor fornecido diretamente. O dado derivado ainda é
  pseudônimo e requer controle de acesso.
- **Alternativas consideradas**: IP observado pelo servidor; redes compartilhadas
  bloqueiam usuários diferentes e não correspondem à especificação. UUID bruto;
  desnecessário para contagem e expõe um identificador estável. HMAC; pode ser
  considerado se a gestão de chave e rotação se justificar.
- **Risco aceito**: O cliente pode substituir o UUID e contornar o limite, ou
  reutilizar um UUID para provocar bloqueio de outras contas. O contrato aprovado
  não prova a identidade física do dispositivo. Alterá-lo para dispositivo registrado
  requer revisão da especificação.
- **Fontes**: [RFC 9562: UUID](https://www.rfc-editor.org/rfc/rfc9562.html);
  [tipo UUID no PostgreSQL 17](https://www.postgresql.org/docs/17/datatype-uuid.html);
  [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html#authentication-and-error-messages).

### PHPUnit com PostgreSQL e isolamento

- **Decisão**: Usar PHPUnit e testar persistência/middleware contra PostgreSQL 17
  isolado. Criar Compose de teste independente, com armazenamento efêmero; usar um
  serviço PostgreSQL também efêmero no CI. Usar `RefreshDatabase` e não executar
  `migrate:fresh` no banco persistente de desenvolvimento.
- **Racional**: A configuração PHPUnit atual força SQLite em memória, o que não
  verifica a semântica real do PostgreSQL nem seus locks. O PostgreSQL já é a
  dependência de runtime aprovada.
- **Alternativas consideradas**: SQLite para toda a suíte; rejeitado para os testes
  de integração que dependem de transação, índices e locking PostgreSQL. Banco de
  desenvolvimento compartilhado; rejeitado por risco de apagar ou contaminar dados.
- **Fontes**: [Laravel: bancos suportados](https://laravel.com/docs/13.x/database#introduction);
  [serviços PostgreSQL em GitHub Actions](https://docs.github.com/en/actions/tutorials/use-containerized-services/create-postgresql-service-containers);
  [ordem de inicialização e healthcheck no Compose](https://docs.docker.com/compose/how-tos/startup-order/).

### Análise estática

- **Decisão**: Incluir Larastan 3/PHPStan 2 como dependência de desenvolvimento e
  configurar análise do código `app/` no nível 5, sem gerar baseline
  automaticamente.
- **Racional**: O Larastan 3 declara suporte a PHP 8.2+ e componentes Illuminate 13;
  a integração é pequena (dependência e configuração), adequada ao pedido condicional.
  O CI torna sua execução verificável.
- **Alternativas consideradas**: PHPStan sem extensão Laravel, que perde compreensão
  de container, Eloquent e facades; omitir análise estática, contrário à decisão
  técnica informada.
- **Fontes**: [Larastan 3.x composer.json](https://github.com/larastan/larastan/blob/3.x/composer.json);
  [Larastan](https://github.com/larastan/larastan).

### CI e contexto Docker

- **Decisão**: Criar GitHub Actions com PHP 8.4, `pdo_pgsql`, PostgreSQL 17 healthcheck,
  testes PHPUnit, Pint em modo de verificação e PHPStan. Adicionar `.dockerignore`
  excluindo `.env` e artefatos locais.
- **Racional**: Não há workflow atual, e o Dockerfile usa `COPY . .`; sem exclusões, o
  `.env` local pode entrar no contexto/imagem. O workflow cobre os gates de qualidade
  exigidos pela constituição.
- **Alternativas consideradas**: Executar testes contra o volume Compose persistente;
  rejeitado por isolamento e proteção de dados. Deixar segredos de produção no CI;
  desnecessário para testes.
- **Fontes**: [serviços PostgreSQL em GitHub Actions](https://docs.github.com/en/actions/tutorials/use-containerized-services/create-postgresql-service-containers);
  [Docker build context](https://docs.docker.com/build/concepts/context/).

## Decisões fora do escopo

- Não adicionar throttling secundário por IP ou por conta nesta implementação. Pode
  reduzir abuso por troca de UUID, mas altera comportamento, pode causar bloqueio
  compartilhado e precisa de aprovação como requisito adicional.
- Não incluir agendamento de limpeza de PATs ou de falhas até haver necessidade
  operacional demonstrada; limpeza de registros expirados pode ser adicionada sem
  alterar autenticação, desde que não afete o volume persistente sem autorização.
