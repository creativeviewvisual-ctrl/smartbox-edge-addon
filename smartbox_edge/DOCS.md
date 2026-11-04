# SmartBox Edge — operação offline do IoT

Add-on do Home Assistant que roda **no próprio Raspberry** e mantém a **porta (palma) funcionando mesmo sem internet**. Ele baixa periodicamente um *snapshot* de autorização da nuvem (`GET /edge/snapshot`), cacheia local, e quando a internet cai decide a abertura da palma sozinho (**fail-safe** — abre com o último snapshot conhecido, respeitando a validade da estadia).

> Ver o desenho completo no Roadmap → "Operação OFFLINE do IoT". Backend: `services/api/src/routes/edge.ts`. Fatias 1–3 prontas; luzes offline (Fatia 2b/4) é a próxima.

## Como funciona (resumo)

```
ESP do sensor  ──HTTP──►  Edge (Raspberry, :8377)  ──HTTPS──►  Nuvem (Render)
   palma                    decide local se offline              /edge/snapshot
                            (cache + fail-safe)                   /edge/logs
```

- **Online:** o edge repassa o `recognize` pra nuvem (decisão fresca; a nuvem loga).
- **Offline:** o edge decide pelo cache e **enfileira o log** pra subir quando reconectar.
- A porta pelo **app** continua via MQTT/HA como hoje (não muda).

## Acesso técnico (0.5.0 · sem prazo na 0.5.1)

A palma do **técnico** não se valida por estadia — a autoridade dele é a autorização que o admin
concedeu (migrations 0244/0248), com prazo ou, desde a 0288, **sem prazo** (até revogar). Até a
0.4.0 o edge só sabia decidir por ocupação, então a palma do prestador caía em `sem_ocupacao` e a
porta não abria em prédio com edge. O snapshot agora traz `acessosTecnicos[]` e a credencial carrega
a âncora (`acessoTecnicoId` + `validoAte`).

