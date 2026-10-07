# Especificação Técnica e Funcional do Projeto — Complemento "Cálculos Previdenciários"

**Documento:** SPEC / Análise completa do repositório
**Data da análise:** 08/10/2026
**Objeto:** Add-on para Google Planilhas ("Cálculos Previdenciários"), desenvolvido em Google Apps Script pela **Contadoria** (organização no GitHub), publicado na Chrome Web Store com ID `cjeahogdijapbbnndojpidgdneeepnla`.
**Perspectiva da análise:** tripla — (I) desenvolvedor sênior (arquitetura/código), (II) advogado previdenciário (domínio jurídico-calcístico), (III) especialista em RPPS (regime próprio).

---

## 1. Sumário executivo

O projeto é o **front-end de automação** das planilhas de cálculo da Contadoria — ferramenta já consagrada na Justiça Federal (JEFs) para liquidação de sentenças previdenciárias. O add-on **não calcula**: ele prepara, higieniza e parametriza as planilhas-mestre (TC, RMI, ATRASADOS), por meio de:

1. extração automática de dados de texto colado (**CNIS, SisJEF, PLENUS, HISCRE**);
2. atualização remota de **índices de correção monetária/juros** e **tabelas de domínio** (salário-de-benefício, tempo mínimo, carência, pontuação da regra de transição do art. 15 da EC 103/2019, tábuas de mortalidade — aposentadoria por incapacidade permanente/progama);
3. área de transferência entre planilhas via cache de usuário;
4. "padrões do usuário" (dados repetitivos como DER, NB, juizos/seções salvos em `UserProperties`);
5. calculadoras auxiliares (intervalo em dias, prescrição, soma/subtração de tempo);
6. registro estatístico anônimo dos cálculos realizados (serviço POST configurável por planilha).

**Estado geral:** produto funcional, maduro em UX, mas com **dívida técnica relevante**: dependências críticas ausentes do repositório (bibliotecas `Utils`, `Planilhas`, `TextParser`), manifesto incompleto, sem testes, sem controle de versão do código, bibliotecas externas não pinadas, e regras de negócio enterradas em *magic numbers*. Para uso pericial (art. 465 CPC), a ausência de trilha de auditoria robusta e de versionamento de tabelas é o ponto mais sensível.

---

## 2. Visão geral e posicionamento no ecossistema

### 2.1 O que é
Add-on (extensão de Scripts do Google Docs) que roda dentro do contexto de uma planilha da Contadoria ou de qualquer planilha compatível. Arquitetura em três camadas:

```
[Google Sheets]
   └─ Sidebar HTML (Vue.js 2 + Vuex)  ← _index/_app/_config/_view-*/*_ui-*
        └─ google.script.run           ← Ui.gs (obterDados/registrarErro)
             └─ Módulos server-side (.gs): Preenchimento, Atualizacoes, AreaTransferencia,
                Registro, Estatistica, Erros, Notificacao, Http, Config, Inicializacao
                  └─ Bibliotecas EXTERNAS (não estão neste repo): Utils, Planilhas, TextParser
                  └─ Webservices externos (URLs em ScriptProperties):
                       • Webservice de índices/tabelas (API REST JSON {status, payload})
                       • Serviço central de erros (form POST → planilha/log)
                       • Serviço estatístico (por planilha, URL gravada em intervalo nomeado)
```

### 2.2 Contratos implícitos (acoplamento com as planilhas-mestre)
Todo o comportamento depende de **intervalos nomeados** convencionados nas planilhas da Contadoria:

| Prefixo/Intervalo | Função |
|---|---|
| `_p_*` / `_p_f_*` | Intervalos candidatos a "padrão do usuário" (valor / fórmula) |
| `_c_*` / `_c_f_*` | Intervalos candidatos à área de transferência comum entre planilhas |
| `_limpar_` | Flag de limpeza (definida em `Config.PREFIXO`, sem uso no código deste repo) |
| `_v_Tipo` | Identifica o tipo da planilha: `TC`, `RMI` ou `ATRASADOS` — chave de roteamento das atualizações |
| `_v_VersaoAtual`, `_v_Data` | Versionamento da planilha (usado em logs de erro) |
| `DER`, `Citacao` | Termo inicial e marco interruptivo pré-carregados na calculadora de prescrição |
| `Processo`, `Juizo`, `Secao`, `Subsecao` | Metadados para registro estatístico |
| `CompetenciaIndices`, `CompetenciaDescontos`, `SalariosContribuicao`, `TCInformado`, `DescontosNB1..5(+Renda/Abono)` | Alvos de preenchimento automático |
| `DadosTemplate`, `LogTemplate` | Páginas-modelo clonadas pelo módulo `Planilhas` (externo) |
| `EstatisticaUrl` | Guarda a URL do serviço estatístico como hiperfórmula (`HYPERLINK`) |

