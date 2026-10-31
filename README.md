# SmartBox Edge — repositório de add-on do Home Assistant

Repositório **só de distribuição**: o manifesto (`smartbox_edge/config.yaml`) aponta para a imagem
pronta publicada no GHCR. O código-fonte vive no monorepo privado do SmartBox e é publicado aqui
pelo GitHub Actions a cada versão (`edge/vX.Y.Z`), com um clique de aprovação. **Nada aqui é
editado à mão.**

## Instalar no Home Assistant do prédio

1. Configurações → Add-ons → Loja de add-ons → ⋮ → **Repositórios** → colar
   `https://github.com/creativeviewvisual-ctrl/smartbox-edge-addon` → Adicionar.
2. Loja → **SmartBox Edge** → Instalar.
3. Configuração: `cloud_url`, `edge_key` (gerada no painel, card "Edge local"), `esp_portal_user`,
   `esp_portal_pass`, `healthchecks_ping_url` → Salvar → Iniciar.
4. Log: `[sync] snapshot ok`.

## Atualizar

Prédio de bancada: "Atualizar automaticamente" ligado. Prédio em operação: a Central avisa quando
há versão nova; atualizar pelo botão do add-on, fora do horário de check-in.

Histórico de versões: `smartbox_edge/CHANGELOG.md` (traz o digest da imagem de cada versão).
