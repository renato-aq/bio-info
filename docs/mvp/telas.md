# Telas

Especificação **funcional**: o que cada tela mostra, as ações e os estados. Layout, cores e componentes visuais ficam para o design.

## Elementos globais (todas as telas depois do login)

- **Cabeçalho:** nome do site, link "Histórico", link "Privacidade", foto/e-mail do usuário e "Sair".
- **Indicador do motor:** enquanto o Pyodide carrega após o login, mostra "Preparando o motor de análise..." com progresso. As etapas que dependem do motor ficam bloqueadas até ele terminar.
- **Dentro de uma sessão de análise:**
  - **Etapas** (navegáveis para trás): 1 Arquivo → 2 Estrutura → 3 Colunas → 4 Preparação → 5 Análises → 6 Resultados.
  - **Selo do modo:** "Modo padrão" ou "Modo protegido", sempre visível, com uma explicação curta ao passar o mouse.
  - **Expiração:** "Disponível por mais 23 h 40 min".
- **Voltar a uma etapa anterior** e mudar algo invalida os resultados das etapas seguintes. Antes, a tela confirma: "Isso vai refazer a preparação e apagar 4 resultados desta sessão."

## T1. Login

- **Conteúdo:** nome e descrição curta do site; botão "Entrar com Google".
- **Ação:** login Google → volta para T2.
- **Estados:** carregando (após o clique); erro de login ("Não foi possível entrar. Tente de novo.").

## T2. Acesso negado

- **Quando:** o e-mail Google não está na allowlist.
- **Conteúdo:** "Este e-mail ({email}) não tem acesso. O Bio-Info está em fase fechada." Botão "Entrar com outra conta".
- **Regra:** nenhuma outra tela é acessível; a sessão Google é encerrada no site.

## T3. Início

- **Conteúdo:**
  - Botão principal "Nova análise".
  - Lista das sessões das últimas 24 h, juntando as do servidor (modo padrão) e as deste navegador (modo protegido). Cada item mostra: nome do arquivo, modo, quando foi criada, tempo restante, número de análises e etapa em que parou.
- **Ações:** abrir uma sessão (vai para a etapa em que parou, ver regra abaixo); excluir uma sessão (com confirmação).
- **Estados:** vazio ("Nenhuma análise nas últimas 24 horas"); carregando; erro ao buscar o histórico do servidor (as sessões do navegador ainda aparecem).
- **Regra ao reabrir:** o histórico guarda os dados já preparados, não o arquivo original. Uma sessão reaberta abre na etapa 5 ou 6. Para mudar a estrutura, as colunas ou a preparação é preciso enviar o arquivo de novo.
- **Nota:** sessões do modo protegido só aparecem no navegador onde foram criadas. A lista explica isso numa linha.

## T4. Arquivo (etapa 1)

- **Conteúdo:**
  - Área de arrastar e soltar + botão "Escolher arquivo". Formatos aceitos e limite (10 MB).
  - Pergunta obrigatória, sem opção marcada por padrão: **"Esta planilha tem dados sensíveis?"**
    - "Sim: dados de pacientes ou pessoas, ou pesquisa não publicada" → modo protegido.
    - "Não" → modo padrão.
  - Só no modo protegido: opção "Não guardar nem neste navegador" (a sessão some ao fechar a aba).
  - Texto curto comparando os modos: o que fica guardado e onde (ver [`backend.md`](backend.md)).
  - Dica de formato com um exemplo de planilha "uma linha por paciente/amostra".
- **Ação:** "Continuar" (habilitado com arquivo + resposta) → lê o arquivo → T5.
- **Estados:** lendo o arquivo (progresso); erros: formato não aceito, arquivo maior que 10 MB, arquivo protegido por senha, arquivo vazio ou corrompido.

## T5. Estrutura (etapa 2)

- **Conteúdo:**
  - Lista de abas com n de linhas e colunas, para escolher quais usar (abas vazias vêm desmarcadas).
  - Para cada aba escolhida: prévia das primeiras ~15 linhas com a linha de cabeçalho e a de grupo destacadas; controles para mudar essas linhas.
  - Séries temporais detectadas: "Encontramos medidas repetidas em 7 tempos (0, 6, 12, 18, 24, 30, 36 meses): carga viral, CD4, CD8..." com opção de desfazer a detecção.
  - Com 2+ abas: escolha da aba principal e da coluna-chave de cada aba (pré-selecionadas pela sugestão).
- **Ação:** "Continuar" → aplica a junção e mostra o resumo ("398 de 401 pacientes encontrados em Resultados SNPS; 5 linhas dessa aba sem correspondência") → T6.
- **Estados e erros:** chave com valores duplicados na aba secundária (bloqueia, lista exemplos); nenhuma correspondência entre as chaves (bloqueia).

