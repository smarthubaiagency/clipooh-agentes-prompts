--- name: joey-cor-e-escala ## description: Fechar cor com o Marcelo e gerar a escala da marca. Como pedir papel e hex, quando chamar a ação scale, o que ela devolve, quais campos gravar em brands.colors e brands.color_scales sem gate, e como ajustar um fundo sem inventar hex. Use sempre que houver item de cor na lista. # Cor e escala da marca

Cor é o único item da lista em que **você não propõe nada**. O Marcelo define papel e hex; o resto é aritmética e você grava.

## O que é decisão dele e o que é medição sua

| Decisão do Marcelo (gate) | Medição sua (grava direto, sem perguntar) |
|---|---|
| qual cor é primária, secundária, terciária e acentuada | a escala 100–900 de cada cor |
| o hex de cada uma | o fundo claro e o fundo escuro |
| | a cor de texto de cada fundo |
| | o contraste medido de cada par |

Isso vale porque o fundo não é escolha estética: ele é o degrau da escala, e o degrau fixa a luminosidade. O contraste sai igual para qualquer matiz. Perguntar seria pedir aprovação de uma conta.

## A conversa

A paleta tem quatro papéis obrigatórios. Cada um é um item, e cada item precisa do *sim* dele.

Abra mostrando o que o cliente declarou em `brands.colors_by_owner` e a cor proibida do envio. Peça papel e hex:

``` O cliente declarou *azul e cinza* e disse que vermelho não pode.

Me passa o papel e o hex de cada uma? São quatro: primária, secundária, terciária e acentuada. ```

Quando ele informar, repita em uma linha com a cor em palavras — *#**1E5BC6**, azul médio, como primária* — e peça o *sim* daquele item.

**Nunca proponha hex.** Nem para a cor da marca, nem para fundo, nem para texto. Se ele pedir sugestão, diga que cor é decisão dele e que você gera o resto a partir do que ele der.

## Depois que os quatro hexes estiverem aprovados

Uma chamada só, com as quatro cores juntas:

```
action: *scale*
brand_colors: [
    { role: *primary*,   hex: *#...* },
    { role: *secondary*, hex: *#...* },
    { role: *tertiary*,  hex: *#...* },
    { role: *accent*,    hex: *#...* }
]
threshold: 7
```

A resposta traz, além da escala completa, um bloco `directus` com duas listas prontas — `colors` e `color_scales` — nos mesmos nomes dos campos. Grave como veio, sem transformar nada no meio.

Em `brands.colors`, por cor: `bg_light_hex`, `text_on_light_hex`, `contrast_light`, `bg_dark_hex`, `text_on_dark_hex`, `contrast_dark`. O `hex`, o `confirmed_by` e o `confirmed_at` são os que ele aprovou — não sobrescreva.

Em `brands.color_scales`, a escala inteira de cada cor com o degrau-âncora.

Confirme para ele em uma linha, sem despejar hex:

``` Fechado. Gerei a escala das quatro e os fundos claro e escuro de cada uma, todos acima de 7:1. Está gravado. ```

## O padrão de fundo

Fundo claro é o **degrau **100**** com texto `#**000000**`. Fundo escuro é o **degrau **800**** com texto `#**FFFFFF**`. É padrão da fábrica, não escolha por marca.

A cor da marca continua intacta em `hex` — ela vive no logo e nos detalhes. Os fundos existem só para ter texto legível por cima.

Se `todos_os_fundos_passam` vier `false`, não improvise: diga quais cores reprovaram, registre no relatório e deixe essas cores pendentes.

## Ajuste manual

Se algum fundo não servir visualmente, a correção **nunca** é um hex novo: é outro degrau da escala já gravada em `brands.color_scales`. Mostre as opções vizinhas com o contraste de cada uma e peça o *sim* daquele item. Degrau abaixo de 7:1 não entra, mesmo que ele peça — nesse caso explique e ofereça o degrau mais próximo que passa.

## Reconfirmação de cor

Se ele mudar o hex de uma cor já confirmada, isso é reconfirmação: mensagem própria, um item só, com o valor aprovado antes, o proposto e a pergunta *tem certeza que quer atualizar?*. Com o *sim*, rode a `scale` **só daquela cor** — a escala de cada cor deriva apenas dela mesma — e regrave a linha dela em `colors` e em `color_scales`. As outras três não mudam.

## O que nunca fazer

- propor ou inventar hex de marca, de fundo ou de texto;
- calcular contraste de cabeça — o número vem sempre da ferramenta;
- alterar o `hex` da marca para ganhar contraste;
- gravar fundo que não passa em 7:1;
- perguntar aprovação de fundo, texto ou escala: é medição;
- chamar a `scale` antes de os quatro hexes estarem aprovados.
