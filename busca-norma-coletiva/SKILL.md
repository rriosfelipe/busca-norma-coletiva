---
name: busca-norma-coletiva
description: Busca convenções coletivas, acordos coletivos e termos aditivos registrados no Sistema Mediador do MTE (mediador.trabalho.gov.br, "Consultar Instrumentos Coletivos Registrados") e extrai as cláusulas pedidas (piso salarial, reajuste, jornada, banco de horas, adicionais, estabilidades, contribuições etc.). Use sempre que o usuário pedir para achar/conferir/baixar a CCT, ACT, norma coletiva ou instrumento coletivo aplicável a uma categoria, sindicato, empresa (CNPJ), UF ou período, perguntar o piso ou a data-base de uma categoria, ou precisar da cláusula de uma norma coletiva para contestação, cálculo ou consultoria — mesmo sem citar o Mediador.
---

# Busca de norma coletiva no Mediador (MTE)

O Mediador é a base oficial dos instrumentos coletivos registrados no MTE. O site fica atrás do Cloudflare e usa reCAPTCHA v3 invisível, então **requisições diretas (curl, WebFetch, scripts) recebem 403**. Use um navegador de verdade: o navegador integrado (`mcp__Claude_Browser__*`) por padrão, ou o Claude in Chrome se o usuário pedir. O reCAPTCHA roda sozinho quando o botão "Pesquisar" do próprio site é clicado; você só preenche o formulário e clica no botão.

Se aparecer um desafio visível (caixa "Não sou um robô", seleção de imagens, tela "Verify you are human" que não some sozinha), **não tente resolvê-lo**: peça ao usuário que resolva no painel do navegador e continue depois.

Detalhes de campos, valores de selects e códigos de grupos de cláusulas: `references/campos.md`.

## Fluxo

### 1. Entender o pedido

Extraia do pedido o que for possível: sindicato/empresa (razão social), CNPJ, categoria, tipo (convenção, acordo, aditivo), UF de registro, vigência e as cláusulas de interesse.

Vigência: "vigentes" só mostra o que está em vigor hoje. Isso esconde a CCT que acabou de vencer e cuja renovação ainda não foi registrada (comum nos meses após a data-base). Se a busca de vigentes não trouxer a norma geral da categoria, refaça com vigência "Todos" e o período de vigência indo de um ano atrás até hoje. Para fatos passados, use as datas do contrato. **O período aceita no máximo 1 ano**; para contratos mais longos, faça uma busca por ano. Não pesquise com vigência "Todos" sem período: o servidor varre o histórico inteiro e devolve 504 (timeout).

Histórico de vários anos (ex.: "últimos 5 anos"): faça uma busca por janela de 1 ano, junte as listas e elimine repetidos pelo nº de registro (um instrumento aparece em todas as janelas que ele cruza). Depois organize por unidade/sindicato e data-base e procure buracos na sequência anual. Se faltar um ano, repita a busca daquele período com um trecho menor do nome (ex.: "CERRADINHO" em vez de "CERRADINHO BIOENERGIA"), porque o acordo pode ter sido assinado por outra empresa do grupo. Se continuar faltando, diga que não foi encontrado. Lembre que a ultratividade é vedada (art. 614, §3º, CLT): cada norma só vale para fatos dentro da própria vigência.

Uma mesma categoria profissional costuma ter várias convenções, uma por sindicato patronal (ex.: os comerciários de SP têm CCTs distintas com Sindilojas, Fecomercio/atacadistas, Sincovaga, Sincamesp etc.). A norma aplicável depende do enquadramento pela atividade preponderante da empresa. Se a atividade não estiver clara, apresente as principais com a abrangência de cada uma e diga isso, em vez de escolher uma. Não pergunte o que dá para inferir; pergunte só se não houver nenhum filtro de participante nem categoria (a busca ficaria ampla demais).