## T6. Colunas (etapa 3)

- **Conteúdo:**
  - **Aviso de identificadores** no topo, se houver: lista das colunas detectadas, que serão removidas. No modo padrão, oferece "Mudar para o modo protegido".
  - Tabela de colunas: nome (com o grupo), tipo detectado (editável), papel (editável), n válidos, n ausentes, marca "repetida", ícone de problemas e uma amostra de valores. **Colunas identificadoras não mostram amostra de valores.**
  - Filtros: por tipo, por papel, "só com problemas", busca por nome.
  - Seleção múltipla para mudar papel em lote (por exemplo, marcar várias como "ignorar").
- **Ações:** desmarcar um identificador (pede confirmação explícita); "Continuar" → T7.
- **Estados:** ~300 colunas precisam caber com rolagem e filtros sem travar.

## T7. Preparação (etapa 4)

- **Conteúdo:** a receita como uma lista de cartões, um por passo, na ordem de aplicação. Cada cartão mostra:
  - o nome do passo e o resumo do efeito ("68 células viraram ausente em 12 colunas");
  - "ver detalhes" (por coluna);
  - opções do passo (por exemplo, o modo dos censurados: metade do limite, limite ou categoria);
  - remover o passo.
- **Seções especiais:**
  - **Valores estranhos:** por coluna, os valores que não se encaixam no tipo (`X`, `REPETIR`), com "marcar como ausente" ou "manter".
  - **Fora da faixa:** casos sinalizados, com "manter" ou "marcar como ausente".
  - **Variáveis derivadas sugeridas:** razão CD4/CD8, "carga viral indetectável (< 40)", com "adicionar". Botão "criar variável" para fazer manualmente (razão ou limiar).
- **Prévia:** tabela com as primeiras linhas do resultado, alternando entre a tabela principal e a longa.
- **Ação:** "Continuar" → T8.
- **Estado:** reaplicando a receita (progresso).

## T8. Análises (etapa 5)

- **Conteúdo:**
  - **Sugestões**, agrupadas por categoria (Visão geral, Genética, Evolução no tempo, Genética × clínica, Comparações, Correlações), com título, justificativa e botão "adicionar ao plano". "Ver mais" em cada categoria.
  - **Criar análise manualmente:** escolher o tipo; o formulário mostra os parâmetros do tipo, e cada seletor de coluna lista só as colunas compatíveis.
  - **Plano:** lista dos itens escolhidos, com título, tipo e parâmetros; editar, remover e reordenar.
- **Ação:** "Rodar análises" → executa os itens do plano em ordem → T9. Itens já executados e não alterados não rodam de novo.
- **Estados:** item inválido (mostra o motivo e não deixa rodar até corrigir); plano vazio (botão desabilitado).

## T9. Resultados (etapa 6)

- **Conteúdo:** um cartão por análise, na ordem do plano. Cada cartão tem:
  - título e o tipo de análise;
  - **avisos** em destaque (n pequeno, Hardy-Weinberg fora do equilíbrio, Monte Carlo usado...);
  - **gráfico** interativo (zoom, valores ao passar o mouse, legenda clicável);
  - **texto** explicativo;
  - **tabelas** (abas, se houver mais de uma);
  - **método** (recolhido por padrão) com "copiar";
  - ações: ajustar gráfico (título, eixos, paleta, pontos, escala log), baixar PNG/SVG, baixar CSV, editar parâmetros (volta para T8 com o item aberto) e remover.
- **No topo:** "Baixar tudo (.zip)", "Adicionar análises" (volta para T8) e o aviso de comparações múltiplas quando a sessão tiver 5 ou mais testes.
- **Estados por cartão:** rodando (com o lugar do cartão já reservado); erro na execução (mensagem + "tentar de novo"); concluído.

## T10. Privacidade

Texto fixo: o que é guardado em cada modo, onde, por quanto tempo, quem tem acesso e o que **nunca** sai do navegador (o arquivo original e as colunas identificadoras).

## Mensagens de erro gerais

| Situação | Mensagem |
|---|---|
| Sem conexão (modo padrão) | "Sem conexão. Sua análise continua aqui, mas não será salva no histórico até a conexão voltar." |
| Motor falhou ao carregar | "Não foi possível carregar o motor de análise. Recarregue a página." |
| Sessão de login expirou | Leva ao login e volta para a mesma tela depois. |
| Erro inesperado no motor | "Algo deu errado nesta análise." + código do erro para relatar. Os outros cartões continuam. |
