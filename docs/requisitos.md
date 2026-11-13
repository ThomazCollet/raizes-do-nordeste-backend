# Especificação de Requisitos — Projeto Multidisciplinar (Trilha Back-End)

**Rede “Raízes do Nordeste”**
**Versão:** 1.0 (rascunho de trabalho) | **Local no repositório:** `docs/requisitos.md`
**Thomaz de Aquino Collet e Silva - RU: 5107245**
---

## 1. Identificação e escopo

| Item                    | Descrição                                                                                           |
| :---------------------- | :-------------------------------------------------------------------------------------------------- |
| Trilha                  | Back-End                                                                                            |
| Fluxo crítico escolhido | **Fluxo A:** Pedido → Pagamento mock → Atualização de status                                        |
| Stack                   | Java 17, Spring Boot 3, Spring Data JPA, Spring Security + JWT, PostgreSQL, Flyway, Swagger/OpenAPI |
| Arquitetura             | Em camadas: `api`, `application`, `domain`, `infrastructure`                                        |
| Entrega                 | API executável + README + OpenAPI + coleção Postman + PDF único (ABNT)                              |

**Objetivo do projeto.** Projetar e implementar o back-end de uma API REST que sustente a operação multicanal da rede (pedidos, estoque por unidade, pagamento simulado, fidelidade com LGPD), priorizando **um fluxo principal fechado e testável** em vez de muitos endpoints.

**Dentro do escopo (MVP):** autenticação com perfis; unidades, cardápio e estoque por unidade; pedido com `canalPedido`; pagamento mock (aprovado, recusado, falha); atualização de status e cancelamento; fidelidade com consentimento; auditoria; documentação e testes.

**Fora do escopo (declarado):** pagamento real, front-end/telas, estorno financeiro, entrega por motoboy/rotas, emissão fiscal, gestão de receitas e de capacidade da cozinha, notificações push/e-mail.

---

## 2. Contexto e problema

A rede nasceu como negócio familiar em Recife e hoje tem unidades em várias capitais e no interior. Cada unidade tem equipe, **estoque local** e regras próprias; algumas têm cozinha completa, outras operam em formato reduzido; certos produtos são **sazonais** (período junino) e algumas receitas variam por região. A **matriz** precisa de vendas por unidade/região, produtos mais consumidos, relatórios financeiros e **auditoria** de operações sensíveis (cancelamentos, descontos, ajustes).

O cliente pode pedir por **App, Totem, Balcão, Pick-up ou Web**, e espera ver o cardápio da unidade, pedir, acompanhar o status e receber o pedido correto. O pagamento é feito por **serviço externo desacoplado**: o sistema solicita, recebe aprovação ou recusa, registra e atualiza o pedido. A empresa lançou um programa de **fidelização** sob a diretriz de que respeitar a privacidade é obrigatório (**LGPD**). Nos horários de pico, o sistema não pode falhar.

**Problema a resolver:** unificar a jornada entre canais mantendo controle por unidade, rastreabilidade, privacidade e resiliência, sem acoplar o sistema ao processamento de pagamento.

---

## 3. Glossário

| Termo          | Significado                                                 |
| :------------- | :---------------------------------------------------------- |
| Matriz         | Administração central da rede (visão consolidada)           |
| Unidade        | Loja da rede, com estoque, equipe e cardápio próprios       |
| Canal          | Origem do pedido: `APP`, `TOTEM`, `BALCAO`, `PICKUP`, `WEB` |
| Sazonal        | Produto disponível só em certas épocas (ex.: junino)        |
| Gateway (mock) | Serviço externo de pagamento simulado                       |
| Titular        | Pessoa a quem os dados pessoais se referem (LGPD)           |
| RBAC           | Controle de acesso por perfil (role)                        |

---

## 4. Atores e perfis

| Ator                    | Perfil (role)     | Vínculo com unidade    | O que faz                                                                             |
| :---------------------- | :---------------- | :--------------------- | :------------------------------------------------------------------------------------ |
| Cliente (App/Web/Totem) | `CLIENTE`         | Nenhum                 | Cadastra-se, vê cardápio, pede, acompanha status, gerencia consentimento e fidelidade |
| Atendente (Balcão)      | `ATENDENTE`       | Uma unidade            | Registra pedidos no balcão, entrega/retirada, consulta estoque                        |
| Cozinha                 | `COZINHA`         | Uma unidade            | Vê pedidos pagos e atualiza o preparo                                                 |
| Gerente                 | `GERENTE`         | Uma unidade            | Gerencia produtos/estoque da unidade, cancelamentos, auditoria da unidade             |
| Administrador (Matriz)  | `ADMIN`           | Nenhum (visão da rede) | Gestão global, auditoria, relatórios, cadastro de unidades e de usuários internos     |
| Gateway de pagamento    | (sistema externo) | —                      | Recebe a cobrança e devolve aprovado/recusado/falha                                   |