Tipos de planilha suportados (`Config_.ATUALIZACOES.CONSULTAS`):
- **TC** (tempo de contribuição): tabelas `ListaJuizos`, `ListaBeneficios`, `TabelaPontuacao`, `TabelaConversao`, `TabelaCarencia`, `ListaMotivos` (índices: lista vazia).
- **RMI** (renda mensal inicial): índices `IndicesConsolidados`; tabelas incl. `OpcoesPC`, `OpcoesAtividadesConcomitantes`, `TabuasMortalidade`.
- **ATRASADOS** (valores atrasados/cumprimento de sentença): índices `IndicesConsolidados`; tabelas de opções de ação/reajuste/correção/juros/alcada (fase de cumprimento).

### 2.3 Dependências críticas AUSENTES do repositório (achado nº 1)
O README afirma que o repo "hospeda a documentação", mas o código **não executa sem**:
- **`Utils`** (GS library): `CustomError`, `isNumeric`, `tentarConverterEmData`, `isTable`, `posicaoCompetencia`, `mensagemErro`, `nomeErro`, `stackErro`, `isString`, `isObject`, `isCustomError`, `humanizar`, `atualizarIntervalo`, `load()`. No browser: `debounce`, `calcularPrescricao`, `dias360`, `isDate`.
- **`Planilhas`** (GS library): `atualizarIntervalo`, `criarPagina`, `atualizarPagina`, `atualizarLog`, `criarRelatorio`, `ultimaLinhaNaoVazia`, `inserirUrlComoFormula`, `extrairUrlDeFormula`.
- **`TextParser`** (GS library carregada no client): `testarOrigem`, `processar` — o motor que reconhece CNIS/SisJEF/PLENUS/HISCRE e produz as tabelas/opções de destino.
- **Webservice de índices/tabelas** e **serviços de log/estatística** (URLs em `ScriptProperties`).

Consequência: este repo sozinho **não é buildável nem auditável por completo**; a revisão das fórmulas financeiras (o coração do cálculo) está fora dele.

---

## 3. Inventário do que está pronto (módulo a módulo)

### 3.1 Server-side (Apps Script)

