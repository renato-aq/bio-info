# Análises, sugestões, plano e resultado

## 1. Regras gerais

- **Nível de significância:** α = 0,05, fixo no MVP.
- **p-valor na tela:** 3 algarismos significativos, e `p < 0,001` abaixo disso.
- **n mínimo:** grupos com n < 3 aparecem no gráfico, mas ficam fora do teste, com aviso. Se sobrarem menos de 2 grupos válidos, o teste não roda e o resultado mostra só a parte descritiva.
- **Aviso de n pequeno:** qualquer grupo ou tempo com n < 10.
- **Ausentes:** cada análise usa só as linhas com valor nas colunas envolvidas e informa quantas ficaram de fora.
- **Comparações múltiplas:** os pós-testes dentro de uma análise usam correção de Holm. Entre análises diferentes não há correção, mas a tela de resultados avisa quando houver 5 ou mais testes de hipótese na sessão.
- **Escolha paramétrico × não paramétrico:** Shapiro-Wilk em cada grupo (3 ≤ n ≤ 5000). Se todos tiverem p ≥ 0,05, usa o teste paramétrico. Senão, o não paramétrico. O usuário pode forçar a outra opção no plano (`"teste": "parametrico" | "nao_parametrico" | "auto"`).

## 2. Catálogo

Cada análise tem um `tipo`, os `parametros` aceitos e as saídas. Os `parametros` com colunas usam o `id` da coluna (ver [`dados.md`](dados.md)).

### 2.1 `descritiva`: estatística descritiva

- **Parâmetros:** `colunas` (lista; padrão: todas com papel `medida` ou `grupo`), `por_grupo` (opcional).
- **Numéricas:** n, ausentes, média, desvio-padrão, mediana, 1º e 3º quartis, mínimo, máximo e n de censurados.
- **Categóricas:** n e % de cada valor.
- **Saída:** tabelas. Não tem gráfico.

### 2.2 `distribuicao`: distribuição de uma variável numérica

- **Parâmetros:** `variavel`.
- **Cálculo:** histograma (regra de Freedman-Diaconis), boxplot marginal e teste de Shapiro-Wilk.
- **Saída:** figura (histograma + boxplot), tabela de resumo e texto sobre a normalidade.

### 2.3 `frequencias`: frequências de uma variável categórica

- **Parâmetros:** `variavel`, `por_grupo` (opcional, gera barras empilhadas em %).
- **Saída:** barras com n e %, ordenadas por frequência, e tabela.

### 2.4 `comparacao_grupos`: variável numérica entre grupos

- **Parâmetros:** `variavel`, `grupo`, `teste` (`auto` por padrão).

| Situação | Paramétrico | Não paramétrico |
|---|---|---|
| 2 grupos | Teste t de Welch | Mann-Whitney |
| 3+ grupos | ANOVA (Welch se Levene p < 0,05) + pós-teste de Tukey (Games-Howell se Welch) | Kruskal-Wallis + pós-teste de Dunn com Holm |

- **Tamanho de efeito:** d de Cohen (t), r = Z/√n (Mann-Whitney), η² (ANOVA), ε² (Kruskal-Wallis).
- **Saída:** boxplot com pontos sobrepostos, tabela por grupo (n, média ± DP, mediana [Q1–Q3]), tabela do teste e dos pós-testes, e texto.

### 2.5 `correlacao`: duas variáveis numéricas

- **Parâmetros:** `x`, `y`, `metodo` (`auto`, `pearson` ou `spearman`), `por_grupo` (opcional, para colorir os pontos).
- **Regra do `auto`:** Pearson se as duas variáveis passarem no Shapiro-Wilk; senão, Spearman.
- **Saída:** dispersão com reta de regressão e IC 95% da reta, coeficiente (r ou ρ), IC 95%, p, R², equação da reta e n de pares.

### 2.6 `contingencia`: duas variáveis categóricas

- **Parâmetros:** `linha`, `coluna`. Se `linha` for genótipo: `modelo_genetico` (`codominante` (padrão), `dominante` ou `recessivo`).
- **Testes:**
  - Tabela 2×2: qui-quadrado se todas as frequências esperadas forem ≥ 5; senão, exato de Fisher. Calcula a razão de chances (OR) com IC 95%.
  - Tabela maior: qui-quadrado. Se mais de 20% das esperadas forem < 5, o p é calculado por simulação de Monte Carlo (10 mil permutações), com aviso sugerindo o modelo dominante ou recessivo.
- **Tamanho de efeito:** V de Cramér.
- **Saída:** barras empilhadas em %, tabela com n e % por linha, tabela do teste e texto.

