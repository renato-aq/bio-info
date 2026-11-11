# Especificação do MVP

> Status: **rascunho para revisão**, v1. Data: 11/11/2026. Base: [`../proposta.md`](../proposta.md) (v4).

O MVP é a Fase 1 da proposta: um site, com login Google restrito a 3 e-mails, que lê uma planilha de biologia, ajuda a limpá-la, sugere análises típicas de bioinformática e gera gráficos e estatísticas para download. Ainda não usa IA.

## Documentos

| Arquivo | Conteúdo |
|---|---|
| [`telas.md`](telas.md) | Telas, o que cada uma mostra, ações e estados. Funcional, sem decisões visuais (o design é feito em outra ferramenta). |
| [`dados.md`](dados.md) | Leitura da planilha, detecção de tipos e identificadores, preparação dos dados |
| [`analises.md`](analises.md) | Catálogo de análises, regras de sugestão, formato do plano e do resultado |
| [`backend.md`](backend.md) | Login, allowlist, banco, histórico de 24 h, limpeza automática, deploy |

## Escopo

### Entra no MVP

- Login Google com allowlist de e-mails.
- Upload de `.xlsx`, `.xls`, `.csv` e `.tsv`, com a escolha de modo padrão ou protegido.
- Leitura de cabeçalho em 2 níveis com células mescladas e junção de abas por código.
- Detecção de tipos (número, categoria, data, genótipo de SNP, alelo HLA, valor censurado) e de colunas identificadoras.
- Preparação dos dados: ausentes, censurados, genótipos, texto, formato largo → longo e variáveis derivadas.
- 9 análises (ver [`analises.md`](analises.md)), sugestões automáticas por regras e plano editável.
- Resultados com gráfico interativo, tabela, texto explicativo, descrição do método e avisos.
- Download em PNG, SVG, CSV e "baixar tudo" (.zip).
- Histórico de 24 h: no servidor (modo padrão) ou no navegador (modo protegido).
- Deploy na Vercel como último passo.

### Fica fora do MVP

| Item | Quando |
|---|---|
| IA (pedido em texto livre, explicações geradas) | Fase 2 |
| Regressão logística, forest plot, Kaplan-Meier, diagrama de trocas de terapia | Fase 2 |
| Expressão gênica, dose-resposta, ecologia | Fase 3 |
| Relatório PDF | Fase 3 |
| Celular/tablet (o MVP é só para desktop) | A definir |
| Compartilhar análises entre usuários | A definir |

## Mudança em relação à proposta v4

**A planilha original nunca sai do navegador, em nenhum dos modos.** No modo padrão, o que vai para o servidor é a **versão preparada**: sem as colunas identificadoras e com os códigos de paciente trocados por pseudônimos (`P0001`, `P0002`...). Assim, mesmo se o usuário marcar "não sensível" por engano, nomes e prontuários não chegam ao servidor. A tabela da seção 5 da proposta foi atualizada.

## Arquitetura técnica

```
Navegador
├── Next.js (interface)
├── Web Worker de leitura ── SheetJS: arquivo → grade de células + células mescladas
├── Web Worker do motor ──── Pyodide (Python no navegador) com o pacote `motor`:
│                            estrutura, tipos, preparação, sugestões, análises
│                            usa pandas, numpy, scipy e statsmodels
├── Plotly.js ────────────── desenha as figuras que o motor devolve
└── IndexedDB ────────────── histórico do modo protegido

Servidor
├── Vercel ───────────────── hospeda o Next.js
└── Supabase ─────────────── Auth (Google), Postgres, Storage, Edge Function de limpeza
```

### Por que o motor é em Python

- A estatística vem de bibliotecas validadas (SciPy, statsmodels) em vez de implementações próprias em JavaScript.
- O mesmo código roda no navegador (Pyodide) e no computador do desenvolvedor, então é testado com `pytest` contra valores de referência do R.
- Custo: o Pyodide e os pacotes somam cerca de 15–25 MB no primeiro acesso. O carregamento começa logo após o login, em segundo plano, e o navegador guarda os arquivos em cache para os acessos seguintes.

### Comunicação com o motor

A interface conversa com o motor por mensagens (`postMessage`) com funções de entrada e saída em JSON:

| Função | Entrada | Saída |
|---|---|---|
| `detectar_estrutura` | grade de células de cada aba | abas, linhas de cabeçalho, grupos, blocos repetidos por tempo |
| `inferir_colunas` | estrutura confirmada | lista de colunas com tipo, papel e problemas encontrados |
| `aplicar_preparacao` | receita de preparação | conjunto de dados preparado + resumo do que mudou |
| `sugerir` | metadados do conjunto preparado | lista de análises sugeridas (itens do plano) |
| `executar` | um item do plano | resultado (figura, tabelas, texto, método, avisos) |