---

## 5. Requisitos funcionais (RF)

**Escalas.** *Importância:* Essencial (sem isso o fluxo não fecha), Importante, Desejável. *Status no MVP:* **Previsto** (será implementado), **Parcial** (implementado em versão mínima), **Proposto** (apenas documentado/justificado). O status só vira “Implementado” quando houver evidência (ver seção 12).

| ID    | Requisito                       | Descrição (comportamento)                                                                                                                                                                          | Ator(es)                             | Importância | Status MVP                  |
| :---- | :------------------------------ | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----------------------------------- | :---------- | :-------------------------- |
| RF-01 | Cadastro de usuários            | Cliente se cadastra sozinho (perfil sempre `CLIENTE`). Usuários internos são criados por `ADMIN`/`GERENTE` com perfil e unidade definidos.                                                         | Cliente, Admin, Gerente              | Essencial   | Previsto                    |
| RF-02 | Autenticação e perfis           | Login com e-mail e senha devolve token com tempo de expiração; rotas protegidas conforme o perfil.                                                                                                 | Todos                                | Essencial   | Previsto                    |
| RF-03 | Gestão de unidades              | Listar unidades ativas (público, paginado); cadastrar e ativar/inativar unidades.                                                                                                                  | Todos (consulta); Admin (gestão)     | Importante  | Previsto                    |
| RF-04 | Cardápio por unidade            | Consultar produtos disponíveis em uma unidade, com filtro por sazonais.                                                                                                                            | Todos                                | Essencial   | Previsto                    |
| RF-05 | Gestão de produtos              | Cadastrar, editar e inativar produtos do catálogo da rede.                                                                                                                                         | Gerente, Admin                       | Importante  | Previsto                    |
| RF-06 | Controle de estoque por unidade | Consultar saldo da unidade; registrar entradas e saídas manuais (com registro de movimentação); bloquear venda sem saldo.                                                                          | Gerente, Atendente (consulta), Admin | Essencial   | Previsto                    |
| RF-07 | Registro de pedido com canal    | Criar pedido para uma unidade com itens, forma de pagamento e `canalPedido` obrigatório; validar itens e estoque; calcular total; status inicial `AGUARDANDO_PAGAMENTO`; suportar idempotência.    | Cliente, Atendente                   | Essencial   | Previsto                    |
| RF-08 | Consulta de pedidos             | Consultar pedido por id; listar com paginação e filtros (`canalPedido`, `status`, `unidadeId`). Cliente vê só os seus; funcionário, os da sua unidade.                                             | Todos autenticados                   | Essencial   | Previsto                    |
| RF-09 | Atualização de status           | Alterar o status seguindo a máquina de estados (seção 7), com o perfil adequado a cada transição.                                                                                                  | Cozinha, Atendente, Gerente, Sistema | Essencial   | Previsto                    |
| RF-10 | Cancelamento de pedido          | Cancelar conforme perfil e estado do pedido, devolvendo o estoque e registrando auditoria.                                                                                                         | Cliente, Atendente, Gerente          | Essencial   | Previsto                    |
| RF-11 | Pagamento externo (mock)        | Solicitar pagamento ao gateway simulado; tratar **aprovado**, **recusado** e **falha/indisponibilidade**; registrar transação e payload; atualizar o pedido; devolver status e payload ao cliente. | Cliente, Atendente, Gateway          | Essencial   | Previsto                    |
| RF-12 | Programa de fidelidade          | Acumular pontos por pedido pago (com consentimento vigente), consultar saldo e histórico; resgate simples de pontos como desconto.                                                                 | Cliente, Gerente (consulta)          | Importante  | Parcial (resgate: Proposto) |
| RF-13 | Promoções e campanhas           | Regra documentada de aplicação de desconto (itens sazonais e/ou resgate de pontos), registrada no pedido.                                                                                          | Sistema                              | Desejável   | Parcial                     |
| RF-14 | Auditoria                       | Registrar ações sensíveis e permitir consulta filtrada.                                                                                                                                            | Gerente (própria unidade), Admin     | Essencial   | Previsto                    |
| RF-15 | Relatórios para a matriz        | Vendas por unidade/região/canal e produtos mais vendidos.                                                                                                                                          | Admin                                | Desejável   | Proposto                    |
| RF-16 | Consentimento LGPD              | Conceder, consultar e revogar consentimento de uso de dados na fidelidade, com data e versão do termo.                                                                                             | Cliente                              | Essencial   | Previsto                    |