### 2.7 `frequencia_genotipica`: SNP (genótipos, alelos e Hardy-Weinberg)

- **Parâmetros:** `variavel` (coluna `genotipo_snp`).
- **Cálculo:**
  - frequências genotípicas (n, %);
  - frequências alélicas, com IC 95% de Wilson; o alelo menos frequente é o alelo menor (MAF);
  - equilíbrio de Hardy-Weinberg pelo **teste exato** (Wigginton, Cutler e Abecasis, 2005), com as contagens observadas e esperadas.
- **Avisos:** mais de 2 alelos na coluna (o SNP não é bialélico) e p de Hardy-Weinberg < 0,05 ("pode indicar erro de genotipagem ou estrutura populacional").
- **Saída:** barras dos genótipos com observado × esperado, tabela de genótipos, tabela de alelos e texto.

### 2.8 `frequencia_alelica`: alelos HLA

- **Parâmetros:** `alelo_1`, `alelo_2` (as duas colunas do mesmo locus), `status` (opcional, coluna homo/heterozigoto), `top` (padrão 15).
- **Contagem:** cada pessoa contribui com 2 alelos. Se `alelo_2` estiver vazio e o `status` for "homozigoto", conta `alelo_1` duas vezes. Se estiver vazio sem `status`, a pessoa fica fora, com aviso. Valores fora do formato HLA (por exemplo, `AMBIGO`) são ausentes.
- **Cálculo:** frequência alélica (contagem / 2N) com IC 95% de Wilson, n de pessoas portadoras e % de homozigotos.
- **Saída:** barras horizontais ordenadas (os `top` mais frequentes + "outros"), tabela completa e texto.

### 2.9 `evolucao_temporal`: medida ao longo do tempo

- **Parâmetros:** `medida` (da tabela longa), `tempos` (padrão: tempos com n ≥ 30% do n do primeiro tempo), `por_grupo` (opcional), `resumo` (`mediana_iqr` (padrão) ou `media_ic95`), `escala_log` (ver observação no fim desta seção).
- **Cálculo:**
  - por tempo (e grupo): n, média, DP, IC 95%, mediana, Q1 e Q3;
  - **sem grupo:** Friedman com os pacientes que têm valor em todos os `tempos`, e Wilcoxon pareado de cada tempo contra o primeiro, com Holm;
  - **com grupo:** em cada tempo, comparação entre grupos (Mann-Whitney ou Kruskal-Wallis), com Holm entre os tempos.
- **Avisos:** n por tempo sempre visível no gráfico; aviso para tempos com n < 10; aviso quando o Friedman usar menos de 50% dos pacientes do primeiro tempo.
- **Saída:** linhas com barras de erro (ou faixa), n abaixo de cada tempo, tabela por tempo e grupo, tabela dos testes e texto.
- Observação: `escala_log` vem ligado por padrão para colunas brutas de carga viral (nome com `carga viral` e sem `log`), porque esses valores variam de dezenas a milhões. Colunas que já estão em log10 ficam em escala linear.

## 3. Sugestões automáticas

`sugerir` recebe os metadados do conjunto preparado e devolve itens de plano com `prioridade` (1 = mais alta) e `categoria`. A tela mostra até 12 sugestões agrupadas por categoria, com "ver mais".

| Condição | Sugestão | Categoria | Prioridade |
|---|---|---|---|
| Sempre | `descritiva` de todas as colunas | Visão geral | 1 |
| Cada coluna `genotipo_snp` | `frequencia_genotipica` | Genética | 1 |
| Cada locus HLA (par de colunas `alelo_hla` com o mesmo prefixo de locus) | `frequencia_alelica` | Genética | 1 |
| Tabela longa existe | `evolucao_temporal` de cada medida, sem grupo | Evolução no tempo | 1 |
| Tabela longa + coluna `genotipo_snp` | `evolucao_temporal` da medida principal por genótipo* | Evolução no tempo | 2 |
| Coluna `genotipo_snp` × medida numérica | `comparacao_grupos` no primeiro tempo e no último tempo com n ≥ 30 | Genética × clínica | 2 |
| Coluna `genotipo_snp` × coluna `binario` (inclui as derivadas por limiar) | `contingencia` | Genética × clínica | 2 |
| Coluna categórica com 2–10 valores × medida numérica | `comparacao_grupos` | Comparações | 3 |
| Pares de medidas numéricas | `correlacao` dos 3 pares com maior \|ρ de Spearman\| (\|ρ\| ≥ 0,3) | Correlações | 3 |
| Coluna categórica (não genótipo) | `frequencias` | Visão geral | 4 |

