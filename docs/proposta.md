# Bio-Info: proposta do projeto

> Status: **proposta fechada**, v4. Data: 11/11/2026.

## 1. Visão

Um **site** que deixa pesquisadores e estudantes de biologia analisarem planilhas **sem programar**. A pessoa envia um arquivo Excel/CSV, descreve em português o que quer descobrir (ou deixa o site sugerir) e recebe gráficos e estatísticas prontos na tela, com opção de download.

### Problema que resolve

- Muitos biólogos dependem de R/Python ou de ferramentas pagas (GraphPad Prism, Origin) para gráficos e testes estatísticos básicos.
- O Excel faz gráficos simples, mas não faz volcano plot, PCA, heatmap com clusterização nem a escolha do teste estatístico certo.
- Muitas vezes a pessoa nem sabe **que análise faz sentido** para os dados que tem.

### Público e domínios de dados

Fase fechada com **3 usuários autorizados** (e-mails Google definidos depois).

Domínios cobertos, em ordem de prioridade (revista depois da planilha de referência, seção 3):

1. **Coortes clínicas com genética** (pacientes acompanhados no tempo, exames laboratoriais, tratamento, genotipagem de SNPs e HLA). É o caso real de referência.
2. **Expressão gênica** (RNA-seq/qPCR já processados: genes × amostras, log2FC, p-valor)
3. **Ensaios de laboratório** (tratamentos × medidas, dose-resposta, curvas de crescimento)
4. **Ecologia/campo** (contagens de espécies, variáveis ambientais, locais × amostras)

## 2. Decisões já tomadas

| Tema | Decisão |
|---|---|
| Plataforma | Site (web app), não extensão do Chrome |
| Login | Google, restrito a uma allowlist de 3 e-mails |
| Dados sensíveis | O **usuário informa** no upload se os dados são sensíveis, e isso define o fluxo (seção 5) |
| Histórico | Análises ficam disponíveis por **24 horas** e depois são apagadas automaticamente |
| Idioma | Somente português |
| Objetivo de uso | Exploratório: extrair gráficos e estatísticas típicos de bioinformática dos dados, sem uma pergunta de pesquisa fixa. As sugestões automáticas são o caminho principal. |
| Hospedagem | Vercel. O deploy é o **último passo** do MVP; até lá o desenvolvimento é local. |
| E-mails autorizados | Cadastrados na allowlist no momento do deploy na Vercel |

## 3. Caso de referência: planilha do projeto HIV-EBV

Arquivo em `docs/references/` (**fora do git**: contém dados reais de pacientes, ver seção 9).

### O que tem na planilha

| Aba | Conteúdo |
|---|---|
| `Plan1` | ~400 pacientes × ~160 colunas usadas. Identificação (nome, código, prontuário, Sigaa), genotipagem, acompanhamento laboratorial, terapia e HLA. |
| `Resultados SNPS` | ~400 linhas. Genótipos de SNPs farmacogenéticos e controle do laboratório (extração de DNA, reteste). |
| `Plan3` | Vazia |

Blocos de dados da `Plan1`:

- **Acompanhamento no tempo** (meses 0, 6, 12, 18, 24, 30 e 36): carga viral do HIV (bruta e log10), linfócitos T CD4, CD8, duplo-negativos e duplo-positivos (absoluto e %). O número de pacientes cai com o tempo: ~400 no mês 0, ~110 no mês 24 e ~7 no mês 36.
- **Terapia antirretroviral:** até 8 dispensações, com esquema e motivo de troca (tuberculose, gestação, reação adversa etc.).
- **Genotipagem HLA:** locus A, B e DRB1 (dois alelos cada) e status alélico (homo/heterozigoto).
- **SNPs** (aba `Resultados SNPS`): ABCB1 rs1128503, ABCG2 rs2231142, CYP1A2 rs2472677 e rs1800629.

### Problemas de formato encontrados

Esta é uma planilha real típica, e o site precisa lidar com tudo isto:

