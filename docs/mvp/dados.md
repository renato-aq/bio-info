# Dados: leitura, colunas e preparação

Tudo neste documento roda **no navegador**. A planilha original nunca é enviada ao servidor.

## 1. Leitura do arquivo

O worker de leitura usa o SheetJS e devolve, para cada aba, uma **grade de células**:

```json
{
  "aba": "Plan1",
  "linhas": 427,
  "colunas": 209,
  "celulas": [["NOME", "CÓDIGO", "..."], ["...", "PAC 001", "..."]],
  "mescladas": [{ "linha_ini": 1, "col_ini": 87, "linha_fim": 1, "col_fim": 96 }]
}
```

- Datas do Excel chegam como texto ISO (`2025-09-10`). Os demais valores chegam como estão (número ou texto).
- CSV/TSV: o separador é detectado automaticamente (`,`, `;` ou tabulação), com codificação UTF-8 ou Latin-1.
- Linhas e colunas totalmente vazias nas bordas são descartadas.
- Observação de instalação: o pacote `xlsx` no registro npm está desatualizado. O SheetJS deve ser instalado pelo endereço oficial do próprio SheetJS (`cdn.sheetjs.com`), com a versão fixada.

## 2. Estrutura

`detectar_estrutura` sugere, para cada aba, a estrutura abaixo. O usuário pode corrigir tudo na tela **Estrutura**.

### 2.1 Linhas de cabeçalho

- **Linha principal:** a primeira linha em que ≥ 50% das células não vazias são texto e a linha seguinte tem predominância de valores de dados.
- **Linha de grupo** (opcional): a linha imediatamente acima, se tiver células mescladas ou textos esparsos sobre blocos de colunas.
- O grupo é propagado para todas as colunas cobertas pela célula mesclada ou, se não houver mescla, até o próximo grupo.
- Nome final da coluna: `<cabeçalho> · <grupo>` quando há grupo, por exemplo `CARGA VIRAL HIV · 6 MESES`.

### 2.2 Blocos repetidos por tempo (formato largo)

Um conjunto de grupos é marcado como **série temporal** quando:

- os grupos têm o mesmo conjunto de cabeçalhos (com tolerância: ignorar maiúsculas, acentos e espaços);
- e o texto do grupo contém um número com unidade de tempo, pela expressão `(\d+)\s*(m[eê]s(es|ses)?|meses|dias?|semanas?|anos?|h(oras?)?)` (aceita erros comuns como "MESSES").

Saída: unidade (`meses`), valores (`0, 6, 12, 18, 24, 30, 36`) e as medidas repetidas (`CARGA VIRAL HIV`, `LINF. T CD4` ...). Uma medida que não aparece em todos os tempos entra só nos tempos em que existe (por exemplo, `LINF. T DUPLO (-)` só no mês 0).

### 2.3 Junção de abas

- Quando há mais de uma aba com dados, a tela Estrutura pergunta quais usar.
- Para juntar, o usuário escolhe a coluna-chave de cada aba. O site sugere colunas cujo cabeçalho contém `código`, `id`, `amostra` ou `paciente` e cujos valores são únicos em ≥ 95% das linhas.
- Normalização da chave antes de juntar: remover espaços nas pontas, maiúsculas e espaços internos múltiplos (`PAC  001` → `PAC 001`).
- Tipo de junção: à esquerda, a partir da aba principal. Colunas com o mesmo nome recebem o sufixo do nome da aba.
- Resumo mostrado: linhas com correspondência, linhas da principal sem correspondência e linhas da secundária descartadas. Chaves duplicadas na aba secundária são um **erro bloqueante** (o usuário precisa escolher outra chave ou remover as duplicatas na planilha).

## 3. Colunas

`inferir_colunas` devolve uma lista de colunas:

```json
{
  "id": "carga_viral_hiv__6_meses",
  "nome": "CARGA VIRAL HIV · 6 MESES",
  "origem": { "aba": "Plan1", "coluna": "CT" },
  "tipo": "numerico",
  "censurado": true,
  "papel": "medida",
  "tempo": { "serie": "s1", "medida": "CARGA VIRAL HIV", "valor": 6, "unidade": "meses" },
  "n_validos": 334,
  "n_ausentes": 67,
  "valores_estranhos": [{ "valor": "não realizado", "n": 8 }],
  "problemas": ["log_fora_da_faixa"]
}
```

