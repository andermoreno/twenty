# Fork Twenty CRM — guia de customização e operação

Documentação do fork `andermoreno/twenty`. Este arquivo existe só no fork (não no upstream)
e serve como referência do ambiente local, da estratégia de branches e do fluxo de atualização.

## Estratégia de branches

- `main`: espelha o upstream (`twentyhq/twenty`). Nunca recebe edição manual nem commit de customização.
- `twenty-custom`: branch de produção do fork. Concentra as customizações estáveis. Branch de trabalho padrão.
- `feature/*`: branches temporárias para desenvolvimento pontual.

Regra de ouro: preferir arquivos/módulos novos e o `twenty-sdk` (`defineObject`) a editar o core.
Quanto menos linha do core alterada, mais limpo o merge com o upstream.

## Remotes

- `origin` = https://github.com/andermoreno/twenty (o fork)
- `upstream` = https://github.com/twentyhq/twenty (oficial)

## Fluxo de atualização a partir do upstream (validado)

```bash
# 1. Atualizar main (espelho) a partir do upstream — fast-forward
git checkout main
git fetch upstream
git merge --ff-only upstream/main
git push origin main

# 2. Trazer o upstream para a branch customizada
git checkout twenty-custom
git merge main --no-edit
git push origin twenty-custom

# 3. Refrescar o ambiente (dependencias + schema)
nvm install          # instala a versao fixada no .nvmrc
yarn install
npx nx database:reset twenty-server   # em dev; recria schema e seed
```

Antes de mergear, inspecionar o que vem: `git log --oneline main..upstream/main` e
`git diff --stat main..upstream/main`. Sincronizar com frequência (semanal/quinzenal) para
manter os merges pequenos. Revisar migrations de banco manualmente ao resolver conflitos.

## Ambiente local

A máquina compartilha serviços com o projeto `bedrock-enterprise`.

- Node: fixado no `.nvmrc` (24.16.0). O Twenty exige `^24.5.0`. Rodar `nvm install` uma vez
  para instalar a versao exata; depois `nvm use` funciona ao entrar na pasta.
- Postgres: reusa o container Docker `erp-postgres` (porta 5432, user/senha `postgres`/`postgres`).
  Bancos do Twenty: `default` e `test`.
- Redis: container dedicado `twenty-redis` na porta 6380 (isolado do `erp-redis` do bedrock).
  Costuma ficar parado; reiniciar com `docker start twenty-redis` antes de subir.
- `.env` do server: `PG_DATABASE_URL=postgres://postgres:postgres@localhost:5432/default`
  e `REDIS_URL=redis://localhost:6380`.

### Ajuste de sistema necessário

O modo `--watch` do NestJS estoura o limite de file watchers do kernel. Corrigido de forma
permanente em `/etc/sysctl.d/40-max-user-watches.conf` com `fs.inotify.max_user_watches=524288`.

## Subir a aplicação

```bash
docker start twenty-redis                     # garantir Redis de pe
# tres terminais, cada um com: cd ~/claude-projects/twenty && nvm use
npx nx start twenty-server                     # Terminal 1 (esperar estabilizar)
npx nx worker twenty-server                    # Terminal 2
npx nx start twenty-front                       # Terminal 3
```

- Front: http://localhost:3001 · API/GraphQL: http://localhost:3000/graphql
- Credenciais do seed dev: `tim@apple.dev` / `Applecar2025`

## Banco de dados: seed vs prefill

`database:reset` tem duas configurações:

- padrão (`--configuration=seed`): recria schema + roda `workspace:seed:dev` (dataset demo grande).
- `--configuration=no-seed`: recria schema sem o dataset demo grande.

Atenção: mesmo com `no-seed`, todo workspace novo nasce com ~5 registros de exemplo por objeto
(Airbnb, Anthropic, Stripe, Figma, Notion em Companies, mais People e Opportunities ligadas).
Isso vem do módulo `standard-objects-prefill-data`, que roda na criação de qualquer workspace
(inclusive em produção), e não é controlado pela flag de seed. Para um workspace 100% vazio,
apagar esses registros na interface.
