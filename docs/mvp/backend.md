# Backend: login, histórico e deploy

O servidor é pequeno de propósito: **não processa planilhas**. Ele só faz login, aplica a allowlist e guarda o histórico de 24 h do modo padrão.

## 1. Login e allowlist

- **Provedor:** Supabase Auth com Google. Escopos: só `openid`, `email` e `profile`.
- **Allowlist:** tabela `allowlist` no Postgres. Adicionar ou remover alguém é um `insert`/`delete` nessa tabela, sem novo deploy.

A allowlist é aplicada em 3 camadas, para que uma falha numa delas não libere o acesso:

1. **Criação de conta:** um hook de autenticação do Supabase ("before user created") recusa e-mails fora da allowlist. A conta nem chega a existir. *Confirmar na implementação se o hook está disponível no plano usado; se não estiver, as camadas 2 e 3 bastam.*
2. **Banco:** toda política de acesso (RLS) chama a função `tem_acesso()`, que confere se o e-mail do token está na allowlist **naquele momento**. Remover um e-mail corta o acesso aos dados na hora.
3. **Interface:** o middleware do Next.js confere `tem_acesso()` a cada navegação e manda para a tela de acesso negado (T2).

```sql
create table allowlist (
  email        text primary key check (email = lower(email)),
  adicionado_em timestamptz not null default now()
);

create function tem_acesso() returns boolean
language sql stable security definer set search_path = public as $$
  select exists (select 1 from allowlist where email = lower(auth.jwt() ->> 'email'));
$$;
```

## 2. O que fica guardado em cada modo

| | Modo padrão | Modo protegido |
|---|---|---|
| Arquivo original | Não guardado (nunca sai do navegador) | Não guardado (nunca sai do navegador) |
| Dados preparados (sem identificadores, chaves pseudonimizadas) | Supabase Storage | IndexedDB do navegador |
| Metadados, receita, plano e resultados | Postgres | IndexedDB do navegador |
| Visível em outro computador | Sim | Não |
| Duração | 24 h a partir da criação da sessão | 24 h a partir da criação da sessão |

- Continuar trabalhando numa sessão **não** estende o prazo.
- No modo protegido, a tela Arquivo oferece "Não guardar nem neste navegador": a sessão vive só enquanto a aba estiver aberta.
- Os dados no IndexedDB não são criptografados: quem usar o mesmo perfil do navegador consegue vê-los. A página de Privacidade diz isso.

## 3. Banco (modo padrão)

```sql
create table sessoes (
  id            uuid primary key default gen_random_uuid(),
  usuario_id    uuid not null references auth.users on delete cascade,
  nome_arquivo  text not null,
  etapa         smallint not null,             -- 5 = Análises, 6 = Resultados
  metadados     jsonb not null,                -- colunas, receita, versão do motor
  plano         jsonb not null default '{"versao":1,"itens":[]}',
  dados_path    text,                          -- caminho no Storage
  criada_em     timestamptz not null default now(),
  expira_em     timestamptz not null default now() + interval '24 hours'
);

create table resultados (
  id          uuid primary key default gen_random_uuid(),
  sessao_id   uuid not null references sessoes on delete cascade,
  item_id     text not null,
  item        jsonb not null,                  -- item do plano usado na execução
  resultado   jsonb not null,                  -- ver analises.md, seção 5
  ajustes     jsonb not null default '{}',     -- título, eixos, paleta...
  criado_em   timestamptz not null default now(),
  unique (sessao_id, item_id)
);
```

### Políticas de acesso (RLS)

- `sessoes`: o usuário lê, cria, altera e apaga só as próprias linhas, e só se `tem_acesso()` e `expira_em > now()`.
- `resultados`: as mesmas regras, verificadas pela sessão dona do resultado.
- Como a política filtra `expira_em > now()`, **uma sessão vencida some na hora**, mesmo antes de a limpeza física rodar.
- Limite: no máximo 20 sessões ativas por usuário (verificado no `insert`).

### Storage

- Bucket privado `conjuntos`, com limite de 10 MB por arquivo.
- Caminho: `{usuario_id}/{sessao_id}.json.gz` (dados preparados em JSON por colunas, compactado).
- Política: só o dono (primeira pasta = `auth.uid()`) com `tem_acesso()`.

### Quando salvar

| Momento | O que é salvo |
|---|---|
| "Continuar" na Preparação (T7) | Cria a sessão: metadados + dados preparados |
| Mudança no plano | `plano` (agrupando mudanças seguidas, no máximo 1 gravação a cada 2 s) |
| Análise executada ou gráfico ajustado | Linha em `resultados` |

As etapas 1 a 4 não são salvas: até a preparação terminar, nada vai para o servidor.

## 4. Limpeza de 24 h

- **Edge Function `limpar-expirados`** do Supabase, agendada a cada hora pelo `pg_cron` (com `pg_net`):
  1. busca as sessões com `expira_em <= now()`;
  2. apaga os arquivos correspondentes no Storage pela API do Storage;
  3. apaga as linhas de `sessoes` (os `resultados` caem em cascata).
- **Por que não um cron da Vercel:** no plano gratuito da Vercel, os crons rodam no máximo uma vez por dia.
- **Navegador (modo protegido):** a limpeza do IndexedDB roda ao abrir o site e a cada 10 minutos com a aba aberta. Sessões vencidas nunca são mostradas, mesmo antes de apagadas.
- **Logs:** nenhum log (servidor ou função) registra conteúdo de dados, só ids e contagens.

## 5. IndexedDB (modo protegido)

Banco `bioinfo` no navegador, com as mesmas formas do servidor:

| Store | Chave | Conteúdo |
|---|---|---|
| `sessoes` | `id` | Mesmos campos da tabela `sessoes`, sem `usuario_id`, mais o e-mail de quem criou |
| `resultados` | `[sessao_id, item_id]` | Mesmos campos da tabela `resultados` |
| `conjuntos` | `sessao_id` | Dados preparados |

- A tela Início só mostra as sessões do e-mail logado.
- Ao sair ("Sair"), as sessões protegidas daquele e-mail **continuam** no navegador até vencer. A página de Privacidade diz isso, e T3 tem "excluir" em cada sessão.

## 6. Ambiente de desenvolvimento

- **Supabase local** pela CLI (`supabase start`, requer Docker), com as migrations em `supabase/migrations/`.
- Login Google local: um cliente OAuth de desenvolvimento no Google Cloud, configurado em `supabase/config.toml`.
- O e-mail do desenvolvedor entra na allowlist local por comando SQL documentado no README. **Nenhum e-mail real é commitado no repositório**, que é público.
- Variáveis do Next.js: `NEXT_PUBLIC_SUPABASE_URL` e `NEXT_PUBLIC_SUPABASE_ANON_KEY`. O Next.js **não** usa a chave de serviço do Supabase; só a Edge Function de limpeza usa, e ela já vem configurada no próprio Supabase.

## 7. Deploy (último passo do MVP)

1. Criar o projeto Supabase de produção na região **São Paulo** (`sa-east-1`), aplicar as migrations, publicar a Edge Function e agendar a limpeza.
2. Criar o cliente OAuth de produção no Google Cloud (tela de consentimento com os escopos básicos) e configurá-lo no Supabase.
3. Criar o projeto na Vercel ligado ao repositório do GitHub, com as variáveis de ambiente e a região de funções `gru1` (São Paulo).
4. Cadastrar a URL da Vercel nas URLs de redirecionamento do Supabase e do Google.
5. Inserir os **3 e-mails** na tabela `allowlist` de produção.
6. Rodar os critérios de aceite do [`README.md`](README.md) em produção, com a planilha de referência.
