# Campos do formulário (ConsultarInstColetivo)

Levantado em 28/10/2026. Se algum ID deixar de existir, mapeie de novo com `javascript_tool` listando `input, select, button` da página.

## Participante (mutuamente exclusivos; ative o checkbox antes)

| Filtro | Checkbox | Campo | Formato |
|---|---|---|---|
| CNPJ/CAEPF | `chkNRCNPJ` | `txtNRCNPJ` | `99.999.999/9999-99` |
| Matrícula CEI | `chkNRCEI` | `txtNRCEI` | `99999999999-9` |
| Razão Social | `chkNORazaoSocial` | `txtNORazaoSocial` | texto (trecho) |
| Categoria | `chkCategoria` | `txtDSCategoria` | texto **com acento** ("Açúcar"); desmarcado pela razão social |

Razão social: sem acento, em maiúsculas. Categoria: com acento (sem acento volta "Nenhum registro encontrado.").

## Instrumento

`cboTPRequerimento` (tipo; vazio = todos):

- `acordo` — Acordo Coletivo
- `acordoColetivoEspecificoPPE` — Acordo Coletivo Específico (PPE)
- `acordoColetivoEspecificoDomingosFeriados` — Acordo Específico, domingos e feriados
- `convencao` — Convenção Coletiva
- `termoAditivoAcordo` — Termo Aditivo de Acordo
- `termoAditivoConvecao` — Termo Aditivo de Convenção (grafia do site, sem "n")
- `termoAditivoAcordoEspecificoPPE`
- `termoAditivoAcordoEspecificoDomingoFeriado`

`cboSTVigencia`: `2` Todos, `1` Vigentes, `0` Não vigentes (vazio = "Selecione").

`cboUFRegistro`: `SRT` (registro nacional) ou sigla da UF. É a UF onde foi registrado, não necessariamente a base territorial.

Datas (`dd/mm/aaaa`):

- Período do registro: `chkPeriodoRegistro`, `txtDTInicioRegistro`, `txtDTFimRegistro`
- Vigência no período: `chkVigencia`, `txtDTInicioVigencia`, `txtDTFimVigencia`

Os dois períodos aceitam no máximo 1 ano. Com vigência "Todos" e sem período, a busca costuma dar 504. Acima disso, o site abre o alerta "O intervalo máximo de pesquisa é de um ano." e não pesquisa.

Diálogos: o DOM tem sempre um `.ui-dialog` oculto ("Município por UF"). Para achar um alerta real, filtre pelos diálogos visíveis (`offsetWidth > 0`).

Abrangência (checkboxes, todos marcados por padrão): `chkMunicipal`, `chkEstadual`, `chkNacional`, `chkIntermunicipal`, `chkInterestadual`. A abrangência territorial por UF/município (`cboUFAbrangenciaTerritorial`, botões `btnAdicionarUF` / `btnSelecionarMunicipio`) abre um pop-up; use só se for indispensável.

## Cláusulas (só com filtro estreito, como CNPJ — com filtros amplos o endpoint dá 504)

`cboGrupoClausula`: 1 Salários, Reajustes e Pagamento · 2 Gratificações, Adicionais, Auxílios e Outros · 3 Contrato de Trabalho · 4 Relações de Trabalho, Condições, Estabilidades · 5 Jornada de Trabalho · 6 Férias e Licenças · 7 Saúde e Segurança · 8 Relações Sindicais · 9 Disposições Gerais.

`cboSubGrupoClausula` é carregado por AJAX após `change` no grupo (ex.: grupo 5 → 81 Compensação de Jornada, 84 Controle da Jornada, 86 Turnos Ininterruptos...). `txtNOTituloClausula`: palavra do título.

Os títulos de grupo aparecem no texto do extrato como cabeçalhos em maiúsculas (ex.: "SALÁRIOS, REAJUSTES E PAGAMENTO", "PISO SALARIAL") antes das cláusulas; servem para localizar o assunto quando a busca por termo falhar.

## Endpoints (só para diagnóstico; não chame direto)

- `POST /sistemas/mediador/ConsultarInstColetivo/getConsultaAvancada` — busca normal (exige `recaptchaToken`)
- `POST .../getConsultaAvancadaClausula` — busca por cláusula (lenta, 504)
- `GET /sistemas/mediador/Resumo/ResumoVisualizar?NrSolicitacao=MRxxxxxx%2Faaaa` — extrato completo
- Paginação: `funcPaginar(pagina, total)` via links do resultado.