Razão social, CNPJ e CEI são mutuamente exclusivos (o site desmarca os outros). Razão social também desmarca categoria. A busca por razão social é por trecho do nome, sem acento e em maiúsculas (os nomes estão cadastrados assim); se não vier nada, tente um trecho menor ("COMERCIARIOS DE SAO PAULO"). **A categoria é o contrário: precisa de acento** ("Açúcar" acha centenas de instrumentos; "acucar" não acha nenhum).

#### Normas aplicáveis a uma empresa (ACT + CCT)

Buscar pelo nome ou CNPJ da empresa só traz os instrumentos que ela **assinou** (acordos coletivos e, raramente, convenções). A convenção da categoria, assinada pelos sindicatos sem a empresa, não aparece. Quando o pedido for "as normas da empresa X" ou a norma aplicável a um empregado dela, levante as duas:

1. Acordos da empresa: busca por CNPJ ou razão social (passos 2 e 3).
2. Na lista e nos extratos dos acordos, anote o **sindicato profissional** (com CNPJ) e a **base territorial** (cláusula de abrangência) de cada unidade.
3. Convenções da categoria: nova busca pelo **CNPJ do sindicato profissional**, com tipo `convencao` e o mesmo período. O mesmo sindicato também aparece em acordos de outras empresas, por isso o tipo `convencao` é necessário.
4. Se não vier convenção, tente pela categoria (com acento, ex.: "Açúcar", "Alimentação"), tipo `convencao` e UF de registro. A CCT pode ter sido assinada pela federação, e não pelo sindicato local. Confira na abrangência se o município da unidade está incluído.
5. Se a empresa não tiver acordo, pergunte ou deduza a atividade e o município, e vá direto ao item 4.

Na resposta, diga qual instrumento prevalece. Pelo art. 620 da CLT (redação da Lei 13.467/2017), as condições do acordo coletivo sempre prevalecem sobre as da convenção. Se a CCT trouxer cláusulas sobre temas que o ACT não regula, mostre-as e avise que a aplicação delas é discutível. Se nenhuma convenção for encontrada, diga isso; em alguns setores a negociação é só por acordo.

### 2. Abrir a página e pesquisar

Abra `https://mediador.trabalho.gov.br/sistemas/mediador/ConsultarInstColetivo` (no navegador integrado: `preview_start` com `url`; para buscas seguintes, `navigate` para a mesma URL).

**Na primeira busca da conversa, confira se o formulário continua igual.** Os snippets dependem dos IDs abaixo; se o MTE mudar a página, eles falham sem erro e a busca volta vazia, o que pareceria "nenhuma norma":

```js
const ids = ['chkNRCNPJ','txtNRCNPJ','chkNORazaoSocial','txtNORazaoSocial','chkCategoria','txtDSCategoria',
  'cboTPRequerimento','cboSTVigencia','cboUFRegistro','chkVigencia','txtDTInicioVigencia','txtDTFimVigencia',
  'cboGrupoClausula','cboSubGrupoClausula','txtNOTituloClausula','btnPesquisar','divConsultaDetalhada'];
const tipos = [...document.querySelectorAll('#cboTPRequerimento option')].map(o => o.value);
({ faltando: ids.filter(id => !document.getElementById(id)),
   tiposOk: ['acordo','convencao','termoAditivoAcordo','termoAditivoConvecao'].every(v => tipos.includes(v)),
   jquery: typeof $ === 'function', recaptcha: typeof grecaptcha === 'object' });
```

Esperado: `faltando: []`, `tiposOk`, `jquery` e `recaptcha` como `true`. Se algo falhar, não confie nos snippets: mapeie a página de novo (liste `input, select, button` com id, name e o texto da linha), adapte os IDs e avise o usuário que o site mudou e a skill precisa ser atualizada.

Depois use duas chamadas de `javascript_tool`: a primeira preenche e clica, a segunda espera e lê. Uma só chamada pode passar do limite de ~45 s da ferramenta quando o servidor demora.

