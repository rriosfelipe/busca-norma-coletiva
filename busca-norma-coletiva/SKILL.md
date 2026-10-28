---
name: busca-norma-coletiva
description: Busca convenções coletivas, acordos coletivos e termos aditivos registrados no Sistema Mediador do MTE (mediador.trabalho.gov.br, "Consultar Instrumentos Coletivos Registrados") e extrai as cláusulas pedidas (piso salarial, reajuste, jornada, banco de horas, adicionais, estabilidades, contribuições etc.). Use sempre que o usuário pedir para achar/conferir/baixar a CCT, ACT, norma coletiva ou instrumento coletivo aplicável a uma categoria, sindicato, empresa (CNPJ), UF ou período, perguntar o piso ou a data-base de uma categoria, ou precisar da cláusula de uma norma coletiva para contestação, cálculo ou consultoria — mesmo sem citar o Mediador.
---

# Busca de norma coletiva no Mediador (MTE)

O Mediador é a base oficial dos instrumentos coletivos registrados no MTE. O site fica atrás do Cloudflare e usa reCAPTCHA v3 invisível, então **requisições diretas (curl, WebFetch, scripts) recebem 403**. Use um navegador de verdade: o navegador integrado (`mcp__Claude_Browser__*`) por padrão, ou o Claude in Chrome se o usuário pedir. O reCAPTCHA roda sozinho quando o botão "Pesquisar" do próprio site é clicado; você só preenche o formulário e clica no botão.

Se aparecer um desafio visível (caixa "Não sou um robô", seleção de imagens, tela "Verify you are human" que não some sozinha), **não tente resolvê-lo**: peça ao usuário que resolva no painel do navegador e continue depois.

Arquivos de apoio:

- `references/snippets.md` — o código para `javascript_tool` (S1 a S5). Copie de lá; não reescreva.
- `references/campos.md` — IDs dos campos, valores dos selects, grupos de cláusulas, endpoints.

## 1. Entender o pedido

Extraia do pedido o que for possível: empresa ou sindicato (razão social), CNPJ, categoria, tipo (convenção, acordo, aditivo), UF de registro, período e as cláusulas de interesse. Não pergunte o que dá para inferir; pergunte só se não houver nenhum filtro de participante nem categoria (a busca ficaria ampla demais).

**Filtros de participante.** Razão social, CNPJ e CEI são mutuamente exclusivos (o site desmarca os outros), e razão social também desmarca categoria. Razão social: trecho do nome, **sem acento, em maiúsculas** (é como os nomes estão cadastrados); se não vier nada, tente um trecho menor ("COMERCIARIOS DE SAO PAULO"). Categoria: o contrário, **precisa de acento** ("Açúcar" acha centenas de instrumentos; "acucar", nenhum).

**Vigência.** "Vigentes" só mostra o que está em vigor hoje, e isso esconde a CCT que acabou de vencer e cuja renovação ainda não foi registrada (comum nos meses após a data-base). Se a busca de vigentes não trouxer a norma geral da categoria, refaça com "Todos" e o período de um ano atrás até hoje. Para fatos passados, use as datas do contrato. Regras do site:

- o período aceita **no máximo 1 ano**;
- "Todos" **sem período** varre o histórico inteiro e dá 504 (timeout).

A ultratividade é vedada (art. 614, §3º, CLT): cada norma só vale para fatos dentro da própria vigência.

**Histórico de vários anos** (ex.: "últimos 5 anos"): uma busca por janela de 1 ano. Junte as listas e elimine repetidos pelo nº de registro (um instrumento aparece em todas as janelas que cruza). Organize por unidade/sindicato e data-base e procure buracos na sequência anual. Se faltar um ano, repita aquele período com um trecho menor do nome (ex.: "CERRADINHO" em vez de "CERRADINHO BIOENERGIA"), porque o acordo pode estar em nome de outra empresa do grupo. Se continuar faltando, diga que não foi encontrado.

