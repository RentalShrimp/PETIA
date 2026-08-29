# PETIA — Command OS (primeira fatia)

**Status:** rascunho para revisão  
**Data:** 2026-08-29  
**Produto:** PETIA (Pet Shop Operating System), produto proprietário da HYPERION  
**Fase:** pré-implementação — spec da primeira fatia, não do produto inteiro  
**Fonte:** `PETSHOP_OS_FEATURE_DISCOVERY.md` + brainstorming desta sessão

Este documento congela o desenho aprovado da **primeira fatia operacional**. Não substitui o discovery completo. Não autoriza implementar os 22 domínios do discovery.

---

## 1. O que estamos construindo

Um núcleo **Command OS** para **uma unidade** de petshop: cadastro 360 operacional (tutor + pet), agenda, ordem de serviço de banho & tosa, copiloto no WhatsApp e cobrança PIX/link com baixa automática.

A PETIA continua sendo um OS vertical (Record → Action → Intelligence). Esta fatia entrega Record + Action no chão da loja. Intelligence preditiva fica para depois.

**Público desta fatia**

- Loja independente = um `Tenant` com uma `Unidade`
- Rede/franquia = mesmo modelo; **não** opera holding, permissões multiunidade, estoque entre lojas nem DRE consolidado nesta fatia

**Dupla missão**

- Produto real: uma loja-piloto opera **uma semana** de banho & tosa só na PETIA
- Série no YouTube: o episódio mostra **esse** loop, não um demo mais rico que o produto

---

## 2. Critério de sucesso

A fatia está pronta quando:

1. Recepção e tosador operam o dia na tela (cadastro, agenda, OS, fotos, cobrança).
2. O tutor agenda, reagenda, cancela, confirma e recebe cobrança/avisos no WhatsApp.
3. PIX ou link é emitido; o pagamento **baixa sozinho** quando o PSP confirma.
4. A timeline do pet mostra os fatos (serviço, fotos, pagamento, conversa).
5. Se o agente cair, a tela da loja continua gravando pelos **mesmos comandos**.
6. O episódio filma o loop que a loja-piloto realmente usou.

---

## 3. Princípios desta fatia

1. **Pet-first.** Pet é entidade de primeira classe, não um campo do cliente.
2. **Um caminho de escrita.** Tela e WhatsApp disparam o mesmo catálogo de comandos.
3. **LLM só no chat.** A tela não conversa para persistir.
4. **Nada grava sem confirmação no chat** (padrão v1). A loja poderá configurar o nível por ação depois.
5. **Orquestrador obrigatório** para chamada de agente. O orquestrador não é o dono da regra de negócio.
6. **SQL só no handler.** O modelo não redige SQL. O audit pode mostrar o SQL parametrizado que o handler vai executar (visível na série).
7. **Tenant → Unidade no modelo desde o dia 1.** Isolamento por tenant em toda consulta e comando; operação do dia filtrada por unidade.
8. **Comando é o produto; LLM é um cliente.**

---

## 4. Arquitetura

```text
Tela da loja  ──┐
                ├──► Comando ──► Orquestrador ──► Handler ──► SQL parametrizado
WhatsApp+LLM ──┘         ▲                              │
                         │                              ▼
                   confirmação no chat                 Evento + ComandoLog
                   (antes de gravar)                    │
                                                        ▼
                                                   Timeline (projeção)
```

- **Leitura** (agenda do dia, ficha do pet, timeline): API de consulta ao estado. Não passa pelo agente.
- **Escrita:** somente comando nomeado, com `tenant_id` + `unidade_id`.
- **WhatsApp:** o orquestrador chama o agente com contexto; o agente devolve uma intenção (comando + argumentos); o orquestrador valida, pede confirmação, e só então executa o handler.
- **PIX/link:** comando `EmitirCobranca` (com confirmação no chat). A baixa é **evento de webhook** (`BaixarPagamento`), não conversa.

Este spec **não** escolhe linguagem, framework, banco concreto nem fornecedor de WhatsApp/PSP. Escolhe o contrato: comandos, handlers, orquestrador, SQL parametrizado, webhook de baixa.

---

## 5. Catálogo de comandos

Todo comando carrega `tenant_id` e, quando opera a loja, `unidade_id`.