| Problema | Exemplo | Como o site deve tratar |
|---|---|---|
| Cabeçalho em 2 linhas com células mescladas | Linha 2 diz "MÊS 0", "6 MESES"; linha 3 diz "CARGA VIRAL HIV" | Juntar os dois níveis: `CARGA VIRAL HIV · 6 meses` |
| Formato "largo" (um bloco de colunas por tempo) | Mesmas 6 colunas repetidas para 0, 6, 12... 36 meses | Oferecer conversão para formato "longo" (paciente, tempo, medida), necessária para gráficos de evolução |
| Valores censurados (abaixo do limite de detecção) | `<40`, `<1,6`, `<1.6`, `>40` | Reconhecer como "abaixo/acima do limite" e perguntar como tratar: categoria "indetectável", o próprio limite ou limite/2 |
| Vários marcadores de "sem dado" | vazio, `não realizado`, `sem dados`, `não se aplica`, `Não informado`, `X`, `REPETIR` | Unificar como ausente, mostrando a contagem de cada um |
| Vírgula e ponto decimal misturados | `<1,6` e `<1.6` | Normalizar |
| Datas em formatos mistos | data real do Excel e texto `16.01.18` | Converter tudo para data |
| Genótipos equivalentes escritos diferente | `CT` e `TC` | Normalizar (ordem alfabética ou ordem ref/alt) |
| Texto livre com variações | `Não informado` e `não informado` | Agrupar ignorando maiúsculas/acentos |
| Erros prováveis de digitação | Valores como `1973` na coluna de **log10** da carga viral | Sinalizar valores fora da faixa esperada para revisão |
| Blocos de colunas aparentemente duplicados/desalinhados | Grupo "RESULTADO DA QUANTIFICAÇÃO" repete as colunas de identificação, com genótipos aparecendo sob o título "Origem" | Detectar colunas com cabeçalho repetido e conteúdo incoerente e pedir que o usuário escolha quais usar |
| Colunas de controle do laboratório misturadas com dados | `Situação 06/05`, `Extração de DNA`, `Prioridade` | Permitir excluir colunas da análise |
| **Identificadores diretos de pacientes** | Nome completo, prontuário, Sigaa | Detectar automaticamente, forçar o modo protegido e **remover essas colunas antes de qualquer análise** |

### Análises que fazem sentido para estes dados

| Pergunta de pesquisa provável | Análise | Gráfico |
|---|---|---|
| As frequências genotípicas estão em equilíbrio? | Frequências alélicas e genotípicas, teste de Hardy-Weinberg | Barras de frequência |
| O genótipo influencia a resposta ao tratamento? | Supressão viral (carga < 40) aos 6/12 meses por genótipo: qui-quadrado/Fisher, regressão logística com OR e IC 95% nos modelos codominante, dominante e recessivo | Barras empilhadas, forest plot |
| Como CD4 e carga viral evoluem? | Média ± IC por tempo, comparação entre tempos (Friedman ou modelo misto) | Linhas ao longo do tempo, por grupo |
| O genótipo afeta a recuperação de CD4? | CD4 por genótipo em cada tempo (Kruskal-Wallis) | Boxplot por tempo × genótipo |
| Quanto tempo até a carga ficar indetectável? | Kaplan-Meier, log-rank por genótipo | Curva de sobrevivência |
| Quais alelos HLA são mais comuns? | Frequências alélicas por locus | Barras ordenadas |
| Quais esquemas e trocas de terapia? | Frequência de esquemas e motivos de troca | Barras; diagrama de fluxo (Sankey) das trocas |
| Relação CD4/CD8 | Variável derivada CD4/CD8 e sua evolução | Linhas, boxplot |

## 4. Fluxo do usuário

```
Login com Google (só e-mails autorizados)
        ▼
Upload da planilha
        ▼
"Estes dados são sensíveis?"  ── Sim ──> Modo protegido
        │ Não                                    │
        ▼                                        ▼
Modo padrão                              (ver seção 5)
        └────────────────┬──────────────────────┘
                         ▼
Pré-visualização: abas, cabeçalho, tipo de cada coluna (o usuário corrige)
                         ▼
Preparação dos dados: remover identificadores, unificar "sem dado",
tratar valores censurados (<40), converter formato largo → longo
                         ▼
     ┌───────────────────┴───────────────────┐
O usuário descreve o                Nada foi descrito:
que quer extrair                    o site sugere análises
     └───────────────────┬───────────────────┘
                         ▼
Plano de análise (o usuário confirma ou ajusta)
                         ▼
Gráficos + estatísticas + explicação curta
                         ▼
Download (PNG / SVG / PDF / CSV) e histórico por 24 h
```

Por que cada etapa existe:

1. **Pré-visualização.** Mostra as primeiras linhas e o tipo detectado de cada coluna (numérica, categórica, data, identificador), e o usuário corrige o que estiver errado. Isso evita a maior fonte de erro: planilha mal interpretada.
2. **Preparação dos dados.** A planilha de referência (seção 3) mostra que dados reais quase nunca chegam prontos. O site propõe cada limpeza com um resumo ("23 células `não realizado` serão tratadas como ausentes"), o usuário aceita ou ajusta, e o arquivo original nunca é alterado.
3. **Plano de análise antes de executar.** O site mostra o que vai fazer (por exemplo, "boxplot de `peso` por `tratamento` + ANOVA com pós-teste de Tukey"), e o usuário aprova ou ajusta. Isso dá transparência para quem precisa justificar a metodologia.
4. **Explicação em linguagem simples.** Cada resultado vem com 2 ou 3 frases interpretando o que foi encontrado.
5. **Reprodutibilidade.** O download inclui um resumo do método (teste, parâmetros, versão da ferramenta) para a seção de métodos de artigo ou TCC.