**Várias convenções por categoria.** Uma mesma categoria profissional costuma ter várias CCTs, uma por sindicato patronal (ex.: os comerciários de SP têm CCTs distintas com Sindilojas, Fecomercio/atacadistas, Sincovaga, Sincamesp etc.). A norma aplicável depende do enquadramento pela atividade preponderante da empresa. Se a atividade não estiver clara, apresente as principais com a abrangência de cada uma e diga isso, em vez de escolher uma.

### Normas aplicáveis a uma empresa (ACT + CCT)

Buscar pelo nome ou CNPJ da empresa só traz os instrumentos que ela **assinou** (acordos coletivos e, raramente, convenções). A convenção da categoria, assinada pelos sindicatos sem a empresa, não aparece. Quando o pedido for "as normas da empresa X" ou a norma aplicável a um empregado dela, levante as duas:

1. Acordos da empresa: busca por CNPJ ou razão social.
2. Na lista e nos extratos dos acordos, anote o **sindicato profissional** (com CNPJ) e a **base territorial** (cláusula de abrangência) de cada unidade.
3. Convenções da categoria: nova busca pelo **CNPJ do sindicato profissional**, com tipo `convencao` e o mesmo período. O mesmo sindicato também aparece em acordos de outras empresas, por isso o tipo `convencao` é necessário.
4. Se não vier convenção, tente pela categoria (com acento, ex.: "Açúcar", "Alimentação"), tipo `convencao` e UF de registro. A CCT pode ter sido assinada pela federação, e não pelo sindicato local. Confira na abrangência se o município da unidade está incluído.
5. Se a empresa não tiver acordo, pergunte ou deduza a atividade e o município, e vá direto ao item 4.

Na resposta, diga qual instrumento prevalece. Pelo art. 620 da CLT (redação da Lei 13.467/2017), as condições do acordo coletivo sempre prevalecem sobre as da convenção. Se a CCT trouxer cláusulas sobre temas que o ACT não regula, mostre-as e avise que a aplicação delas é discutível. Se nenhuma convenção for encontrada, diga isso; em alguns setores a negociação é só por acordo.

## 2. Pesquisar

Abra `https://mediador.trabalho.gov.br/sistemas/mediador/ConsultarInstColetivo` (no navegador integrado: `preview_start` com `url`; para buscas seguintes, `navigate` para a mesma URL, que também limpa os filtros).

1. **Na primeira busca da conversa, rode S1** (checagem do formulário). Os snippets dependem dos IDs da página; se o MTE mudar o site, eles falham sem erro e a busca volta vazia, o que pareceria "nenhuma norma". Se S1 acusar algo, não confie nos snippets: mapeie a página de novo (liste `input, select, button` com id, name e o texto da linha), adapte os IDs e avise o usuário que o site mudou e a skill precisa ser atualizada.
2. **S2** preenche e clica; **S3**, em outra chamada, espera e lê a lista. Uma chamada só pode passar do limite de ~45 s da ferramenta. Se S3 voltar "ainda processando", rode-o de novo.
3. Leia o retorno de S3:
   - "Resultado: N Instrumento(s) ... Página X de Y" → a `lista` traz registro, solicitação, tipo, vigência, partes e link (10 por página);
   - alerta "Nenhum registro encontrado." → resultado válido, zero instrumentos;
   - outro alerta (ex.: "O intervalo máximo de pesquisa é de um ano.") → a busca não foi feita; corrija o filtro, recarregue e repita.

**Próxima página:** clique no link "Próximo >>" de `#divConsultaDetalhada` (não pede novo reCAPTCHA) e rode S3 de novo. Com mais de ~5 páginas, não percorra tudo: mostre o total e sugira refinar (UF, tipo, razão social, período).

**Filtro de cláusula do site** (Grupo/SubGrupo/Palavra do Título): usa outro endpoint e devolve direto o texto das cláusulas, sem a `lista`. Só use com filtro estreito (CNPJ); com razão social ou categoria costuma dar 504 e a tela fica em "Processando". Na dúvida, busque sem cláusula e filtre no passo 3.

## 3. Ler os instrumentos

Cada resultado tem um link para o extrato completo em HTML, ex.:
`https://mediador.trabalho.gov.br/sistemas/mediador/Resumo/ResumoVisualizar?NrSolicitacao=MR071249%2F2024`