| Comando | Efeito | Confirmação no chat (v1) |
|---|---|---|
| `RegistrarTutor` | Cria/atualiza tutor | Sim |
| `RegistrarPet` | Cria/atualiza pet | Sim |
| `VincularPet` | Liga tutor ↔ pet (v1: um tutor principal) | Sim |
| `Agendar` | Pet + serviço + profissional + horário | Sim |
| `Reagendar` | Altera horário/profissional | Sim |
| `CancelarAgendamento` | Cancela | Sim |
| `ConfirmarAgendamento` | Tutor confirma que vem (RSVP); não é check-in | Sim |
| `ConfirmarPresenca` | Check-in na unidade | Sim no chat; na tela, um toque |
| `AbrirOS` | Abre ordem de serviço a partir do agendamento | Sim no chat; na recepção pode seguir o check-in |
| `RegistrarEntradaOS` | Checklist de entrada + fotos | Tela do tosador; chat só se o tutor mandar observação |
| `AtualizarExecucaoOS` | Início, profissional, serviços, produtos usados, ocorrências | Tela |
| `RegistrarSaidaOS` | Checklist de saída, fotos depois, próxima visita sugerida | Tela |
| `RecomendarProduto` | Sugestão na saída; **não** movimenta estoque | Sim, se for pelo chat |
| `NotificarPetPronto` | Avisa o tutor no WhatsApp que o pet pode ser retirado | Disparo pela tela/OS; se o tutor pedir no chat, confirma antes |
| `EmitirCobranca` | Gera PIX ou link da OS | Sim no chat |
| `BaixarPagamento` | Marca a cobrança como paga | Não — webhook |
| `ConfigurarNivelAcao` | Loja altera confirmação por comando | Só tela / admin |

**Agenda no v1:** por profissional; duração **fixa por serviço**. Não calcula duração por porte/peso. Sem lista de espera, encaixe inteligente, recorrência automática nem capacidade de equipamentos.

**Cadastro 360 no v1:** operacional — dados do tutor, pets, preferências/restrições que afetam o atendimento, agenda, OS, pagamentos, conversas. Fora: LTV, churn, campanhas, tickets, fidelidade, score.

**WhatsApp no v1 (via comando, nunca SQL solto):** agendar, reagendar, cancelar, RSVP (`ConfirmarAgendamento`), check-in se fizer sentido no chat, avisar que o pet está pronto (`NotificarPetPronto`), emitir cobrança, recomendar produto ou próxima visita.

---

## 6. Loop operacional (semana piloto)

```text
Tutor/Pet 360
    → Agendar (tela ou WhatsApp)
    → ConfirmarAgendamento (RSVP do tutor, se o pedido veio do chat ou a loja pediu)
    → Chegada → ConfirmarPresenca → AbrirOS
    → Entrada (fotos / checklist)
    → Execução
    → Saída (fotos, próxima visita, recomendação)
    → EmitirCobranca → PIX/link → BaixarPagamento (webhook)
    → Timeline do pet
```

Esse é o loop que a loja-piloto deve fechar em uma semana e que o episódio deve mostrar.

---

## 7. Dados, eventos e timeline

O banco guarda **estado**. A timeline do pet é **projeção de eventos**, não uma tabela editada na mão.

**Entidades** (sempre com `tenant_id`; operação com `unidade_id`):

- `Tenant` — conta (rede futura ou loja única)
- `Unidade` — loja; v1 = uma
- `Usuario` — quem opera a tela (recepção, tosador, dono)
- `Tutor`
- `Pet`
- `VinculoTutorPet` — v1: um tutor principal; modelo já permite mais de um depois
- `Servico` — duração e preço fixos na unidade
- `Profissional`
- `Agendamento`
- `OrdemServico` — entrada → execução → saída
- `Cobranca`
- `Pagamento`
- `Conversa` / `Mensagem`
- `ComandoLog` — audit: ator, comando, payload, SQL parametrizado, resultado
- `Evento` — fato imutável para a timeline

**Eventos mínimos:** `PetRegistrado`, `AgendamentoCriado`, `AgendamentoConfirmado`, `PresencaConfirmada`, `OSAberta`, `OSEntradaRegistrada`, `OSConcluida`, `PetProntoNotificado`, `CobrancaEmitida`, `PagamentoBaixado`, `ProdutoRecomendado`, `MensagemRecebida`.

Fluxo após comando aceito:

1. Handler grava estado (SQL parametrizado).
2. Emite `Evento`.
3. Atualiza a projeção da timeline.
4. Efeitos de mensagem (ex.: “Thor ficou pronto”) são **comandos**, não o LLM disparando texto solto contra o WhatsApp à revelia do catálogo.

**Isolamento:** agendamento e OS não atravessam unidade. Sem transferência entre lojas nesta fatia.

**Idempotência:** `EmitirCobranca` e `BaixarPagamento` usam chave de idempotência (OS + tipo, e ID do PSP). Webhook repetido não duplica baixa. Reenvio de “sim” no chat não cria segundo banho no mesmo slot.

---

## 8. Copiloto

O orquestrador chama o agente com:

- texto da mensagem
- tutor/pet conhecidos na unidade
- agenda relevante
- catálogo de comandos permitidos
- nível de confirmação por ação

O agente devolve **uma intenção** (`comando` + argumentos). O orquestrador:

1. valida contra o catálogo e o tenant/unidade;
2. no v1, envia confirmação no chat (“Confirmo o banho do Thor sábado 11:30?”);
3. no “sim”, executa o handler;
4. no “não” ou silêncio, não grava.

Há duas camadas de “confirmar”, e não se misturam:

- **Protocolo:** todo comando proposto no chat pede “sim” antes de gravar (padrão v1).
- **Domínio:** `ConfirmarAgendamento` é o RSVP (“vou no sábado”); `ConfirmarPresenca` é a chegada na loja.