Os formatos estão definidos em [`dados.md`](dados.md) e [`analises.md`](analises.md).

### Estrutura do repositório

```
bio-info/
├── app/                  # Next.js (TypeScript)
├── motor/                # pacote Python do motor de análises
│   ├── src/motor/
│   └── tests/            # pytest; valores de referência do R
├── supabase/
│   ├── migrations/       # tabelas, RLS, allowlist
│   └── functions/        # limpeza de 24 h
├── scripts/
│   └── gerar_planilha_sintetica.py
└── docs/
```

## Requisitos não funcionais

| Tema | Requisito |
|---|---|
| Navegadores | Chrome, Edge e Firefox de desktop, versões atuais |
| Tamanho | Até 10 MB, ~5 mil linhas e ~300 colunas por arquivo |
| Desempenho | Motor pronto em até ~15 s no primeiro acesso e ~3 s com cache. Cada análise em até 5 s no tamanho máximo. |
| Interface durante o processamento | Nunca trava: o processamento roda em workers, e a tela mostra o progresso |
| Formatação | Números com vírgula decimal e datas em `dd/mm/aaaa` na interface |
| Acessibilidade | Paleta segura para daltônicos disponível; gráficos não dependem só da cor (formas e rótulos) |
| Privacidade | Página "Privacidade" com o que é guardado, onde e por quanto tempo |

## Testes

- **Motor:** `pytest` com uma planilha sintética que tem a mesma estrutura e os mesmos problemas da referência HIV-EBV, mas dados inventados. Cada análise é comparada com o resultado do R ou do SciPy para os mesmos dados (tolerância de 1e-6).
- **Motor no Pyodide:** teste que roda o motor dentro do Pyodide (Node) para garantir que funciona no navegador.
- **Ponta a ponta:** Playwright com login simulado, cobrindo upload → preparação → análise → download.
- **A planilha real nunca entra nos testes automatizados.** Ela fica só em `docs/references/`, fora do git, para testes manuais.

## Critérios de aceite

O MVP está pronto quando, com a planilha de referência HIV-EBV:

1. Um e-mail fora da allowlist não consegue entrar, e um autorizado entra.
2. As abas `Plan1` e `Resultados SNPS` são lidas, com os dois níveis de cabeçalho juntados ("CARGA VIRAL HIV · 6 meses").
3. Nome, prontuário e Sigaa são detectados como identificadores e removidos, e o código do paciente vira pseudônimo.
4. As abas são unidas pelo código do paciente, com o número de linhas sem correspondência informado.
5. `<40`, `<1,6`, `não realizado`, `sem dados` e `TC`/`CT` são tratados conforme a receita, com o resumo de cada mudança.
6. Os blocos de 0 a 36 meses são convertidos para o formato longo.
7. Sem nenhum pedido, o site sugere ao menos: frequências genotípicas com Hardy-Weinberg de cada SNP, frequências alélicas de HLA, evolução de CD4 e carga viral no tempo e comparação de CD4 por genótipo.
8. Cada análise mostra gráfico, tabela, texto, método e avisos (por exemplo, n pequeno no mês 36).
9. PNG, SVG, CSV e .zip baixam corretamente, e o CSV abre no Excel em português sem ajuste.
10. No modo padrão, a sessão aparece no histórico em outro navegador. No modo protegido, só no mesmo navegador. Em ambos, some após 24 h.
11. Em nenhum momento nomes ou prontuários aparecem em requisições de rede (verificado nas ferramentas de desenvolvedor do navegador).

## Ordem de implementação

| # | Marco | Entrega |
|---|---|---|
| 1 | Esqueleto | Next.js + Supabase local + login Google + allowlist + tela de acesso negado |
| 2 | Motor no navegador | Worker com Pyodide carregando o pacote `motor`; planilha sintética; CI com `pytest` |
| 3 | Leitura | Upload, escolha de modo, estrutura (abas, cabeçalho, junção) |
| 4 | Colunas | Inferência de tipos, identificadores, pseudônimos |
| 5 | Preparação | Receita completa com resumo e desfazer |
| 6 | Análises | As 9 análises com testes de referência |
| 7 | Sugestões e plano | Regras de sugestão, edição do plano |
| 8 | Resultados e exportação | Tela de resultados, ajustes do gráfico, downloads |
| 9 | Histórico | Servidor (padrão), IndexedDB (protegido), limpeza de 24 h |
| 10 | Deploy | Projeto na Vercel, Supabase de produção, Google OAuth de produção, os 3 e-mails na allowlist |