| Arquivo | Linhas | Status | Descrição |
|---|---|---|---|
| `Inicializacao.gs` | 24 | ✅ pronto | `onOpen`/`onInstall`: menu "Cálculos Previdenciários → Abrir"; abre sidebar automaticamente se a preferência `opcaoAbrir` estiver salva. |
| `Ui.gs` | 70 | ✅ pronto | `abrirSidebar()` (HtmlService), `incluir()` (partials), `obterDados()` (bootstrap do client: preferências, mapa de padrões, lista de intervalos, URL estatística, datas de prescrição), `registrarErro()` (ping-back de erros do client). |
| `Config.gs` | 107 | ✅ pronto | Constantes globais: status, tipos de atualização, consultas por tipo de planilha, nomes de intervalos, chaves de storage, prefixos. |
| `Notificacao.gs` | 41 | ✅ pronto | Envelope padrão `Notificacao_.criar(fn, ...args)`: serializa resultado em JSON ou converte exceção em erro de usuário (log local) ou de sistema (log central+local). |
| `Erros.gs` | 113 | ✅ pronto | API fluente `informar(erro).alertar().logLocal().logCentral().dados()`; payload com timestamp, versão da planilha, e-mails ativo/efetivo e URL. |
| `Http.gs` | 89 | ✅ pronto | `consultarServicoInterno(url)` (GET com envelope `{status,payload}`, reconversão de datas) e `criarUrl(base, consultas)` (query string `consulta=X&...`). |
| `AreaTransferencia.gs` | 105 | ✅ pronto | Copiar/colar intervalos `_c_*` entre planilhas via `CacheService.getUserCache()` (TTL 600 s). Bloqueia colar na mesma planilha e em planilhas de desenvolvimento (IDs em `ScriptProperties`). |
| `Registro.gs` | 230 | ✅ pronto | CRUD de "padrões do usuário" em `UserProperties` (mapear/salvar/baixar/atualizar/remover) + preferências (`opcaoAbrir`). |
| `Preenchimento.gs` | 209 | ✅ pronto | Roteador de preenchimento tabular: `BENEFICIO_RECEBIDO` (descontos de NB recebido em até 5 grupos), `SALARIOS` (salários de contribuição no intervalo `SalariosContribuicao`, alinhando competências), `PERIODOS` (appender em `TCInformado`), `NOVA_PAGINA` (cria aba com os dados brutos importados) e modo simples (não tabular). |
| `Atualizacoes.gs` | 209 | ✅ pronto | Orquestra atualização de ÍNDICES/TABELAS: detecta tipo pelo intervalo `_v_Tipo`, monta URL multi-consulta, clona páginas ocultas a partir de `DadosTemplate`, grava relatório em log e devolve tabela HTML de sucessos/erros. |
| `Estatistica.gs` | 211 | ✅ pronto | Obter/testar/registrar serviço estatístico (URL guardada como hiperfórmula no intervalo `EstatisticaUrl`); coleta payload do cálculo (usuário, tipo, URL, processo, juízo, seção, subseção, complemento, observações), confirma em modal e envia por POST. |
| `Descricao.gs` | 13 | ✅ pronto | Metadados/escopos OAuth (`@OnlyCurrentDoc`, external_request, scriptapp, storage, spreadsheets.currentonly, userinfo.email). |

### 3.2 Client-side (sidebar Vue 2)

| Arquivo | Conteúdo pronto |
|---|---|
| `_index.html` | Shell: FontAwesome 4.x (CDN), w3.css 4, add-ons CSS do Google, Vue 2.4.2, Vuex 2.4.0, Velocity 1.5.0; injeta libs via `<?!= ?>`. |
| `_config.html` | Constantes de UI (`MODO_DESENVOLVIMENTO=false`), links institucionais (privacidade/termos/bugs/tutoriais), menu principal (Principal, Calculadoras, Avançado, Atualizar dados, Ajuda), menu de ajuda e **store Vuex** (intervalos, padrões, preferências, URL estatística, datas de prescrição, filas de notificação). |
| `_app.html` | Componente raiz: toolbar + `<keep-alive>` com troca dinâmica de views, spinner global, dialogs de erro/alerta/info e toast de sucesso. |
| `_view-principal.html` | Abas: **Área de transferência** (copiar/colar), **Padrões do usuário** (salvar/baixar com seleção múltipla), **Extrair dados de texto** (textarea com detecção automática de origem via `TextParser.testarOrigem`, resumo do parse, escolha do destino e envio a `preencher`), **Registro estatístico** (status online/offline, teste, edição do serviço, envio com preview). |
| `_view-calculadoras.html` | **Calculadora de intervalo** (métodos EXCEL/EUR/US-NASD, base 360, anos/meses/dias + total, sinal negativo se invertida), **Calculadora de prescrição** (prazo em meses, termo inicial [DER], marco interruptivo [citação], período suspenso → dias suspensos e marco prescricional; usa `$utils.calcularPrescricao`), **Calculadora de tempo** (soma/subtração de anos/meses/dias em base 360). |
| `_view-avancado.html` | **Atualizações** (botões Índices/Tabelas com relatório HTML retornado), **Gerenciamento de padrões** (editar/excluir registros salvos individualmente), **Preferências** (abertura automática do add-on). |
| `_ui-components.html` | Design system interno: 16 componentes (`ui-icon`, `ui-button`, `ui-icon-button`, `ui-fab`, `ui-button-group`, `ui-modal`, `ui-spinner`, `ui-close`, `ui-dialog` [tipos checkbox/radio/info/error], `ui-toast`, `ui-input-date`, `ui-input-number`, `ui-toolbar`, `ui-sidebar`, `ui-menu`, `ui-tabs`). |
| `_ui-directives.html` | `v-ui-click-outside`, `v-ui-auto-select`, `v-ui-input`, `v-ui-input-label`, `v-ui-flash` (animação com Velocity). |
| `_ui-animations.html` | Transições slide/fade/offcanvas (Velocity). |
| `_estilos.html` | CSS da sidebar (largura 300 px, scrollbar fina, ajustes w3-modal, spinner SVG). |