Regras, espelhando a nuvem: confere a linha **âncora** (não "alguma autorização que cobre a
unidade" — com duas vigentes isso daria falso negativo); prazo vencido **nega mesmo offline**;
`validoAte`/`expiraEm` **null = sem prazo** (0.5.1): não vence pelo relógio, só a revogação encerra
(a autorização some do snapshot) — e por isso esse ramo tem **teto de 48 h de idade do snapshot**:
sem sync com a nuvem por mais que isso, o técnico sem prazo é negado offline (o hóspede segue
fail-safe com dado velho; com prazo o teto era o próprio prazo). No cache do ESP a chave
`validoAte` é **omitida** (não `null`) — caminho que o firmware já trata. Credencial **sem o
campo** (snapshot de API antiga) nega, e a nuvem segue autorizando pelo caminho dela. ⚠️ Edge
**0.5.0** ainda nega o técnico sem prazo — rebuild obrigatório pra esse acesso valer offline.

## OTA por MQTT (0.4.0) — atualizar o firmware do prédio inteiro

O edge também distribui firmware. Ele varre a pasta compartilhada do Raspberry, descobre a
quem cada `.bin` se destina **pelo nome do arquivo** e anuncia num tópico MQTT retido por
variante; cada ESP assina só o da própria família+placa, confere o MD5, grava e reinicia.

```
\\<ip>\share\smartbox-ota\luzes-esp30-V21.bin
        │
        ├─ MQTT retido: smartbox/ota/luzes/esp30  {versao, url, md5, tam}
        └─ HTTP:        GET :8377/ota/luzes-esp30-V21.bin
```

- **Código:** [`src/ota.ts`](src/ota.ts) (descoberta + MD5 + anúncio) e a rota `/ota/` em
  [`src/server.ts`](src/server.ts). Roda no mesmo intervalo do sync, mas **independente da
  nuvem** — atualizar o prédio não precisa de internet, só do Pi e do broker.
- **Nome do arquivo é contrato:** `<familia>-<placa>-<versao>.bin`. A `versao` tem que bater
  com o `OTA_VERSAO` de dentro do sketch — a versão *canônica*, sem o sufixo de placa que o
  `FW_VERSION` carrega pro portal (`V4 (30p)` no portal → `V4` no nome). Se não bater, o ESP
  reinicia achando que não atualizou (desiste após 3 tentativas, mas o lote não sobe).
- **Canário/pausa:** `ota.json` opcional na mesma pasta —
  `{ "luzes/esp30": { "unidades": [3], "ativo": true } }`.
- **Tirar o `.bin` da pasta** limpa o aviso retido (ninguém mais atualiza).
- **Firmware compatível:** as **5 famílias** (luzes V21/V22 — e V23/V24 quando validada · medidor V8 · ventilador V7 ·
  palm-vein V2.13 · portaria V6), nas duas placas de cada. A versão do nome do arquivo tem que
  bater com o `OTA_VERSAO` do sketch (a canônica, sem o sufixo de placa do `FW_VERSION`).
- Manual de uso (é o que o instalador lê): [`firmware/EDGE-RASPBERRY.md`](../../firmware/EDGE-RASPBERRY.md).
- **Firmware pela nuvem (0.10.0+):** segunda pasta, `/data/ota`, que [`src/ota-nuvem.ts`](src/ota-nuvem.ts)
  enche a partir da lista `ota` do snapshot (`GET /edge/snapshot`; ausente = não tocar, `[]` = retirar tudo):
  baixa pela API (`GET /edge/firmware/<pacoteId>`, chave do edge), grava em `.part`, confere tamanho,
  sha256, md5 e a assinatura ed25519 (0.11.0: com `firmware_chave_publica` nas options **todo** pacote tem de vir assinado e bater; sem a option, pacote assinado é recusado e o sem assinatura entra — bancada), e só então renomeia.
  `listarPacotes()` lê as duas pastas; **mesma chave nas duas = conflito** (chave não anunciada, fato sobe em
  `otaConflitos` no `POST /edge/contatos`). Servida em `GET :8377/ota/nuvem/<arquivo>`. Testes:
  `npm test` (`src/ota-nuvem.test.mjs`).
- **Versão (0.9.0+):** `dist/versao.json` (ARG `BUILD_VERSION` do Dockerfile) vai no header `x-edge-version`.
- **Embalagem e release (0.8.1+):** `npm run conferir-versao` (config.yaml = package.json = tag), workflows
  `.github/workflows/edge-release.yml` (tag `edge/vX.Y.Z` na main → imagem no GHCR) e `edge-publicar.yml`
  (Run workflow → manifesto gerado por `scripts/gerar-manifesto.mjs` no repositório público de add-on).

## Batimento da central (0.6.2) — saber que o Pi caiu sem esperar reclamação

A cada ciclo de sync (`sync_interval_s`, 60 s por padrão) o add-on dá um ping num check do
[healthchecks.io](https://healthchecks.io) (plano gratuito, conta `dev.sistema.smartbox`). Se o Pi
desligar, perder a internet ou o add-on parar, o ping deixa de chegar e o healthchecks manda o
alerta por e-mail. Não depende do `online` retido do MQTT (que o HA mostra mesmo com o Pi morto).

**Configurar (uma vez por prédio, na aba Configuração do add-on):**

```yaml
healthchecks_ping_url: "https://hc-ping.com/<uuid-do-check-deste-predio>"
```

No healthchecks: *Add Check* → nome `Pi <prédio>` → *Period* 5 min, *Grace* 10 min → copiar a URL
do ping. Vazio = sem batimento. Sync falhando por **10 ciclos seguidos** (10 min) manda `/fail` (o check
acusa na hora, não só por atraso); até lá o ping segue `ok` — um deploy da API derruba o sync por 2 a 5
ciclos e não é falha do prédio (0.6.3). Página canônica: `obsidian-docs/features/Monitoramento e Alertas.md`.

## Sonda dos ESPs (0.7.0) — saber que um ESP do prédio caiu

> **0.7.1:** a sonda e os comandos recusam também 172.30.32.0/23 (rede interna do Supervisor e dos
> add-ons). O cadastro no painel e o banco (migration 0357) recusam o mesmo.

Luzes, ar, ventilador, medidor, portaria, lavanderia e acessos comuns só falam com o broker do
prédio: a nuvem não os enxerga, e quando um some ninguém é avisado (caso da u03 do Centro em
13/10: 4 h fora). A central está na mesma rede — então é ela quem pergunta.

A cada ciclo de sync o add-on abre e fecha **uma conexão TCP** (porta 80, teto 2 s, em paralelo) no
portal de cada ESP cadastrado no card **"ESPs remotos"** do prédio no painel (a lista desce no
snapshot). Conexão aceita prova que o ESP está ligado, na rede e com o servidor web de pé; recusa ou
silêncio contam como "não respondeu". **Não é uma requisição HTTP**, de propósito: um GET anônimo em
`/` passa pelo `requireAuth()` do firmware e carimba "portal em uso", o que no ar V6 segura a
autocura — sondar por HTTP desarmaria o watchdog. Nenhum byte é enviado, nenhuma senha trafega. Quem
respondeu sobe em `POST /edge/contatos` e vira o "último contato" do ESP; a rotina de 5 min do
painel avisa o admin (sino + push) quando um ESP passa 10 min sem contato e de novo quando volta,
com o efeito em português ("as luzes da u03 não acendem pelo app; o interruptor funciona").

**Não tem opção nova.** Basta cadastrar os ESPs no card (esp_id, família, host na LAN). **Prefira o
IP com reserva no DHCP do roteador:** nome `.local` (mDNS) pode não resolver de dentro do container do
add-on, e aí o ESP consta como "nunca respondeu" e fica fora da vigilância sem aviso. Nomes do próprio
Home Assistant (`supervisor`, `homeassistant`, `core-*`, `addon_*`, `localhost`) são recusados. Prédio
sem ESP cadastrado = sem sonda. Log: `[sonda] N de M ESPs responderam`. Página canônica:
`obsidian-docs/features/Aviso de ESP sem Contato.md`.

## Saúde pelo broker (0.12.0) — o ESP não "morre" porque o IP mudou

A sonda acima pergunta ao **IP cadastrado**. Quando o roteador dá outro IP ao ESP (queda de energia
sem reserva de DHCP), ela bate no vazio e a Central acusa "mudo" com o ESP funcionando pelo app —
foram 4 dias no ar da u03 do Centro (30/10 a 03/11/2026).

Com a chave do prédio em **sombra** ou **ligado** (painel → card "ESPs do prédio", com PIN), a cada
ciclo o add-on faz **uma** chamada `POST /api/template` ao Home Assistant local e lê, para o aparelho
de cada ESP (o `device.identifiers` do firmware; a nuvem manda o identificador e os domínios que
contam no snapshot), se ele aparece disponível e o **IP que o próprio ESP anuncia** (hoje só o
`sensor.*_ar_reinicio` do ar V6+). Se esse IP é diferente do cadastrado, o add-on **pergunta por TCP
nele**, como a sonda. Tudo sobe no bloco `broker` do `POST /edge/contatos`.

- **Só a resposta no IP anunciado conta como vida** (chave `ligado`): a Central mostra amarelo
  "endereço mudou" em vez de "não responde". **Estar disponível no broker é só informação**: o
  `online` retido de um ESP que morreu junto com o broker volta para sempre, e nada que o HA mostre
  separa isso de um ESP vivo (reinício do HA, reconexão ao broker e até o hóspede apertando o botão da
  porta morta mudam o que o HA mostra). Revisão de segurança de 2026-11-04.
- Comando com login (Ler status, Ler config, Reiniciar) vai **sempre** para o IP cadastrado, que só
  muda com PIN no painel. Nunca para o anunciado.
- **Sem cliente MQTT, sem credencial nova, sem opção nova.** Nunca publica `offline` retido.
- HA fora ou template recusado: `estado: 'sem'`, fica só a sonda. Famílias que ainda não anunciam IP
  (todas menos o ar) ficam só com a sonda até a próxima versão de firmware de cada uma (F1).
- Log: `[broker] N de M ESPs disponíveis no HA · K respondendo em outro endereço · J sem elo com o HA`.

Página canônica: `obsidian-docs/features/Saude do ESP pelo Broker.md`.

## Comandos remotos nos ESPs (0.6.0) — mexer num ESP sem ir ao prédio

O portal web dos ESPs só responde na LAN. A partir da **0.6.0**, o painel enfileira um comando e
**este add-on executa aqui dentro**, no mesmo ciclo em que busca o snapshot — o edge é quem sai
para a nuvem, então **nenhuma porta é aberta no prédio** e o portal continua invisível para a
internet. (Requisito do Vinicius, 21/09: todo ESP deve poder ser acessado remotamente.)

**Configurar (uma vez, na aba Configuração do add-on):**

```yaml
esp_portal_user: "admin"      # o login do portal dos ESPs deste prédio
esp_portal_pass: "…"          # NÃO deixe a senha de fábrica (admin123)
```

Sem isso, todo comando volta com *"Login do portal dos ESPs não configurado"*. A credencial fica
**só aqui, no Pi** — nunca viaja no comando que vem da nuvem.

> ⚠️ **A senha do portal é UMA por prédio no add-on** — o par `esp_portal_user`/`esp_portal_pass`
> vale para todos os ESPs daquele Raspberry. Se os ESPs tiverem senhas diferentes, só respondem os
> que baterem com a que está no add-on. **Padronize a senha do portal em todos os ESPs do prédio.**
>
> 🔴 **E há um impedimento de firmware para trocar a senha de fábrica hoje:** `medidor` e `luzes`
> têm a aba **Painel** (`/config/panel` → `panel_user`/`panel_pass`), mas o **ventilador NÃO tem
> tela para trocar** — ele nasce com `admin`/`admin123` e não há como mudar pelo portal (conferido
> no V24: existe `cfg.panel_pass`, não existe rota que grave). Então, hoje, trocar a senha nos
> medidores e nas luzes **deixaria o ventilador inalcançável** pelo add-on.
>
> **O que fazer agora:** preencher o add-on com a senha que os ESPs já têm (de fábrica,
> `admin`/`admin123`). **Follow-up de firmware:** dar a aba Painel ao ventilador (e conferir
> palma/portaria) e só então padronizar uma senha nova em todo o prédio.

**Cadastrar os ESPs:** painel → *Empreendimento → Obra & Operação → Automação* → card **"ESPs do
prédio (acesso remoto)"** → ID do ESP (o mesmo do portal, ex. `medidor-a`), família e o IP dele na
rede do prédio.

**O que cada família aceita** (elas não expõem as mesmas rotas — o card só oferece o que o firmware
atende): `medidor` e `luzes` = ler status, ler config e reiniciar; `ventilador` = status e
reiniciar; `palm-vein-fechadura` e `portaria` = só saber se respondem.

**Garantias, e por que elas existem** (`src/portal-esp.ts`):
- O comando traz um **tipo** de vocabulário fechado, nunca uma URL — é o que impede a fila de virar
  um SSRF dentro da LAN. `/factory-open` **não é alcançável** pela fila.
- O host é revalidado aqui: **só IP privado ou nome `.local`**, porta 80.
- O `/export/json` do ESP traz `wifi_pass`, `mqtt_pass` e `panel_pass` **em texto puro** — são
  **redigidas** antes de o resultado subir.
- `config_set` lê a config atual, aplica o patch e reenvia o formulário **completo**: o
  `/save/canais` do firmware trata campo ausente como *desativar*, e um envio parcial desligaria os
  8 canais do medidor de uma vez.

Teste: `npx tsx src/portal-esp.test.mjs` (25 asserts sobre host, redação e merge de canais).

## Pré-requisitos

1. **Backend com as rotas `/edge/*` no ar** (deploy do `services/api` no Render).
2. **Migration `0218_edge_devices`** aplicada (✅ já em prod).
3. Um registro em `edge_devices` pra este prédio (passo abaixo) → gera o `edge_key`.

## Passo 1 — registrar o edge do prédio (gera o `edge_key`)

**Pelo painel** (2026-07-15 — antes era node + SQL manual): *Empreendimento → Obra & Operação →
Automação → card "Edge local (Raspberry)"* → digite um nome (ex.: `raspberry-centro-vv`) →
**Registrar edge** (pede PIN). A **`edge_key` aparece uma única vez** — copie na hora: o token em
claro só existe no add-on; o banco guarda o sha256.

> **Revogar / mudar de prédio:** botão **Remover** no mesmo card (PIN) — apaga o registro e a chave
> deixa de valer. Raspberry que muda de prédio = Registrar no empreendimento novo (chave nova no
> add-on) + Remover no antigo. O `update ... set revoked_at = now()` via SQL segue funcionando como
> alternativa de emergência.

## Passo 2 — instalar o add-on no Home Assistant

Como **add-on local**:

1. No Raspberry, copie a pasta `services/edge/` para `/addons/smartbox_edge/` (via Samba/SSH add-on, ou File editor).
2. HA → **Configurações → Add-ons → Add-on Store → ⋮ → Check for updates**. O "SmartBox Edge" aparece em *Local add-ons*.
3. Abra → **Install** → aba **Configuration**:
   - `cloud_url`: `https://smartbox-api.onrender.com`
   - `edge_key`: o **TOKEN** do Passo 1
   - `sync_interval_s`: `60` (padrão)
4. **Start**. Nos **Logs** deve aparecer:
   ```
   [sync] snapshot ok: N sensores, N credenciais, N ocupacoes
   [edge] ouvindo na porta 8377 (o ESP do sensor aponta pra ca)
   ```
5. Teste o health (de qualquer PC na rede): `http://<ip-do-raspberry>:8377/health`.

## Atualizar o add-on (novo deploy do edge)

Quando o código do edge muda (ex.: **0.2.0 → 0.3.0**, que passou a publicar o nome do empreendimento no
tópico MQTT retido `smartbox/predio`), o add-on **já instalado** se atualiza assim — sem re-registrar o
edge nem reconfigurar (`edge_key`/`cloud_url` continuam):

1. **Bump de versão obrigatório** no `config.yaml` (`version:`) — já vem no commit. **Sem bump o Supervisor
   reusa a imagem cacheada** e o Rebuild não pega o código novo (visto 2026-07-03).
2. **Backend no ar primeiro:** garanta que o `services/api` do deploy novo já subiu no Render — senão o
   `/edge/snapshot` ainda não traz o campo novo (ex.: `empreendimentoNome`) e o edge não tem o que publicar.
3. **Copie a pasta `services/edge/` por cima** de `/addons/smartbox_edge/` no Raspberry (Samba/File editor/SSH).
4. HA → **Configurações → Add-ons → Add-on Store → ⋮ → Verificar atualizações** → o **SmartBox Edge** mostra
   **Atualizar (para a versão nova)** → clique. Se o botão não aparecer, abra o add-on e use **Reconstruir**.
5. **Iniciar** (se não reiniciar sozinho) e conferir nos **Logs**: além do `[sync] snapshot ok: …`, na 0.3.0+
   deve aparecer `[sync] smartbox/predio publicado: <Nome do Empreendimento>`. Se aparecer
   `falha ao publicar smartbox/predio`, a integração **MQTT do HA** não está conectada ao broker.

Não precisa mexer no ESP (ele assina o tópico e recebe o nome sozinho) nem re-registrar o edge.

## Passo 3 — apontar o ESP do sensor pro edge

No portal do ESP (firmware **V2.2+**) → **Backend URL**:

```
http://<ip-do-raspberry>:8377
```

(em vez de `https://smartbox-api.onrender.com`). Salvar → reiniciar. O firmware detecta `http://` e usa a LAN sem TLS. **Regrave o V2.2** se o ESP estiver numa versão anterior (o suporte a `http://` entrou na V2.2).

## Passo 4 — validar offline

1. Abra a porta com a palma normalmente (com internet) → funciona via edge→nuvem.
2. **Tire a internet do prédio** (só o WAN; a LAN/Wi-Fi continua).
3. Abra com a palma de novo → **deve abrir** (o edge decide local). No log do add-on:
   ```
   [recognize] OFFLINE palm=NNN -> AUTORIZADO (ok)
   ```
4. Volte a internet → em até `sync_interval_s` os logs offline sobem (`[sync] N logs subiram`).

## Notas de segurança (do audit da Fatia 1)

- **Fail-safe respeita a validade:** estadia vencida (`validoAte < hoje`) **NEGA** mesmo offline.
- O `edge_key` autentica e **escopa ao empreendimento** — um edge nunca lê/loga outro prédio.
- Cache em disco (`/data`) contém PII (palmas/ocupações) — o `/data` do add-on é local ao Pi; criptografia at-rest é follow-up.
- Enroll/cadastro **não** opera offline (é raro e tolera internet).

## Dev local (sem o Raspberry)

```bash
cd services/edge
CLOUD_URL=http://localhost:3013 EDGE_KEY=<token> PORT=8377 npx tsx src/index.ts
# health: curl http://localhost:8377/health
```

Testes da lógica de autorização: `npx tsx src/authz.test.mjs`.