### 5.1 Critérios de aceite dos RFs críticos

- **RF-07 (pedido):**
  - Dado cliente autenticado e produtos com saldo, quando envia `canalPedido` válido e itens, então recebe **201** com status `AGUARDANDO_PAGAMENTO`, total calculado e estoque baixado.
  - Sem `canalPedido` ou com valor fora do enum → **422** no erro padrão.
  - Unidade/produto inexistente → **404**. Saldo insuficiente → **409** `ESTOQUE_INSUFICIENTE`, sem baixar nada (tudo ou nada).
  - Repetição com a mesma `Idempotency-Key` e mesmo corpo → devolve o pedido original, sem baixar estoque de novo.
- **RF-11 (pagamento):**
  - Aprovado → pagamento `APROVADO`, pedido `PAGO`, resposta 200 com status e payload.
  - Recusado → pagamento `RECUSADO`, pedido `PAGAMENTO_RECUSADO`, estoque devolvido, mensagem clara.
  - Falha/indisponibilidade → pagamento `FALHA`, pedido permanece `AGUARDANDO_PAGAMENTO` (consistente), nova tentativa permitida.
  - Pedido já pago ou cancelado → **409**.
- **RF-09 (status):** transição fora da tabela da seção 7 → **409** `TRANSICAO_STATUS_INVALIDA`; perfil sem permissão → **403**.
- **RF-06 (estoque):** saldo nunca fica negativo; toda entrada/saída gera movimentação rastreável.
- **RF-12/RF-16 (fidelidade):** sem consentimento vigente não há acúmulo; após revogar, o acúmulo para e o histórico é preservado.
- **RF-14 (auditoria):** criar pedido, mudar status, cancelar e movimentar estoque geram registro com usuário, ação, entidade, canal e `requestId`.

---

## 6. Regras de negócio (RN)

| ID    | Regra                                                                                                                                            |
| :---- | :----------------------------------------------------------------------------------------------------------------------------------------------- |
| RN-01 | Todo pedido possui `canalPedido` ∈ {`APP`, `TOTEM`, `BALCAO`, `PICKUP`, `WEB`}; ausente ou inválido é rejeitado.                                 |
| RN-02 | Um pedido pertence a **uma única unidade**; só aceita produtos do cardápio dessa unidade, com quantidade ≥ 1.                                    |
| RN-03 | O saldo de estoque nunca fica negativo; sem saldo suficiente a venda é bloqueada (409).                                                          |
| RN-04 | O preço unitário é **congelado** no item do pedido no momento da compra.                                                                         |
| RN-05 | Total = soma(quantidade × preço unitário) − desconto; valores em `DECIMAL(10,2)`, arredondamento `HALF_UP`.                                      |
| RN-06 | A baixa de estoque ocorre **na criação do pedido**, na mesma transação; é devolvida em recusa, cancelamento ou expiração.                        |
| RN-07 | Somente as transições da seção 7 são permitidas; as demais retornam 409.                                                                         |
| RN-08 | Um pedido tem no máximo **um** pagamento `APROVADO`.                                                                                             |
| RN-09 | Recusa → `PAGAMENTO_RECUSADO` e devolução de estoque. Falha técnica do gateway → pedido continua `AGUARDANDO_PAGAMENTO` e aceita nova tentativa. |
| RN-10 | Cliente acessa apenas os próprios pedidos e o próprio saldo; funcionário de unidade, apenas dados da sua unidade; `ADMIN` vê toda a rede.        |
| RN-11 | Pontos só acumulam após pagamento **aprovado** e com consentimento **vigente**; regra de cálculo na seção 9.                                     |
| RN-12 | Consentimento revogado interrompe acúmulo e uso em campanhas segmentadas; o histórico é preservado para prova.                                   |
| RN-13 | `Idempotency-Key` repetida com o mesmo corpo devolve o pedido original; com corpo diferente retorna 409.                                         |
| RN-14 | Pedido `AGUARDANDO_PAGAMENTO` além do prazo de expiração (configurável) vira `CANCELADO` e devolve estoque.                                      |
| RN-15 | Criação/cancelamento de pedido, mudança de status, movimentação de estoque, tentativa de pagamento e alteração de consentimento geram auditoria. |
| RN-16 | No autocadastro o perfil é sempre `CLIENTE`; o cliente nunca escolhe o próprio perfil.                                                           |
| RN-17 | O cliente é identificado **pelo token**, não por `clienteId` no corpo (exceto atendente registrando pedido para cliente cadastrado).             |
| RN-18 | O campo de simulação de resultado do pagamento só é aceito em ambiente de desenvolvimento/teste.                                                 |

