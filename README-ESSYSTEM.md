# ESSystem Suporte Remoto

Client de acesso remoto da ESSystem, baseado no [RustDesk](https://github.com/rustdesk/rustdesk) (AGPL-3.0),
customizado e pré-configurado para o servidor próprio da empresa.

> **Licença:** este projeto deriva do RustDesk, licenciado sob **AGPL-3.0** (ver `LICENCE`).
> A distribuição do executável modificado obriga a disponibilizar o código-fonte
> correspondente a quem recebe o binário.

---

## Branches

| Branch | Produto | Uso |
|---|---|---|
| `master` | **ESSystem-Cliente.exe** | Máquinas dos clientes. Janela travada (330x480), só recebe conexão. |
| `tecnico` | **ESSystem-Tecnico.exe** | Máquinas da equipe. Painel de conexão, catálogo/login, janela normal. |

As duas compartilham o submódulo `libs/hbb_common`, onde ficam servidor, chave e nome do app.
Mudanças comuns são feitas no `master` e replicadas com `git cherry-pick` para `tecnico`.

## Build

Pelo GitHub Actions (não há build local configurado):

> Actions → **ESSystem - Build Windows (portable, teste)** → Run workflow → escolher a branch

O workflow nomeia o executável conforme a branch e publica o artefato
`essystem-<cliente|tecnico>-windows-x86_64` (pasta portable pronta para uso).

## Instalação nas máquinas

O executável roda como **portable** por padrão. Para instalar como serviço
(necessário para **acesso desatendido**):

```bat
ESSystem-Cliente.exe --silent-install
```

Ou renomeie o arquivo para terminar em `install.exe` e dê duplo clique — o RustDesk
abre o instalador automaticamente quando o nome do executável casa com esse padrão
(ver `is_setup()` em `src/common.rs`).

Depois de instalar, para acesso desatendido: **Configurações → Segurança → Senha permanente**.

---

## Decisões de projeto (o porquê)

Registro do que não é óbvio olhando só o código.

### 1. `libs/hbb_common` é um fork próprio

O submódulo aponta para o fork da empresa, branch `essystem`, e **não** para o
`rustdesk/hbb_common` original. Motivo: servidor, chave pública, nome do app e o
formato da senha são cravados em tempo de compilação lá dentro
(`src/config.rs` e `src/password_security.rs`). Sem o fork, o CI baixaria a versão
original e o build sairia sem as credenciais.

### 2. Credenciais cravadas no build

Cliente e técnico já saem configurados — o usuário final não digita nada:

- Servidor de ID e API: `acesso.essystem.com.br`
- Chave pública: definida em `RENDEZVOUS_SERVERS` / `RS_PUB_KEY` no `hbb_common`
- Reforço em runtime no `desktop_home_page.dart` (`initState`)

A opção correta para a chave é **`key`** — não `custom-key`, que não é lida por
`get_key()` em `src/common.rs`.

### 3. Senha de uso único: 4 dígitos numéricos

Alterado em `hbb_common/src/password_security.rs` para o padrão TeamViewer.
Segurança depende da proteção anti-brute-force do RustDesk (bloqueio por IP após
tentativas erradas). **Não use 4 dígitos em máquinas críticas expostas** — nessas,
prefira senha permanente forte.

### 4. Servidor usa o fork `lejianwen/rustdesk-server`

O `hbbs`/`hbbr` **não** usam a imagem oficial `rustdesk/rustdesk-server`. Com a
oficial, clientes **logados na conta** falhavam com `Failed to secure tcp: deadline
has elapsed` (bug conhecido: lejianwen/rustdesk-api#482). O fork corrige o timeout
de conexão com contas da API. Trocar de volta reintroduz o bug.

### 5. Executável é renomeado com `mv`, nunca `cp`

No workflow, o `rustdesk.exe` é **renomeado** (não copiado) para o nome de marca.
Motivo: o instalador do Windows executa
`move /Y "{pasta}\{exe_origem}" "{pasta}\{app_name}.exe"` e registra o serviço
apontando para `APP_NAME.exe`. Com dois executáveis na pasta, o serviço acabava
apontando para um binário inexistente e nunca iniciava.

### 6. Ícone multi-resolução

`app_icon.ico`, `tray-icon.ico` e `icon.ico` contêm 16/24/32/48/64/128/256 px.
Um `.ico` só com 256 px fica borrado na barra de tarefas e ilegível na bandeja.

### 7. Gatilhos automáticos de CI desativados

`Full Flutter CI` e `CI` rodavam a matriz completa (macOS, Android, Linux) a cada
push, gastando ~50 min sem gerar artefato utilizável. Ficaram apenas com
`workflow_dispatch`.

---

## Servidor

Docker Compose em `/opt/rustdesk` (Ubuntu 22). Componentes: `hbbs`, `hbbr` e
`rustdesk-api` (painel web, porta 21114) atrás de Nginx com TLS em
`https://acesso.essystem.com.br`.

Armadilhas conhecidas:

- **`docker-compose` 1.29.2** tem o bug `KeyError: 'ContainerConfig'` ao recriar
  container com volume. Sempre `docker rm -f <nome>` antes de `docker-compose up -d`.
- A API lê `conf/config.yaml` montado no container, e **o arquivo tem precedência
  sobre variáveis de ambiente**. Alterações em env que parecem ignoradas geralmente
  estão sendo sobrescritas por ele.
- Em `config.yaml`, `web-client` é um **inteiro** (`1` = ligado, `0` = desligado).
  Colocar uma URL ali derruba a API em loop de crash (resulta em 502 no painel).
- O container da API roda em `network_mode: host` porque a checagem de status do
  ServerCmd usa `127.0.0.1` fixo.

Backup diário de `/opt/rustdesk/data` (chave privada + banco de usuários e catálogo)
via cron. **Copie os arquivos para fora do servidor periodicamente** — backup local
não protege contra perda da instância.
