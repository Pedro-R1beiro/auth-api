# Implementation Plan: Autenticação de usuários pela API

**Branch**: `001-user-auth` | **Date**: 2026-10-22 | **Spec**: [spec.md](./spec.md)

**Input**: Especificação em `specs/001-user-auth/spec.md` e decisões técnicas fornecidas
pelo usuário.

## Summary

Implementar cadastro, login, consulta da identidade e logout com tokens Bearer do
Laravel Sanctum, incluindo respostas JSON consistentes e proteção contra força bruta
por dispositivo. Usar Laravel 13 com PHP 8.4, PostgreSQL 17 e migrations; persistir
falhas de login no PostgreSQL para cumprir a janela móvel, o bloqueio compartilhado
entre contas e a concorrência. Executar testes HTTP de integração no PostgreSQL
isolado, formatar com Laravel Pint e analisar com Larastan/PHPStan. Manter o Compose de
desenvolvimento existente; acrescentar somente os arquivos de configuração necessários
para testes isolados, proteção do contexto Docker e CI.

## Technical Context

**Language/Version**: PHP 8.4; Laravel 13, conforme `composer.json` e Dockerfile
existentes.

**Primary Dependencies**: Laravel Sanctum 4 para personal access tokens; Laravel
Framework 13; Larastan 3/PHPStan 2 para análise estática, compatíveis com o framework.

**Storage**: PostgreSQL 17. As falhas por dispositivo e o prazo de bloqueio serão
persistidos no banco para compartilhar o estado entre processos e instâncias.

**Testing**: PHPUnit 12 existente; testes de feature HTTP com banco PostgreSQL de
teste isolado e migrations. Laravel Pint existente. Larastan/PHPStan no CI.

**Target Platform**: Serviço HTTP Laravel em container Linux PHP 8.4; execução local
via Docker Compose e integração contínua no GitHub Actions.

**Project Type**: API REST Laravel de projeto único.

**Performance Goals**: Não há meta de latência ou throughput aprovada na especificação.
O fluxo de login acrescentará consultas indexadas e uma transação curta por
identificador de dispositivo.

**Constraints**: Respeitar os contratos, os códigos HTTP e a política de bloqueio da
especificação. Os tokens expiram após 60 minutos sem renovação por atividade. Não
adicionar funcionalidades fora do escopo. Não usar a base de desenvolvimento nem
apagar volumes persistentes para executar testes.

**Scale/Scope**: Primeira versão, sem volume de usuários ou meta de concorrência
definidos. A regra de bloqueio deve ser correta entre requisições simultâneas e
processos da aplicação.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Princípio | Avaliação |
|-----------|-----------|
| Segurança em primeiro lugar | Bloqueado: a constituição proíbe expor tokens em qualquer resposta, enquanto FR-006/FR-007 exigem retornar o token no sucesso de cadastro/login. O desenho só poderá passar após alinhar formalmente a constituição e a especificação. |
| Arquitetura Laravel simples e modular | Passa: controllers HTTP enxutos, Form Requests, Resources e um serviço focado no fluxo de tentativas concorrentes. Sem repositórios ou interfaces genéricas. |
| Qualidade verificável | Passa: testes HTTP com PostgreSQL isolado cobrem sucesso, falhas, validação, bloqueio, expiração e revogação; Pint e PHPStan serão gates de CI. |
| Contratos de API consistentes | Passa: respostas, status e campos estão descritos em `contracts/auth-api.md`. |
| Especificação e documentação em pt-BR | Passa: o plano, os contratos e os guias deste recurso serão escritos em português brasileiro; os identificadores literais do protocolo permanecem inalterados. |
| PostgreSQL e integridade | Passa: migrations versionadas, índices e transação PostgreSQL para estado e tentativas de login. |
| Docker, segredos e CI | Passa após adicionar contexto Docker que exclui `.env`, serviço PostgreSQL efêmero de teste e pipeline de CI. O Compose de desenvolvimento e seu volume persistente não serão apagados nem substituídos. |
| Escopo aprovado | Pendente do gate de segurança: não incluir recuperação de senha, verificação de email, refresh, login social, MFA, gestão avançada de permissões ou frontend. Throttling adicional por IP é registrado como risco, não como requisito novo. |

## Research Decisions

As decisões e alternativas técnicas estão em [research.md](./research.md). A pesquisa
resolveu a compatibilidade de Sanctum e Larastan com a stack, a expiração dos tokens,
o uso de PostgreSQL para a janela móvel e o isolamento de testes.

## Project Structure

### Documentation (this feature)

```text
specs/001-user-auth/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   └── auth-api.md
├── checklists/
│   └── requirements.md
└── tasks.md                 # Não criado nesta etapa; depende da revisão do plano
```