### 3.3 Infraestrutura de projeto
- ✅ Licença MIT (2017, Contadoria).
- ✅ README com características e passo a passo de instalação (via convite a grupo de usuários + Web Store).
- ❌ **Sem `appsscript.json`** (manifesto) no repo — obrigatório para deploy com `clasp`; sem ele não há declaração formal de timezone, escopos, `addon` config (title, layouts), ou vínculo às libs (`dependencies.libraries` IDs+versões).
- ❌ Sem `.gitignore`, sem CI, sem testes, sem changelog, sem instruções de desenvolvimento (somente instalação).

---

## 4. Como funciona — fluxos principais

### 4.1 Ciclo de vida
1. Usuário instala o add-on (Web Store) → `onInstall` chama `onOpen`.
2. `onOpen` cria item "Abrir" no menu customizado e, conforme preferência salva, abre a sidebar automaticamente.
3. Sidebar renderiza `_index.html`; `vm.obterDados()` dispara `google.script.run.obterDados()` que devolve bootstrap JSON (uma única viagem redonda).

### 4.2 Extração de texto (recurso mais valioso)
1. Usuário cola conteúdo (extrato CNIS, tela do SisJEF, relatórios PLENUS/HISCRE) na textarea.
2. **No cliente**, com debounce de 500 ms, `TextParser.testarOrigem(texto)` identifica a origem e `TextParser.processar(origem, texto, listaIntervalos)` devolve resumo HTML + lista de grupos/opções de destino.
3. Usuário escolhe a interpretação (radio dialog) → `preencher(payload)` no servidor.
4. Servidor roteia: dados simples → `Planilhas.atualizarIntervalo`; tabulares → funções específicas (posicionamento por competência via `Utils.posicaoCompetencia`, offsets fixos linha 5, append em `TCInformado`, criação de página bruta).
5. Resultado (HTML de sucesso) vira toast/dialog na store.

### 4.3 Atualização de índices e tabelas
1. Leitura de `_v_Tipo` → validação contra `Config_.ATUALIZACOES.CONSULTAS[tipo]`.
2. URL base vem de `ScriptProperties` (`INDICES_URL`/`TABELAS_URL`); monta query multi-consulta.
3. `Http_.consultarServicoInterno` espera envelope `{status:'ok', payload:{Consulta:[...]}}`.
4. Para cada consulta: `Planilhas.criarPagina(nome, {sobrescrever:true, manterOculta:true})` a partir de `DadosTemplate` + `atualizarPagina`.
5. Relatório (sucessos × erros) é registrado em log (aba `LogTemplate`) e devolvido como tabela HTML para dialog "Info".
6. Falha isolada por consulta não aborta o lote (try/catch por item) — bom comportamento.

### 4.4 Área de transferência
`copiarDados`: varre `getNamedRanges()` com `_c_*`, captura valor (ou fórmula para `_c_f_*`), empacota com o ID da planilha de origem e grava no UserCache (10 min). `colarDados`: valida origem (mesma planilha = bloqueado; planilha de desenvolvimento = bloqueado), reconverte tipos (número/data ISO) e aplica via `Planilhas.atualizarIntervalo`; remove o cache após colar.

### 4.5 Padrões do usuário
Mapa bidirecional planilha×registro (`obterMapaInfoPadrao`): permite salvar N intervalos `_p_*`/`_p_f_*` selecionados, baixar só os existentes no registro, editar/remover individualmente (aba Avançado). Persistência em `UserProperties` (chave = nome do intervalo).

### 4.6 Prescrição (calculadora)
Campos: prazo (meses; default 60 = art. 103, § único da Lei 8.213/91), termo inicial (pré-lê do intervalo `DER`), interrupção (pré-lê `Citacao`), início/fim de suspensão (ex.: exigência indevida, doença equiparada, decisão liminar). Cálculo delegado a `Utils.calcularPrescricao` (lib externa) → dias suspensos somados ao marco. Pré-preenchimento inteligente a partir da planilha é um diferencial real para petição de impugnação ao cumprimento/cálculo.