**Chamada A — preencher e pesquisar** (ajuste o objeto `f`):

```js
const f = {
  cnpj: '',            // '60.989.944/0001-65' (com máscara)
  razao: 'SINDICATO DOS COMERCIARIOS DE SAO PAULO',
  categoria: '',       // não combina com razão social
  tipo: 'convencao',   // '' = todos; ver references/campos.md
  vigencia: '1',       // '1' vigentes, '0' não vigentes, '2' todos
  ufRegistro: 'SP',    // '' = qualquer; 'SRT' = registro nacional
  vigDe: '', vigAte: '', // 'dd/mm/aaaa' — vigentes nesse intervalo; máximo 1 ano
  grupo: '', subgrupo: '', tituloClausula: '' // filtro de cláusula; ver abaixo
};
const ck = id => { const c = document.getElementById(id); if (!c.checked) c.click(); };
const val = (id, v) => { $('#' + id).val(v); };
if (f.cnpj)  { ck('chkNRCNPJ'); val('txtNRCNPJ', f.cnpj); }
if (f.razao) { ck('chkNORazaoSocial'); val('txtNORazaoSocial', f.razao); }
if (f.categoria && !f.razao) { ck('chkCategoria'); val('txtDSCategoria', f.categoria); }
val('cboTPRequerimento', f.tipo); val('cboSTVigencia', f.vigencia); val('cboUFRegistro', f.ufRegistro);
if (f.vigDe) { ck('chkVigencia'); val('txtDTInicioVigencia', f.vigDe); val('txtDTFimVigencia', f.vigAte || f.vigDe); }
// sempre definir os campos de cláusula: o navegador restaura valores da busca anterior
$('#cboGrupoClausula').val(f.grupo).trigger('change');
await new Promise(r => setTimeout(r, f.grupo ? 2500 : 300));
val('cboSubGrupoClausula', f.subgrupo); val('txtNOTituloClausula', f.tituloClausula);
document.getElementById('btnPesquisar').click();
'pesquisa enviada';
```

**Chamada B — esperar e ler** (se voltar "ainda processando", rode de novo):

```js
const dialogo = () => [...document.querySelectorAll('.ui-dialog')]
  .find(d => d.offsetWidth > 0 && !/Processando/.test(d.innerText) && d.innerText.trim());
let h = '', alerta;
for (let i = 0; i < 30 && !h && !alerta; i++) {
  await new Promise(r => setTimeout(r, 1000));
  h = document.querySelector('#divConsultaDetalhada h2')?.innerText || '';
  alerta = dialogo()?.innerText.trim();
}
if (h) await new Promise(r => setTimeout(r, 1000)); // a lista termina de renderizar depois do h2
const lista = [...document.querySelectorAll('#divConsultaDetalhada a[href*="ResumoVisualizar"]')].map(a => {
  const t = a.closest('table').innerText;
  const g = re => (t.match(re) || [])[1]?.trim();
  return {
    registro: g(/Registro\s+(\S+)/), solicitacao: g(/Solicitação\s+(\S+)/),
    tipo: g(/Instrumento\s*\n\s*(.+?)\t/), vigencia: g(/Vigência\s+([\d\/ \-]+)/),
    partes: t.split(/Partes\s*/)[1]?.split(/\s*Download/)[0].trim().split(/\n+/),
    url: a.href
  };
});
({ resultado: h || (alerta ? 'ALERTA: ' + alerta : 'ainda processando'), lista });
```

Um alerta "Nenhum registro encontrado." é um resultado válido (zero instrumentos). Outros alertas (ex.: "O intervalo máximo de pesquisa é de um ano.") significam que a busca não foi feita: corrija o filtro, recarregue e repita.

Os IDs são estáveis; não use refs de `read_page` (mudam a cada recarga). Recarregue a página antes de cada nova busca.

