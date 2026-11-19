# Regras para conversas neste repositório

Fale sempre em português.

## Antes

- `git pull`. Se houver mudança local que não é sua, pare e pergunte.
- Este repositório é **público**: nada de domínio, IP, e-mail ou segredo reais em código, docs ou exemplos. Use `example.com`.
- O que é de um servidor específico (rotas do proxy, redes externas, rótulos de backup) não entra aqui: fica num arquivo de ajuste no repositório do servidor, por cima do `docker-compose.yml`.

## Testes com Docker

Um teste não pode deixar nada para trás nem encostar no que está no ar.

- **Nunca num servidor de produção** (os `container_name` seriam os mesmos).
- **Projeto `teste-drop`:** `docker compose -p teste-drop ...`.
- **Dados em `~/.cache/bergatrix-teste/drop`**, pelas variáveis de pasta do `.env` (`APPDATA_PATH`, `CACHE_PATH`...). Nunca na pasta temporária do Claude (`/tmp/claude-*`): o Docker recria a pasta como root a cada boot e o Claude Code para de funcionar.
- **`restart: "no"`** nos serviços do teste (num arquivo de ajuste do teste, fora do git), **porta alta** (`WEB_PORT=18080`) e **segredos gerados na hora**.
- **Testes rápidos sem Docker:** `PYTHONPATH=app python -m unittest discover -s tests` e `node --test tests/*.test.mjs`.

Exemplo:

```bash
mkdir -p ~/.cache/bergatrix-teste/drop
APPDATA_PATH=~/.cache/bergatrix-teste/drop CACHE_PATH=~/.cache/bergatrix-teste/drop/cache WEB_PORT=18080 \
  docker compose -p teste-drop up -d --build
```

## Depois

1. Limpar o teste:
   ```bash
   docker compose -p teste-drop down -v --remove-orphans --rmi local
   # a pasta pode ter arquivos do root: apague por dentro de um container
   docker run --rm -v ~/.cache/bergatrix-teste:/d alpine:3.20 rm -rf /d/drop
   docker image prune -f
   ```
   Confira com `docker ps -a` que não sobrou nada `teste-*`.
2. Atualize o README e os docs que a mudança deixou errados, no mesmo commit.
3. Diga o que foi e o que não foi testado.

## Commits

- Só com autorização de quem pediu a mudança.
- E-mail pessoal, nunca o do trabalho: confira `git config user.email` contra `git log -3 --format=%ae`.
- Nunca `.env`, chaves (`*.jks`, `key.properties`) nem dados de volume.