\* Medida principal: a primeira medida da série temporal que não é percentual (por exemplo, `CARGA VIRAL HIV`, depois `LINF. T CD4`).

Regras para não explodir o número de sugestões:

- Combinações genótipo × medida usam só as medidas principais (no máximo 3 por série).
- Não sugerir análises cujas colunas tenham n válido < 10.
- Não repetir uma análise que já está no plano.

Cada sugestão traz uma `justificativa` curta, por exemplo: "ABCB1 rs1128503 é um genótipo de SNP: veja as frequências e se estão em equilíbrio de Hardy-Weinberg."

## 4. Plano de análise

O plano é a lista de análises que o usuário quer rodar. Hoje é preenchido pelas sugestões e pelo usuário. Na Fase 2, a IA vai gerar **o mesmo formato**.

```json
{
  "versao": 1,
  "itens": [
    {
      "id": "a1b2c3",
      "tipo": "comparacao_grupos",
      "origem": "sugestao",
      "titulo": "CD4 no mês 0 por genótipo ABCB1 rs1128503",
      "parametros": {
        "tabela": "principal",
        "variavel": "linf_t_cd4__mes_0",
        "grupo": "abcb1_rs1128503",
        "teste": "auto"
      },
      "justificativa": "Compara a contagem de CD4 entre os genótipos."
    }
  ]
}
```

- `origem`: `sugestao`, `usuario` ou, na Fase 2, `ia`.
- `tabela`: `principal` ou `longa`.
- **Validação** antes de executar: o `tipo` existe; os `parametros` seguem o esquema do tipo; as colunas existem na tabela; e os tipos de coluna são compatíveis (por exemplo, `variavel` numérica, `grupo` categórico). Um item inválido não roda, e a tela mostra o motivo.
- Os esquemas de parâmetros ficam em JSON Schema no pacote `motor` e são exportados para o TypeScript da interface, para que ambos usem a mesma definição.

## 5. Resultado

Cada execução de um item do plano gera um resultado:

```json
{
  "item_id": "a1b2c3",
  "versao_motor": "0.1.0",
  "figuras": [{ "id": "f1", "plotly": { "data": [], "layout": {} } }],
  "tabelas": [{ "id": "t1", "titulo": "Resumo por grupo", "colunas": ["Grupo", "n", "Mediana [Q1–Q3]"], "linhas": [["CC", 47, "412 [290–580]"]] }],
  "texto": ["A mediana de CD4 no mês 0 foi 412 células/µL no genótipo CC ..."],
  "metodo": "Comparação entre 3 grupos pelo teste de Kruskal-Wallis, escolhido porque os dados não seguiram distribuição normal (Shapiro-Wilk, p < 0,05) ...",
  "avisos": [{ "codigo": "n_pequeno", "texto": "O grupo TT tem só 8 pacientes." }],
  "estatisticas": { "teste": "kruskal_wallis", "estatistica": 6.21, "p": 0.0448, "efeito": { "nome": "epsilon2", "valor": 0.026 } }
}
```

- `figuras` saem do motor no formato JSON do Plotly. A interface aplica por cima o tema (fontes, paleta) e os ajustes do usuário.
- `texto` vem de modelos fixos por tipo de análise, preenchidos com os números. Exemplo para `comparacao_grupos` com 2 grupos: "A mediana de {variavel} foi {m1} em {g1} e {m2} em {g2}. A diferença {foi | não foi} estatisticamente significativa ({teste}, p = {p})."
- `metodo` é um parágrafo pronto para copiar para a seção de métodos, citando o teste, por que foi escolhido, a correção usada e as versões (`motor`, SciPy, statsmodels).
- `estatisticas` guarda os números brutos (com ponto decimal) para os testes automatizados e para a Fase 2.

### 5.1 Ajustes do gráfico (na tela de resultados)

- Título e rótulos dos eixos.
- Paleta: `padrão`, `daltônico-segura` (Okabe-Ito) e `tons de cinza`.
- Mostrar/ocultar pontos individuais (boxplot) e escala log (eixo Y).

Os ajustes ficam guardados junto do resultado no histórico.

### 5.2 Downloads

| Formato | Conteúdo |
|---|---|
| PNG | Figura em 2× (≈ 2400 px de largura), fundo branco |
| SVG | Figura vetorial |
| CSV | Cada tabela, com `;` como separador, vírgula decimal e UTF-8 com BOM (abre direto no Excel em português) |
| Método | Botão "copiar texto do método" |
| Baixar tudo (.zip) | Todas as figuras (PNG e SVG), tabelas (CSV) e um `metodos.txt` com os métodos de todas as análises da sessão |