---

## 7. Máquina de estados do pedido [PROPOSTA]

Status: `AGUARDANDO_PAGAMENTO`, `PAGO`, `COZINHA`, `PRONTO`, `ENTREGUE`, `CANCELADO`, `PAGAMENTO_RECUSADO`.

| De                     | Para                 | Quem executa                                            | Condição / efeito                                    |
| :--------------------- | :------------------- | :------------------------------------------------------ | :--------------------------------------------------- |
| `AGUARDANDO_PAGAMENTO` | `PAGO`               | Sistema                                                 | Pagamento aprovado                                   |
| `AGUARDANDO_PAGAMENTO` | `PAGAMENTO_RECUSADO` | Sistema                                                 | Pagamento recusado; estoque devolvido                |
| `AGUARDANDO_PAGAMENTO` | `CANCELADO`          | Cliente (dono), Atendente, Gerente, Sistema (expiração) | Estoque devolvido                                    |
| `PAGO`                 | `COZINHA`            | Cozinha                                                 | Início do preparo                                    |
| `PAGO`                 | `CANCELADO`          | Gerente, Admin                                          | Estoque devolvido; estorno financeiro fora do escopo |
| `COZINHA`              | `PRONTO`             | Cozinha                                                 | Preparo concluído                                    |
| `PRONTO`               | `ENTREGUE`           | Atendente                                               | Entrega ou retirada (inclui `PICKUP`)                |

- Estados finais: `ENTREGUE`, `CANCELADO`, `PAGAMENTO_RECUSADO`.
- Falha técnica do gateway **não muda o status** (ver RN-09).
- Qualquer outra transição → 409.

---

## 8. Multicanalidade

- `canalPedido` é dado de domínio obrigatório, gravado no pedido e propagado para auditoria (rastreabilidade entre canais).
- `GET /pedidos?canalPedido=TOTEM&status=AGUARDANDO_PAGAMENTO` filtra por canal e status.
- **Interpretação adotada [PROPOSTA]:** `BALCAO` = pedido digitado por atendente no balcão; `PICKUP` = pedido antecipado (app/web) para retirada rápida; `TOTEM` = autoatendimento na loja; `APP`/`WEB` = canais digitais do cliente.
- Os canais mudam **a origem**, não as regras: estoque, pagamento e status valem igual para todos.

---

## 9. Fidelidade, promoções e LGPD

### 9.1 Fidelidade [PROPOSTA]

- Acúmulo: **1 ponto por R$ 1,00 pago** (parte inteira), creditado após pagamento aprovado e somente com consentimento vigente.
- Histórico de pontos em tabela própria (créditos e débitos).
- Resgate simples: desconto fixo ao trocar pontos (valor da conversão a definir). Regra documentada; implementação se o tempo permitir (**Parcial/Proposto**).
- Cliente anônimo (totem/balcão sem cadastro) **não acumula** pontos.

### 9.2 Promoções e campanhas [PROPOSTA]

- **Promoção sazonal:** desconto percentual configurável por produto sazonal, aplicado no cálculo do pedido.
- **Desconto por pontos:** resgate abate o total do pedido.
- O desconto aplicado fica registrado no pedido (campo de desconto) para auditoria.
- **Campanha segmentada** (frequência, idade, perfil): descrita como regra; segmentação por idade só com consentimento específico (ver A-06).

### 9.3 Mapa preliminar de dados pessoais (LGPD)