### Source Code (repository root)

```text
app/
├── Http/
│   ├── Controllers/Api/AuthController.php
│   ├── Requests/Auth/
│   │   ├── RegisterRequest.php
│   │   └── LoginRequest.php
│   └── Resources/
│       └── UserResource.php
├── Models/
│   ├── User.php
│   ├── LoginDevice.php
│   └── LoginAttempt.php
└── Services/Auth/
    └── LoginAttemptService.php

bootstrap/app.php
config/
├── auth.php
├── database.php
└── sanctum.php
database/migrations/
├── 2026_10_22_000001_create_personal_access_tokens_table.php
├── 2026_10_22_000002_create_login_devices_table.php
└── 2026_10_22_000003_create_login_attempts_table.php
routes/
└── api.php
tests/
└── Feature/Api/
    ├── RegisterTest.php
    ├── LoginTest.php
    ├── LoginLockoutTest.php
    ├── MeTest.php
    └── LogoutTest.php
.dockerignore
.github/workflows/ci.yml
docker-compose.testing.yml
phpstan.neon
composer.json
composer.lock
phpunit.xml
.env.example
README.md
```

**Structure Decision**: Aplicação Laravel única, mantendo os diretórios idiomáticos já
presentes. `AuthController` coordena endpoints; Form Requests validam e normalizam
entradas; `UserResource` limita os dados públicos; `LoginAttemptService` contém somente
a regra com estado concorrente. Os Models Eloquent representam estado persistido.
Adicionar `routes/api.php` e registrar o arquivo na configuração moderna em
`bootstrap/app.php`. Não criar camadas Repository/Interface nem serviço genérico de
respostas. O `docker-compose.yml` e o Dockerfile atuais já selecionam PHP 8.4 e
PostgreSQL 17; mantê-los sem alteração, salvo descoberta de bloqueio durante a
implementação.

### Arquivos a criar ou alterar

| Arquivo | Mudança planejada |
|---------|-------------------|
| `composer.json`, `composer.lock` | Adicionar Sanctum e Larastan como dependências; restringir o runtime ao PHP 8.4 conforme a stack aprovada. |
| `bootstrap/app.php`, `routes/api.php` | Registrar rotas API, middleware Sanctum e renderização JSON das falhas da API. |
| `config/sanctum.php` | Definir expiração de PATs em 60 minutos. |
| `config/database.php`, `.env.example` | Tornar PostgreSQL a conexão padrão/documentada, coerente com a constituição. |
| `app/Models/User.php` | Adicionar `HasApiTokens`; manter `password` oculto e o cast de hash já existente. |
| `app/Http/Controllers/Api/AuthController.php` | Expor cadastro, login, identidade e logout, delegando regra de bloqueio. |
| `app/Http/Requests/Auth/*` | Validar payloads, confirmação de senha, email normalizado e UUID no `X-Device-ID`. |
| `app/Http/Resources/UserResource.php` | Serializar apenas `id`, `name` e `email`; incluir o token explicitamente somente no envelope de cadastro/login. |
| `app/Models/LoginDevice.php`, `app/Models/LoginAttempt.php`, `app/Services/Auth/LoginAttemptService.php` | Persistir e aplicar o bloqueio por identificador de dispositivo de forma transacional. |
| `database/migrations/*` | Criar PATs Sanctum, estado de bloqueio por dispositivo e histórico indexado de falhas. |
| `tests/Feature/Api/*` | Cobrir contrato HTTP e persistência real em PostgreSQL. |
| `phpunit.xml` | Remover a dependência implícita de SQLite e usar configuração explícita de PostgreSQL para testes. |
| `docker-compose.testing.yml` | Criar serviço de aplicação e PostgreSQL efêmero, separado do banco persistente de desenvolvimento. |
| `.dockerignore` | Excluir `.env`, `.git`, `vendor`, `node_modules` e artefatos locais do contexto de build. |
| `.github/workflows/ci.yml` | Adicionar gates para instalação, migrações/testes PostgreSQL, Pint e PHPStan. |
| `phpstan.neon` | Configurar Larastan para analisar `app/` no nível 5, sem baseline automático. |
| `specs/001-user-auth/contracts/auth-api.md` | Registrar os contratos de requisição e resposta para implementação e revisão. |
| `README.md` | Documentar inicialização da API, endpoints de autenticação e comandos de testes/qualidade. |

### Plano de migrações

1. Publicar a migration oficial do Sanctum para `personal_access_tokens`; preservar o
   formato e os índices definidos pelo pacote.
2. Criar `login_devices`, chaveado pelo SHA-256 do UUID canônico, com `blocked_until`
   nullable e timestamps. Não persistir o UUID em claro.
