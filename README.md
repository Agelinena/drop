# Drop

Passa senhas, textos (até 16 KiB) e arquivos (até 25 MiB por envio) entre dois dispositivos, cifrados no navegador. O servidor é um relay cego: não vê o código, a chave nem o conteúdo, e nada vai para o disco.

Público e sem conta, por decisão: quem tem o código entra na sala, e mais ninguém.

[![CI](https://github.com/Agelinena/drop/actions/workflows/ci.yml/badge.svg)](https://github.com/Agelinena/drop/actions/workflows/ci.yml) · Licença: [AGPL-3.0](LICENSE).

## Como funciona

1. **Receber** gera um código de 20 símbolos (100 bits) e mostra o código e um QR com o link `https://drop.example.com/#code=…`. O código fica no `#`, que o navegador não envia ao servidor.
2. O navegador passa o código por HKDF-SHA256 e tira dele duas coisas independentes:
   - o **ID da sala**, que vai ao servidor;
   - a **chave AES-256-GCM**, que nunca sai do navegador.
3. **Enviar**, no outro dispositivo, lê o QR ou recebe o código digitado, deriva a mesma chave e entra na sala.
4. Cada mensagem vira um frame binário: `versão | IV | AES-GCM(mensagem)`.
   - Os dados associados amarram o frame à sala e ao lado que o mandou: o servidor não consegue devolvê-lo nem trocá-lo de sala.
   - Textos têm padding por faixas de tamanho.
   - Arquivos vão em pedaços de 64 KiB, sempre cheios, com SHA-256 conferido no destino.
5. O receptor confirma cada item, também cifrado. O emissor só mostra **Entregue e conferido** depois disso.

O protocolo está em [`app/static/assets/js/protocol.js`](app/static/assets/js/protocol.js), e o relay em [`app/server.py`](app/server.py).

### Inverter

Com os dois conectados, qualquer um pode tocar em **Inverter** para trocar quem envia e quem recebe. O outro precisa tocar em **Aceitar e inverter**. Sem resposta em 30 segundos, o pedido cai.

- **Lugar e papel são coisas separadas.** O lugar na sala (quem criou e quem entrou) não muda, e é ele que vai nos dados associados de cada frame. Só o papel na tela troca. O servidor não fica sabendo da inversão: é tudo combinado em mensagens `SWAP` cifradas.
- **Pedido → aceite → confirmação.** Quem pediu só troca ao mandar a confirmação, e o outro só troca ao recebê-la. Como os frames chegam em ordem, os dois nunca ficam enviando ao mesmo tempo.
- **Contador de inversões.** O papel vem da paridade dele. Ao reconectar, os dois trocam o contador, e o maior vale. Assim uma confirmação perdida não deixa os dois com o mesmo papel.
- **Só sem nada em andamento:** com um envio ou recebimento no meio, o pedido é recusado.
- **O que estava na tela some:** os itens recebidos, o texto e os arquivos escolhidos. Os dois lados são avisados antes de aceitar.

### Limites

| O quê | Limite |
|---|---|
| Sessão | 10 minutos a partir da criação, sem renovação |
| Texto | 16 KiB |
| Arquivos | 25 MiB e 20 arquivos por envio; 40 MiB por sessão, somando os dois sentidos (também depois de inverter) |
| Salas | 256 no total, 4 por IP |
| Frame | 66 KiB (WebSocket) |
| Página | No proxy da frente (sugestão: 60 pedidos/min por IP, com rajada de 30; abrir a página faz ~13 pedidos) |

Na tela do receptor, textos somem em 2 minutos e arquivos em 5. Recarregar a página mantém a sessão, mas não os itens recebidos.

## Segurança

- **Rede isolada (recomendado):** o Drop não precisa falar com ninguém. Atrás de um proxy em container, troque a porta publicada por uma rede `internal: true` só dele e do proxy (exemplo em [Subir](#subir)): assim ele não alcança outros containers, a rede local, a internet nem o DNS externo.
- **Container:**
  - read-only, `cap_drop: ALL`, `no-new-privileges` e usuário 10001;
  - código do root e sem permissão de escrita;
  - sem pip na imagem;
  - dependências com hash (`--require-hashes`, só wheels).
- **Página:**
  - tudo servido pelo próprio Drop (fontes e bibliotecas em [`assets/vendor`](app/static/assets/vendor/README.md));
  - CSP `default-src 'none'`, com scripts só de `'self'`;
  - Trusted Types (`innerHTML` bloqueado);
  - COOP, COEP e CORP;
  - WebSocket só da mesma origem.
- **Arquivos recebidos** viram download (`application/octet-stream`). O navegador não os abre dentro do Drop.

O que a criptografia **não** protege: um servidor comprometido pode trocar o `app.js` que entrega e capturar o código e o conteúdo. O navegador, o sistema e o histórico da área de transferência também podem guardar cópias. O código é a credencial da sessão: mostre só para quem vai enviar.

### Texto e quebras de linha

O campo de texto segue a especificação do HTML: quebras `\r\n` viram `\n`. O resto chega byte a byte (emoji, acentos, tabs, aspas, espaços nas pontas). Para manter um arquivo exatamente igual, como um `.env` do Windows, mande como **arquivo**.

## Subir

```bash
git clone https://github.com/Agelinena/drop.git && cd drop
cp .env.example .env   # opcional: só a porta
docker compose up -d --build
```

Na frente, um proxy com HTTPS que repasse WebSocket e mande o IP de quem acessa no `X-Real-IP` (o limite de salas por IP usa esse cabeçalho). Exemplo com Caddy:

```
drop.example.com {
    reverse_proxy 127.0.0.1:8080 {
        header_up X-Real-IP {remote_host}
    }
}
```

Como o `X-Real-IP` é aceito sem conferir, não deixe a porta 8080 aberta para fora: só o proxy deve alcançá-la.

**Isolado atrás de um proxy em container** (ex.: Traefik). Um arquivo por cima do compose, sem porta e numa rede interna em que só o proxy também entra:

```yaml
# drop.local.yml — use com COMPOSE_FILE=docker-compose.yml:drop.local.yml no .env
services:
  drop:
    ports: !reset []
    networks: !override [drop-isolada]
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.drop.rule=Host(`drop.example.com`)"
      - "traefik.http.services.drop.loadbalancer.server.port=8000"
networks:
  drop-isolada:
    external: true   # criada antes com: docker network create --internal drop-isolada
```

O `!reset` e o `!override` precisam do Docker Compose 2.24+. O Traefik já manda o `X-Real-IP`.

### Atualizar

```bash
git pull && docker compose up -d --build
```

## Testes

```bash
# Relay (precisa de starlette, uvicorn, websockets e httpx)
PYTHONPATH=app python -m unittest discover -s tests
# Protocolo e QR (Node 20+)
node --test tests/*.test.mjs
```

## Atualizar dependências

- **Python:** edite o `requirements.in` e rode `pip-compile --generate-hashes --strip-extras requirements.in`.
- **Bibliotecas do navegador:** siga o [`assets/vendor/README.md`](app/static/assets/vendor/README.md).