**Filtro de cláusula do site** (Grupo/SubGrupo/Palavra do Título): usa outro endpoint (`getConsultaAvancadaClausula`) e devolve direto o texto das cláusulas, sem links "Visualizar" (a `lista` vem vazia; leia `#divConsultaDetalhada` com `innerText`). Só use com um filtro estreito (CNPJ); com razão social ou categoria ele costuma dar 504 e a tela fica em "Processando". Na dúvida, faça a busca sem cláusula e filtre no passo 4.

### 3. Ler a lista de resultados

O cabeçalho diz "Resultado: N Instrumento(s) ... Página X de Y" (10 por página); a `lista` traz registro, solicitação, tipo, vigência, partes e o link de cada instrumento.

Próxima página: clique no link "Próximo >>" de `#divConsultaDetalhada` (não pede novo reCAPTCHA), espere o `h2` mudar de página e rode de novo só o trecho que monta a `lista`. Se houver muitas páginas (mais de ~5), não percorra tudo: mostre o total e sugira refinar (UF, tipo, razão social, período).

Quando houver vários resultados para a mesma categoria, o mais recente costuma ser o que interessa. Para a vigência certa num caso concreto, compare o período de vigência com as datas do contrato de trabalho. Aditivos alteram a convenção principal; quando o usuário precisa do texto vigente, procure também os termos aditivos.

### 4. Abrir o instrumento e extrair cláusulas

Cada resultado tem um link direto que abre o extrato completo em HTML:
`https://mediador.trabalho.gov.br/sistemas/mediador/Resumo/ResumoVisualizar?NrSolicitacao=MR071249%2F2024`

Abra numa aba separada (`tabs_create` + `navigate`) para não perder a lista. Prefira "Visualizar" a "Download": o download gera um arquivo e exige autorização do usuário.

Para não despejar o texto inteiro (60+ cláusulas, dezenas de milhares de caracteres), separe por cláusula e filtre **primeiro pelo título**. O filtro pelo corpo é só o plano B, porque termos comuns ("piso", "data-base") aparecem em muitas cláusulas:

```js
const termos = ['piso', 'salario normativo', 'salarios de admissao'];
const noCorpo = false; // true só se o filtro por título não achar nada
const norm = s => s.normalize('NFD').replace(/[̀-ͯ]/g, '').toLowerCase();
const partes = document.body.innerText.split(/\n(?=CLÁUSULA [A-ZÇÃÉÊÍÓÔÚ\- ]+?(?: -|–))/);
const casa = p => termos.some(t => norm(noCorpo ? p : p.split('\n')[0]).includes(norm(t)));
({ cabecalho: partes[0].trim().slice(0, 1500), totalClausulas: partes.length - 1,
   titulos: partes.slice(1).map(p => p.split('\n')[0].trim()),
   achadas: partes.slice(1).filter(casa).map(p => p.trim()) });
```

A lista `titulos` sempre volta: se `achadas` vier vazia ou errada, escolha pelos títulos e rode de novo com o termo exato. O mesmo assunto tem nomes diferentes de uma norma para outra. Use sinônimos:

- piso: "piso", "salário normativo", "salário de admissão", "salário mínimo profissional"
- banco de horas: "banco de horas", "compensação"
- insalubridade: "insalubr"

Se o usuário pedir o texto integral, use `get_page_text`.

**Vários instrumentos de uma vez** (ex.: levantar data de registro, partes e abrangência de 20 acordos): em vez de abrir aba por aba, use `fetch` a partir de uma aba que já esteja no Mediador (mesma origem, a sessão do navegador passa pelo Cloudflare). O HTML vindo do `fetch` não é renderizado, então `innerText` não separa as cláusulas em linhas e o snippet de cláusulas acima não funciona nele. Use o snippet abaixo (as partes vêm do `innerText`, que preserva as quebras do código-fonte; o resto, do `textContent`):