### 4.7 Registro estatístico e telemetria de erros
POST ao serviço registrado na própria planilha (preview antes do envio, consentimento em dialog) e POST assíncrono ao `ERROS_URL` central em falhas de sistema. Nota: `Timestamp: ''` é enviado deliberadamente vazio para o servidor preencher — anonimização parcial; ainda assim trafica e-mail do usuário efetivo (ver §6.2).

---

## 5. Qualidade do código (olhar de dev sênior)

### 5.1 Pontos fortes
- Arquitetura modular clara com IIFE de namespace (`Módulo_ = (function(x){...})(Módulo_ || Object.create(null))`) — idiomático para GAS, evita colisão global.
- Envelope uniforme de resposta/erro (`Notificacao_.criar`) separando **erro de usuário** (mensagem formatada em HTML, log local) de **erro de sistema** (mensagem genérica ao usuário, stack no log central) — condizente com LGPD/minimização de exposição.
- JSDoc consistente nos módulos públicos; design system próprio reutilizável; feedback visual completo (loading/disabled/spinner/toast).
- Resiliência no lote de atualizações (falha por consulta não interrompe as demais) e dupla verificação de segurança na área de transferência.
- Validação de tipos na fronteira HTTP↔planilha (`tentarConverterEmData` no reviver do `JSON.parse`).

### 5.2 Defeitos identificados no código deste repo (verificados)

| # | Local | Problema | Severidade |
|---|---|---|---|
| D1 | `AreaTransferencia.gs:77,80` | `throw new Utils.CustomError(...)` **sem `()`** — lança a classe, não a instância; o `catch` de `Notificacao_` tratará como erro de **sistema** (mensagem genérica) em vez de exibir o aviso específico. | Alta |
| D2 | `Preenchimento.gs:134` | `return new Utils.CustomError('...')` em `preencherPeriodos` — retorna o objeto em vez de `throw`; serializa como `{}` e a UI mostra sucesso ambíguo. Também falta `()` aqui. | Alta |
| D3 | `Preenchimento.gs:57-63` | `preencherBeneficioRecebido` usa `Utils.atualizarIntervalo` enquanto todo o resto usa `Planilhas.atualizarIntervalo` — inconsistência provável (Utils não expõe esse método no contrato observado); risco de `TypeError` em runtime. | Média |
| D4 | `Registro.gs:109` | `delete dados[Config_.STORAGE.CHAVE_OPCOES_USUARIO]` — constante **inexistente** (`Config.STORAGE` define `CHAVE_PREFERENCIAS_USUARIO`); a exclusão é no-op e a chave `PREFERENCIAS` pode ser listada como "padrão salvo" se contiver `_p_` no filtro… (na prática filtrada depois, mas bug latente). | Média |
| D5 | `Estatistica.gs:101` | `salvarUrl(planilha, dadosServico.url, dadosServico.nome)` — `consultarServicoInterno` retorna apenas `payload`; campo `nome` provavelmente indefinido → rótulo `undefined` na hiperfórmula. | Baixa |
| D6 | `Atualizacoes.gs:52-64` | `var registro` redeclarado 3× dentro do mesmo callback (hoisting); estilo frágil, embora funcione. | Baixa |
| D7 | `Config.gs:17` | `CORES.SUCESSO = '#FFFFFF'` (branco!) usado em mensagens de sucesso exibidas sobre fundo branco/claro — texto invisível em vários dialogs (herança de época em que o fundo era verde). | Baixa (UX) |
| D8 | `Inicializacao.gs:18` | Precedência `e.authMode == LIMITED || FULL` sem parênteses + acesso a propriedade de `preferencias` sem optional chaining quando JSON é `null` — protegido por `if (preferencias...)`, ok, mas `JSON.parse(null-safe)` quebra se a propriedade corromper (sem try/catch). | Baixa |
| D9 | `*` | **Magia duplicada**: `Preenchimento.gs` redefine localmente `INTERVALOS_ROTULOS/RENDA/ABONO` e `PRIMEIRA_LINHA=5` já parcialmente previstos em `Config.PLANILHA.INTERVALO.ROTULOS_NB` (que nunca é usada); offsets hardcoded (linha 5) espalhados. | Média |
| D10 | `*` | Config declara `PLANILHA.PAGINAS_OCULTAS`, `PREFIXO.LIMPAR`, `STORAGE.ID.INDICES/TABELAS/ERROS`, `STORAGE.PICKER_KEY/CLIENT_ID` — **dead config** (Picker API legada removida do código, chaves órfãs). Confunde manutenção. | Baixa |
| D11 | `*.html` | CDN sem SRI e versões antigas: Vue 2.4.2 (2017), Vuex 2.4.0, Velocity 1.5.0, FontAwesome 4 via endpoint `use.fontawesome.com/6cd8335dee.js` (kit descontinuado — ícones podem sumir); w3css sempre "latest 4". Se o CDN falhar, a sidebar inteira fica branca (sem fallback). | Alta (op.) |
| D12 | Repo | Sem `appsscript.json`, sem lock de versões das libs GAS (`Utils/Planilhas/TextParser` por ID desconhecido), sem testes, sem lint, sem CI, sem docs de dev. Git history = 1 commit ("Update README"). | Alta |
| D13 | `README.md` | Desactualizado: menciona ferramentas que dependem de recursos hoje inexistentes (ex.: Picker) e não documenta arquitetura, deploy ou as libs externas. | Média |
| D14 | Segurança | `colocar` dados de cache por chave fixa `'clipboard'` com TTL 10 min: qualquer add-on do mesmo effective user? Não — UserCache é por usuário, aceitável; porém `MODO_DESENVOLVIMENTO` é a única guarda de logging; `Logger.log` de dados do usuário em `AreaTransferencia.colar` (linha 90) viola minimização LGPD. | Média |