| Dado                                             | Finalidade                                         | Base legal (LGPD, art. 7º)                      | Onde fica                        | Retenção / anonimização [PROPOSTA]                          |
| :----------------------------------------------- | :------------------------------------------------- | :---------------------------------------------- | :------------------------------- | :---------------------------------------------------------- |
| Nome e e-mail                                    | Identificação, autenticação, comunicação do pedido | Execução de contrato (V)                        | `usuarios`                       | Enquanto a conta existir; anonimizar ao excluir a conta     |
| Senha                                            | Autenticação                                       | Execução de contrato (V)                        | `usuarios` (somente hash BCrypt) | Nunca exposta em resposta ou log                            |
| Pedidos, itens e valores                         | Prestação do serviço e obrigações contábeis        | Execução de contrato (V) e obrigação legal (II) | `pedidos`, `itens_pedido`        | Prazo a definir; depois desvincular do cliente (anonimizar) |
| Pontos e histórico                               | Programa de fidelidade                             | Consentimento (I)                               | `fidelidades` e histórico        | Até revogação + prazo de guarda a definir                   |
| Consentimento (data, versão do termo, revogação) | Prova do consentimento (art. 8º, §2º)              | Obrigação do controlador                        | `consentimentos`                 | Preservado como prova                                       |
| Logs de auditoria                                | Segurança e rastreabilidade                        | Legítimo interesse (IX)                         | `auditoria`                      | Prazo a definir; sem senha, token ou dado desnecessário     |
| Data de nascimento / idade                       | Segmentação de campanhas                           | Consentimento específico (I)                    | **Não coletado no MVP**          | —                                                           |
| Payload do gateway mock                          | Registro da transação                              | Execução de contrato (V)                        | `pagamentos`                     | Sem dados de cartão (mock)                                  |

**Controles:** hash BCrypt; JWT; RBAC; DTOs que nunca devolvem hash/token; minimização (somente o necessário); consentimento com data, versão e revogação; logs de acesso a dados sensíveis (ex.: gerente consultando fidelidade de cliente).
**Direitos do titular (art. 18):** acesso e revogação previstos (RF-16); exclusão/anonimização documentada como **Proposto**.

---

## 10. Requisitos não funcionais (RNF)

| ID     | Categoria                        | Requisito                                                                     | Meta / critério verificável                                                                                                                                                       | Status MVP                  |
| :----- | :------------------------------- | :---------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------- |
| RNF-01 | Segurança                        | Senhas armazenadas somente como **hash BCrypt**                               | Nenhuma senha em texto puro no banco, log ou resposta                                                                                                                             | Previsto                    |
| RNF-02 | Segurança                        | Autenticação stateless com JWT **assinado** e expiração                       | Token expira em tempo definido; segredo só em variável de ambiente                                                                                                                | Previsto                    |
| RNF-03 | Segurança                        | Autorização por perfil e por unidade                                          | 401 sem token, 403 sem permissão, ambos no JSON padrão                                                                                                                            | Previsto                    |
| RNF-04 | Privacidade                      | Conformidade LGPD: finalidade, minimização, consentimento, retenção           | Tabela da seção 9.3 aplicada; DTOs sem dados sensíveis                                                                                                                            | Previsto                    |
| RNF-05 | Interoperabilidade               | Erro padronizado para **todas** as falhas (400, 401, 403, 404, 409, 422, 500) | Mesmo JSON: `error`, `message`, `details`, `timestamp`, `path`, `requestId`                                                                                                       | Previsto                    |
| RNF-06 | Documentação                     | OpenAPI/Swagger refletindo o contrato real                                    | Exemplos e status codes por endpoint; `openapi.yaml` no repositório e prints no PDF                                                                                               | Previsto                    |
| RNF-07 | Confiabilidade                   | Tolerância a falhas no pagamento                                              | Timeout/erro do gateway não corrompe o pedido; estado consistente e nova tentativa possível                                                                                       | Previsto                    |
| RNF-08 | Desempenho                       | Resposta adequada em horário de pico                                          | Paginação obrigatória, índices em FKs e filtros (`canal_pedido`, `status`, `unidade_id`), pool de conexões, evitar N+1. **Meta numérica a definir após medição** (ex.: JMeter/k6) | Parcial (medição: Proposto) |
| RNF-09 | Disponibilidade e escalabilidade | API stateless e escalável horizontalmente                                     | Sem estado em memória da sessão; endpoint de saúde (Actuator). Múltiplas instâncias e balanceador: **Proposto**                                                                   | Parcial                     |
| RNF-10 | Consistência                     | Baixa de estoque atômica, sem venda acima do saldo                            | Transação com bloqueio; teste de concorrência (desejável)                                                                                                                         | Previsto                    |
| RNF-11 | Rastreabilidade                  | Logs estruturados com `requestId`                                             | Logs sem dados pessoais, senha ou token                                                                                                                                           | Previsto                    |
| RNF-12 | Reprodutibilidade                | Execução simples por terceiros                                                | README passo a passo, `.env.example`, Docker Compose, Flyway + seed                                                                                                               | Previsto                    |
| RNF-13 | Manutenibilidade                 | Arquitetura em camadas com dependências controladas                           | Regra de dependência do Anexo A; (opcional) teste ArchUnit                                                                                                                        | Previsto                    |
| RNF-14 | Testabilidade                    | Cenários executáveis e repetíveis                                             | Coleção Postman/Insomnia com ≥ 10 cenários; testes JUnit das regras críticas (desejável)                                                                                          | Previsto                    |