## 5. Dados sensíveis: dois modos de trabalho

No upload, o usuário marca se os dados são sensíveis (pacientes, pesquisa não publicada, dados sob LGPD). A escolha aparece sempre visível durante a análise e não pode ser trocada no meio. Para mudar, é preciso enviar o arquivo de novo.

| | Modo padrão | Modo protegido (sensível) |
|---|---|---|
| Onde a planilha é processada | No navegador | No navegador |
| A planilha original vai para o servidor? | **Não, nunca** | **Não, nunca** |
| Os dados preparados (sem identificadores, códigos pseudonimizados) vão para o servidor? | Sim, para o histórico de 24 h | **Não, nunca** |
| O que a IA recebe | Nomes e tipos das colunas, estatísticas resumidas e até ~20 linhas de amostra | **Só nomes e tipos das colunas** e contagens (n de linhas, n de valores ausentes). Nenhum valor das células. |
| Histórico de 24 h | No servidor, acessível de qualquer computador | **Só no navegador** (IndexedDB), apagado após 24 h. Some se o usuário trocar de computador ou limpar os dados do site. |
| Explicação dos resultados pela IA | Recebe os resultados agregados (médias, p-valores) | Gerada por modelos de texto locais (templates), sem enviar resultados à IA |

Observações:

- Em ambos os modos, **os cálculos e os gráficos rodam no navegador**. O servidor nunca processa dados, só guarda a versão preparada (no modo padrão).
- No modo protegido, até nomes de colunas podem ser identificáveis (por exemplo, `nome_paciente`). O site avisa sobre isso e permite renomear colunas antes de usar a IA, ou desligar a IA e usar só as sugestões por regras.

## 6. Funcionalidades

### 6.1 Entrada de dados

- Formatos: `.xlsx`, `.xls`, `.csv`, `.tsv`.
- Seleção de aba e da linha de cabeçalho, incluindo **cabeçalho em 2 níveis com células mescladas**.
- Detecção automática de tipos, com correção manual. Tipos específicos de biologia: genótipo de SNP (`CT`), alelo HLA (`A*02`), valor censurado (`<40`).
- Detecção de **colunas identificadoras** (nome, prontuário, CPF, código de registro), que são removidas antes da análise.
- Vírgula decimal (padrão brasileiro) e células vazias.
- Junção de abas pelo código do paciente (por exemplo, `Plan1` + `Resultados SNPS`).
- Tamanho esperado: centenas a poucos milhares de linhas e até ~300 colunas. Limite inicial: 10 MB.

### 6.2 Preparação dos dados

Cada passo mostra o que vai mudar e pode ser desfeito:

- Unificar marcadores de ausente (`não realizado`, `sem dados`, `X` etc.).
- Valores censurados (`<40`): tratar como categoria, como o limite ou como limite/2.
- Normalizar genótipos (`TC` → `CT`) e texto livre (maiúsculas, acentos).
- Converter formato largo → longo (blocos repetidos por tempo).
- Criar variáveis derivadas: razão CD4/CD8, "carga viral indetectável" (sim/não), modelos genéticos (dominante/recessivo).
- Sinalizar valores fora da faixa esperada (por exemplo, log10 > 8).

### 6.3 Análises guiadas por texto

Exemplos:

- "O genótipo do ABCB1 influencia a supressão viral aos 12 meses?"
- "Mostre a evolução do CD4 ao longo do tempo, separado por genótipo do CYP1A2."
- "Quais os alelos HLA-B mais frequentes?"
- "Quais genes estão mais expressos no grupo tratado?"
- "Compare o crescimento das plantas entre os tratamentos."

### 6.4 Sugestões automáticas (quando nada é descrito)

Primeiro por regras baseadas na estrutura dos dados, depois refinadas pela IA (respeitando o modo escolhido):