- `id`: nome em minúsculas, sem acentos, com `_` no lugar de símbolos e `__` antes do grupo. É único: colunas repetidas ganham `_2`, `_3`.
- Colunas com cabeçalho repetido dentro da mesma aba recebem o problema `cabecalho_repetido`, e a interface mostra uma marca de "repetida".

### 3.1 Tipos

A inferência considera só os valores não ausentes (seção 4.1). Um tipo é atribuído quando **≥ 90%** dos valores se encaixam. Os outros valores viram `valores_estranhos`, e a interface oferece marcá-los como ausentes.

| Tipo | Regra | Exemplos |
|---|---|---|
| `identificador` | Ver seção 3.2 | nome, prontuário, CPF |
| `chave` | Cabeçalho contém `código`, `id`, `amostra` ou `paciente`, valores únicos em ≥ 95% das linhas, e não é identificador | `PAC 001` |
| `genotipo_snp` | Duas letras de `ACGT`, opcionalmente separadas por `/`: `^[ACGT]/?[ACGT]$` | `CT`, `C/T`, `AA` |
| `alelo_hla` | `^(HLA-)?[A-Z]+\d?\*\d{2,}(:\d{2,})*$` | `A*02`, `DRB1*04`, `B*15:01` |
| `data` | Data do Excel ou texto `dd.mm.aa`, `dd/mm/aaaa` ou `aaaa-mm-dd` | `16.01.18` |
| `numerico` | Número com vírgula ou ponto, ou valor censurado (`<40`, `> 1,6`). Se houver algum censurado, `censurado: true`. | `5,42`, `<40` |
| `categorico` | Até 20 valores distintos, ou distintos ≤ 50% do n | `heterozigoto`, esquema de terapia |
| `texto_livre` | O que sobrar | observações |

Um `categorico` com exatamente 2 valores recebe também `binario: true` (candidato a desfecho).

### 3.2 Identificadores

Uma coluna é `identificador` quando **qualquer** regra abaixo vale:

- O cabeçalho (sem acentos e maiúsculas) contém: `nome`, `paciente` (quando não é chave), `prontuario`, `cpf`, `rg`, `sus`, `cns`, `sigaa`, `matricula`, `nascimento`, `e-mail`/`email`, `telefone`/`celular`, `endereco`, `cep` ou `mae`.
- ≥ 50% dos valores casam com padrões de CPF (`\d{3}\.?\d{3}\.?\d{3}-?\d{2}`), e-mail ou telefone.
- ≥ 80% dos valores parecem nome de pessoa (2 ou mais palavras com inicial maiúscula, aceitando `de`, `da`, `dos`, `das`, `do`, `e` entre elas) e ≥ 90% são únicos.

Regras de tratamento:

- Colunas `identificador` são **removidas antes de qualquer outro passo** e não aparecem em análises, downloads nem no histórico.
- O usuário pode desmarcar uma detecção errada, mas a tela pede confirmação explícita ("Esta coluna não identifica pessoas").
- Colunas `chave` são mantidas, mas os valores são trocados por pseudônimos (`P0001`, `P0002`... na ordem de aparição) logo depois da junção de abas. A correspondência entre código e pseudônimo existe só na memória da página e não é salva.
- Se o usuário escolheu o **modo padrão** e há identificadores, a tela Colunas avisa: "Encontramos colunas que identificam pessoas. Elas serão removidas. Quer mudar para o modo protegido?" O modo só pode mudar neste momento, antes de qualquer análise.

### 3.3 Papel da coluna

| Papel | Uso |
|---|---|
| `medida` | Variável numérica a analisar |
| `grupo` | Variável categórica usada para separar grupos (inclui genótipos) |
| `tempo` | Faz parte de uma série temporal |
| `chave` | Liga linhas entre abas e entre tempos |
| `ignorar` | Fora das análises (o usuário escolhe; por exemplo, `Situação 06/05`, `Extração de DNA`) |

O papel inicial vem do tipo: `numerico` → `medida`; `categorico`, `genotipo_snp` e `alelo_hla` → `grupo`; `data` e `texto_livre` → `ignorar`. O usuário pode mudar.

## 4. Preparação dos dados

A preparação é uma **receita**: uma lista ordenada de passos aplicada sobre os dados lidos. A planilha original não muda. Cada passo devolve um resumo e pode ser desfeito (removido da receita, com a receita reaplicada do zero).