---

## 11. Ambiguidades, premissas e interpretação adotada

### 11.1 Ambiguidades do estudo de caso

| ID   | Ambiguidade                                           | Interpretação adotada [PROPOSTA]                                                                  | Impacto / decisão    |
| :--- | :---------------------------------------------------- | :------------------------------------------------------------------------------------------------ | :------------------- |
| A-01 | Pedido nunca pago segura o estoque (baixa na criação) | Expira após prazo configurável, vira `CANCELADO` e devolve estoque (RN-14)                        | Estoque, status — D3 |
| A-02 | Falha técnica do gateway ≠ recusa                     | Falha mantém `AGUARDANDO_PAGAMENTO` e registra tentativa `FALHA`                                  | Pagamentos — D1, D6  |
| A-03 | Pagamento síncrono ou assíncrono                      | **Síncrono** (mock responde na mesma chamada); webhook assíncrono fica **Proposto**               | Fluxo A — D1         |
| A-04 | O que coloca um produto no cardápio da unidade        | Registro ativo em `estoque_unidades` para aquela unidade                                          | DER — D5             |
| A-05 | Controle de sazonalidade                              | Flag `sazonal` + produto ativo/inativo por unidade; período (início/fim) **Proposto**             | DER                  |
| A-06 | Segmentação por idade/perfil × minimização de dados   | Idade **não coletada** no MVP; segmentação por frequência (derivada de pedidos) com consentimento | LGPD — D7            |
| A-07 | Como o totem se autentica                             | Usuário de serviço por unidade (perfil `ATENDENTE`), credencial por variável de ambiente/seed     | Segurança — D4       |
| A-08 | Cliente anônimo e fidelidade                          | Pedido vinculado a um cliente genérico de sistema; **não acumula pontos**                         | DER — D4             |
| A-09 | Diferença entre `BALCAO` e `PICKUP`                   | Ver seção 8                                                                                       | Domínio              |
| A-10 | Escopo do gerente                                     | Restrito à própria unidade; `ADMIN` enxerga a rede                                                | Autorização — D5     |
| A-11 | Unidades com cozinha reduzida                         | Tratado pelo cardápio/estoque por unidade; capacidade da cozinha fora do escopo                   | Escopo               |
| A-12 | Receitas que variam por região                        | Produto regional é um produto do catálogo; modelagem de receitas fora do escopo                   | Escopo               |
| A-13 | Estorno de pagamento                                  | Fora do escopo (gateway mock)                                                                     | Escopo               |
| A-14 | Pedido com itens de unidades diferentes               | Não permitido (RN-02)                                                                             | Regra                |

### 11.2 Premissas técnicas

1. **Cliente anônimo (totem/balcão):** associado a um usuário genérico de sistema para preservar a integridade referencial do pedido.
2. **Baixa de estoque na criação do pedido:** devolvida de forma atômica em recusa, cancelamento ou expiração (trade-off: evita vender acima do saldo, ao custo de segurar estoque de pedidos não pagos; mitigado por RN-14).
3. **Moeda:** BRL, `DECIMAL(10,2)`, arredondamento `HALF_UP`.
4. **Datas:** armazenadas em UTC e expostas em ISO-8601.
5. **Mock:** o resultado simulado é um recurso de teste, restrito a ambiente não produtivo (RN-18).

---

## 12. Quadro Implementado × Proposto (atualizar ao fim de cada fase)