3. Criar `login_attempts` com referência ao hash do dispositivo e `failed_at`, mais
   índice composto `(device_hash, failed_at)` para a janela móvel de 15 minutos.
4. Remover eventos expirados de forma segura durante o fluxo ou por limpeza dirigida;
   não executar `migrate:fresh` contra o banco persistente de desenvolvimento.

## Implementation Outline

1. Incluir Sanctum, configurar PATs e aplicar migrations versionadas.
2. Construir requests, Resources e endpoints JSON para cadastro, login, `/api/me` e
   logout; usar `auth:sanctum` nas rotas protegidas. Mapear a violação da restrição
   única do email concorrente para a resposta `409` definida no contrato.
3. Implementar o controle de tentativas no PostgreSQL: normalizar/hash do UUID,
   garantir e bloquear a linha do dispositivo dentro de transação, contar falhas da
   janela móvel, ativar bloqueio ao atingir cinco falhas e preservar o prazo original.
4. Adicionar os testes de feature com banco PostgreSQL dedicado, incluindo concorrência
   de requisições que compartilham o mesmo UUID.
5. Adicionar Pint, Larastan/PHPStan, CI e orientações de validação; corrigir o contexto
   Docker para não incluir configuração local ou segredo.

## Operational Commands

Os comandos reproduzíveis e resultados esperados estão em
[quickstart.md](./quickstart.md). Os testes usam o serviço efêmero de
`docker-compose.testing.yml`, nunca o volume persistente `postgres_data`.

## Security Decisions and Risks

- Sanctum armazena o segredo do PAT em hash; a aplicação o devolve somente na emissão
  bem-sucedida. O middleware recusa token revogado e token com mais de 60 minutos.
- **Bloqueio de governança**: retornar um token Bearer em cadastro/login conflita com a
  proibição literal da constituição de expor tokens em respostas. Não prosseguir para
  tarefas ou implementação até a constituição e a especificação serem alinhadas e a
  revisão confirmar qual comportamento foi aprovado.
- Não registrar senha, token, UUID bruto nem payloads de autenticação. O hash do UUID
  continua sendo dado pseudônimo e deve ter acesso restrito.
- Como `X-Device-ID` é controlado pelo cliente, pode ser trocado para contornar o
  limite ou reutilizado para provocar bloqueio de outras contas. O contrato aprovado
  bloqueia todas as contas com o mesmo UUID; reduzir esse risco exigiria revisão da
  especificação. Não tratar o UUID como identidade confiável do hardware.
- O estado por dispositivo deve ser sincronizado com transação/lock PostgreSQL;
  armazenamento em memória local não cumpre o comportamento em várias instâncias.
- Não foi aprovado limite adicional por IP/conta. Esse controle pode reduzir o abuso
  por troca de UUID, mas também altera a política observável e requer revisão de escopo.
- O Compose de desenvolvimento monta o repositório e mantém `postgres_data`. O build
  atual usa `COPY . .`; o `.dockerignore` é necessário para impedir que `.env` local
  entre na imagem. Não executar comandos que removam volumes ou limpem dados locais.
- A pipeline usa serviço PostgreSQL efêmero; segredos de produção não serão definidos
  na pipeline de teste.

## Constitution Check (post-design)

| Gate | Resultado |
|------|-----------|
| Requisitos rastreáveis à especificação | Passa; os contratos e entidades são derivados de `spec.md`, sem requisito funcional novo. |
| PostgreSQL, migrations e integridade | Passa; PostgreSQL 17 é o banco de execução e integração; schema via migrations; lockout atômico. |
| Segurança, validação e respostas previsíveis | Passa; Sanctum, hashes, validação, serialização pública explícita e erros JSON documentados. |
| Simplicidade e convenções Laravel | Passa; divisão em Requests/Resources/Controller/um serviço focado, sem abstração genérica. |
| Testes antes da conclusão | Passa; PHPUnit executado em PostgreSQL isolado, além de Pint e PHPStan no CI. |
| Docker, segredos e volumes | Passa com `.dockerignore` e Compose isolado de teste; o volume de desenvolvimento é preservado. |
| Documentação em pt-BR | Passa; os artefatos e a documentação dos endpoints serão em pt-BR, mantendo nomes de protocolo literais. |
| Limites de escopo | Passa; login social, recuperação de senha, refresh, MFA, perfis avançados e frontend não serão implementados. |

## Complexity Tracking

Nenhuma violação da constituição identificada. A tabela de tentativas existe para
implementar a janela móvel e o bloqueio concorrente especificados; Redis e camadas
genéricas foram rejeitados por adicionarem infraestrutura ou abstração não necessária.
