# ClipOOH — Prompts e Skills dos Agentes

Versionamento dos prompts e skills dos agentes de produto Amy, Joey e Max.

Cada agente mantém duas versões independentes:

- `multica/`: configuração usada no Multica;
- `n8n/`: configuração usada no n8n.

Em cada versão:

- `prompt.md`: prompt do agente;
- `skills/`: skills específicas daquela versão.

## Quem é a fonte da verdade

**A plataforma é quem roda; este repositório é o histórico revisável.**

O que está no Multica e no n8n é o que de fato executa. Este repositório existe para que toda mudança tenha diff, autor e data — não para substituir a plataforma.

Na prática:

- mudou o prompt ou a skill na plataforma, **comite aqui na mesma sessão**;
- ao editar um arquivo daqui, aplique na plataforma antes de fechar a tarefa;
- divergência entre os dois é bug, não versão alternativa.

Prompt que descreve um caminho que não existe mais é a falha mais cara deste repositório: o agente segue a instrução e o fluxo não responde.

## Estado

| Agente | Multica | n8n |
|---|---|---|
| Amy | prompt + 1 skill | prompt |
| Joey | prompt + 1 skill | prompt + 2 skills |
| Anna | a preencher | — |
| Dawson | a preencher | — |
| House | a preencher | — |
| Max | a preencher | a preencher |
| Pacey | a preencher | — |

`amy/n8n/skills/` está vazio — as skills do agente da Amy no n8n ainda não foram versionadas.