```js
const sols = ['MR054219/2026', 'MR041586/2026']; // nº da solicitação de cada resultado
const out = {};
for (const s of sols) {
  const html = await (await fetch('/sistemas/mediador/Resumo/ResumoVisualizar?NrSolicitacao=' + encodeURIComponent(s))).text();
  const body = new DOMParser().parseFromString(html, 'text/html').body;
  const t = body.textContent.replace(/\s+/g, ' ');
  const g = re => (t.match(re) || [])[1]?.trim();
  out[s] = {
    dataRegistro: g(/DATA DE REGISTRO NO MTE:\s*([\d\/]+)/),
    partes: [...body.innerText.matchAll(/\n\s*([^\n]{5,200}?), CNPJ n\. ([\d\.\/\-]+)/g)].map(m => m[1].trim() + ' (' + m[2] + ')'),
    abrangencia: g(/abrangerá a\(s\) categoria\(s\) (.{0,900}?abrangência territorial em [^.]*\.)/),
    aditivos: [...(t.match(/ADITIVO\(S\) VINCULADO\(S\)(.*?)CNPJ n\./) || ['', ''])[1]
      .matchAll(/Processo nº:\s*(\S+?)\s*e Registro nº:\s*([A-Z]{2,3}\d{6}\/\d{4})?/g)]
      .map(m => ({ processo: m[1], registro: m[2] || 'sem registro' })),
    clausulas: [...t.matchAll(/CLÁUSULA [A-ZÇÃÉÊÍÓÔÚ]+(?: [A-ZÇÃÉÊÍÓÔÚ]+)? [-–] ([A-ZÇÃÉÊÍÓÔÚÕÂ0-9 ,\/\.\(\)ªº]{3,90}?)(?= [A-Z][a-zçãéáíóú])/g)].map(m => m[1].trim())
  };
}
JSON.stringify(out);
```

**Termos aditivos vinculados:** o campo `aditivos` lê a seção "Termos aditivo(s) vinculado(s)" do cabeçalho do extrato. Aditivos alteram cláusulas da norma principal, então, sempre que for transcrever uma cláusula ou dizer o "texto vigente", confira essa lista. Para cada registro: procure-o na lista de resultados que você já tem (aditivos aparecem nas mesmas buscas por partes e período). Se não estiver, busque pelo CNPJ de uma das partes com tipo `termoAditivoAcordo` ou `termoAditivoConvecao` e o período da norma principal. Abra o aditivo e veja se ele altera a cláusula pedida. "sem registro" indica um aditivo protocolado mas ainda não registrado; informe isso ao usuário.

Os títulos de cláusula mostram o assunto de cada instrumento. Isso ajuda a separar acordos paralelos do mesmo período (ex.: um acordo geral de salários e benefícios e outro só de jornada/turnos). Para transcrever uma cláusula, abra o instrumento numa aba e use o snippet de cláusulas acima.

### 5. Responder

Para cada instrumento usado, informe:

- tipo e período (ex.: Convenção Coletiva 2024/2026), vigência e data-base;
- partes (sindicatos/empresa);
- nº de registro no MTE, data de registro e nº da solicitação;
- abrangência (categoria e base territorial, da cláusula de abrangência);
- link `ResumoVisualizar` para conferência;
- termos aditivos vinculados (registro e se alteram as cláusulas transcritas).

Para levantamentos de vários anos, use tabelas por unidade/sindicato (período, registro, solicitação com link, vigência, data de registro), marcando o vigente e os anos sem instrumento encontrado.

Depois, as cláusulas pedidas, transcritas na íntegra com o número e o título (ex.: "CLÁUSULA TERCEIRA - PISOS SALARIAIS..."), porque peças e cálculos precisam do texto exato. Não resuma valores, percentuais ou condições; transcreva.

Diga com clareza o que não foi verificado: o Mediador só traz instrumentos registrados no sistema; ausência de resultado não prova que não exista norma (pode ter outro nome de sindicato, outra UF de registro, ou não ter sido registrada). Confira a abrangência antes de afirmar que a norma se aplica a um empregado.
