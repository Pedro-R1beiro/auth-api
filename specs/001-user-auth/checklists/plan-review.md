# Revisão do plano: autenticação de usuários pela API

**Purpose**: Avaliar a qualidade, a consistência e a prontidão dos requisitos e decisões documentados no plano técnico.
**Created**: 2026-10-22
**Feature**: [plan.md](../plan.md)

**Note**: Esta checklist personalizada valida a qualidade dos requisitos e do desenho documentado; não verifica se a implementação funciona.
**Review Ownership**: Esta checklist pertence ao revisor. Marque um item `[x]` somente após avaliar e aprovar a qualidade do requisito correspondente.
**Marker Semantics**: `[x]` significa que o critério de qualidade do requisito foi revisado e satisfeito; não significa que a implementação foi concluída.

## Completude dos requisitos

- [ ] CHK001 - O plano relaciona cada endpoint e responsabilidade de entrada/serialização aos requisitos funcionais correspondentes? [Rastreabilidade, Plan §Project Structure, Spec §Functional Requirements]
- [ ] CHK002 - A lista de arquivos a criar ou alterar cobre todos os elementos mencionados na estrutura, migrações, CI, testes e documentação? [Completude, Plan §Project Structure]
- [ ] CHK003 - O plano define como serão preservados dados existentes ao tornar PostgreSQL a conexão padrão, sem presumir que o banco SQLite atual esteja vazio? [Dependência, Plan §Technical Context, Spec §Restrições técnicas]
- [ ] CHK004 - As responsabilidades de retenção e limpeza dos registros de tentativas e dispositivos estão definidas com limites compatíveis com o volume de dados esperado? [Gap, Plan §Plano de migrações, Data Model §Integridade e concorrência]
- [ ] CHK005 - O plano distingue claramente os testes de integração que exigem PostgreSQL daqueles que podem usar outro banco, se houver? [Clareza, Plan §Technical Context, Quickstart §Banco de teste isolado]

## Clareza do desenho

- [ ] CHK006 - A sequência entre validar formato, verificar bloqueio, validar credenciais, adquirir lock, registrar falha e emitir token está definida sem deixar uma janela de corrida? [Clareza, Plan §Implementation Outline, Data Model §Regra de contagem]
- [ ] CHK007 - Está definido como a quinta falha, exatamente no limite da janela móvel de 15 minutos, inicia o bloqueio sem permitir autenticações concorrentes acima do limite? [Clareza, Spec §FR-020, Plan §Implementation Outline]
- [ ] CHK008 - A especificação da resposta `429` define com precisão a origem, arredondamento e valor mínimo de `retry_after_seconds` até o horário original de desbloqueio? [Clareza, Spec §FR-024, Contract §POST /api/login]
- [ ] CHK009 - Está claro como a expiração de 60 minutos é aplicada pelo Sanctum mesmo quando `expires_at` está vazio, e como isso se relaciona à limpeza de tokens expirados? [Clareza, Spec §FR-016, Research §Tokens e expiração, Data Model §Token pessoal]
- [ ] CHK010 - A regra para falha do PostgreSQL durante autenticação ou atualização do bloqueio define se o login falha de forma segura e qual resposta JSON é esperada? [Gap, Spec §Functional Requirements, Plan §Security Decisions and Risks]

## Consistência entre artefatos

- [ ] CHK011 - A avaliação do gate constitucional no plano foi atualizada para a versão 1.1.0 e remove a afirmação de que a devolução única do token conflita com a constituição? [Conflict, Plan §Constitution Check, Constitution §I]
- [ ] CHK012 - Os status HTTP, envelopes JSON, campos públicos e erros do contrato correspondem sem divergências à especificação? [Consistency, Spec §FR-006–FR-015, Contract §Convenções]
- [ ] CHK013 - A regra de que login bem-sucedido não apaga falhas recentes aparece de forma consistente na especificação, no modelo de dados, no contrato e nos critérios de validação? [Consistency, Spec §FR-022, Data Model §Regra de contagem, Contract §POST /api/login]
- [ ] CHK014 - O prazo de bloqueio não prorrogável e o tratamento de tentativas durante o bloqueio são consistentes em todos os artefatos? [Consistency, Spec §FR-024/FR-028, Contract §POST /api/login, Quickstart §Validação manual]
- [ ] CHK015 - O requisito de teste com banco isolado protege explicitamente o volume de desenvolvimento e evita comandos destrutivos ou uso acidental da base persistente? [Consistency, Constitution §Restrições técnicas e operacionais, Plan §Operational Commands, Quickstart §Banco de teste isolado]