| Domínio | Estrutura detectada | Sugestões |
|---|---|---|
| Clínico + genética | Coluna de genótipo de SNP | Frequências alélicas/genotípicas, Hardy-Weinberg |
| Clínico + genética | Genótipo + desfecho (sim/não) | Qui-quadrado/Fisher, regressão logística com OR e IC 95% (modelos codominante, dominante e recessivo), forest plot |
| Clínico + genética | Genótipo + medida numérica | Boxplot por genótipo, Kruskal-Wallis/ANOVA |
| Clínico | Blocos de colunas por tempo (0, 6, 12... meses) | Evolução média ± IC, comparação entre tempos, linhas por grupo |
| Clínico | Alelos HLA | Frequências alélicas por locus, homo/heterozigose |
| Clínico | Esquemas de tratamento em sequência | Frequência de esquemas, motivos de troca, diagrama de fluxo |
| Expressão gênica | Colunas `log2FC` e `p-value`/`padj` | Volcano plot, lista de genes significativos |
| Expressão gênica | Matriz genes × amostras | Heatmap com clusterização, PCA das amostras |
| Laboratório | 1 numérica + 1 categórica (grupos) | Boxplot/violino, teste t ou ANOVA (Mann-Whitney/Kruskal-Wallis sem normalidade) |
| Laboratório | Concentração + resposta | Curva dose-resposta (IC50/EC50) |
| Laboratório | Tempo + medida por grupo | Curva de crescimento, média ± desvio |
| Ecologia | Matriz locais × espécies (contagens) | Riqueza, índice de Shannon/Simpson, barras empilhadas de composição |
| Ecologia | 2 variáveis numéricas | Dispersão com regressão, correlação de Pearson/Spearman |
| Clínico | Tempo até evento + status | Kaplan-Meier, teste log-rank |
| Clínico | Só categóricas | Tabela de contingência, qui-quadrado/Fisher |

Sempre mostrar também a **estatística descritiva** (média, mediana, desvio, n por grupo, valores ausentes).

### 6.5 Visualização e exportação

- Gráficos interativos (zoom, hover com valores, legenda clicável).
- Ajustes: título, rótulos dos eixos, paleta de cores (incluindo paleta segura para daltônicos).
- Download: **PNG** em alta resolução (slides), **SVG/PDF** vetorial (artigos), **CSV** dos resultados e relatório PDF com gráficos e método.

### 6.6 Autenticação

- **Login com Google** (OAuth).
- Allowlist de 3 e-mails verificada **no servidor**: um e-mail fora da lista nem chega a ter conta criada e vê uma mensagem de "acesso não autorizado".
- Sem senha para gerenciar e sem fluxo de recuperação de senha.
- Para adicionar ou remover alguém, basta editar a allowlist.

### 6.7 Histórico de 24 horas

- Lista das análises das últimas 24 h, com o nome do arquivo, data e hora, modo (padrão/protegido) e gráficos gerados.
- Ao abrir uma análise do histórico, o usuário pode continuar (pedir novos gráficos) ou baixar de novo.
- Expiração automática: um job agendado apaga do servidor tudo o que tiver mais de 24 h (arquivo, plano e resultados). No modo protegido, o próprio site apaga do navegador ao abrir.
- A tela mostra quanto tempo falta para cada análise expirar.

## 7. Arquitetura

```
┌──────────────────── Navegador ─────────────────────────────┐
│  Leitura da planilha (SheetJS)                             │
│  Cálculos estatísticos e gráficos                          │
│  Histórico do modo protegido (IndexedDB, 24 h)             │
└───────────────┬────────────────────────────────────────────┘
                │ modo padrão: arquivo + plano + resultados
                │ ambos os modos: contexto para a IA
                │ (o conteúdo depende do modo, ver seção 5)
                ▼
┌──────────────────── Servidor ──────────────────────────────┐
│  Login Google + allowlist                                  │
│  Endpoint de IA: devolve um "plano de análise" em JSON     │
│  Banco + armazenamento do histórico (modo padrão)          │
│  Job de limpeza a cada hora (apaga o que passou de 24 h)   │
│  Chave da API de IA (nunca vai para o navegador)           │
└────────────────────────────────────────────────────────────┘
```

### Princípios

1. **Cálculos no navegador, sempre.** O mesmo código serve os dois modos, o servidor fica simples e barato, e o modo protegido sai "de graça".
2. **A IA devolve um plano em JSON, não código.** O site só executa análises de uma lista fechada e testada. A IA escolhe *qual* análise e *com quais colunas*, mas nunca inventa estatística.
3. **Chave da IA só no servidor**, e toda chamada exige usuário autenticado e autorizado.

### Stack sugerida (a validar)