Se faltar dado, o agente pergunta. Não inventa horário, preço ou PIX. Pedido fora do catálogo (NFe, transferência de estoque, holding) é recusa explícita.

`ConfigurarNivelAcao` existe no modelo para a loja afrouxar confirmação por comando; o **padrão desta fatia** permanece confirmar antes de gravar no chat.

---

## 9. Cobrança

- `EmitirCobranca` gera PIX ou link atrelado à OS, depois da confirmação no chat (quando o pedido veio do copiloto).
- A loja também pode emitir o mesmo comando pela tela.
- Quando o PSP confirma, `BaixarPagamento` atualiza cobrança e OS. Não pede “sim” no chat.
- PSP fora ou falha ao gerar PIX: comando falha de forma visível; a OS **não** trava; retry é idempotente.
- Conciliação bancária manual, split, assinatura e plano recorrente estão fora desta fatia.

O fornecedor de PSP não está escolhido neste spec; o contrato (emitir → webhook → baixa única) está.

---

## 10. Erros

| Situação | Comportamento |
|---|---|
| Slot ocupado ou serviço inexistente | Comando rejeitado; chat explica e só oferece alternativa que existe de verdade |
| Tutor desconhecido | Pede identificação ou cai na recepção; não cria cadastro fantasma sem confirmação |
| PSP fora / PIX não gerado | Falha visível em `EmitirCobranca`; OS segue; retry idempotente |
| Webhook atrasado ou duplicado | Uma baixa; o sistema não “desbaixa” sozinho |
| Agente fora / timeout | Chat avisa; tela opera normalmente |
| Confirmação expirada | “Sim” tardio não grava; pede confirmação de novo |

---

## 11. Testes mínimos

Prova de contrato, não teatro:

1. O mesmo comando disparado pela tela e pelo chat produz o mesmo efeito no banco e na timeline.
2. “Sim” duplicado não duplica agendamento nem cobrança.
3. Webhook duplicado não duplica baixa.
4. Consulta ou comando de outro tenant não lê nem altera dados.
5. Agente indisponível: agendar na tela funciona.
6. Pedido fora do catálogo é recusado.
7. Confirmação expirada não persiste.

A semana piloto é o teste de aceitação: o loop da seção 6 fecha sem ferramenta paralela para banho & tosa.

---

## 12. Fora desta fatia

Não implementar agora (permanecem no discovery, não neste spec):

- Holding, franquia, permissões por unidade, consolidação financeira, transferência de estoque
- Duração dinâmica de serviço, lista de espera, otimização de ocupação, equipamentos como recurso
- Estoque, compras, fornecedores, varejo completo, POS de prateleira
- Planos/assinatura, ração recorrente, delivery, rotas
- CRM de campanha, LTV, churn, fidelidade, pipeline comercial
- Copiloto executivo (“como está a loja hoje?”), previsões, motor de inteligência
- Portal/app do tutor além do WhatsApp
- Emissão fiscal, folha, comissões complexas

---

## 13. Recomendação de CTO (congelada)

Tratar o catálogo de comandos como o produto. O LLM é cliente do catálogo, não autor do banco.

A semana piloto prova Record + Action. “Copiloto completo” é o teto do **canal** (marcar, reagendar, cobrar, recomendar), não um cheque em branco para gateway mal definido, CRM 360 e inteligência no mesmo corte.

SQL visível no audit da série; SQL composto pelo modelo, nunca.

Recomendação de produto no chat é sugestão com confirmação. Venda que baixa estoque espera o módulo de varejo.

Tenant → Unidade no modelo; holding não entra na semana piloto.

---

## 14. Decisões desta sessão

| Decisão | Escolha |
|---|---|
| Natureza do projeto | Produto real + construção em público (YouTube) |
| Cliente | Rede/franquia **e** loja independente; mesmo sistema |
| v1 operacional | Uma unidade excelente; modelo Tenant → Unidade |
| Loop | Agenda + banho & tosa + cadastro 360 operacional + WhatsApp |
| Copiloto | Linguagem natural: agendar, reagendar, cobrar, recomendar |
| Gravação no chat | Sempre confirma; loja pode configurar nível por ação |
| Escrita | Telas e WhatsApp → mesmos comandos; LLM só no chat |
| Persistência | Orquestrador chama agente; handler edita via SQL parametrizado |
| Cobrança | PIX ou link; baixa automática no webhook |
| Arquitetura | Command OS (não Workflow OS, não Agent-SQL livre) |
| Sucesso | Loja-piloto usa de verdade **e** o episódio mostra o mesmo loop |

Hipóteses ainda abertas (não bloqueiam este spec): linguagem, framework, banco, orquestrador concreto, API de WhatsApp, PSP.

---

## 15. Próximo passo

Depois da revisão humana deste arquivo: plano de implementação da **primeira fatia apenas** (skill writing-plans). Arquitetura de linguagem/stack entra no plano ou em `DECISIONS.md`, não por acréscimo silencioso de escopo.