## Qualidade dos critérios de aceitação

- [ ] CHK016 - Os critérios de sucesso definem resultados verificáveis para bloqueio, liberação, concorrência, revogação e expiração sem depender de detalhes internos? [Measurabilidade, Spec §Success Criteria, Plan §Implementation Outline]
- [ ] CHK017 - O plano define metas ou declara explicitamente que latência, volume e disponibilidade não são critérios de aceite desta versão? [Clareza, Spec §Success Criteria, Plan §Technical Context]
- [ ] CHK018 - Os gates de CI estão descritos com comandos, pré-condições e condições de sucesso consistentes com o ambiente PHP 8.4/PostgreSQL 17? [Completude, Plan §Operational Commands, Quickstart §CI]

## Cobertura de cenários e falhas

- [ ] CHK019 - Os cenários cobrem identificador ausente/malformado, troca de UUID, compartilhamento entre contas, credenciais válidas durante bloqueio e identificadores distintos? [Cobertura, Spec §Edge Cases, Spec §FR-019–FR-027]
- [ ] CHK020 - Os requisitos esclarecem se o bloqueio por UUID controlado pelo cliente é proteção suficiente apesar de permitir troca do identificador e bloqueio provocado contra outras contas? [Risco, Spec §Assumptions, Research §Identificador e privacidade]
- [ ] CHK021 - Os critérios documentam a resposta quando duas solicitações válidas concorrentes disputam o mesmo identificador e uma delas ativa o bloqueio antes da conclusão da outra? [Cobertura, Spec §FR-020–FR-024, Data Model §Integridade e concorrência]
- [ ] CHK022 - Os cenários diferenciam a quinta falha (`401`) das tentativas seguintes bloqueadas (`429`) e definem o comportamento quando `retry_after_seconds` chega a zero? [Clareza, Spec §FR-024, Contract §POST /api/login]
- [ ] CHK023 - Os requisitos de cadastro tratam conflito de email causado por duas criações concorrentes e mantêm o resultado `409` sem token? [Cobertura, Spec §FR-004/FR-006, Plan §Implementation Outline]

## Segurança e privacidade

- [ ] CHK024 - A exceção de exposição única do token recém-emitido está alinhada entre constituição, especificação, plano e contrato, mantendo a proibição de tokens em logs e erros? [Consistency, Constitution §I, Spec §FR-006/FR-007, Plan §Security Decisions and Risks, Contract §Convenções]
- [ ] CHK025 - Os requisitos distinguem explicitamente UUID bruto, hash do UUID, senha, hash de senha e token quanto a armazenamento, logs e respostas? [Completeness, Constitution §I, Spec §Edge Cases, Data Model §Falha de login]
- [ ] CHK026 - Os riscos de negação de serviço por identificador compartilhado e de evasão por troca de UUID têm tratamento ou aceitação de risco documentados, sem introduzir silenciosamente limites fora do escopo? [Risk, Spec §Assumptions, Research §Identificador e privacidade, Research §Decisões fora do escopo]

## Dependências e premissas

- [ ] CHK027 - A compatibilidade declarada entre PHP 8.4, Laravel 13, Sanctum 4, Larastan 3 e PHPUnit 12 está sustentada por versões e requisitos reproduzíveis? [Dependência, Plan §Technical Context, Research §Laravel 13/PHP 8.4]
- [ ] CHK028 - A decisão de usar bloqueio de linha PostgreSQL para sincronizar tentativas está ligada a uma semântica transacional que continua correta em várias instâncias? [Dependência, Plan §Implementation Outline, Research §Bloqueio por identificador, Data Model §Integridade e concorrência]

## Notes

- Marque itens `[x]` somente após revisar a qualidade dos requisitos e decisões documentados.
- Itens não são testes de implementação nem evidência de que código ou pipeline já existem.
- Referências usam os títulos das seções em `spec.md`, `plan.md`, `research.md`, `data-model.md` e `contracts/auth-api.md`.
- `/speckit-implement` lê o estado das checklists como gate e não deve alterar seus marcadores.
