# Issue Journeys — padrão de execução de issue no ClipOOH

Atualizado em 24/09/**2026**, ao final da sessão que reconstruiu a camada de entrada.

## Por que este documento existe

Uma *issue journey* é o caminho completo de uma issue no Multica: quem a recebe, o que ela produz, quem a fecha e o que fica registrado depois que ela morre.

O ClipOOH tem hoje três agentes que percorrem esse caminho — Amy, Joey e Max — e cada um foi construído numa sessão diferente. Sem um padrão escrito, cada um vira um desenho próprio: contratos com nomes diferentes para o mesmo campo, códigos de resposta que significam coisas distintas, e a mesma decisão sendo retomada do zero a cada agente novo.

Este documento é a fonte da verdade desse padrão. Ele descreve o comportamento correto, não necessariamente o que já está implementado — a última seção registra a distância entre os dois. Quando o código divergir daqui, o código é que está errado, a menos que a decisão tenha sido mudada de propósito e este documento atualizado junto.

O padrão foi extraído do fluxo da Amy, que foi o primeiro a rodar de ponta a ponta. Em 23/09 o fluxo da Joey também rodou completo, e o que aquele teste ensinou está incorporado aqui — inclusive a falha que ele revelou e que motivou a reconstrução em curso.

### Onde este arquivo mora

O caminho canônico é `CLIPOOH/claude-code/issue-journeys-padrao-de-execucao-de-issue.md`. Em 24/09 existiam **três** cópias divergentes deste documento na máquina:

| Caminho | Atualizado | Situação |
| --- | --- | --- |
| `claude-code/issue-journeys-padrao-de-execucao-de-issue.md` | 24/09 16:00 | canônico |
| `github/Issue_Journeys_padrao_de_execucao_de_issue_no_ClipOOH.md` | 24/09 13:53 | desatualizado |
| `Issue Journeys — padrão de execução de issue no ClipOOH.md` (raiz) | 23/09 16:47 | desatualizado |

Um documento que existe em três versões não é fonte da verdade de nada. As duas cópias desatualizadas devem ser apagadas ou substituídas por um ponteiro para o caminho canônico. Vale a mesma regra que o documento aplica ao código: quando divergirem, o canônico vence.

## Os quatro momentos de uma issue

Toda issue de agente passa pelos mesmos quatro momentos. O conteúdo muda; a forma não.

**1. Abertura.** A Anna cria a subtarefa e atribui ao agente. A issue carrega o contexto mínimo: a empresa, a fase, o `rise_project_id` da jornada e o que se espera como resultado. O agente lê a issue, não adivinha.

**2. Execução.** O agente faz a parte que lhe cabe, com as próprias ferramentas, e grava o que produziu nos sistemas de registro — Directus para conteúdo, `journey_events` para histórico. Essa gravação é dele, com o `actor` dele. Nada é deixado para *o workflow gravar depois*.

**3. Passagem de bastão.** Quando a etapa dele depende de outro ator — tipicamente um humano no WhatsApp — ele chama uma ferramenta de abertura de canal e **para**. Ele não envia mensagem, não conduz conversa, não fecha a própria issue. A ferramenta devolve um código, e o código diz o que ele faz a seguir.

**4. Fechamento.** Quem conduziu a etapa final é quem fecha a issue — na prática, o workflow do n8n, porque é ele que sabe o instante exato em que a conversa acabou. O fechamento escreve o relatório na issue, muda o status e avisa quem depende dessa issue, sem reacionar o agente que já terminou.

A regra que amarra os quatro: **cada ator executa a parte que lhe cabe, registra o que fez e passa o bastão.** Ninguém refaz o trabalho do anterior e ninguém grava em nome de outro.

## A regra que faltava: gravação declarada não é gravação

Adicionada em 24/09, depois do caso da BRD-000007.

O momento 2 diz que o agente grava. Ele não diz o que fazer quando o agente **afirma** ter gravado e não gravou — e foi exatamente isso que aconteceu. O relatório final da Joey do n8n dizia, literalmente, *"Registro do item 7 feito em brands.logo_min_width_px"*. O campo estava nulo. Nenhum dos 7 itens aprovados foi gravado. A execução do n8n terminou com status `success`, a issue fechou como `done`, a Anna foi avisada, e o padrão foi cumprido em todos os pontos verificáveis pelo fluxo — porque nenhum ponto do fluxo verificava o que importava.

Daí a regra:

> **Nenhuma etapa encerra com base no que o agente diz ter feito. O fluxo relê o registro e compara antes de aceitar `concluido = true`.**

O que isso implica em cada camada:

- Antes de fechar a etapa, o fechamento **relê** no Directus os campos que a etapa deveria ter escrito e confere contra o critério de pronto declarado na issue.
- Se o que deveria estar gravado não está, a etapa **não** fecha como `done`. Fecha como `blocked`, com a divergência na nota: o que o agente afirmou, o que o Directus mostra.
- Sucesso de execução do n8n **não** é sinal de sucesso da etapa. Todo nó que grava por conta de terceiro precisa de leitura de volta, ou a falha vira `success`.

**Sobre o modelo do agente.** Houve duas trocas de motor e elas não se confundem. Na corrida de 23/09 — a que aprovou sete itens na BRD-000007 e não gravou nenhum campo — a Joey rodava `openai/gpt-5.6-terra`, conforme a versão `909505ca` publicada em 23/09 às 15:38. O motor era capaz e ainda assim ela declarou uma gravação que não fez. O `openrouter/xiaomi/mimo-v2.6-flash` só foi publicado em 24/09 às 17:03 e responde por uma falha de outra natureza: saída estruturada inválida, que abortava a rodada no `Message an Agent` e deixava estado órfão (conversa aberta e linha de fila em andamento). A troca para `mimo-v2.6-pro`, em 24/09, resolveu a falha de schema. Nenhuma das duas é a causa a corrigir: qualquer modelo pode relatar uma gravação que não fez, e um desenho que só funciona com o motor bom não é um desenho. A correção é a verificação acima, que vale para todos os motores. A política de motor por agente continua valendo e a troca para teste deve voltar ao canônico antes de qualquer corrida que conte como válida.

### Estado declarado não pode apagar estado guardado

Em 25/09, com a Joey rodando `deepseek-v4.1-flash`, o agente devolveu saída estruturada **válida e vazia**:

```json
{"mensagem_para_marcelo": "placeholder", "concluido": false, "itens": []}
```

Passou por toda validação, porque é JSON correto. O nó de gravação da fila substituiu os sete itens pela lista vazia, e **as quatro cores já aprovadas sumiram do estado** — sem erro, sem aviso, sem execução falhada.

A lição não é sobre o modelo. Qualquer motor com soluço aciona isso, e o desenho tem de sobreviver a soluço. O que faltava era a mesma regra que as instruções do agente já carregam para o Directus — *nunca apague; valor antigo não se perde* — aplicada ao próprio estado da rodada.

**A regra, agora implementada no `Conferir o que mudou` da camada 2:** a lista guardada é o **piso**. O retorno do agente só pode **promover** item de `pendente` para decidido. Não pode encolher a lista, não pode desfazer decisão, não pode inventar item fora da lista fixa. Cada recusa vira aviso no `avisos` da rodada, em vez de sumir.

Dois efeitos colaterais que valem por si:

- **Recibo só para o que sobreviveu ao guarda.** Os recém-decididos passaram a sair da lista já guardada, não do retorno cru. Item que o guarda recusou não ganha recibo.
- **Fechamento fail-closed.** O `concluido` do agente só vale se a lista guardada não tiver mais nenhum pendente. Antes, o `Concluiu a aprovacao?` lia direto a saída do modelo — bastava ele declarar para a etapa fechar.

Provado em três cenários com o código real do nó: lista vazia mantém tudo; promoção de pendente passa; tentativa de desfazer decisão é recusada com aviso.

## A reconstrução em quatro camadas

Em curso desde 24/09. O motivo: Amy e Joey são hoje dois monólitos de ~103 e ~105 nós, quase idênticos, construídos por cópia manual. Cada bug estrutural precisa ser lembrado e corrigido duas vezes à mão, sem herança nenhuma entre eles, e o Max no mesmo molde multiplicaria isso por três.

O fluxo passa a ser quatro camadas:

| Camada | O que faz | Compartilhada? |
| --- | --- | --- |
| **1. Entrada do Multica** | recebe o pedido de abertura, valida o contrato daquele agente, decide a admissão, resolve o interlocutor e abre a conversa | sim, uma para os três |
| **2. Agente de IA** | conduz a conversa, decide e grava o que for decidido | não — um por agente |
| **3. Canal (Chatwoot)** | recebe a mensagem, normaliza o contexto, monta o estado da rodada e entrega a resposta | sim |
| **4. Fechamento** | confere o registro, fecha a issue no Multica, desbloqueia a jornada | sim |

O que é genuinamente específico de cada agente: as `instructions` e skills (que já vivem no objeto Agent do n8n, não nos nós) e a gravação no Directus, porque a Amy grava briefing, a Joey grava marca e o Max grava veredicto de aprovação. Todo o resto é a mesma lógica e deve existir uma vez só.

Já existia um subfluxo nessa direção — `Contexto do contato [CLIPOOH]`, cuja descrição diz "usado por Amy; depois Joey e Max". A intenção estava certa e não foi longe o bastante.

### As quatro camadas substituem os fluxos individuais — não convivem com eles

Registrado em 25/09. Até aqui o documento tratava a reconstrução como obra paralela; ela não é. **As quatro camadas existem para aposentar `JOEY Assistente Marcelo [CLIPOOH]` e `AMY Head of Art — Briefing [CLIPOOH]`**, e o Max nasce nelas em vez de virar um terceiro monólito.

A conferência por execução, feita em 25/09, mostra o tamanho do engano que estava em curso:

| Workflow | Execuções | Modo |
| --- | --- | --- |
| `JOEY Assistente Marcelo` | 598 | webhook, produção |
| `AMY Head of Art — Briefing` | 159 | webhook, produção |
| `Entrada MULTICA F1` | 13 | todas manuais |
| `Agentes MULTICA F2` | 12 | todas manuais |
| `Fechamento MULTICA F4` | 2 | todas manuais |
| `Chatwoot MULTICA F3` | **0** | nunca executou |

**Nenhuma mensagem real jamais passou pelas camadas novas.** As 27 execuções delas são testes manuais, nó a nó. Os quatro workflows estão `active: true`, o que sugere "no ar" e não é: ativo significa que o workflow aceita gatilho, não que alguém entregue nele.

### A numeração descreve a abertura, e a camada 2 é um centro

A ordem é **entrada (1) → agente (2) → canal (3) → fechamento (4)**, que é o caminho de nascimento de uma conversa: o Multica pede, a entrada admite, o agente redige, o canal entrega.

O caminho de resposta corre na outra direção — mensagem chega pelo canal e vai ao agente. Por isso **a camada 2 não é uma estação numa fila: é um centro chamado pelas duas portas.** A camada 1 a chama na abertura, a camada 3 a chama a cada resposta. Quem tentar ler os quatro números como um encadeamento linear vai se confundir, e foi o que aconteceu.

Fica uma dívida de nomenclatura: o código do `Normalizar contexto` da F2 ainda se descreve como "camada 3" e chama a recepção de "camada 2", da numeração antiga. Corrigir junto com a fiação.

### Os três pontos que faltam ligar

| Ligação | Estado |
| --- | --- |
| F1 → F2 (abertura) | aponta para `WORKFLOW TESTE`, um placeholder, com mapeamento vazio |
| F3 → F2 (resposta) | aponta certo, mapeamento vazio |
| F2 → F4 (fechamento) | **não existe** — nenhum nó chama a F4 |

Os dois primeiros são o mesmo defeito: o `Execute Workflow` foi criado e o contrato de entrada nunca foi preenchido. Como o `Normalizar contexto` falha alto por desenho, hoje as junções quebram na primeira passagem — que é o comportamento certo, só falta o conteúdo.

O contrato da camada 2, lido do código em 25/09: obrigatórios `agente`, `conversation_id`, `account_id`, `contact_id`; mais `modo` (`abertura` ou `continuacao`), e os opcionais `inbox_id`, `evolution_instance`, `origem`, `base_url`, `memory_key`, `celPhone`, `nome`, `texto_do_marcelo`, `responder_em_voz`, `message_id_verificado`, `multica_issue_id`, `issue_ref` e `marca`. O trigger é `passthrough`: o chamador entrega o objeto inteiro, então cada porta precisa de um nó que monte exatamente esse contrato antes da chamada.

### Duas decisões que a fiação obriga

**O envio mora na camada errada.** A camada 3 recebe do Chatwoot, mas quem envia são os nós `Chatwoot - Enviar (Joey/Amy/Max)` dentro da camada 2, com leque de credencial por bot. Com isso a camada do canal tem só metade do trabalho — e a verificação de entrega, que este documento define como sendo dela, nasceria no lugar errado. A saída limpa é a camada 2 devolver o texto e a camada 3 entregar.

**A porta da camada 3 é Joey-only.** O `Verificar remetente` dela tem `INBOX_JOEY = '8'` e o número do Marcelo cravados, e descarta em silêncio o que não bate. Do jeito que está, virar o Chatwoot para ela faria sumir toda mensagem da Amy (inbox 9) e do Max (10). Precisa ser indexada por agente, como o `CONFIG` da camada 1 já é.

### A ordem da virada

1. Preencher o contrato nas três junções (F1→F2, F3→F2, F2→F4).
2. Generalizar a porta da camada 3 para as três inboxes.
3. Mover o envio da camada 2 para a camada 3.
4. Guarda de gravação da fila: nunca aceitar lista de itens vazia ou menor que a guardada — ver a seção do caso de 25/09.
5. Só então mover os dois ponteiros: o webhook do Chatwoot de `cam-SyV1y2a0Gq57jlhxghdqYkI1bdq321IW` para `agentes-multica`, e a tool de abertura dos agentes de `joey-abrir-validacao` para `multica-abrir-conversa`.
6. Arquivar os dois monólitos — não desativar e esquecer: arquivar, para não voltarem a atender por engano.

**Feito em 25/09:** os passos 1 e 4 estão aplicados. As três junções têm nó de contrato próprio — `Montar contrato da camada 2` na F1 e na F3, `Montar contrato da camada 4` na F2 — e a chamada da F1 deixou de apontar para o placeholder `WORKFLOW TESTE`. A F3 passou a derivar o `agente` da inbox (8 joey, 9 amy, 10 max) no nó de contrato, o que começa a generalização do passo 2 mas **não** substitui o conserto do `Verificar remetente`, que continua Joey-only e ainda descarta Amy e Max antes de chegar ali. Os três workflows ficaram em rascunho, não publicados.

A Amy precisa de plano próprio nessa virada. Ela não é uma camada: é um fluxo inteiro paralelo, com dois gatilhos e a lógica de briefing dentro. Migrar a Joey não a move junto.

## Camada 1 — entrada compartilhada

Implementada e testada em 24/09 no workflow `Entrada MULTICA` (`do644VHyIhBGn3Mg`), 43 nós. **Não está em produção**: o webhook dela é `multica-abrir-conversa`, e as tools dos agentes continuam chamando os fluxos antigos.

A camada tem **credencial própria** no Directus — `Header Auth Directus - Multica Access [CLIPOOH]` — com exatamente quatro permissões: ler `contacts`, ler `journeys`, atualizar `journeys`, criar `journey_events`. Nada de escrita em `brands` ou `brand_assets`, de propósito: assim a camada não consegue gravar resultado de etapa em nome de agente nenhum, nem por engano. Antes ela usava o token do agente Joey, que não lia `contacts` e por isso recusava todo pedido do perfil cliente.

### Perfis

Os três agentes não são três casos especiais. São **dois perfis**, e a Joey é a exceção:

| | Joey | Amy | Max |
| --- | --- | --- | --- |
| perfil | `casa` | `cliente` | `cliente` |
| com quem fala | Marcelo | contato técnico do cliente | contato técnico do cliente |
| inbox do Chatwoot | 8 | 9 | 10 |
| instância Evolution | `joey-d31fec` | `amy-clipooh-9d9008` | `max-clipooh-0f6121` |
| escopo de ocupação | `global` | `empresa` | `projeto` |
| quem destrava a jornada | `marcelo` | `cliente` | `cliente` |

O perfil `casa` fala com uma única pessoa e por isso é estritamente serial. O perfil `cliente` fala com vários clientes em paralelo e trava por empresa ou por projeto, nunca entre clientes diferentes. O Max é "mais um agente do perfil cliente", não um terceiro desenho — é isso que o faz sair barato.

### Passe e regras

O pedido carrega um **passe** com identidade e canal: `agente`, e opcionalmente `inbox_id` e `evolution_instance`. Esses valores são **estáticos na definição da tool** de cada agente, nunca `$fromAI` — um modelo não escolhe por qual inbox se fala.

As **regras de admissão não viajam no passe.** Elas vivem na entrada, indexadas pelo agente. Se o pedido trouxesse as próprias regras, o agente declararia a si mesmo aceitável — é a regra dura "não aprove sua própria proposta" aplicada à infra.

A entrada confere o passe contra a configuração da casa. Passe de agente desconhecido, ou `inbox_id`/`evolution_instance` divergentes do que aquele agente opera, são **400**. Nunca "abre e vê no que dá".

### O interlocutor sai do Directus, não do pedido

No perfil `cliente`, quem descobre o número é a entrada, lendo `contacts` filtrado por `company`, com `types` contendo `technical` e `archived = false`, preferindo `is_primary`. O agente diz **de qual empresa/projeto é**; a entrada resolve com quem se fala.

O contrato continua prevendo `contact_name` e `contact_whatsapp`, e eles continuam aceitos — mas o significado mudou: são **o que o agente acredita**, não o número a usar. Divergência entre o declarado e o Directus gera aviso e o Directus vence. Não recusa o pedido.

O motivo é o perfil `cliente`: um número errado manda o material de um cliente para o contato de outro. Resolver na entrada torna isso estruturalmente impossível, em vez de depender de o modelo acertar o campo. No perfil `casa` o par é fixo na configuração, porque o interlocutor é sempre o mesmo.

Sem contato técnico cadastrado não existe com quem falar: **400** com `erro: sem_contato_tecnico`. A recusa acontece antes de criar linha na fila — nada foi aberto, nada a limpar.

Duas armadilhas desse trecho, as duas descobertas no teste de 24/09 e as duas silenciosas:

**`contacts.types` é campo `json`, e este Directus recusa o operador `_contains` em `json`** (`"json field type does not contain the _contains filter operator"`). Filtrar o papel na consulta devolve erro, e com `neverError` o erro chega como resposta vazia: o pedido cai em `sem_contato_tecnico`, uma recusa plausível apontando para o lugar errado. Por isso a consulta traz os contatos não arquivados da empresa e a escolha do `technical` acontece no código. E por isso o nó que escolhe também inspeciona `errors` na resposta e leva o texto do erro para os `avisos` — falha de permissão não pode virar "não achei contato".

**Ordenar por `is_primary` não serve para escolher.** Na CLI-000003 a contato principal é a Viviane, que é `administrative`; o `technical` é o Marcelo, com `is_primary = false`. Um `limit 1` no Directus abriria conversa com a pessoa errada. O papel decide primeiro; `is_primary` só desempata entre vários `technical`.

### O vínculo com a inbox do Chatwoot não é idempotente

`POST /contacts/{id}/contact_inboxes` **cria um vínculo novo a cada chamada**, com um `source_id` novo. Chamar sempre faz o contato acumular vínculos duplicados para a mesma inbox — em 24/09 o contato de teste já tinha quatro para a inbox 9, um por corrida anterior da Amy. Com vários vínculos, escolher o `source_id` vira sorteio, e um `source_id` que a instância do Evolution não conhece não entrega mensagem: o sintoma é o agente ficar mudo, sem erro nenhum.

A entrada confere antes e só cria o vínculo quando ele falta.

### A fila

Tabela `fila_conversas` (`RGslCGgQgLpsU05a`), compartilhada pelos três. Substitui a `joey_fila_aprovacoes`, que fica como histórico da Joey e não deve ser estendida.

A ocupação conta **apenas** linhas `em_andamento`, e dentro do escopo do agente:

- `global` — existe linha `em_andamento` deste agente? (Joey)
- `empresa` — deste agente **e** mesma `escopo_chave` = `company_id` (Amy)
- `projeto` — deste agente **e** mesma `escopo_chave` = `rise_project_id` (Max)

Sem `rise_project_id`, o escopo `projeto` cai para `empresa`, com aviso. É mais restritivo, portanto seguro, e mantém intacta a regra dura de que `rise_project_id` nunca bloqueia.

### `em_bloqueio` — o estado que destrava

Linha com problema recebe `status = em_bloqueio`, um `bloqueio_motivo` e um aviso ao Marcelo. Como a ocupação só conta `em_andamento`, isso **libera a fila** e ao mesmo tempo deixa o caso visível para inspeção. Não reprocessa nada: a etapa não é retomada sozinha.

São duas classes de travamento, e as duas precisam de tratamento:

**Falha detectada durante a execução** — Chatwoot fora, Directus recusando. Há alguém vivo para marcar `em_bloqueio`, comentar na issue do Multica e devolver o código ao agente.

**Execução que morreu** — crash, timeout, n8n reiniciado. Ninguém sobrou para marcar nada e a linha fica `em_andamento` órfã, travando o escopo para sempre. Resolvido sem cron: cada linha nasce com `expires_at`, e **a própria entrada varre na passada**. Ao receber um pedido novo, linha `em_andamento` vencida conta como morta, é marcada `em_bloqueio` e não ocupa. A fila se cura na próxima tentativa, sem depender de nada rodando por fora.

TTL atual: **120 minutos** para os três. Número provisório — precisa de decisão.

### Liberar a linha não basta: encerre a conversa no canal

Esta é a lição do incidente de 23/09 aplicada à varredura, e é a parte que erra fácil.

Marcar a linha `em_bloqueio` libera a fila. Se a conversa daquela linha ainda estiver **aberta no canal**, o pedido seguinte abre uma **segunda** conversa com a mesma pessoa, as mensagens se misturam e a resposta pode ser registrada no cliente errado. Na Joey, cujo interlocutor é sempre o Marcelo na inbox 8, isso é garantido.

Por isso a varredura leva o `conversation_id` da linha vencida adiante e **resolve a conversa no Chatwoot** antes de seguir. Encerrar uma etapa significa encerrar a conversa no canal, não só marcar a linha.

### Ordem da resposta

O `200` é o **último** nó, depois de gravar `conversation_id` na fila, bloquear a jornada e gravar o evento. Um `200` significa que tudo aterrou.

Antes, o `200` saía antes das gravações do Directus: o agente recebia "conversa iniciada" e o bloqueio da jornada acontecia depois, sem ninguém esperando por ele. É o mesmo padrão de reportar sucesso sem conferir que a nova regra proíbe.

## Camada 2 — o fluxo do agente

Extraída em 24/09 para o workflow `Agentes MULTICA F2` (`x5LFoVcCM4d8TjEc`), 40 nós. **Não está ligada**: nada a chama ainda, e um webhook temporário faz o papel da camada 3 nos testes.

### O contrato chega pelo trigger, e contrato quebrado falha alto

Dentro do monólito, o nó de normalização reconstruía o contexto lendo **nós irmãos**: `$('Settings v2')`, `$('Message Debounce')`, `$('Chatwoot - Criar conversa da aprovacao')`, `$('Validar pedido Paperclip')`. Extraída como subworkflow, esses nós deixaram de existir — e o resultado foi instrutivo: o ramo do Chatwoot lançava exceção, e o do Paperclip degradava **em silêncio** dentro de um `try/catch`. Nesse silêncio, o `message_id_verificado` desaparecia, e o agente passava a ser instruído a *não gravar aprovação nenhuma* sem que erro nenhum aparecesse.

Leitura implícita de vizinho não sobrevive à modularização. O contexto passa a chegar pelo trigger, e a normalização só valida:

`agente`, `modo` (`abertura`/`continuacao`), `base_url`, `account_id`, `inbox_id`, `contact_id`, `conversation_id`, `memory_key`, `celPhone`, `nome`, `texto_do_marcelo`, `responder_em_voz`, `message_id_verificado`, e na abertura `multica_issue_id`, `issue_ref` e `marca`.

Campo faltando, agente desconhecido, agente sem Agent criado no n8n, ou `modo` inválido: **exceção com mensagem específica**. Nenhum caminho degrada. O trigger de subworkflow entrega os campos no topo do `json` e um webhook os embrulha em `.body`; a normalização aceita as duas formas, para o disparador de teste exercitar o mesmo caminho da camada 3.

### Credencial é estática por nó — o fan-out mora aqui

Duas coisas precisam de credencial diferente por agente, e credencial no n8n **não aceita expressão**:

- **Evolution** (indicador de digitando): cada instância tem API key própria, em `Header Auth {Joey|Amy|Max} Evolution GO Instancia [CLIPOOH]`.
- **Chatwoot** (envio da mensagem): cada agente tem o token do próprio AgentBot.

A saída é um `Switch` por `agente` com três nós em cada ponto. Isso é feito **uma vez, na camada compartilhada** — replicar esse fan-out em três workflows de agente é exatamente o custo que a reconstrução existe para eliminar.

### Os agentes falam como bot, não como pessoa

Antes as mensagens saíam com o token de usuário do Chatwoot e apareciam atribuídas a *Marcelo Menezes*. Passam a sair com o `access_token` do AgentBot de cada um, e o `sender` volta como `type: agent_bot`.

Duas armadilhas no caminho, as duas confirmadas por teste:

- A tela "Alterar Robô" do Chatwoot tem **dois** segredos: **Token de acesso** (o que autentica o bot na API) e **Segredo do Webhook** (que serve para o n8n verificar que a chamada veio do Chatwoot). Só o primeiro serve para enviar.
- O **nome do header** na credencial tem de ser exatamente `api_access_token`. Com outro nome o Chatwoot trata a chamada como anônima.

E vale registrar como o diagnóstico se fez, porque a distinção economiza tempo: **401** (`"You need to sign in"`) é header ou token não aceito; **404** (`"Resource could not be found"`) é token aceito mas o recurso não existe ou não é visível. Um 404 na prova não era credencial — eram conversas de teste já apagadas. AgentBot **não** é restrito à própria inbox: o token da Amy posta numa conversa da inbox da Joey.

### A conferência do recibo

É a aplicação da regra "gravação declarada não é gravação" no lugar mais cedo possível: entre o agente e a fila, antes de o item contar como decidido.

A cada rodada, o fluxo separa os itens que o agente **acabou** de declarar decididos — item fechado em rodada anterior não é reconferido, ele tem o recibo da rodada dele. Para esses, relê `brands.notes` e procura o **id da mensagem daquela rodada**, que a skill de registro já obriga o agente a citar no bloco de procedência.

- Recibo presente: o que o agente declarou vale.
- Recibo ausente: o item **volta a `pendente`** e a divergência é registrada.
- Leitura do Directus falhou: também volta a pendente. Falha de leitura não pode passar por "gravou".
- Sem `brand_id` ou sem `message_id_verificado`: não há como conferir, e aí é **fail-closed** — o item não é marcado. Deixar passar seria reintroduzir a falha da BRD-000007 tal e qual.

O bloco de procedência é o **recibo universal**: uma leitura por rodada, sem mapa de campo por agente. Conferências de campo específicas podem ser somadas por cima, incrementalmente, sem que a rede de proteção dependa delas.

E a divergência **volta ao agente** na rodada seguinte, dentro do bloco `ESTADO DA APROVAÇÃO`, com instrução explícita: *não repita a pergunta ao Marcelo, a decisão dele já existe; o que falta é a gravação*. Assim a correção acontece na conversa, com o humano presente, e não no fechamento, quando ninguém mais está lendo.

Por isso o contrato da camada 1 ganhou `brand_id`, obrigatório para a Joey: sem ele a conferência não tem alvo.

## Camada 3 — a porta compartilhada

Generalizada em 25/09. Antes era a porta da Joey copiada: `INBOX_JOEY = '8'` e o telefone do Marcelo cravados no código, descartando em silêncio qualquer mensagem da Amy (inbox 9) ou do Max (10). Virar o canal para ela teria feito sumir todo atendimento de cliente sem um erro sequer.

A porta faz **duas perguntas, e elas não se confundem**:

**1. A mensagem é autêntica?** O webhook do Chatwoot pode ser forjado; a API, que exige credencial, não. A porta confere contra a API: a mensagem existe, é desta conversa, o conteúdo bate com o do webhook, tem menos de 10 minutos, e o anexo é o mesmo. Isso não tem nada a ver com quem enviou.

**2. Quem enviou pode decidir nesta conversa?** Aqui estava o telefone cravado, e aqui está a mudança.

### A fila é que autoriza, não o cadastro

Quando a camada 1 abre a conversa, ela grava na linha da fila o `contact_id` do interlocutor — inclusive o da casa, que vem do `contato_fixo` da configuração. A autorização virou então uma **comparação de id**: o remetente que o Chatwoot confirma tem de ser o mesmo `contact_id` que a linha registrou para aquela conversa.

Três ganhos, e o terceiro é o que importa:

- acaba o número de telefone no código, e a porta passa a servir os três agentes sem configuração nova;
- acaba a ambiguidade de formato — id não tem nono dígito, `+`, nem variante;
- **o escopo passa a ser a conversa, não a pessoa.** Um contato técnico legítimo da empresa A deixa de poder responder numa conversa aberta para a empresa B, porque o id não bate.

**Ficha do contato não autoriza.** O subfluxo `Contexto do contato [CLIPOOH]` (`yAn4zwYrpN8bli2O`) recebe um telefone e devolve nome, empresas, papéis, fase e formulários da fase. Ele responde *"quem é esse telefone?"*, que é pergunta de contexto, não de credencial: contato conhecido não é contato autorizado para uma conversa específica. O lugar dele é a ficha injetada no payload do agente — como a Amy já recebe, e como a própria descrição dele antecipa ("usado por Amy; depois Joey e Max").

Há ainda um detalhe que fecha o argumento para o perfil casa: o interlocutor da Joey é da casa e **não sai do Directus** — é por isso que a camada 1 tem `contato_fixo`. Uma busca por telefone no cadastro devolveria "não encontrado" e a porta barraria justamente quem manda.

Ao usar o subfluxo para a ficha, trocar a credencial: ele hoje usa `Header Auth Amy Directus [CLIPOOH]`, e Joey e Max herdariam a credencial da Amy — a mesma que morreu em 24/09. A credencial da camada é a certa, e precisa ganhar leitura de `forms`, que hoje ela não tem.

### O que continua em aberto

**Conversa solta no perfil cliente.** Sem linha de fila aberta, nada autoriza a mensagem e a porta recusa — decisão conservadora, registrada no código. Isso colide com a regra *"o agente nunca emudece"*: quando a Amy migrar para esta porta, a conversa fora de etapa dela entra por aqui e essa regra precisa de decisão própria. No perfil casa não há o impasse: sem linha de fila, só o contato da casa fala.

**O mapa `inbox → agente` vive num lugar só**, dentro da porta. O nó de contrato lê o agente já resolvido em vez de repetir o mapa — duas cópias divergem, e foi assim que a divergência de numeração das camadas nasceu.

## Camada 4 — o fechamento

Extraída em 24/09 para o workflow `Fechamento MULTICA F4` (`1nYLjBiANZom8dEn`), 35 nós. **Não está ligada.** É o ramo de `resolved` do fluxo antigo, e era o mais limpo dos quatro para extrair — o que não a tornou correta: três defeitos graves apareceram na análise.

### O critério de pronto se relê, e é ele que decide o desfecho

Era aqui o buraco. A decisão entre `done` e `blocked` saía **só** da contagem de itens pendentes na fila. Nada conferia o critério escrito na issue.

A conferência do recibo da camada 2 garante que cada item decidido teve registro na rodada dele. Isso **não** garante o critério da etapa: o agente pode ter os itens todos com recibo e nunca ter levado a marca a `kit_status = confirmed`. Era exatamente esse o critério declarado na CLIPOOH-42, e foi por não conferi-lo que ela fechou como `done` contra o próprio relatório.

O critério é por agente:

| Agente | Critério | Onde se confere |
| --- | --- | --- |
| Joey | `brands.kit_status = confirmed` e `confirmed_by` diferente de `sem_confirmacao` | releitura de `brands/{brand_id}` |
| Amy | nenhum — o acesso dela ao Directus é somente leitura, ela não grava conteúdo próprio | desfecho sai dos itens e das divergências |
| Max | **ainda não definido** | fail-closed: não fecha como `done` |

Além do critério, bloqueiam o `done`: item ainda pendente, divergência não resolvida vinda da camada 2, `brand_id` ausente, `pedido` ilegível, e falha na releitura do Directus. **Fechar como `done` um agente sem critério definido é assinar em branco** — daí o fail-closed do Max.

### A jornada só desbloqueia se a etapa cumpriu

O desbloqueio era incondicional: conversa encerrada com pendências fechava a subtarefa como `blocked` **e** devolvia a jornada para `ativa`. O painel passava a mentir na direção oposta à de 23/09 — etapa travada, jornada verde.

Agora o desfecho decide:

- **`done`** → `status = ativa`, campos de bloqueio limpos, e evento `desbloqueio` com `actor: n8n`.
- **`blocked`** → jornada **mantida** `bloqueada`, `blocked_reason` **atualizado** com o que falta, `blocked_owner = house`, e evento `bloqueio` com `actor: n8n`.

O evento da etapa é gravado nos dois casos, porque a etapa aconteceu de um jeito ou de outro. O `type` e o `actor` dele vêm do agente — estavam chumbados como `gate`/`marcelo_joey_n8n`, o que só serve para a Joey.

E o que esta camada **não** faz, de propósito: não avança `journeys.phase` e não grava `closed_at`. Fase é passo da Anna, e `closed_at` só com o resultado do gate do Max. A camada habilita o fechamento do ciclo; quem fecha é a Anna.

### O contrato, outra vez

`Mudou para resolved?` lia `$json.body.changed_attributes` e a busca na fila lia `$json.body.id` — dependência do payload cru do Chatwoot chegar embrulhado em `.body`. Mesma correção da camada 2: o contrato chega pelo trigger e é validado, aceitando tanto a forma normalizada (`{conversation_id, evento: 'resolved', agente}`) quanto o evento cru, no topo ou em `.body`. Sem `conversation_id`, exceção.

O `agente` pode vir no contrato ou nos `custom_attributes` que a camada 1 grava na conversa, mas **a linha da fila é a autoridade** sobre de quem é a conversa.

### O defeito que a tabela nova escancarou

Os dois nós de fila apontavam para a `joey_fila_aprovacoes` e o fechamento lia `fila.itens`. Na `fila_conversas` o pedido vive em `pedido` e não existe coluna `itens` — então o parse devolvia `{}`, a lista saía vazia, `pendentes` era zero e **toda** subtarefa fecharia como `done` com "0 de 0 itens decididos", sem erro nenhum. É o mesmo padrão de falha silenciosa que a reconstrução existe para eliminar, e só apareceu porque a mudança de tabela foi conferida campo a campo em vez de por analogia.

### Vocabulário de `status` da fila

`em_andamento` (ocupa a fila), `em_bloqueio` (liberada, aguardando inspeção humana) e `concluido` (encerrada pelo fechamento). Só `em_andamento` conta para ocupação.

## Contrato da ferramenta de abertura

Toda ferramenta que abre um canal com um humano usa os mesmos nomes de campo. A Amy chama a dela de `abrir_briefing`; a Joey, de `abrir_validacao`. O verbo muda, o contrato não.

| Campo | O que é |
| --- | --- |
| `agente` | `amy`, `joey` ou `max`. É por ele que a entrada escolhe as regras |
| `brand_id` | UUID da marca no Directus. Obrigatório para a Joey: é o alvo da conferência do recibo |
| `multica_issue_id` | UUID da subtarefa do próprio agente. Nunca o da issue pai |
| `issue_ref` | Identificador legível, ex. `CLIPOOH-42`. Só para leitura humana |
| `rise_project_id` | Chave da jornada no Directus. É por ele que a jornada é resolvida |
| `company_id` | UUID da empresa no Directus |
| `company_code` | Código da empresa, ex. `CLI-000003` |
| `trade_name` | Nome fantasia da empresa ou marca |
| `documento` | CNPJ, só dígitos |
| `objetivo` | Uma frase dizendo o que o humano precisa decidir ou fazer |
| `inbox_id`, `evolution_instance` | Passe do canal. Estáticos na tool. Divergência é 400 |

Campos específicos de cada agente entram além destes — a Amy leva `contact_name`, `contact_whatsapp`, `stage` e `form_url`; a Joey leva `itens`.

Obrigatórios por agente, como a entrada valida hoje:

- **Amy** — `multica_issue_id`, `company_id`, `objetivo`, `stage`, `form_url`
- **Joey** — `multica_issue_id`, `company_id` ou `company_code`, `brand_id`, `trade_name`, `documento`, `objetivo`, `itens`
- **Max** — `multica_issue_id`, `company_id`, `objetivo`

Três regras que valem para qualquer ferramenta desse tipo:

- **`rise_project_id` nunca bloqueia.** Sem ele a conversa abre normalmente e só não fica registrada na jornada. É aviso, não recusa. Vale inclusive para quem tem escopo por projeto.
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

O **400** tem mais de um motivo, no campo `erro`:

- `contrato_invalido` — falta campo obrigatório daquele agente. O corpo lista em `faltando`.
- `agente_desconhecido` — o `agente` do passe não está na configuração da entrada.
- `sem_contato_tecnico` — a empresa não tem contato técnico ativo no Directus. Não é erro do agente; é cadastro faltando.

O **409** também:

- `pedido_ja_aberto` — a mesma issue já abriu esse canal. O agente **não** chama de novo.
- `ocupado` — outra conversa está em andamento no mesmo escopo. O corpo devolve `ocupacao_escopo`, `ocupacao_chave` e a marca de quem está na frente, para o agente registrar na issue.

O motivo de existir o caso `ocupado`: o WhatsApp é um canal único por interlocutor. Duas conversas abertas ao mesmo tempo com a mesma pessoa se misturam, e a resposta dela pode ser registrada no cliente errado. O `multica_issue_id` sozinho não protege disso — ele só evita a mesma issue duplicar.

**Nota de implementação:** o webhook precisa estar em `responseMode: responseNode`. No modo padrão ele responde `Workflow got started` a qualquer coisa, os nós de resposta rodam sem devolver nada, e o contrato inteiro existe só no papel.

## Como o n8n fecha a issue no Multica

O Multica **não** é chamado por **HTTP**. O acesso é por nó **MCP** client, endpoint `https://mcp.multi.clipooh.cloud/mcp`, transporte `httpStreamable`, com a credencial `Multica **API** [n8n → Multica]`. As ferramentas são `multica_get_task`, `multica_add_comment` e `multica_update_task`.

O fechamento tem sete passos, nesta ordem. O primeiro é novo e vem da regra de gravação conferida:

1. **Reler o registro** no Directus e conferir contra o critério de pronto da issue. Divergência entre o que o agente afirmou e o que está gravado define o status do passo 4 e entra na nota do passo 3.
2. **Ler a subtarefa** (`multica_get_task`) para descobrir o identificador legível e o `parent_issue_id`.
3. **Postar o relatório na própria subtarefa** (`multica_add_comment`) com o texto prefixado por `/note`. O prefixo faz o comentário ficar registrado sem redisparar o agente dono da subtarefa — que já terminou e não tem o que refazer.
4. **Mudar o status da subtarefa** (`multica_update_task`) com `suppress_run: true`, pelo mesmo motivo.
5. **Ler a issue pai.**
6. **Comentar na issue pai só se necessário.** Quando a subtarefa vira `done` e todas as irmãs já estão em `done` ou `cancelled`, o próprio Multica posta um comentário de sistema na pai que aciona a Anna. Comentar junto a acionaria duas vezes. Nesse caso o resumo fica só na `/note` da subtarefa.
7. **Se qualquer passo falhar**, deixar nota privada na conversa do Chatwoot avisando que o Multica não foi notificado e que a Anna precisa ser avisada à mão.

Todos os nós dessa cadeia usam `onError: continueErrorOutput` com saída para a nota de falha, e `retryOnFail` com 3 tentativas. Uma falha de rede no Multica não pode perder o resultado de uma conversa que já aconteceu.

O status que a subtarefa recebe depende do desfecho, não do fato de ter terminado:

- objetivo cumprido **e registro conferido** → `done`;
- conversa encerrada sem o objetivo → `blocked`, com o motivo na nota;
- conversa completa mas **registro divergente** → `blocked`, com a divergência na nota. Este é o caso que faltava e que deixou a CLIPOOH-42 fechar como `done` indevidamente;
- agente pediu apoio no meio → sem mudança de status, só o comentário para quem vai apoiar.

**O que dispara o fechamento.** O agente da conversa marca `concluido = true` quando todos os itens têm resposta, e o fluxo resolve a conversa no canal. Isso emite o mesmo evento que o encerramento manual, então existe um caminho só com dois gatilhos: o agente terminando, ou a pessoa encerrando à mão como rede de segurança. Encerramento manual antes do fim fecha a subtarefa como `blocked`, com os pendentes na nota. E `concluido = true` **não** encerra a etapa por si: passa pelo passo 1.

### Retomada de uma etapa que falhou

Regra escrita em 25/09, depois do primeiro ciclo real do projeto 19.

**A subtarefa de retomada pende da issue de entrada, nunca da tentativa que falhou.** O motivo é mecânico: o aviso de conclusão do Multica acorda o *assignee do pai*. Pendurada sob a subtarefa bloqueada, a retomada acorda o agente da etapa que falhou — numa issue bloqueada — e a Anna nunca volta ao circuito. A jornada para sem ninguém perceber, com tudo parecendo saudável. Quem retoma o fluxo é a orquestradora.

**Uma chamada por subtarefa, não por etapa.** A ferramenta de abertura dispara mensagem real para uma pessoa, e por isso só pode ser chamada uma vez por subtarefa. Consequência: uma etapa que precise ser refeita exige subtarefa nova — não existe caminho de reenvio dentro da mesma. Isso é proteção contra contatar o cliente duas vezes, e o preço é histórico duplicado quando a falha foi de infraestrutura, não de conversa.

Aconteceu duas vezes em 25/09, e a segunda mostrou o custo real. A Joey, diante de uma etapa que precisava ser refeita, **recusou a retomada** citando corretamente a regra de uma chamada por subtarefa. Ela estava certa, e mesmo assim a jornada ficou sem gate: ninguém podia reabrir o canal, e a única saída foi a Anna criar subtarefa nova (CLIPOOH-48). Uma regra de segurança que só existe como disciplina do agente produz esse impasse — o agente obedece, e o fluxo para.

**Proposta: mover a garantia da disciplina do agente para a camada de entrada.** A camada 1 já conta apenas linhas `em_andamento` por escopo. Uma segunda chamada na mesma issue bate em `409 pedido_ja_aberto` por mecânica, não por boa vontade. Com isso o agente deixa de precisar recusar: ele pode tentar, e a camada decide. A proteção contra contatar a pessoa duas vezes continua de pé — e fica mais forte, porque passa a valer também quando o agente erra. Além disso, separa os dois casos que hoje estão colados: reabrir canal já aberto é 409; reenviar dentro da conversa existente é outra operação, que não abre nada e portanto não deveria esbarrar nessa regra.

**Antes de recontatar, confira se o trabalho já chegou.** A retomada deve reler a fonte — `briefings`, `brands`, o que a etapa produz — com corte na abertura da subtarefa original. Se o registro apareceu no intervalo, a retomada informa o código e **não** contata ninguém. Sem essa conferência, uma queda de ponte vira uma segunda mensagem para o cliente sobre algo que ele já fez.

### O envio do formulário encerra a etapa, não a conversa

O cliente não conhece o processo interno. Ele vê um WhatsApp com o nome ClipOOH e não tem como saber o que pode pedir, o que está fora de alçada, nem que existe uma etapa acontecendo do outro lado. Por isso:

- **O agente nunca emudece.** Confirmar o envio fecha a linha da fila, não a comunicação. Mensagem que chegue depois disso continua sendo respondida — fora do roteiro da etapa, mas respondida.
- **O agente diz que não sabe, em vez de improvisar.** Quando a pergunta sai do limite dele ou da base de conhecimento, a resposta honesta é dizer que não tem essa informação e **oferecer o transbordo** para um membro da equipe.
- **Transbordo é ação, não frase.** Só vale quando o cliente pede ou aceita, e precisa mover a conversa de fato: nota privada com o resumo e reatribuição a uma pessoa no Chatwoot. Dizer "já encaminhei" sem mover é pior que não oferecer.
- **Escalação interna é outra coisa.** Dúvida que depende de decisão da casa não vai ao cliente: vira sinal para a orquestradora, sem sair do canal.

Verificado em 25/09: está tudo implementado no fluxo da Amy, com nó executando cada parte — `transbordo_humano` e `resumo_transbordo` na saída estruturada, nota privada, reatribuição a um humano, `precisa_escalar` separado, e o ramo de conversa solta que atende mensagem sem etapa ativa. A ficha do contato acompanha toda mensagem, e quando falta declara a ausência em vez de deixar o agente presumir.

### Credencial morta não é permissão negada

Em 25/09 o token do n8n para o Directus da Amy respondeu **401 `INVALID_CREDENTIALS`**, não 403. A distinção economiza horas:

| Código | Significado | Onde olhar |
| --- | --- | --- |
| **401** | o token não é reconhecido | a credencial: expirada, rotacionada ou revogada |
| **403** | o token vale, o papel não alcança aquela collection | a policy no Directus |
| **404** | autenticado, mas o recurso não existe ou não é visível | o próprio recurso |

E um detalhe que confundiu o diagnóstico: **a Amy tem dois acessos ao Directus**, não um. O conector MCP do agente no Multica (`mcpDirectusAmy`) e a credencial do n8n (`Header Auth Amy Directus`), esta última usada tanto pelo nó de conferência do envio quanto pelo servidor MCP do Agent Amy do n8n. Um pode estar de pé enquanto o outro está morto — e foi o que aconteceu: ela lia empresa e contatos pelo primeiro enquanto o segundo devolvia 401.

**O que o incidente provou de bom:** a credencial morreu no meio da etapa e nada mentiu. A Amy não agradeceu um envio que não pôde verificar, não marcou `envio_confirmado`, escalou com o motivo correto e deixou a conversa aberta. Depois, conferindo por outro caminho, achou o registro e explicitamente não mudou status nem recontatou o cliente. A regra do recibo já estava embutida nesse fluxo antes de ser escrita aqui, e segurou uma falha de infraestrutura sem produzir dado falso.

### O buraco que continua aberto: entrega não é envio

`200 OK` do Chatwoot significa *aceito*, não *entregue*. Entre o Chatwoot e o WhatsApp há a ponte do Evolution, e **nada no desenho hoje verifica esse degrau**.

Em 25/09 isso aconteceu de verdade: a Amy chamou `abrir_briefing` uma vez, o Chatwoot aceitou, a conversa 43 foi criada, a fila registrou, a issue registrou, e a mensagem **não chegou no WhatsApp** porque a instância estava desconectada. Todos os atores relataram sucesso com razão — cada um verificou o que lhe cabia. A falha estava um degrau abaixo do que qualquer um deles enxerga.

O efeito composto é o pior: sem verificação de entrega, a falha só aparece quando a pessoa reclama; e como só há uma chamada por subtarefa, a correção exige abrir subtarefa nova e escalar até o House. Um problema de infraestrutura de segundos consumiu quatro agentes e trinta minutos.

Com verificação de entrega, a Amy teria bloqueado na hora com "aceito pelo Chatwoot, não entregue no WhatsApp", e a retomada seria reenvio na mesma conversa, dentro da mesma subtarefa, sem esbarrar em regra nenhuma. É por isso que essa verificação pertence à camada 3, que é quem fala com o canal.

**Estado vem da fila, não da memória do agente.** Em toda rodada o fluxo lê o estado dos itens e injeta no que chega ao agente, marcado como fonte oficial. Memória de conversa serve para tom e contexto; o que já foi decidido é estado determinístico. Sem isso, numa conversa longa o agente reconstrói a lista de cabeça e reapresenta item já decidido. E quando chega mensagem numa conversa sem etapa ativa, o agente recebe instrução explícita de não improvisar lista — o desenho não pode deixar essa porta aberta.

## Registro na jornada do Directus

A issue morre; a jornada fica. É nela que se lê, meses depois, o que aconteceu com aquele cliente.

**Cada ator grava o próprio passo, com o próprio `actor`.** O agente que executou uma etapa é quem registra o evento dela — não o workflow em nome dele, e não um agente em nome de outro.

O vocabulário de `actor` é o mesmo do campo `confirmed_by` dos `brand_assets`, para não existirem duas listas divergindo: `marcelo_direto`, `joey_multica`, `marcelo_joey_n8n`, `cliente_marcelo`. Para eventos que não são de confirmação, valem os nomes dos agentes: `anna`, `amy`, `max`, `n8n`.

A semântica é **quem decidiu**, não quem digitou. `marcelo_joey_n8n` significa o Marcelo decidindo através da Joey do n8n — a autoridade é dele, o agente é o meio. Registrar `joey_n8n` atribuiria a decisão ao agente.

Divisão entre agente e workflow:

- **Eventos de etapa** são do agente que executou a etapa.
- **Mudanças de estado da jornada** — bloqueio e desbloqueio — são do workflow, porque só ele sabe o instante em que a conversa abriu e encerrou. O evento correspondente vai com `actor: n8n`, e o agente aparece nomeado no `summary`.

Quando uma etapa passa a depender de um humano, a jornada é marcada `bloqueada`, com `blocked_owner` e `blocked_since`. Sem isso a jornada aparece como ativa enquanto espera decisão humana, e o painel mente sobre o estado real. No fechamento, desbloqueia.

Duas regras que evitam retrabalho:

- O resumo vai em `journey_events.summary`, **nunca** num campo único em `journeys` — um campo único é sobrescrito pela passada seguinte, e há pelo menos duas por jornada (design system e campanha).
- O registro na jornada **nunca** bloqueia a conversa. Sem `rise_project_id` ou sem jornada encontrada, o fluxo segue e registra só na issue. Os nós de jornada usam `neverError`, com uma consequência a tratar: hoje um 403 do Directus é indistinguível de "esta empresa não tem jornada", e os dois caem no mesmo ramo silencioso. A distinção deve aparecer em `avisos`.

## Quem faz o quê

| Momento | Agente do Multica | Workflow do n8n | Marcelo |
| --- | --- | --- | --- |
| Abertura | lê a issue e o contexto no Directus | — | — |
| Execução | usa as próprias ferramentas, grava o resultado no Directus | — | — |
| Registro da etapa | grava o evento na jornada com o próprio `actor` | — | — |
| Passagem de bastão | chama a ferramenta de abertura e para | recebe, valida, resolve o interlocutor, devolve o código | — |
| Conversa | não participa | conduz, grava cada decisão na hora | decide item a item |
| Estado da jornada | — | bloqueia na abertura, desbloqueia no fim | — |
| Conferência do registro | — | relê o Directus antes de fechar | — |
| Fechamento | não fecha a própria issue | `/note` com o relatório, status, avisa a pai | resolve a conversa só se quiser interromper antes do fim |

O que cada um **nunca** faz:

- O agente do Multica não envia mensagem, não conduz conversa e não fecha a própria issue depois de passar o bastão.
- O workflow não grava eventos de etapa em nome do agente, e não reaciona um agente que já terminou (daí `/note` e `suppress_run`).
- O workflow não aceita como feito o que não releu.
- Nenhum deles grava julgamento sem aprovação humana quando a decisão é do Marcelo.

## Estado atual contra este padrão

Situação em 24/09/**2026**, com os dados conferidos direto na fonte.

> **Nota sobre os dados do Directus.** Ao final de 24/09, depois desta análise, as treze collections de negócio foram zeradas para recomeçar o ciclo de teste: `companies`, `contacts`, `brands`, `brand_assets`, `assets`, `briefings`, `campaigns`, `jobs`, `job_renders`, `approvals`, `journeys`, `journey_events` e `templates`. Ficaram de pé só `forms` (o catálogo de formulários da Amy), `fonts` (o catálogo curado de onde a Joey escolhe) e os arquivos de avatar e marca do sistema — apagar qualquer um dos três travaria um agente por falta de configuração, não por bug.
>
> Portanto **os valores citados abaixo não são mais reproduzíveis no Directus**: este documento é o registro deles. Foi sorte de ordem terem sido transcritos antes; se o zeramento tivesse vindo primeiro, a razão de existir a regra do recibo teria virado lembrança em vez de evidência. Lição para a próxima limpeza: transcrever o que prova uma regra antes de apagar o que a provou.

**Amy — conforme no que o padrão já cobre.** É de onde o padrão veio. Contrato completo, quatro códigos, fechamento com os seis passos originais, tratamento de falha do Multica. Não grava eventos na jornada: a etapa dela é anterior ao modelo de jornada e isso ainda não foi retroalimentado. Não tem a conferência do passo 1, porque a regra é de hoje — mas o risco nela é menor: o acesso dela ao Directus é **somente leitura** (`items`, `system-prompt`), então ela não tem o que afirmar ter gravado. E ela já pratica o princípio: é proibida de agradecer o envio sem conferir em `briefings`.

**Joey — não conforme no momento 2.** É a correção mais importante desta versão do documento; a versão anterior a classificava como "conforme, e testada".

O que rodou em 23/09 na CLIPOOH-42 é verdade: 7 de 7 itens decididos na conversa, relatório como `/note` na subtarefa, issue em `done`, Anna avisada, evento de gate registrado com `marcelo_joey_n8n`, auto-resolve da conversa. Todos os passos verificáveis pelo fluxo passaram.

E o resultado da etapa não existe. Conferido no Directus em 24/09:

- `brands` BRD-000007 segue `kit_status = extracted` e `confirmed_by = sem_confirmacao`;
- `logo_safe_area` e `logo_min_width_px` continuam nulos, apesar de 0,5× da altura e 220 px terem sido aprovados na conversa;
- `colors` e `fonts` nulos, apesar de a paleta e a fonte Archivo terem sido aprovadas;
- `usage_rights` nulo nos **cinco** `brand_assets`, inclusive no `logo_principal`;
- `brands.notes` não tem **nenhum** bloco `[JOEY-N8N ...]`, que a skill `joey-registro-directus` exige para cada item aprovado. Só existem os dois blocos `[JOEY ...]` da Joey do Multica, de 13:57 e 20:52.

Zero gravações para sete itens aprovados. E o relatório afirmava ter gravado. O agente tinha a capacidade: o MCP do Directus dele libera `schema`, `items`, `files` e `assets`. Na mesma execução o fluxo gravou com sucesso em `journey_events` — não é credencial, não é rede, não é permissão global.

O relatório também afirmava *"não há brand_assets cadastrados"* e *"não há logo principal cadastrado, transparente, utilizável e confirmado"*. Os dois são falsos: há cinco assets, e o `logo_principal` está `approved`, `usable = true`, `has_transparency = true`, confirmado por `joey_multica` desde 20:52. Essa afirmação falsa foi o que levou a Anna a bloquear pelo motivo errado, ainda que a decisão de bloquear estivesse certa.

Consequência em aberto: a **JRN-000001 está `bloqueada`** na fase `gate_material` desde 23/09 22:32, com `blocked_owner = house`, e não existe issue do Pacey. A bola está com a casa, não com a Anna — ela escalou e parou, e só desbloqueia depois que o fechamento do gate for corrigido e reprocessado.

Duas circunstâncias que contribuíram e continuam pendentes:

- A credencial da Joey no Directus **não tem leitura de `directus_relations`**, e a própria Joey do Multica registrou isso: *"schema do Directus indisponível por falta de permissão de leitura em directus_relations"*. A skill manda conferir o schema antes de gravar e nunca adivinhar nome de campo. Schema cego mais essa ordem é um caminho plausível para não gravar. Falta também `create` em `assets`, que o prompt promete e a credencial não permite.
- Os prompts e skills atualizados existem no repositório mas **não foram aplicados no n8n**. Enquanto isso o agente roda a versão anterior — a divergência que este documento proíbe, e que torna incerto se o prompt lido no n8n é o pretendido.

**Max — não construído.** Nenhum agente `Max` existe no n8n; só Amy e Joey. Entra neste padrão desde o início, como segundo agente do perfil `cliente`, sem etapa de adaptação. Terá agente de IA próprio no n8n, como os outros dois. A inbox 10 ("MAX Aprovação"), o AgentBot Max e a instância `max-clipooh-0f6121` já existem.

**Camada 2 — construída e religada, não ligada.** `Agentes MULTICA F2` (`x5LFoVcCM4d8TjEc`), 40 nós. A abertura rodou fim a fim em 24/09: contexto validado pelo contrato, agente escolhido por expressão, saída estruturada, estado gravado na fila e quatro bolhas entregues na conversa, com a pausa entre elas. Os três AgentBots foram provados com `sender type agent_bot`.

A conferência do recibo está construída, com a lógica provada nos seis cenários (recibo presente, recibo ausente, erro de leitura, `brand_id` ausente, `message_id` ausente, nada decidido) e a fiação provada em execução real. **Falta a prova do caminho de reversão com o agente de verdade**, porque em quatro rodadas o agente nunca declarou um item decidido.

Dois defeitos herdados da extração foram corrigidos: a leitura implícita de nós irmãos, descrita acima, e o nó de followup no Postgres, que era o último a referenciar o `Settings v2` do pai. E um defeito da camada 1 apareceu ao ler o dado gravado: a lista de `itens` era montada e não entrava no retorno do validador, então o `pedido` chegava à fila sem itens e o agente receberia lista vazia — sem erro nenhum.

**O motor da Joey bloqueia o teste.** Quatro rodadas, quatro resultados degradados: saudação errada pela hora local de São Paulo (regra com tabela explícita na skill), id interno da mensagem vazado na conversa com o cliente (o prompt proíbe citar mecânica interna), duas rodadas com aprovação explícita sem marcar item nenhum — respondendo *"vou registrar agora"*, que é o padrão que a skill proíbe — e uma falha de saída estruturada (`"Couldn't get structured output matching the schema"`). Nada disso é falha do fluxo, e a conferência não pega nenhuma delas: ela pega a gravação declarada e não feita, que é outra classe. Voltar ao motor canônico é pré-requisito para qualquer teste com agente, não melhoria.

**Camada 4 — construída e corrigida, não ligada.** `Fechamento MULTICA F4` (`1nYLjBiANZom8dEn`), 35 nós. Os sete defeitos da análise de 24/09 foram corrigidos: o contrato pelo trigger, a fila nova com a leitura do `pedido`, a releitura do critério de pronto, o desbloqueio condicional, o evento com `type` e `actor` do agente, o evento de `desbloqueio` que não existia, e a credencial da camada no lugar da do agente.

A lógica de desfecho foi provada em nove cenários, rodando o código dos nós com entradas simuladas: o caso da BRD-000007 (itens todos decididos, marca em `extracted`/`sem_confirmacao`) devolve **`blocked`** com o motivo escrito — *"kit_status esta 'extracted' e o critério exige 'confirmed'; confirmed_by esta 'sem_confirmacao'"*. Antes devolvia `done`. Também provados: marca confirmada fecha `done`; item pendente, divergência da camada 2, erro de leitura do Directus, `brand_id` ausente, `pedido` ilegível e agente sem critério definido todos bloqueiam; e a Amy, que não grava conteúdo próprio, fecha `done` pelo estado dos itens. Falta execução real, que depende de um disparador de teste.

**`evolution - enviar audio` nunca funcionou.** Está com `sendBody: false` também na Amy e na Joey de produção — nunca enviou corpo, em nenhum workflow. Não é resíduo de versão anterior: nasceu morto. O áudio que chega ao WhatsApp chega pelo Chatwoot. Configurá-lo exige o contrato do `/send/media` do gateway.

**Camada 1 — construída e testada, não ligada.** `Entrada MULTICA` (`do644VHyIhBGn3Mg`), 43 nós. O workflow antigo da Joey continua atendendo a produção em `joey-abrir-validacao`; a camada nova só responde em `multica-abrir-conversa`, que nenhuma tool chama ainda.

A bateria de 24/09 cobriu, com execução real e leitura nó a nó:

- as três recusas — `agente_desconhecido`, `contrato_invalido` com o campo faltando, e passe divergente pegando inbox **e** instância;
- o perfil `casa` de ponta a ponta: contato fixo, linha na fila, conversa criada na inbox 8, `conversation_id` gravado, jornada consultada, 200;
- os dois motivos de **409** — `pedido_ja_aberto` na mesma issue e `ocupado` numa issue diferente, provando o escopo global da Joey;
- a **varredura**: linha vencida marcada `em_bloqueio`, conversa encerrada no canal (`current_status: resolved`), aviso postado na issue do Multica, e o pedido novo seguindo — tudo numa passada;
- o perfil `cliente` nas duas inboxes: contato resolvido no Directus, contato localizado no Chatwoot, vínculo criado quando faltava e **reusado** quando existia, conversa aberta;
- a proteção do interlocutor: um pedido com `contact_whatsapp` e `contact_name` deliberadamente errados gerou dois avisos e usou o do Directus. Número errado no pedido não chega ao canal.

Quatro defeitos foram encontrados e corrigidos nessa fase, todos que só apareceriam em produção: referência órfã a um nó renomeado (a jornada nunca seria encontrada); `_contains` em campo `json`; o `conversation_id` que não viajava para a varredura (ela liberaria a fila deixando a conversa aberta — o incidente de 23/09 de volta); e o vínculo de inbox não idempotente.

O que o teste **não** cobriu: o ramo de jornada encontrada — todas as corridas foram sem `rise_project_id` de propósito, para não sobrescrever o bloqueio que a Anna deixou na JRN-000001, que é evidência. Falta exercitar `Directus - Bloquear jornada` e `Directus - Evento bloqueio na abertura` contra uma jornada de teste própria.

## Decisões em aberto

1. **TTL da fila.** 120 minutos para os três, em operação. Número ainda não validado contra a duração real de uma conversa — uma aprovação que passe de duas horas seria varrida no meio. Durante teste vale baixar para 5 e **restaurar depois**.
2. **Credencial do webhook da entrada.** Ainda é `Header Auth Joey Envio [CLIPOOH]`, com header `X-Joey-Secret`, numa camada que serve os três. Precisa de nome e segredo próprios da camada. O lado do Directus já foi resolvido com a `Header Auth Directus - Multica Access [CLIPOOH]`.
3. **O Max conduz a conversa** depois de enviar o link de aprovação, ou só dispara? Ele terá agente de IA próprio no n8n, como os outros dois, o que indica que conduz. A inbox 10 ("MAX Aprovação") e o AgentBot Max já existem no Chatwoot.
4. **Permissões da Joey no Directus**: `directus_relations` (leitura) e `assets` (create).
5. **Aplicar no n8n** os prompts e skills que já estão no repositório, e voltar o motor da Joey ao canônico. Este é o bloqueio de maior prioridade: sem ele nenhum teste com agente fecha.
6. **Reprocessar a BRD-000007** de ponta a ponta depois da correção, em vez de remendar campo à mão. A CLIPOOH-40 é o RISE projeto 18, "TESTE DE FLUXO 4", com CNPJ da própria Clipooh — é fluxo de teste, não entrega de cliente, e não há ninguém esperando.
7. **Apagar as duas cópias divergentes** deste documento.
8. **Limpar os vínculos duplicados da inbox 9** no Chatwoot. O contato 4 acumulou cinco `contact_inbox` para a mesma inbox, cada um com `source_id` diferente, antes de a entrada passar a reusar em vez de criar. O código pega o primeiro, que é o que a Amy de produção já usa — mas se um dia o primeiro não for o que a instância do Evolution conhece, a Amy fica muda sem erro.
9. **Testar o ramo de jornada encontrada** da camada 1 contra uma jornada de teste própria, sem tocar a JRN-000001.
10. **Sobras da bateria de 24/09**: as linhas 2 a 6 da `fila_conversas` ficaram `em_andamento` e a conversa 42 ficou aberta na inbox 8 com mensagens de teste. Não travam nada em produção, porque a tabela é nova; a varredura encerra cada uma na próxima chamada daquele agente.
11. **Contrato do `/send/media`** do gateway Evolution, para configurar o envio de áudio. Quando existir, os dois nós de áudio entram no mesmo `Switch` por agente dos outros dois pontos.
12. **Arquivar o webhook temporário** `temp-camada3-teste` da camada 2 quando a camada 3 passar a chamá-la.
13. **Camada 3**, não construída. É a maior das quatro e a que menos revela problema novo: recepção, deduplicação por Redis, transcrição de áudio, debounce, e o caminho de resposta em bolhas.
14. **Critério de pronto do Max.** Definido junto com a construção do agente dele, porque é a etapa mais difícil das três: quem dá o OK final é **o cliente**, no WhatsApp. Enquanto não existir, a camada 4 se recusa a fechar uma etapa dele como `done` — o fail-closed é o lugar-guardado.

    A collection `approvals` já existe e serve a esse propósito: tem `job` e `job_render`, `type` (`internal`/`client`), `decision` (`approved`/`rejected`/`changes_requested`), `channel` (`multica`/`whatsapp`/`dashboard`), `round` para o limite comercial de ajustes, `approved_by` ("agente, ou nome e telefone de quem respondeu") e `decided_at`. `jobs.status` já tem `client_review` e `approved`. O critério deve ser: existe `approvals` com `type = client`, `channel = whatsapp`, `decision = approved` e `approved_by` preenchido, para o `job_render` que foi enviado.

    Três consequências, que precisam ser resolvidas antes de o Max rodar:

    - **O alvo é o `job_render`, não o job.** A nota do campo é explícita: *"A aprovação vale para este arquivo. Render novo não herda aprovação."* Logo o contrato da camada 1 para o Max precisa levar `job_id` **e** `job_render_id` — é o equivalente do `brand_id` da Joey.
    - **O desfecho do Max é ternário e a camada 4 só sabe dois.** `changes_requested` não é "bloqueado esperando a casa": é volta para produção na rodada N+1, com o `round` incrementado e o agente de produção reacionado. É um terceiro caminho de fechamento, e não existe ainda.
    - **Falta um valor no vocabulário de `actor`.** `cliente_marcelo` significa "o cliente validou, repassado por Marcelo". No Max o cliente decide direto, sem intermediário, e registrar como `cliente_marcelo` atribuiria a decisão a quem não a tomou — contra a regra dura "quem decidiu, não quem digitou". Seguindo o padrão de `marcelo_joey_n8n`, o valor consistente é **`cliente_max_n8n`**. O `approved_by` da `approvals` resolve a identidade de quem respondeu; a lacuna é só no `actor` do evento de jornada.
15. **Verificação de entrega no canal.** Hoje nada confere se a mensagem aceita pelo Chatwoot chegou ao WhatsApp. É o buraco descrito acima, e o lugar dele é a camada 3. Enquanto não existir, toda queda do Evolution vira subtarefa de retomada e escalada até o House.
16. **Instabilidade do Evolution.** A instância caiu durante o ciclo do projeto 19 e vinha derrubando com frequência. É problema de infraestrutura, distinto do buraco de verificação — mas os dois juntos é que produziram o incidente: a ponte caiu **e** caiu em silêncio.
17. **Ajuste aplicado no House em 25/09:** a regra de que subtarefa de retomada pende da issue de entrada foi acrescentada às instruções dele. A `CLIPOOH-46`, criada antes do ajuste, ficou pendurada na `CLIPOOH-45` — decidir se vale reparentar para a `CLIPOOH-44` antes de promovê-la.
18. **Testar as camadas 2 e 4 de ponta a ponta.** As duas têm trigger de subworkflow, que o executor do MCP não dispara; a 3 tem um webhook temporário, a 4 não tem nenhum. Enquanto a camada 3 não existir, cada uma precisa de um disparador de teste próprio.
