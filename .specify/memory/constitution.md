<!--
Sync Impact Report
Version change: 1.0.0 -> 1.1.0
Princípios modificados: I. Segurança em primeiro lugar (exceção explícita para entrega
  única de token de acesso recém-emitido ao cliente).
Seções adicionadas: nenhuma.
Seções removidas: nenhuma.
Pendências: confirmar a data de ratificação original; remover este relatório temporário
antes de criar um commit.
-->
# Constituição da Auth API

## Core Principles

### I. Segurança em primeiro lugar
Senhas MUST ser armazenadas somente por meio de hash seguro, nunca em texto puro.
Senhas, hashes de senha e segredos MUST NOT aparecer em respostas, logs ou mensagens de
erro. Tokens de acesso MUST NOT aparecer em logs, mensagens de erro ou respostas que
não os emitam como resultado de autenticação bem-sucedida. A única exceção é o token
Bearer recém-emitido, que MUST ser devolvido uma única vez ao cliente nas respostas
bem-sucedidas de cadastro ou login; o token MUST NOT ser retornado por `/api/me`,
logout ou qualquer resposta de falha. Entradas MUST ser validadas e normalizadas
conforme o contrato do endpoint. Autenticação MUST usar mecanismos consolidados do
Laravel; rotas privadas MUST exigir autenticação e autorização adequadas. Essas regras
reduzem exposição de dados e acesso indevido.

### II. Arquitetura Laravel simples e modular
A aplicação MUST separar responsabilidades de HTTP, validação, regras de negócio,
persistência e serialização das respostas de forma proporcional à complexidade real.
Controllers MUST NOT concentrar responsabilidades excessivas nem duplicar lógica.
Implementações MUST seguir convenções idiomáticas do Laravel e aplicar princípios
SOLID quando trouxerem benefício concreto, evitando abstrações sem necessidade.

### III. Qualidade verificável
Cada funcionalidade MUST incluir testes automatizados relevantes. A cobertura MUST
considerar, conforme aplicável, sucesso, validação, autenticação, autorização e falhas.
Uma alteração MUST NOT ser considerada concluída antes da execução dos testes
pertinentes. Mudanças MUST preservar o comportamento existente, exceto quando a
especificação aprovada definir explicitamente uma mudança de contrato ou comportamento.

### IV. Contratos de API consistentes
Endpoints MUST manter contratos HTTP consistentes, códigos de status adequados e
respostas JSON previsíveis. Os endpoints e seus requisitos MUST ser documentados.
A API MUST ser versionada quando mudanças incompatíveis ou a evolução do produto
tornarem isso necessário, sem introduzir versionamento prematuro.

### V. Especificação e documentação em pt-BR
Toda funcionalidade nova MUST começar por uma especificação revisável. O plano técnico
MUST respeitar a especificação e esta constituição; as tarefas MUST ser rastreáveis aos
requisitos. Uma revisão de consistência entre especificação, plano, tarefas, código e
testes MUST ocorrer antes da implementação e da conclusão. Implementações MUST permanecer
dentro do escopo aprovado. Toda documentação produzida pelo projeto MUST ser escrita
em português brasileiro natural e tecnicamente consistente. Nomes de comandos,
bibliotecas, classes, métodos, variáveis, endpoints, propriedades de protocolos e
identificadores exigidos por ferramentas MUST preservar a grafia original; estruturas
e palavras-chave literais do Spec Kit MUST permanecer inalteradas.

## Restrições técnicas e operacionais

PostgreSQL MUST ser o banco de dados relacional do projeto. Alterações de esquema MUST
ser feitas por migrations versionadas e MUST preservar a integridade e a consistência
dos dados. Docker MUST ser usado para executar a aplicação e o PostgreSQL. Configuração
sensível e segredos MUST permanecer fora do código versionado. Dados e volumes de
ambientes MUST NOT ser alterados ou apagados sem autorização explícita.

## Fluxo de desenvolvimento

O projeto MUST estar preparado para validação automatizada em CI. Revisões MUST
verificar os requisitos de segurança, contratos, testes e consistência dos artefatos
de especificação. Migrations e testes MUST ser executados nos fluxos de validação
apropriados, sem substituir verificações por suposições. Desvios desta constituição
MUST ser justificados e aprovados por meio de alteração formal da própria constituição.

## Governance

Esta constituição prevalece sobre convenções locais conflitantes. Alterações MUST ser
propostas e revisadas como mudanças explícitas deste documento, com justificativa e
avaliação de impacto sobre especificações, planos, tarefas, código e testes existentes.
Mudanças incompatíveis nos princípios ou remoções exigem incremento MAJOR da versão;
novos princípios, seções ou expansão material exigem incremento MINOR; esclarecimentos
sem mudança semântica exigem incremento PATCH. Toda revisão MUST atualizar a versão,
a data da última alteração e o registro de impacto. A data de ratificação registra a
adoção original e MUST permanecer inalterada em revisões posteriores. Revisões de
código e de artefatos MUST avaliar conformidade com esta constituição e registrar
desvios aprovados ou pendências antes de considerar o trabalho concluído.

**Version**: 1.1.0 | **Ratified**: TODO(RATIFICATION_DATE): confirmar data de adoção original | **Last Amended**: 2026-10-22