### 5.3 Avaliação quantitativa
- ~1.420 linhas de GS (server) + ~2.470 linhas de HTML/JS (client). Complexidade ciclomática moderada; coesão boa por módulo; acoplamento alto com contratos de nomes de intervalos (indebted ao domínio, mas deveria estar em um único dicionário validado na abertura).

---

## 6. Análise jurídica e de domínio (advogado previdenciário)

### 6.1 Valor forense do estado atual
- **ATRASADOS**: atualização de `OpcoesCorrecao/OpcoesJuros/OpcoesAlcada` cobre diretamente a fase pós-**EC 113/2021** (art. 100: SELIC único índice até a data do cálculo, absorvendo correção + juros) e a transição IPCA-E vs. TR (Tema 905/985 STJ/STF, ERE 1488854-RS, ADI 5.715, decisão do Tema 1.197 STJ sobre modulação) — o jurista precisa escolher índices por período com segurança; a base consolidada (`IndicesConsolidados`) é exatamente isso. **Risco:** se a URL de atualização não for mantida com frequência mínima mensal (expurgo de precatórios, revisões de teto, novos temas), o laudo nasce desatualizado.
- **RMI**: `TabuasMortalidade` + `OpcoesAtividadesConcomitantes` servem à aposentadoria por incapacidade permanente (art. 42 ss., L. 8.213 — fator/pontuatura, esperança de sobrevida IBGE/anexos) e à média de todas as contribuições (art. 26 da EC 103/2019), incluindo a regra de descarte de 20% das menores contribuições (art. 26, §2º) — o módulo de salários importa o CNIS integral, deixando o descarte para a planilha. Correto.
- **TC**: `TabelaPontuacao` atende às regras de transição da EC 103/2019 (arts. 15–20: pontos 96/101 progressivos etc.); `TabelaConversão` cobre conversão de tempo especial (decretos 3.048/99, 4.882/03, Anexo IV e fatores multiplicadores — PPP/perfil profissiográfico); `TabelaCarência` a Tabela B anexa à L. 8.213. Tempo de contribuição é o insumo de quase toda demanda (DIB retroativa, revisão de RMI, benefício programado).
- **Prescrição**: a calculadora com termo inicial (DER) + marco interruptivo (citação, art. 240 CPC; requerimento administrativo como causa de interrupção na jurisprudência — REsp 1.380.978/PE, Tema 813: prescrição das parcelas **anteriores ao quinquênio** da propositação, art. 103 §único L. 8.213 c/c Decreto 20.910/32) é adequada ao uso em contestação/impugnação. Suspensões parametrizadas cobrem hipóteses de força maior/doença e exigibilidade suspensa.
- **Extração CNIS/SisJEF/PLENUS/HISCRE**: elimina a principal fonte de erro material do laudo (digitação de competências e remunerações); o resumo prévio + confirmação explícita do destino é boa prática probatória (auditabilidade do que foi importado).

