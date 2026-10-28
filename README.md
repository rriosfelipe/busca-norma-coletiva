# busca-norma-coletiva

Skill do Claude para buscar convenções coletivas, acordos coletivos e termos aditivos registrados no **Sistema Mediador do MTE** ([Consultar Instrumentos Coletivos Registrados](https://mediador.trabalho.gov.br/sistemas/mediador/ConsultarInstColetivo)) e extrair as cláusulas pedidas (piso, reajuste, jornada, banco de horas, adicionais etc.).

## Como funciona

O Mediador fica atrás do Cloudflare e usa reCAPTCHA v3 invisível, então a skill trabalha num navegador de verdade (o navegador integrado do Claude ou o Claude in Chrome): preenche o formulário, clica em "Pesquisar" e lê os resultados e os extratos. Se aparecer um desafio visível, ela para e pede que o usuário resolva.

Cobre:

- busca por razão social, CNPJ, categoria, tipo, UF de registro e período de vigência;
- históricos de vários anos (uma busca por ano, já que o site limita o período a 1 ano);
- normas aplicáveis a uma empresa: os acordos que ela assinou e as convenções da categoria (pelo sindicato profissional), com a regra de prevalência do art. 620 da CLT;
- termos aditivos vinculados a cada instrumento;
- checagem inicial do formulário, para avisar quando o site do MTE mudar;
- leitura dos extratos e filtro das cláusulas por título;
- resposta com registro, solicitação, vigência, partes, abrangência, link e texto integral das cláusulas.

## Estrutura

- `SKILL.md`: o fluxo e as regras (o que buscar, como escolher a norma, como responder).
- `references/snippets.md`: o código que roda no navegador (checagem, busca, leitura em lote, extração de cláusulas).
- `references/campos.md`: IDs do formulário, valores dos selects e endpoints.

## Instalação

**Claude Code:** copie a pasta `busca-norma-coletiva/` para `~/.claude/skills/` (no Windows, `C:\Users\<usuário>\.claude\skills\`).

**claude.ai:** compacte a pasta `busca-norma-coletiva/` num `.zip` e envie em Configurações → Capacidades → Skills.

## Uso

Peça normalmente ("qual o piso da CCT dos comerciários de SP?", "levante os acordos da empresa X dos últimos 5 anos") ou chame `/busca-norma-coletiva`.

## Limites

O Mediador só traz o que foi registrado no sistema; ausência de resultado não prova que não exista norma. A busca por cláusula do próprio site costuma dar timeout com filtros amplos. Os IDs do formulário foram levantados em outubro de 2026 e podem mudar.