| Item                                            | Status   | Justificativa                                                      | Evidência                    |
| :---------------------------------------------- | :------- | :----------------------------------------------------------------- | :--------------------------- |
| Autenticação JWT e perfis                       | Previsto | Base de segurança e pré-requisito de todo o fluxo                  | _a preencher (commit/teste)_ |
| Cadastro de usuários                            | Previsto | Necessário para login, perfis e seed                               | _a preencher_                |
| Unidades e cardápio por unidade (incl. sazonal) | Previsto | Diferença entre unidades é núcleo do caso                          | _a preencher_                |
| Estoque por unidade e movimentações             | Previsto | Regra central de venda e de auditoria                              | _a preencher_                |
| Pedido com `canalPedido` e filtro               | Previsto | Requisito mínimo do roteiro                                        | _a preencher_                |
| Atualização de status e cancelamento            | Previsto | Fecha o Fluxo A                                                    | _a preencher_                |
| Pagamento mock (aprovado, recusado, falha)      | Previsto | Exigido pelo roteiro; trata RNF-07                                 | _a preencher_                |
| Fidelidade: consentimento, acúmulo e saldo      | Previsto | Exigido (fidelização + LGPD)                                       | _a preencher_                |
| Fidelidade: resgate                             | Proposto | Regra documentada; implementação se sobrar tempo                   | _a preencher_                |
| Promoções/campanhas                             | Parcial  | Regra documentada + desconto registrado no pedido                  | _a preencher_                |
| Auditoria de ações sensíveis                    | Previsto | Exigida pelo caso e pela rubrica (15 pts)                          | _a preencher_                |
| LGPD (finalidade, base legal, retenção)         | Previsto | Tabela 9.3 + controles aplicados                                   | _a preencher_                |
| Expiração de pedidos não pagos                  | Proposto | Mitiga A-01; avaliar tarefa agendada simples                       | _a preencher_                |
| Relatórios para a matriz                        | Proposto | Fora do MVP; descrito em RF-15                                     | —                            |
| Desempenho e disponibilidade em pico            | Parcial  | Boas práticas no projeto; medição e múltiplas instâncias propostas | _a preencher_                |
| Testes automatizados (JUnit)                    | Proposto | Coleção Postman é a evidência principal                            | _a preencher_                |
| Deploy público do Swagger                       | Proposto | Opcional; `openapi.yaml` + prints cobrem                           | —                            |

---

## 13. Decisões (portão antes da Fase 4.2) [PROPOSTAS — confirmar ou ajustar]

Cada linha é uma **proposta** com justificativa. Só vira decisão quando eu confirmar. Meta: nenhuma linha em branco antes de escrever migrations.

| #    | Decisão                 | Proposta                                                                                                                                        | Justificativa / trade-off                                                                      |
| :--- | :---------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------- |
| 1    | Fluxo de pagamento      | Pedido criado em `AGUARDANDO_PAGAMENTO`; pagamento em `POST /pagamentos/processar`                                                              | Desacopla pedido de pagamento (como no caso) e permite retentativa; custo: duas chamadas       |
| 2    | Máquina de estados      | Seção 7                                                                                                                                         | Cada transição tem dono e condição; reduz estados inválidos                                    |
| 3    | Baixa de estoque        | Na criação, em transação com bloqueio da linha de estoque; devolução em recusa/cancelamento/expiração                                           | Evita overselling em pico; custo: segura estoque de pedidos não pagos (mitigado por expiração) |
| 4    | Cliente anônimo         | Cliente genérico de sistema; totem usa usuário de serviço por unidade                                                                           | Mantém `cliente_id` obrigatório e a FK íntegra; custo: anônimo não acumula pontos              |
| 5    | Escopo por unidade      | `usuarios.unidade_id` (nulo para `CLIENTE` e `ADMIN`)                                                                                           | Isola gerente/atendente/cozinha por unidade; `ADMIN` representa a matriz                       |
| 6    | Tentativas de pagamento | Várias por pedido; no máximo uma `APROVADA`                                                                                                     | Suporta falha do gateway e nova tentativa; exige remover o UNIQUE de `pedido_id`               |
| 7    | Fidelidade              | 1 ponto/R$ 1; histórico em tabela; consentimento com data, versão e revogação; resgate documentado                                              | Simples, auditável e aderente à LGPD                                                           |
| 8    | Códigos de validação    | **400** = JSON malformado/ilegível; **422** = dados legíveis porém inválidos (campo ausente, enum inválido, quantidade ≤ 0, e-mail inválido)    | Regra única e fácil de testar                                                                  |
| 9    | Paginação               | `page` iniciando em 1 e `limit` (como no roteiro), convertidos internamente; envelope `content`, `page`, `limit`, `totalElements`, `totalPages` | Alinha com o roteiro; custo: conversão com o Spring Data (base 0)                              |
| 10   | Auditoria               | Ações da RN-15; campos: usuário, ação, entidade, id, canal, detalhes, `requestId`, data; consulta por `GERENTE` (unidade) e `ADMIN`             | Cobre rastreabilidade pedida pela rubrica                                                      |
| 11   | Idempotency-Key         | Header em `POST /pedidos`; tabela guarda chave, hash do corpo e pedido; validade de 24 h; mesmo corpo → mesmo pedido; corpo diferente → 409     | Evita pedido duplicado por retry em pico                                                       |
| 12   | Repositórios            | Interface no `domain` + implementação JPA na `infrastructure`                                                                                   | Respeita a regra de dependência exigida; custo: mais classes                                   |
| 13   | Entidades JPA           | Anotações JPA nas entidades do domínio (concessão pragmática documentada)                                                                       | Evita duplicar modelos em um projeto acadêmico                                                 |
| 14   | Idioma do código        | Domínio em português; termos técnicos/framework em inglês; commits em português com prefixo Conventional Commits                                | Alinha ao contrato do caso e à convenção do ecossistema                                        |
| 15   | JWT                     | Expiração de 3600 s; segredo em `JWT_SECRET`; refresh e logout **Propostos**                                                                    | Simples e seguro o bastante para o MVP                                                         |