### 6.2 Lacunas jurídicas relevantes
1. **Trilha de auditoria insuficiente para perícia (art. 473 CPC):** o log local é opcional/oculto e sobrescritível; não há registro imutável de *qual versão de tabela/índice* foi usada em cada cálculo (só `_v_VersaoAtual` da planilha, não do dataset baixado). Recomendar exportação assinada do relatório de atualização (data/hora + hash das tabelas).
2. **Versionamento temporal das tabelas ausente:** `sobrescrever: true` destrói a base anterior — impossível refazer um cálculo antigo com os índices vigentes à época (relevante em modulações de efeitos e revisões da vida toda — Tema 1.133 STF).
3. **Selic e marcos legais mudam por Emenda/tema repetitivo**; nada no add-on avisa defasagem da base (ex.: última atualização > 30 dias → alerta amarelo).
4. **RGPS-only?** As consultas atuais são centradas no INSS/JF. Para RPPS falta suporte específico (ver §7).
5. **Aviso de responsabilidade técnica:** não há disclaimer gerado no ato da atualização lembrando que o laudo exige conferência humana dos marcos (FR, reenquadramento jurídico de períodos rurais/especiais, regime jurídico do servidor).

---

## 7. Perspectiva RPPS (Regime Próprio)

**Diagnóstico: o projeto hoje é ~90% RGPS/Justiça Federal.** O que existe aproveitável e o que falta para RPPS:

| Item | Situação | Ação sugerida |
|---|---|---|
| Correção monetária de atrasados de servidores | Base `IndicesConsolidados` cobre IPCA-E/TR/SELIC usados também em RPPS federais (FONASEF/precatórios) | Validar cobertura de índices estaduais/municipais (LE providos pelas Fazendas) e criar `CONSULTAS` por ente |
| Tempo de contribuição/conversão | `TabelaConversao` (analogia RGPS, Dec. 3.048) não reflete regras UPP/autarquias locais nem EC 103 aplicável aos RPPS (arts. 6º-A, 15–20 c/c art. 1º da EC 103 e LC/leis locais) | Nova tabela de pontuação/idade por ente + paridade/integralidade (art. 40 §§3º,4º e regras de transição) |
| Proventos por incapacidade em RPPS | Tábuas de mortalidade existem (bom) | Adaptar base de cálculo (média desde 07/2004 ou últimas 80% — art. 1º da EC 103 c/c art. 40) |
| Abono de permanência, BC do RPPS, tetos (art. 40, XI) | Ausentes | Incluir `TabelaTeto` e flag de aplicação de teto |
| Certidão de tempo de contribuição (CTC) / averbação entre RPPS-RGPS | Não suportado na extração | Parser de CTC/e-Social/SIAPE seria o divisor de águas para RPPS |
| Prescrição em RPPS | Mesma calculadora serve (Decreto 20.910 + Tema 813) | Adicionar opção "quinqüenal a contar do fato gerador p/ ação anulatória (Súmula 85 STJ)" |

Conclusão: a **plataforma é portável** (contrato por intervalos nomeados + webservice de tabelas parametrizável), mas exige nova matriz `Config_.ATUALIZACOES.CONSULTAS.RPPS` e parsers específicos — esforço estimado em iteração de produto, não em refactor estrutural.

---

## 8. Roadmap de melhorias priorizado

### P0 — Correção imediata (dias)
1. Corrigir D1/D2 (`new Utils.CustomError(...) → throw new Utils.CustomError(...)`; `return → throw` em `preencherPeriodos`).
2. Unificar `Utils.atualizarIntervalo → Planilhas.atualizarIntervalo` (D3) e corrigir chave `CHAVE_OPCOES_USUARIO` (D4).
3. Trocar `CORES.SUCESSO` para cor visível (D7) e remover `Logger.log` de dados pessoais (D14).
4. Publicar `appsscript.json` completo (timezone GMT-3, escopos, `addons.spreadsheets`, `dependencies.libraries` com IDs/versões pinados) e workflow `clasp push/pull` no README.