```json
{
  "versao": 1,
  "passos": [
    { "tipo": "remover_colunas", "colunas": ["nome", "prontuario", "sigaa"], "motivo": "identificador" },
    { "tipo": "unificar_ausentes", "marcadores": ["não realizado", "sem dados", "x", "repetir"] },
    { "tipo": "tratar_censurados", "coluna": "*", "modo": "metade_do_limite", "criar_indicador": true },
    { "tipo": "normalizar_genotipos", "colunas": "*" },
    { "tipo": "normalizar_texto", "colunas": "*" },
    { "tipo": "largo_para_longo", "serie": "s1" },
    { "tipo": "derivar_razao", "nova": "razao_cd4_cd8", "numerador": "linf_t_cd4", "denominador": "linf_t_cd8" },
    { "tipo": "derivar_limiar", "nova": "cv_indetectavel", "coluna": "carga_viral_hiv", "operador": "<", "valor": 40 }
  ]
}
```

`"*"` significa "todas as colunas em que o passo se aplica".

### 4.1 Passos

| Passo | O que faz | Padrão |
|---|---|---|
| `remover_colunas` | Remove colunas | Sempre aplicado para identificadores; também usado para colunas com papel `ignorar` |
| `unificar_ausentes` | Transforma marcadores em ausente, sem diferenciar maiúsculas e acentos | Lista padrão: vazio, `-`, `--`, `na`, `n/a`, `nan`, `null`, `nd`, `?`, `não realizado`, `sem dados`, `sem dado`, `não informado`, `não se aplica`. Valores como `X` e `REPETIR` **não** entram por padrão: aparecem como valores estranhos e o usuário decide. |
| `tratar_censurados` | Converte `<L` e `>L` em número | Modos: `metade_do_limite` (padrão para `<`), `limite` e `categoria` (vira categórica: "abaixo do limite" / valor). Em colunas com `log` no nome, "metade" é `L − log10(2)`. `>L` vira `L` nos modos numéricos. Com `criar_indicador`, cria `<coluna>_censurado` (sim/não). |
| `normalizar_genotipos` | `TC` → `CT`, `C/T` → `CT` (letras em ordem alfabética) | Aplicado a todas as colunas `genotipo_snp` |
| `normalizar_texto` | Agrupa variações de maiúsculas, acentos e espaços (`Não informado` = `não informado`), mantendo a grafia mais frequente | Aplicado a todas as colunas `categorico` |
| `converter_datas` | Converte textos de data para data | Aplicado a todas as colunas `data` |
| `largo_para_longo` | Gera a tabela **longa** a partir de uma série temporal: uma linha por chave × tempo, com uma coluna por medida e a coluna `tempo` | Sugerido quando a estrutura detecta uma série |
| `derivar_razao` | Nova coluna = A / B (divisão por zero vira ausente) | Sugerido para pares CD4 e CD8 |
| `derivar_limiar` | Nova coluna sim/não: `coluna <operador> valor`. Valores censurados `<L` com `L ≤ valor` contam como "sim" quando o operador é `<`. | Sugerido para colunas de carga viral com censurados |
| `derivar_modelo_genetico` | A partir de um genótipo: `dominante` (portador do alelo menor: sim/não) ou `recessivo` (homozigoto do alelo menor: sim/não). O alelo menor é calculado nos dados. | Não aplicado por padrão; criado sob demanda pelas análises |
| `marcar_fora_da_faixa` | Transforma em ausente os valores sinalizados que o usuário confirmar | Ver 4.2 |

### 4.2 Valores fora da faixa

São sinalizados (não removidos):

- colunas com `log` no nome: valores > 10;
- colunas com `%` no nome: valores < 0 ou > 100;
- qualquer coluna numérica: valores além de 3 intervalos interquartis acima do 3º quartil ou abaixo do 1º.

A tela Preparação lista os casos por coluna, e o usuário escolhe "manter" ou "marcar como ausente" (que cria um passo `marcar_fora_da_faixa`).

### 4.3 Resumo de cada passo

Todo passo devolve contagens legíveis, por exemplo:

```json
{ "passo": 2, "texto": "68 células viraram ausente em 12 colunas", "detalhes": [{ "coluna": "linf_t_cd4__12_meses", "n": 48 }] }
```

### 4.4 Conjunto de dados preparado

Saída final da preparação:

- **Tabela principal:** uma linha por paciente/amostra.
- **Tabela longa** (se houve `largo_para_longo`): uma linha por paciente × tempo.
- **Metadados:** colunas finais (id, nome, tipo, papel, n válidos), a receita aplicada e a versão do motor.

É esse conjunto (e nunca o arquivo original) que vai para o histórico, no servidor ou no navegador.
