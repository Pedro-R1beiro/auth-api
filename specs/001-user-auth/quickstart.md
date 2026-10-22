# Guia de validação: autenticação de usuários pela API

**Pré-requisitos**: Docker com Docker Compose; arquivo local `.env` configurado para
o Compose de desenvolvimento quando necessário; acesso ao repositório e dependências
Composer disponíveis dentro da imagem PHP 8.4.

## Banco de teste isolado

O serviço de teste fica em `docker-compose.testing.yml`, usa PostgreSQL 17 e não
compartilha o volume persistente `postgres_data` do Compose de desenvolvimento.

```powershell
docker compose -f docker-compose.testing.yml up -d --wait db-test
docker compose -f docker-compose.testing.yml run --build --rm app-test php artisan test
docker compose -f docker-compose.testing.yml down
```

Os comandos iniciam o banco de teste efêmero, esperam seu healthcheck, executam PHPUnit
no container PHP 8.4 e removem o container/rede de teste. O banco é exclusivo da
execução/testes; não usar `migrate:fresh` no banco configurado no Compose de
desenvolvimento. Para executar várias verificações no mesmo banco efêmero, mantenha-o
em execução e execute os comandos de validação antes de chamar `down`:

```powershell
docker compose -f docker-compose.testing.yml up -d --wait db-test
docker compose -f docker-compose.testing.yml run --build --rm app-test vendor/bin/pint --test
docker compose -f docker-compose.testing.yml run --build --rm app-test vendor/bin/phpstan analyse --no-progress
docker compose -f docker-compose.testing.yml down
```

`down` remove somente os containers e a rede do Compose de teste, que não usa volume
nomeado;
não usar `down -v` e não aplicar esses comandos ao Compose de desenvolvimento.

## Verificações de qualidade

Executar dentro do mesmo container PHP de teste:

```powershell
docker compose -f docker-compose.testing.yml run --build --rm app-test vendor/bin/pint --test
docker compose -f docker-compose.testing.yml run --build --rm app-test vendor/bin/phpstan analyse --no-progress
```

Resultados esperados: PHPUnit sem falhas, Pint sem arquivos fora do padrão e PHPStan
sem erros dentro do escopo configurado.

## Validação manual do contrato

Com o serviço local iniciado conforme o README e a base configurada para PostgreSQL:

1. `POST /api/register` com nome, email, senha e `password_confirmation` válidos:
   esperar `201`, usuário público e token; conferir que senha/hash não aparecem.
2. `POST /api/login` com `X-Device-ID` contendo UUID válido e credenciais corretas:
   esperar `200` com mesmo envelope de sucesso e token.
3. `GET /api/me` com `Authorization: Bearer <token>`: esperar `200` somente com
   `id`, `name` e `email`.
4. `POST /api/logout` com o token: esperar `200`; repetir `/api/me` com o mesmo token
   e esperar `401`.
5. Fazer quatro logins inválidos com o mesmo UUID: cada tentativa deve retornar
   `401`; a quinta também retorna `401` e ativa o bloqueio.
6. Tentar novamente com esse UUID: esperar `429` com `retry_after_seconds`; credenciais
   corretas e incorretas têm o mesmo resultado. Confirmar que outro UUID não está
   bloqueado e que novas tentativas não prorrogam o prazo.
7. Validar campo ausente, UUID malformado, email duplicado, token expirado e token
   revogado conforme [contrato HTTP](./contracts/auth-api.md).

Os testes automatizados devem usar fixtures/factories e migrations em banco isolado,
sem depender de dados ou estado da base de desenvolvimento. Testes de janela móvel e
expiração devem controlar o relógio; não aguardar em tempo real.

## CI

O workflow `.github/workflows/ci.yml` deve executar com PHP 8.4 e PostgreSQL 17
efêmero:

1. Instalar extensões necessárias, incluindo `pdo_pgsql`, e executar `composer install`.
2. Aguardar healthcheck PostgreSQL e executar migrations de teste.
3. Executar testes PHPUnit.
4. Executar `vendor/bin/pint --test`.
5. Executar `vendor/bin/phpstan analyse --no-progress`.

O job não deve receber segredos de produção nem se conectar a volumes persistentes.