### P1 — Robustez operacional (semanas)
5. Pin/SRI dos CDNs ou bundle local das libs Vue/Vuex/Velocity; migrar FontAwesome para kit próprio ou SVG embutido; fallback de boot da sidebar.
6. Snapshot versionado das tabelas/índices baixados (aba `_Hist_<Consulta>_<YYYYMM>` em vez de sobrescrever) + hash/UUID da base gravado no log e imprimível no laudo.
7. Alerta de defasagem da base (idade da última atualização, tabela por tabela) na UI.
8. Testes mínimos: harness de `Preenchimento`/`AreaTransferencia` com mocks (`rosetta`/gas-test), golden files para payloads dos parsers; CI com lint (eslint gas rules).
9. Sanitização centralizada de HTML devolvido ao client (mensagens construídas por concatenação hoje confiam no servidor).

### P2 — Produto/domínio (meses)
10. Matriz RPPS (§7): índices por ente, teto, paridade, CTC parser.
11. Exportação do laudo de parametrização (PDF/CSV: quais tabelas/versões/marcos foram usados) — fortifica o art. 473 CPC.
12. Revisar estatística sob LGPD: opt-in explícito por planilha, pseudonimização do e-mail, política de retenção.
13. i18n-zero: extrair textos de UI para catálogo; substituir magic numbers de layout (linha 5, offsets) por metadados lidos da própria planilha (ex.: intervalo `_cfg_FirstRow`).

---

## 9. Matriz de riscos consolidada

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Libs externas (Utils/TextParser/Planilhas) divergirem do repo público | Alta | Alto (produto quebra silenciosamente) | Monorepo ou lock de versões no manifesto + CHANGELOG |
| Webservice de índices desatualizado (novo tema/SELIC) | Média | **Crítico jurídico** (laudo errado) | Alerta de idade da base + hash/version no laudo (P1.6/P1.7) |
| Kit FontAwesome morto / CDN indisponível | Média | Médio (UI degradada) | Bundling local |
| Bugs D1/D2 mascararem erros de usuário como "erro de sistema" | Certa (já ocorre) | Médio | P0.1 |
| Dados pessoais em logs (LGPD art. 6º III) | Média | Médio | P0.3 + revisão de retenção |
| Colar área de transferência corromper tipos (datas BR × ISO) | Baixa | Alto (silencioso) | Golden tests de round-trip de datas |

---

## 10. Conclusão

O **"Cálculos Previdenciários" é um add-on bem arquiteturado no seu contexto** (GAS + Vue SPA em sidebar), com excelente experiência de uso para os fluxos centrais — extração de texto (CNIS/SisJEF/PLENUS/HISCRE), atualização de índices/tabelas, padrões de usuário e calculadoras de prescrição/tempo — e com forte aderência às necessidades do advogado previdenci federal (RGPS/JEF). Seus calcanhares de aquiles são: **(a)** dependências críticas fora do repositório (impossível auditar/computar o núcleo financeiro sem `TextParser/Utils/Planilhas` e o webservice); **(b)** ausência de versionamento/auditoria temporal das bases de índices (o problema mais grave para uso pericial e para modulações de temas repetitivos); **(c)** higiene operacional de deploy (sem manifesto, CDNs soltos, lib legada de ícones) e pequenos bugs objetivos (D1–D5); **(d)** cobertura RPPS incipiente, embora a arquitetura permita estendê-la por configuração. O roadmap P0→P2 acima fecha esses ciclos sem exigir reescrita.

---

### Apêndice A — Contrato JSON esperado dos serviços externos
```
GET {BASE}?consulta=A&consulta=B   →   { "status": "ok" | "erro usuario" | "erro sistema",
                                         "payload": { "A": [[...]], "B": {...} } }   // datas em ISO, reconvertidas no reviver
POST {ESTATISTICA_URL}?teste=true    →   { status, payload: { url, dadosServico: [{rotulo, valor}] } }
POST {ERROS_URL}                     →   form: Timestamp,Versao,Resultado,Descricao,Detalhes,UsuarioAtivo,UsuarioEfetivo,Url
```

### Apêndice B — Endpoints públicos citados
- Instalação: chrome.google.com/webstore (ID `cjeahogdijapbbnndojpidgdneeepnla`)
- Site/documentação viva: contadoria.github.io (privacidade, termos, tutoriais, issues)
