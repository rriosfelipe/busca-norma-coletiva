# Snippets para `javascript_tool`

Rode cada um na aba do Mediador. Ajuste só o que está marcado como entrada (objeto `f`, `termos`, `sols`). Os IDs são estáveis; não use refs de `read_page` (mudam a cada recarga).

- [S1. Checagem do formulário](#s1-checagem-do-formulário)
- [S2. Busca — preencher e clicar](#s2-busca--preencher-e-clicar)
- [S3. Busca — esperar e ler a lista](#s3-busca--esperar-e-ler-a-lista)
- [S4. Cláusulas de um instrumento](#s4-cláusulas-de-um-instrumento)
- [S5. Vários instrumentos de uma vez (lote)](#s5-vários-instrumentos-de-uma-vez-lote)
- [S6. Comparar com o instrumento anterior](#s6-comparar-com-o-instrumento-anterior)

## S1. Checagem do formulário

Na página `ConsultarInstColetivo`, antes da primeira busca da conversa.

```js
const ids = ['chkNRCNPJ','txtNRCNPJ','chkNORazaoSocial','txtNORazaoSocial','chkCategoria','txtDSCategoria',
  'cboTPRequerimento','cboSTVigencia','cboUFRegistro','chkVigencia','txtDTInicioVigencia','txtDTFimVigencia',
  'cboGrupoClausula','cboSubGrupoClausula','txtNOTituloClausula','btnPesquisar','divConsultaDetalhada'];
const tipos = [...document.querySelectorAll('#cboTPRequerimento option')].map(o => o.value);
({ faltando: ids.filter(id => !document.getElementById(id)),
   tiposOk: ['acordo','convencao','termoAditivoAcordo','termoAditivoConvecao'].every(v => tipos.includes(v)),
   jquery: typeof $ === 'function', recaptcha: typeof grecaptcha === 'object' });
```

Esperado: `faltando: []` e os outros três `true`.

## S2. Busca — preencher e clicar

Recarregue a página (`navigate` para a mesma URL) antes de cada busca. Entradas no objeto `f`; deixe vazio o que não usar.

```js
const f = {
  cnpj: '',            // '60.989.944/0001-65' (com máscara)
  razao: 'SINDICATO DOS COMERCIARIOS DE SAO PAULO', // sem acento, maiúsculas
  categoria: '',       // COM acento; não combina com razão social
  tipo: 'convencao',   // '' = todos; ver campos.md
  vigencia: '1',       // '1' vigentes, '0' não vigentes, '2' todos
  ufRegistro: 'SP',    // '' = qualquer; 'SRT' = registro nacional
  vigDe: '', vigAte: '', // 'dd/mm/aaaa' — vigentes nesse intervalo; máximo 1 ano
  grupo: '', subgrupo: '', tituloClausula: '' // filtro de cláusula do site; só com CNPJ
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

## S3. Busca — esperar e ler a lista

Em chamada separada de S2 (uma só passaria do limite de ~45 s da ferramenta). Se voltar "ainda processando", rode de novo. Também serve para reler a lista depois de clicar em "Próximo >>".

```js
const dialogo = () => [...document.querySelectorAll('.ui-dialog')]
  .find(d => d.offsetWidth > 0 && !/Processando/.test(d.innerText) && d.innerText.trim());
// desafio visível do Cloudflare ou do reCAPTCHA (o selo do v3 não conta)
const desafio = () => /just a moment|verify you are human|verifique se você é humano/i.test(document.title + ' ' + document.body.innerText.slice(0, 1000))
  || [...document.querySelectorAll('iframe[src*="recaptcha"][src*="bframe"], iframe[src*="challenges.cloudflare.com"]')]
    .some(f => f.offsetWidth > 0 && f.offsetHeight > 100);
let h = '', alerta;
for (let i = 0; i < 30 && !h && !alerta; i++) {
  await new Promise(r => setTimeout(r, 1000));
  h = document.querySelector('#divConsultaDetalhada h2')?.innerText || '';
  alerta = dialogo()?.innerText.trim() || (desafio() ? 'DESAFIO: o site pediu verificação humana; peça ao usuário que resolva no painel do navegador' : '');
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
const total = +(h.match(/(\d+) Instrumento/) || [])[1] || 0, pagina = +(h.match(/Página (\d+)/) || [])[1] || 0;
const comClausula = !!($('#cboGrupoClausula').val() || $('#txtNOTituloClausula').val());
// registros já vistos nas páginas anteriores desta busca (a página recarregada zera)
if (pagina <= 1 || !window.__vistos) window.__vistos = new Set();
const repetidos = lista.filter(x => window.__vistos.has(x.registro)).map(x => x.registro);
lista.forEach(x => window.__vistos.add(x.registro));
const aviso = total > 0 && !lista.length && !comClausula
  ? 'LAYOUT: o site diz ' + total + ' instrumento(s), mas nenhum foi lido; o HTML mudou — mapeie a página de novo'
  : repetidos.length === lista.length && lista.length ? 'PAGINAÇÃO: página só com registros já vistos; o clique em "Próximo" não avançou' : '';
({ resultado: h || (alerta ? 'ALERTA: ' + alerta : 'ainda processando'), aviso, pagina, repetidos, lista });
```

Com o filtro de cláusula do site, a `lista` vem vazia: o resultado é o próprio texto das cláusulas; leia `document.querySelector('#divConsultaDetalhada').innerText`.

Rode S3 uma vez por página: rodar de novo na mesma página marca tudo como repetido.

## S4. Cláusulas de um instrumento

De qualquer aba que já esteja no Mediador; não precisa abrir o extrato. O HTML do `fetch` vai para um `div` fora da tela para ser renderizado, e o `innerText` volta com as quebras de linha que separam as cláusulas.

```js
const sol = 'MR054219/2026';                 // nº da solicitação
const termos = ['horas extras', 'hora extra']; // sem acento; ver sinônimos no SKILL.md
const noCorpo = false; // true só se o filtro por título não achar nada
const norm = s => s.normalize('NFD').replace(/[̀-ͯ]/g, '').toLowerCase();
const html = await (await fetch('/sistemas/mediador/Resumo/ResumoVisualizar?NrSolicitacao=' + encodeURIComponent(sol))).text();
const div = document.createElement('div');
div.style.cssText = 'position:absolute;left:-99999px;width:900px';
div.innerHTML = new DOMParser().parseFromString(html, 'text/html').body.innerHTML;
document.body.appendChild(div);
const texto = div.innerText.replace(/[ \t]+\n/g, '\n').replace(/\n{3,}/g, '\n\n'); div.remove();
// o extrato emenda no fim de cada cláusula o título do próximo grupo/subgrupo ("Adicional de Insalubridade");
// corta só linhas curtas, sem números, sem "a)"/"b)" e sem pontuação final — e devolve o que cortou
const titulo = s => s.length < 90 && !/\d/.test(s) && !/^[a-zA-Z]{1,3}\)/.test(s) && !/[.;:,)”"]$/.test(s);
const limpa = p => { const l = p.trim().split('\n').map(x => x.trim()); const cortadas = [];
  while (l.length > 2 && cortadas.length < 3 && (l.at(-1) === '' || titulo(l.at(-1)))) cortadas.unshift(l.pop());
  return { texto: l.join('\n').trim(), cortadas: cortadas.filter(Boolean) }; };
const partes = texto.split(/\n(?=CLÁUSULA [A-ZÇÃÉÊÍÓÔÚ\- ]+?(?: -|–))/);
const casa = p => termos.some(t => norm(noCorpo ? p : p.split('\n')[0]).includes(norm(t)));
({ cabecalho: partes[0].trim().slice(0, 1500), totalClausulas: partes.length - 1,
   titulos: partes.slice(1).map(p => p.split('\n')[0]),
   achadas: partes.slice(1).filter(casa).map(limpa) });
```

Confira `cortadas` de cada cláusula: devem ser só títulos de grupo ou subgrupo. Se houver algo do texto da cláusula ali, devolva-o à transcrição.

## S5. Vários instrumentos de uma vez (lote)

De qualquer aba que já esteja no Mediador (mesma origem; a sessão do navegador passa pelo Cloudflare). O HTML do `fetch` não é renderizado, por isso o snippet usa `textContent` para quase tudo e `innerText` só para as partes.

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

`aditivos` vazio = nenhum aditivo vinculado; `registro: 'sem registro'` = aditivo protocolado mas ainda não registrado.

## S6. Comparar com o instrumento anterior

De qualquer aba que já esteja no Mediador. Alinha as cláusulas dos dois instrumentos pelo título (sem o número, que muda de um ano para outro) e classifica cada uma.

```js
const antes = 'MR033274/2024', depois = 'MR047163/2025'; // nº da solicitação de cada um
const termos = []; // vazio = só o mapa; com termos (do título), devolve também o texto integral das duas versões
const norm = s => s.normalize('NFD').replace(/[̀-ͯ]/g, '').toLowerCase();
const titulo = s => s.length < 90 && !/\d/.test(s) && !/^[a-zA-Z]{1,3}\)/.test(s) && !/[.;:,)”"]$/.test(s);
const limpa = p => { const l = p.trim().split('\n').map(x => x.trim());
  while (l.length > 2 && (l.at(-1) === '' || titulo(l.at(-1)))) l.pop(); return l.join('\n').trim(); };
const chave = t => norm(t.replace(/^CLÁUSULA [A-ZÇÃÉÊÍÓÔÚ\- ]+?(?: -|–)\s*/, '')).replace(/[^a-z0-9]+/g, '');
const ler = async sol => {
  const html = await (await fetch('/sistemas/mediador/Resumo/ResumoVisualizar?NrSolicitacao=' + encodeURIComponent(sol))).text();
  const div = document.createElement('div');
  div.style.cssText = 'position:absolute;left:-99999px;width:900px';
  div.innerHTML = new DOMParser().parseFromString(html, 'text/html').body.innerHTML;
  document.body.appendChild(div);
  const texto = div.innerText.replace(/[ \t]+\n/g, '\n').replace(/\n{3,}/g, '\n\n'); div.remove();
  const m = new Map();
  for (const p of texto.split(/\n(?=CLÁUSULA [A-ZÇÃÉÊÍÓÔÚ\- ]+?(?: -|–))/).slice(1)) {
    const t = p.split('\n')[0]; m.set(chave(t), { titulo: t, corpo: limpa(p).split('\n').slice(1).join('\n').trim() });
  }
  return m;
};
const valores = s => [...s.matchAll(/R\$\s*[\d.]+,\d{2}|\d+(?:,\d+)?\s*%/g)].map(x => x[0].replace(/\s+/g, ' '));
const forma = s => norm(s.replace(/\b(19|20)\d{2}\b/g, '')).replace(/[^a-z0-9]+/g, '');
// do primeiro ao último ponto em que os textos divergem (com 4 palavras de contexto)
const trecho = (a, b) => { const x = a.split(/\s+/), y = b.split(/\s+/); let i = 0, j = 0;
  while (i < x.length && i < y.length && x[i] === y[i]) i++;
  while (j < x.length - i && j < y.length - i && x.at(-1 - j) === y.at(-1 - j)) j++;
  const corta = w => { const s = w.join(' '); return s.length > 300 ? s.slice(0, 300) + '…' : s; };
  return { antes: corta(x.slice(Math.max(0, i - 4), x.length - j)), depois: corta(y.slice(Math.max(0, i - 4), y.length - j)) }; };
const [A, B] = [await ler(antes), await ler(depois)];
const out = { antes: { sol: antes, clausulas: A.size }, depois: { sol: depois, clausulas: B.size },
  novas: [], removidas: [], alteradas: [], soFormaOuAno: [], iguais: 0, textos: [] };
for (const [k, b] of B) {
  const a = A.get(k);
  if (!a) { out.novas.push(b.titulo); continue; }
  if (a.corpo.replace(/\s+/g, ' ') === b.corpo.replace(/\s+/g, ' ')) out.iguais++;
  else if (forma(a.corpo) === forma(b.corpo)) out.soFormaOuAno.push(b.titulo);
  else { const va = valores(a.corpo), vb = valores(b.corpo);
    out.alteradas.push({ titulo: b.titulo, valoresAntes: va.filter(v => !vb.includes(v)),
      valoresDepois: vb.filter(v => !va.includes(v)), trecho: trecho(a.corpo, b.corpo) }); }
  if (termos.some(t => k.includes(norm(t).replace(/[^a-z0-9]+/g, '')))) out.textos.push({ titulo: b.titulo, antes: a.corpo, depois: b.corpo });
}
for (const [k, a] of A) if (!B.has(k)) out.removidas.push(a.titulo);
out;
```

- `alteradas`: o texto mudou. `valoresAntes`/`valoresDepois` trazem os R$ e % que mudaram; `trecho` mostra o pedaço que diverge (pode ser só pontuação no começo e mudança real depois; na dúvida, rode de novo com o título em `termos`).
- `soFormaOuAno`: só mudaram maiúsculas, pontuação ou anos; não precisa listar como alteração.
- `novas` e `removidas`: confira se não é a mesma cláusula com título novo (ex.: "HOMOLOGAÇÃO RESCISÃO CONTRATUAL" → "TERMO DE QUITAÇÃO ANUAL E HOMOLOGAÇÃO DA RESCISÃO"); se for, compare as duas com S4.
