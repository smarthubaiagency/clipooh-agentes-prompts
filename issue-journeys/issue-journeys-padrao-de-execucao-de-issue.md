# Issue Journeys — padrão de execução de issue no ClipOOH

Atualizado em 24/09/**2026**.

## Por que este documento existe

Uma *issue journey* é o caminho completo de uma issue no Multica: quem a recebe, o que ela produz, quem a fecha e o que fica registrado depois que ela morre.

O ClipOOH tem hoje três agentes que percorrem esse caminho — Amy, Joey e Max — e cada um foi construído numa sessão diferente. Sem um padrão escrito, cada um vira um desenho próprio: contratos com nomes diferentes para o mesmo campo, códigos de resposta que significam coisas distintas, e a mesma decisão sendo retomada do zero a cada agente novo.

Este documento é a fonte da verdade desse padrão. Ele descreve o comportamento correto, não necessariamente o que já está implementado — a última seção registra a distância entre os dois. Quando o código divergir daqui, o código é que está errado, a menos que a decisão tenha sido mudada de propósito e este documento atualizado junto.

O padrão foi extraído do fluxo da Amy, que foi o primeiro a rodar de ponta a ponta. Em 23/09 o fluxo da Joey também rodou completo, e as correções que ele exigiu estão incorporadas aqui.

## Os quatro momentos de uma issue

Toda issue de agente passa pelos mesmos quatro momentos. O conteúdo muda; a forma não.

**1. Abertura.** A Anna cria a subtarefa e atribui ao agente. A issue carrega o contexto mínimo: a empresa, a fase, o `rise_project_id` da jornada e o que se espera como resultado. O agente lê a issue, não adivinha.

**2. Execução.** O agente faz a parte que lhe cabe, com as próprias ferramentas, e grava o que produziu nos sistemas de registro — Directus para conteúdo, `journey_events` para histórico. Essa gravação é dele, com o `actor` dele. Nada é deixado para *o workflow gravar depois*.

**3. Passagem de bastão.** Quando a etapa dele depende de outro ator — tipicamente um humano no WhatsApp — ele chama uma ferramenta de abertura de canal e **para**. Ele não envia mensagem, não conduz conversa, não fecha a própria issue. A ferramenta devolve um código, e o código diz o que ele faz a seguir.

**4. Fechamento.** Quem conduziu a etapa final é quem fecha a issue — na prática, o workflow do n8n, porque é ele que sabe o instante exato em que a conversa acabou. O fechamento escreve o relatório na issue, muda o status e avisa quem depende dessa issue, sem reacionar o agente que já terminou.

A regra que amarra os quatro: **cada ator executa a parte que lhe cabe, registra o que fez e passa o bastão.** Ninguém refaz o trabalho do anterior e ninguém grava em nome de outro.

## Contrato da ferramenta de abertura

Toda ferramenta que abre um canal com um humano usa os mesmos nomes de campo. A Amy chama a dela de `abrir_briefing`; a Joey, de `abrir_validacao`. O verbo muda, o contrato não.

| Campo | O que é |
| --- | --- |
| `multica_issue_id` | UUID da subtarefa do próprio agente. Nunca o da issue pai |
| `issue_ref` | Identificador legível, ex. `CLIPOOH-42`. Só para leitura humana |
| `rise_project_id` | Chave da jornada no Directus. É por ele que a jornada é resolvida |
| `company_id` | UUID da empresa no Directus |
| `company_code` | Código da empresa, ex. `CLI-000003` |
| `trade_name` | Nome fantasia da empresa ou marca |
| `documento` | CNPJ, só dígitos |
| `objetivo` | Uma frase dizendo o que o humano precisa decidir ou fazer |

Campos específicos de cada agente entram além destes — a Amy leva `contact_name`, `contact_whatsapp`, `stage` e `form_url`; a Joey leva `itens`.

Três regras que valem para qualquer ferramenta desse tipo:

- **`rise_project_id` nunca bloqueia.** Sem ele a conversa abre normalmente e só não fica registrada na jornada. É aviso, não recusa.
- **A ferramenta dispara mensagem real para uma pessoa.** A descrição dela tem que exigir conferência dos dados antes e uma única chamada por issue.
- **A chamada usa `fullResponse` e `neverError`**, para o agente ler o código **HTTP** em vez de receber uma exceção.

## Códigos de resposta

O agente lê o código **HTTP** e age. Não existe *tenta de novo até dar certo*.

| Código | Significado | O que o agente faz |
| --- | --- | --- |
| 200 | Canal aberto, mensagem a caminho | Registra na issue e aguarda. O retorno chega sozinho |
| 400 | Contrato inválido, nada foi aberto | Lê o campo `faltando`, corrige e chama **uma** vez a mais |
| 409 | Recusado, nada foi aberto | Não insiste. Registra na issue e bloqueia |
| 502 | Canal indisponível, nada foi aberto | Pode tentar uma única vez. Se repetir, bloqueia |
| Outro | Desconhecido | Não repete. Bloqueia |

**Não existe **202**.** O padrão da casa é **recusar e mandar bloquear**, nunca enfileirar. Uma fila que avança sozinha parece mais gentil e é pior: o pedido some de vista, ninguém sabe que está parado, e o sintoma aparece justo quando o volume aumenta. Com a recusa, a issue fica bloqueada e visível.

O **409** tem mais de um motivo, distinguidos no campo `erro` do corpo:

- `pedido_ja_aberto` — a mesma issue já abriu esse canal. O agente **não** chama de novo.
- `ocupado` — outra conversa está em andamento no mesmo canal. O corpo devolve qual (marca ou contato), para o agente registrar na issue quem está na frente.

O motivo de existir o caso `ocupado`: o WhatsApp é um canal único por interlocutor. Duas conversas abertas ao mesmo tempo com a mesma pessoa se misturam, e a resposta dela pode ser registrada no cliente errado. O `multica_issue_id` sozinho não protege disso — ele só evita a mesma issue duplicar.

**Nota de implementação:** o webhook precisa estar em `responseMode: responseNode`. No modo padrão ele responde `Workflow got started` a qualquer coisa, os nós de resposta rodam sem devolver nada, e o contrato inteiro existe só no papel.

## Como o n8n fecha a issue no Multica

O Multica **não** é chamado por **HTTP**. O acesso é por nó **MCP** client, endpoint `[https://mcp.multi.clipooh.cloud/mcp`,](https://mcp.multi.clipooh.cloud/mcp`,) transporte `httpStreamable`, com a credencial `Multica **API** [n8n → Multica]`. As ferramentas são `multica_get_task`, `multica_add_comment` e `multica_update_task`.

O fechamento tem seis passos, nesta ordem:

1. **Ler a subtarefa** (`multica_get_task`) para descobrir o identificador legível e o `parent_issue_id`.
2. **Postar o relatório na própria subtarefa** (`multica_add_comment`) com o texto prefixado por `/note`. O prefixo faz o comentário ficar registrado sem redisparar o agente dono da subtarefa — que já terminou e não tem o que refazer.
3. **Mudar o status da subtarefa** (`multica_update_task`) com `suppress_run: true`, pelo mesmo motivo.
4. **Ler a issue pai.**
5. **Comentar na issue pai só se necessário.** Quando a subtarefa vira `done` e todas as irmãs já estão em `done` ou `cancelled`, o próprio Multica posta um comentário de sistema na pai que aciona a Anna. Comentar junto a acionaria duas vezes. Nesse caso o resumo fica só na `/note` da subtarefa.
6. **Se qualquer passo falhar**, deixar nota privada na conversa do Chatwoot avisando que o Multica não foi notificado e que a Anna precisa ser avisada à mão.

Todos os nós dessa cadeia usam `onError: continueErrorOutput` com saída para a nota de falha, e `retryOnFail` com 3 tentativas. Uma falha de rede no Multica não pode perder o resultado de uma conversa que já aconteceu.

O status que a subtarefa recebe depende do desfecho, não do fato de ter terminado:

- objetivo cumprido → `done`;
- conversa encerrada sem o objetivo → `blocked`, com o motivo na nota;
- agente pediu apoio no meio → sem mudança de status, só o comentário para quem vai apoiar.

**O que dispara o fechamento.** O agente da conversa marca `concluido = true` quando todos os itens têm resposta, e o fluxo resolve a conversa no canal. Isso emite o mesmo evento que o encerramento manual, então existe um caminho só com dois gatilhos: o agente terminando, ou a pessoa encerrando à mão como rede de segurança. Encerramento manual antes do fim fecha a subtarefa como `blocked`, com os pendentes na nota.

**Estado vem da fila, não da memória do agente.** Em toda rodada o fluxo lê o estado dos itens e injeta no que chega ao agente, marcado como fonte oficial. Memória de conversa serve para tom e contexto; o que já foi decidido é estado determinístico. Sem isso, numa conversa longa o agente reconstrói a lista de cabeça e reapresenta item já decidido. E quando chega mensagem numa conversa sem etapa ativa, o agente recebe instrução explícita de não improvisar lista — o desenho não pode deixar essa porta aberta.

## Registro na jornada do Directus

A issue morre; a jornada fica. É nela que se lê, meses depois, o que aconteceu com aquele cliente.

**Cada ator grava o próprio passo, com o próprio `actor`.** O agente que executou uma etapa é quem registra o evento dela — não o workflow em nome dele, e não um agente em nome de outro.

O vocabulário de `actor` é o mesmo do campo `confirmed_by` dos `brand_assets`, para não existirem duas listas divergindo: `marcelo_direto`, `joey_multica`, `marcelo_joey_n8n`, `cliente_marcelo`. Para eventos que não são de confirmação, valem os nomes dos agentes: `anna`, `amy`, `max`, `n8n`.

A semântica é **quem decidiu**, não quem digitou. `marcelo_joey_n8n` significa o Marcelo decidindo através da Joey do n8n — a autoridade é dele, o agente é o meio. Registrar `joey_n8n` atribuiria a decisão ao agente.

Divisão entre agente e workflow:

- **Eventos de etapa** são do agente que executou a etapa.
- **Mudanças de estado da jornada** — bloqueio e desbloqueio — são do workflow, porque só ele sabe o instante em que a conversa abriu e encerrou.

Quando uma etapa passa a depender de um humano, a jornada é marcada `bloqueada`, com `blocked_owner` e `blocked_since`. Sem isso a jornada aparece como ativa enquanto espera decisão humana, e o painel mente sobre o estado real. No fechamento, desbloqueia.

Duas regras que evitam retrabalho:

- O resumo vai em `journey_events.summary`, **nunca** num campo único em `journeys` — um campo único é sobrescrito pela passada seguinte, e há pelo menos duas por jornada (design system e campanha).
- O registro na jornada **nunca** bloqueia a conversa. Sem `rise_project_id` ou sem jornada encontrada, o fluxo segue e registra só na issue.

## Quem faz o quê

| Momento | Agente do Multica | Workflow do n8n | Marcelo |
| --- | --- | --- | --- |
| Abertura | lê a issue e o contexto no Directus | — | — |
| Execução | usa as próprias ferramentas, grava o resultado no Directus | — | — |
| Registro da etapa | grava o evento na jornada com o próprio `actor` | — | — |
| Passagem de bastão | chama a ferramenta de abertura e para | recebe, valida, devolve o código | — |
| Conversa | não participa | conduz, grava cada decisão na hora | decide item a item |
| Estado da jornada | — | bloqueia na abertura, desbloqueia no fim | — |
| Fechamento | não fecha a própria issue | `/note` com o relatório, status, avisa a pai | resolve a conversa só se quiser interromper antes do fim |

O que cada um **nunca** faz:

- O agente do Multica não envia mensagem, não conduz conversa e não fecha a própria issue depois de passar o bastão.
- O workflow não grava eventos de etapa em nome do agente, e não reaciona um agente que já terminou (daí `/note` e `suppress_run`).
- Nenhum dos dois grava julgamento sem aprovação humana quando a decisão é do Marcelo.

## Estado atual contra este padrão

Situação em 24/09/**2026**.

**Amy — conforme.** É de onde o padrão veio. Contrato completo, quatro códigos, fechamento com os seis passos, tratamento de falha do Multica. Não grava eventos na jornada: a etapa dela é anterior ao modelo de jornada e isso ainda não foi retroalimentado.

**Joey — conforme, e testada.** Em 23/09 a **CLIPOOH**-42 rodou completa: 7 de 7 itens decididos na conversa, relatório como `/note` na subtarefa, issue em `done`, Anna avisada. Contrato alinhado, os dois motivos de **409**, bloqueio e desbloqueio da jornada, evento de fechamento com `marcelo_joey_n8n`, fechamento no Multica pelos seis passos e auto-resolve da conversa. Pendências:

- Os prompts e skills atualizados existem no repositório mas ainda não foram aplicados no n8n. Enquanto isso, o agente roda a versão anterior — exatamente a divergência que o repositório passa a proibir.
- A credencial da Joey no Directus tem tudo o que os fluxos precisam, menos `create` em `assets` — o prompt diz que ela pode salvar foto que o Marcelo mandar, e não pode. Falta também leitura de `directus_relations`, sem a qual a ferramenta de schema fica cega e o agente tende a inventar nome de campo.
- O prompt da Joey do Multica ainda descreve cor como trabalho dela, mas cor passou a viver inteira na conversa do WhatsApp. Não quebra nada; carrega peso que não usa.
- O caminho feliz do fechamento no Multica rodou; o caminho de falha também foi exercitado e deixou a nota privada como desenhado.

**Max — não construído.** Entra neste padrão desde o início, sem etapa de adaptação.

**Lição que o teste da Joey deixou para todos os agentes:** o único incidente real da sessão veio de duas conversas abertas ao mesmo tempo no mesmo canal — uma de teste, encerrada só na fila e não no canal, e a da aprovação real. Quando a real foi resolvida, as mensagens voltaram a cair na de teste, que não tinha etapa ativa, e nada do que foi dito ali ficou registrado. Encerrar uma etapa significa encerrar a conversa **no canal**, não só marcar a linha como concluída.
