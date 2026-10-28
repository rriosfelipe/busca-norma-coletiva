# Snippets para `javascript_tool`

Rode cada um na aba do Mediador. Ajuste só o que está marcado como entrada (objeto `f`, `termos`, `sols`). Os IDs são estáveis; não use refs de `read_page` (mudam a cada recarga).

- [S1. Checagem do formulário](#s1-checagem-do-formulário)
- [S2. Busca — preencher e clicar](#s2-busca--preencher-e-clicar)
- [S3. Busca — esperar e ler a lista](#s3-busca--esperar-e-ler-a-lista)
- [S4. Cláusulas de um instrumento](#s4-cláusulas-de-um-instrumento)
- [S5. Vários instrumentos de uma vez (lote)](#s5-vários-instrumentos-de-uma-vez-lote)

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

Com o filtro de cláusula do site, a `lista` vem vazia: o resultado é o próprio texto das cláusulas; leia `document.querySelector('#divConsultaDetalhada').innerText`.

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