Prefira esse link a "Download" (o download gera um arquivo e exige autorização do usuário). Os dois snippets de leitura rodam da aba do Mediador, sem abrir o extrato:

- **S5, levantamento em lote**: data de registro, partes, abrangência, aditivos e títulos das cláusulas de vários instrumentos. Os títulos mostram o assunto de cada um e separam acordos paralelos do mesmo período (ex.: um geral de salários e benefícios e outro só de jornada/turnos).
- **S4, transcrever cláusulas** de um instrumento.

**Escolher pela abrangência.** A mesma empresa pode ter, na mesma unidade, acordos com categorias diferentes (ex.: um com a federação dos trabalhadores rurais e outro com o sindicato da indústria). Use a categoria da cláusula de abrangência para ficar só com os que cobrem o empregado do caso, e liste os outros como "não se aplica", com o motivo. Se a função do empregado não estiver clara (ex.: industrial ou agrícola), diga quais normas valem em cada hipótese.

S4 separa por cláusula e filtra **primeiro pelo título**; o filtro pelo corpo é o plano B, porque termos comuns ("piso", "data-base") aparecem em muitas cláusulas e trariam dezenas de milhares de caracteres. A lista `titulos` sempre volta: se `achadas` vier vazia ou errada, escolha pelos títulos e rode de novo com o termo exato. S4 também corta o título de grupo que o extrato emenda no fim de cada cláusula e devolve o que cortou em `cortadas`; confira antes de transcrever. O mesmo assunto tem nomes diferentes de uma norma para outra; use sinônimos:

- piso: "piso", "salário normativo", "salário de admissão", "salário mínimo profissional"
- horas extras: "horas extras", "hora extra", "jornada extraordinária", "sobrejornada"
- banco de horas: "banco de horas", "compensação"
- insalubridade: "insalubr"

Se o usuário pedir o texto integral, use `get_page_text`.

**Termos aditivos.** Aditivos alteram cláusulas da norma principal. Sempre que for transcrever uma cláusula ou dizer qual é o texto vigente, confira o campo `aditivos` de S5 para aquele instrumento. Para cada registro: procure-o na lista de resultados que você já tem (aditivos aparecem nas mesmas buscas por partes e período); se não estiver, busque pelo CNPJ de uma das partes com tipo `termoAditivoAcordo` ou `termoAditivoConvecao` e o período da norma principal. Abra o aditivo e veja se ele altera a cláusula pedida. "sem registro" indica um aditivo protocolado mas ainda não registrado; informe isso.

## 4. Responder

Para cada instrumento usado, informe:

- tipo e período (ex.: Convenção Coletiva 2024/2026), vigência e data-base;
- partes (sindicatos/empresa);
- nº de registro no MTE, data de registro e nº da solicitação;
- abrangência (categoria e base territorial, da cláusula de abrangência);
- link `ResumoVisualizar` para conferência;
- termos aditivos vinculados (registro e se alteram as cláusulas transcritas).

**Registro tardio.** Compare a data de registro com o início da vigência. Quando o registro sai meses depois (comum: vigência desde março, registro em outubro), transcreva a cláusula de efeitos/vigência do instrumento, que costuma prever retroação, e aponte as datas de protocolo e de registro. Pelo art. 614, §1º, da CLT, o instrumento entra em vigor 3 dias após a entrega no órgão; se isso importa para o caso (fatos entre o início da vigência e o registro), sinalize ao usuário em vez de concluir sozinho.

Para levantamentos de vários anos, use tabelas por unidade/sindicato (período, registro, solicitação com link, vigência, data de registro), marcando o vigente e os anos sem instrumento encontrado.

Depois, as cláusulas pedidas, transcritas na íntegra com o número e o título (ex.: "CLÁUSULA TERCEIRA - PISOS SALARIAIS..."), porque peças e cálculos precisam do texto exato. Não resuma valores, percentuais ou condições; transcreva.

Diga com clareza o que não foi verificado: o Mediador só traz instrumentos registrados no sistema; ausência de resultado não prova que não exista norma (pode ter outro nome de sindicato, outra UF de registro, ou não ter sido registrada). Confira a abrangência antes de afirmar que a norma se aplica a um empregado.