---

## 14. Impactos no DER (a refletir na Fase 2)

O DER atual de 8 tabelas precisa evoluir para cobrir os requisitos acima:

- **Novas tabelas:** `movimentacoes_estoque`, `auditoria`, `consentimentos`, `fidelidade_movimentacoes` (histórico de pontos), `idempotencia_pedidos`.
- **`usuarios`:** adicionar `unidade_id` (nulo) e `ativo`.
- **`pedidos`:** `cliente_id` continua obrigatório (cliente genérico para anônimos); adicionar `desconto` e `expira_em`.
- **`pagamentos`:** remover UNIQUE em `pedido_id`; permitir status `FALHA`; (opcional) renomear a tabela de `pagamentos_mock` para `pagamentos`, pois o mock é o gateway e não a entidade.
- **`fidelidades`:** consentimento sai do booleano e passa para `consentimentos` (data, versão do termo, revogação).
- **`estoque_unidades`:** adicionar `ativo` e restrição única `(unidade_id, produto_id)`; (opcional) controle de versão para concorrência.

---

## 15. Rastreabilidade preliminar (RF → endpoints → tabelas)

> Esboço; refinar na Fase 2 e completar com os IDs de teste na Fase 5.

| RF    | Endpoints previstos                                                                             | Tabelas                                                               |
| :---- | :---------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------- |
| RF-01 | `POST /usuarios`, `POST /usuarios/internos`, `GET /usuarios/me`                                 | `usuarios`                                                            |
| RF-02 | `POST /auth/login`                                                                              | `usuarios`                                                            |
| RF-03 | `GET /unidades`, `POST /unidades`, `PATCH /unidades/{id}`                                       | `unidades`                                                            |
| RF-04 | `GET /produtos?unidadeId=&sazonal=`                                                             | `produtos`, `estoque_unidades`                                        |
| RF-05 | `POST /produtos`, `PUT /produtos/{id}`, `PATCH /produtos/{id}`                                  | `produtos`                                                            |
| RF-06 | `GET /estoque/unidades/{unidadeId}`, `POST /estoque/movimentacoes`                              | `estoque_unidades`, `movimentacoes_estoque`                           |
| RF-07 | `POST /pedidos`                                                                                 | `pedidos`, `itens_pedido`, `estoque_unidades`, `idempotencia_pedidos` |
| RF-08 | `GET /pedidos`, `GET /pedidos/{id}`                                                             | `pedidos`, `itens_pedido`                                             |
| RF-09 | `PATCH /pedidos/{id}/status`                                                                    | `pedidos`, `auditoria`                                                |
| RF-10 | `POST /pedidos/{id}/cancelar`                                                                   | `pedidos`, `estoque_unidades`, `movimentacoes_estoque`, `auditoria`   |
| RF-11 | `POST /pagamentos/processar`, `GET /pagamentos?pedidoId=`                                       | `pagamentos`, `pedidos`                                               |
| RF-12 | `GET /fidelidade/me`, `GET /fidelidade/me/historico`, `POST /fidelidade/me/resgates` (proposto) | `fidelidades`, `fidelidade_movimentacoes`                             |
| RF-13 | (aplicado em `POST /pedidos`)                                                                   | `pedidos`, `produtos`                                                 |
| RF-14 | `GET /auditoria`                                                                                | `auditoria`                                                           |
| RF-15 | `GET /relatorios/vendas` (proposto)                                                             | consultas sobre `pedidos`, `itens_pedido`                             |
| RF-16 | `POST /fidelidade/me/consentimento`, `POST /fidelidade/me/consentimento/revogacao`              | `consentimentos`                                                      |