| Camada | Opção | Motivo |
|---|---|---|
| Front-end | Next.js (React + TypeScript) | Padrão de mercado, fácil de hospedar |
| Hospedagem | Vercel | Deploy automático a cada push, plano gratuito suficiente para 3 usuários |
| Auth + banco + arquivos | Supabase (Auth com Google, Postgres, Storage, pg_cron) | Resolve login, histórico e a limpeza de 24 h num só serviço. A allowlist é aplicada por hook de criação de usuário e por políticas de acesso (RLS). |
| Leitura de planilhas | SheetJS | Padrão para xlsx/xls/csv no navegador |
| Gráficos | Plotly.js | Já tem volcano, heatmap, violino e Kaplan-Meier fáceis, e exporta PNG/SVG |
| Estatística | Pyodide (Python + SciPy/statsmodels no navegador) **ou** bibliotecas JS (simple-statistics, jStat) | Pyodide dá estatística confiável e já validada (SciPy), mas carrega ~10 MB+ na primeira vez. JS é leve, mas não cobre PCA, Kaplan-Meier e dose-resposta prontos. **Recomendo Pyodide**, carregado em segundo plano enquanto o usuário faz o upload. |
| IA | API de LLM (por exemplo, Claude) com saída estruturada (JSON schema) | Garante que o plano venha num formato válido |

## 8. Fases

### Fase 0: Planejamento (atual)
- Proposta fechada. Especificação do MVP em [`mvp/`](mvp/README.md).

### Fase 1: MVP (critério de pronto: analisar a planilha de referência de ponta a ponta)
- Login Google com allowlist.
- Upload, escolha de modo (sensível ou não), pré-visualização com tipos.
- Cabeçalho em 2 níveis, detecção e remoção de identificadores, junção de abas.
- Preparação dos dados: ausentes, censurados, genótipos, formato largo → longo, variáveis derivadas.
- Sugestões por regras (sem IA ainda).
- Análises: estatística descritiva; frequências genotípicas + Hardy-Weinberg; genótipo × desfecho (qui-quadrado/Fisher); boxplot + Kruskal-Wallis/ANOVA; evolução no tempo; frequências HLA; dispersão + correlação.
- Histórico de 24 h nos dois modos.
- Download PNG/SVG/CSV.
- **Último passo:** deploy na Vercel e cadastro dos 3 e-mails na allowlist.

### Fase 2: IA e clínica avançada
- Pedido em texto livre → plano de análise via IA, respeitando o modo.
- Explicação dos resultados em linguagem simples.
- Sugestões refinadas pela IA.
- Regressão logística com OR/IC 95% e forest plot, Kaplan-Meier + log-rank, diagrama de trocas de terapia.

### Fase 3: Outros domínios
- Expressão gênica: volcano, heatmap com clusterização, PCA.
- Laboratório: dose-resposta, curvas de crescimento.
- Ecologia: índices de diversidade, composição.
- Relatório PDF com método.

## 9. Riscos e cuidados

- **Estatística errada é pior que nenhuma.** Testar cada análise contra resultados de referência (R/SciPy) e verificar os pressupostos (normalidade, n mínimo) antes de rodar o teste.
- **Planilhas reais são bagunçadas:** células mescladas, cabeçalhos em várias linhas, unidades no meio dos números ("12 mg"). A pré-visualização detecta e avisa sobre esses casos.
- **Erro na marcação de sensível:** se o usuário marcar "não sensível" para dados de pacientes, o arquivo vai para o servidor. Mitigação: detectar colunas suspeitas (CPF, nome, data de nascimento, e-mail) e perguntar de novo.
- **Alucinação da IA:** a IA só escolhe entre análises existentes, e o plano é sempre mostrado antes de executar.
- **Custo da IA:** limite de requisições por usuário por dia.
- **LGPD:** mesmo com 3 usuários, ter uma página curta de privacidade dizendo o que é guardado, onde e por quanto tempo. Dados de saúde (como sorologia para HIV) são "dados pessoais sensíveis" na LGPD.
- **Planilha de referência no repositório:** o repositório no GitHub é **público** e a planilha tem nomes e prontuários de pacientes. Ela está bloqueada no `.gitignore` (`docs/references/*` e qualquer `.xlsx/.xls/.csv/.tsv`) e **nunca deve ser commitada**. Para testes automatizados, usar uma **planilha sintética** com a mesma estrutura e os mesmos problemas de formato, mas com dados inventados.
- **Pressupostos estatísticos em dados reais:** com o n caindo de ~400 (mês 0) para ~7 (mês 36), comparações nos tempos finais não têm poder estatístico. O site deve avisar quando o n de um grupo for pequeno demais.

## 10. Perguntas em aberto

Nenhuma bloqueando. Pendente apenas para o deploy:

- Os 3 e-mails Google autorizados.
