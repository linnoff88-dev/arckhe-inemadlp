# inemadlp

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

Downloader pessoal de vÃ­deo e Ã¡udio (FastAPI + yt-dlp) que roda na sua VPS. VocÃª
cola o link pelo celular, a VPS baixa e devolve o arquivo â€” que se apaga sozinho
depois de 6 horas.

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/inemadlp/guia/**

---

## Ãndice

1. [As trÃªs chaves do `.env`](#as-trÃªs-chaves-do-env)
2. [Antes de comeÃ§ar](#antes-de-comeÃ§ar)
3. [Passo a passo do deploy](#passo-a-passo-do-deploy)
4. [Acesso remoto e portas](#acesso-remoto-e-portas)
5. [Depois que subiu: o primeiro download](#depois-que-subiu-o-primeiro-download)
6. [Cookies](#cookies)
7. [TranscriÃ§Ã£o](#transcriÃ§Ã£o)
8. [Atualizar](#atualizar)
9. [OperaÃ§Ã£o do dia a dia](#operaÃ§Ã£o-do-dia-a-dia)
10. [Quando algo dÃ¡ errado](#quando-algo-dÃ¡-errado)

---

## As trÃªs chaves do `.env`

O `.env` tem trÃªs segredos com papÃ©is **diferentes**. Vale entender antes de
gerar, porque as consequÃªncias de vazar ou trocar cada um nÃ£o sÃ£o as mesmas.

### `DLP_PASSWORD` â€” a senha que **vocÃª** digita

A Ãºnica que vocÃª usa com as mÃ£os. Ã‰ a que aparece na tela de login. VocÃª digita
uma vez no celular e nunca mais, porque a sessÃ£o nÃ£o expira.

- **Trocar:** edite a linha e rode `docker compose restart app`. As sessÃµes jÃ¡
  abertas continuam valendo (isso Ã© intencional â€” trocar a senha nÃ£o te desloga).
- **Cuidado:** ela Ã© a Ãºnica coisa entre a internet e o seu `cookies.txt`, que
  contÃ©m a sessÃ£o do seu Google/YouTube/Instagram. NÃ£o existe bloqueio por
  tentativas erradas, por design. EntÃ£o ela **precisa ser longa e aleatÃ³ria**,
  nÃ£o uma senha que vocÃª decora.

### `DLP_SECRET_KEY` â€” a assinatura do cookie de sessÃ£o

VocÃª nunca digita isso. Quando vocÃª acerta a senha, o servidor devolve um cookie
assinado com essa chave. A cada requisiÃ§Ã£o ele confere a assinatura para saber
que foi ele mesmo quem emitiu aquele cookie, e nÃ£o alguÃ©m que o inventou.

- **Se vazar:** qualquer um fabrica um cookie vÃ¡lido e entra **sem saber a
  senha**. Ã‰ tÃ£o grave quanto vazar a senha.
- **Se vocÃª trocar:** todo mundo desloga, inclusive vocÃª. Ã‰ justamente o botÃ£o de
  "derrubar todas as sessÃµes" â€” use se perder o celular.
- Ã‰ **independente da senha**, e Ã© por isso que trocar a senha nÃ£o desloga ninguÃ©m.

### `DLP_UPLOAD_TOKEN` â€” a credencial do script

Serve sÃ³ para o `sync-cookies.sh`. Ele roda sozinho no cron, sem navegador e sem
sessÃ£o, entÃ£o precisa provar quem Ã© de outro jeito: manda o token no cabeÃ§alho
`X-Upload-Token`.

Ã‰ uma senha de mÃ¡quina para **uma Ãºnica rota** (`POST /api/cookies`). NÃ£o lista
jobs, nÃ£o baixa arquivo, nÃ£o cria download.

- **Se vazar:** o dano Ã© limitado â€” dÃ¡ para substituir o seu `cookies.txt`, o que
  quebraria seus downloads, mas nÃ£o expÃµe nada (a rota sÃ³ recebe, nunca devolve o
  arquivo).
- **Se vocÃª trocar:** basta atualizar a linha do cron.

**Resumindo:** senha = vocÃª entra Â· secret key = o servidor te reconhece depois Â·
upload token = o script entra sozinho por uma portinha sÃ³.

---

## Antes de comeÃ§ar

VocÃª precisa de trÃªs coisas prontas:

| O quÃª | Por quÃª |
|---|---|
| Uma **VPS Linux** com acesso `ssh` | Ã© onde tudo roda |
| **Docker + Compose** instalados nela | Ãºnica dependÃªncia; ffmpeg, yt-dlp e um runtime JS (Deno) vÃªm na imagem |
| Um **subdomÃ­nio** apontado para o IP da VPS | o Caddy emite o certificado HTTPS sozinho |

> O YouTube passou a exigir um runtime JavaScript para resolver seus desafios
> antes de liberar os formatos de vÃ­deo. Por isso a imagem jÃ¡ vem com Deno e o
> suporte a EJS do yt-dlp, atualizados a cada boot junto com o prÃ³prio yt-dlp.

Instalar o Docker, se ainda nÃ£o tiver:

```bash
curl -fsSL https://get.docker.com | sh
```

O DNS precisa estar apontado **antes** de subir. No painel do seu domÃ­nio, crie
um registro do tipo A:

```
Tipo   Nome            Valor
A      dlp             203.0.113.10      â† o IP da sua VPS
```

Confira que propagou (pode levar alguns minutos):

```bash
dig +short dlp.seudominio    # tem que responder o IP da VPS
```

---

## Passo a passo do deploy

Tudo abaixo roda **dentro da VPS**, via `ssh`.

### Caminho rÃ¡pido: o instalador

Faz tudo â€” gera os trÃªs segredos, escreve o `.env` com a permissÃ£o certa, sobe os
containers e imprime a senha no fim:

```bash
ssh usuario@203.0.113.10

git clone https://github.com/inematds/inemadlp.git
cd inemadlp
./instalar.sh dlp.seudominio
```

Ele avisa se o DNS ainda nÃ£o aponta para a mÃ¡quina, normaliza o domÃ­nio se vocÃª
colar com `https://` por engano, e se jÃ¡ existir um `.env` pergunta antes de
mexer (ajustando sÃ³ o domÃ­nio, sem trocar a sua senha). Se rodar de novo depois,
ele tambÃ©m detecta e conserta linhas `DLP_DOMAIN` duplicadas.

Depois dele, pule para [Depois que subiu](#depois-que-subiu-o-primeiro-download).

### Caminho manual

Se preferir fazer Ã  mÃ£o, ou entender cada peÃ§a:

### 1. Entrar na VPS e clonar

```bash
ssh usuario@203.0.113.10

git clone https://github.com/inematds/inemadlp.git
cd inemadlp
```

### 2. Gerar os trÃªs segredos

Rode os trÃªs comandos e **guarde cada saÃ­da** â€” vocÃª vai colar no `.env` no passo
seguinte:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(24))"   # â†’ DLP_PASSWORD
python3 -c "import secrets; print(secrets.token_urlsafe(32))"   # â†’ DLP_SECRET_KEY
python3 -c "import secrets; print(secrets.token_urlsafe(32))"   # â†’ DLP_UPLOAD_TOKEN
```

A senha do primeiro comando Ã© a que vocÃª vai digitar no celular â€” anote no seu
gerenciador de senhas.

### 3. Preencher o `.env`

```bash
cp .env.example .env
nano .env          # ou vim
```

Deixe assim, com os valores que vocÃª acabou de gerar:

```ini
DLP_PASSWORD=<saÃ­da do 1Âº comando>
DLP_SECRET_KEY=<saÃ­da do 2Âº comando>
DLP_UPLOAD_TOKEN=<saÃ­da do 3Âº comando>
DLP_TTL_HOURS=6
DLP_DATA_DIR=/data
DLP_DOMAIN=dlp.seudominio
```

> **Edite a linha `DLP_DOMAIN` que jÃ¡ existe** â€” nÃ£o acrescente outra com
> `echo ... >> .env`. Com duas linhas iguais o Docker usa a Ãºltima, e o Caddy
> acaba pedindo certificado para o domÃ­nio errado. O sintoma Ã© o site nÃ£o abrir
> por erro de certificado, com tudo o mais aparentemente certo.

Proteja o arquivo â€” ele tem os trÃªs segredos em texto puro:

```bash
chmod 600 .env
```

### 4. Subir

```bash
docker compose up --build -d
```

A primeira construÃ§Ã£o leva alguns minutos (baixa Python e ffmpeg). Acompanhe:

```bash
docker compose logs -f
```

VocÃª quer ver o `uvicorn running on http://0.0.0.0:8000` do app e o Caddy
obtendo o certificado. `Ctrl-C` sai dos logs sem derrubar nada.

### 5. Confirmar que estÃ¡ no ar

```bash
curl -I https://dlp.seudominio
```

Um `HTTP/2 200` significa que o Caddy jÃ¡ pegou o certificado e o app responde.
Se der erro de certificado, quase sempre Ã© DNS que ainda nÃ£o propagou ou a porta
80 fechada â€” veja a seÃ§Ã£o seguinte.

---

## Acesso remoto e portas

**VocÃª nÃ£o configura nada de acesso remoto alÃ©m do DNS.** O acesso Ã© o prÃ³prio
site: vocÃª abre `https://dlp.seudominio` de qualquer lugar, no celular ou no
desktop, e entra com a senha. NÃ£o tem VPN, nÃ£o tem tÃºnel, nÃ£o tem porta
esquisita para lembrar.

### As portas envolvidas

| Porta | Quem usa | Precisa estar aberta? |
|---|---|---|
| **443** | HTTPS â€” Ã© por onde vocÃª usa o inemadlp | **sim** |
| **80** | O Caddy, para emitir/renovar o certificado e redirecionar para o 443 | **sim** (sem ela o HTTPS nÃ£o Ã© emitido) |
| **22** | Seu `ssh` para administrar a VPS | **sim** |
| 8000 | O app, **sÃ³ dentro** da rede do Docker | **nÃ£o** â€” e nÃ£o deve ser |

A porta 8000 merece uma nota: no `docker-compose.yml` o serviÃ§o `app` usa
`expose`, nÃ£o `ports`. Isso significa que ele Ã© alcanÃ§Ã¡vel **apenas** pelo Caddy,
por dentro da rede interna do Docker. Da internet ele nÃ£o existe. Ã‰ isso que
garante que ninguÃ©m chegue no serviÃ§o pulando o HTTPS e a senha.

### Abrir o firewall

Na maioria das VPS hÃ¡ **dois** firewalls: o do sistema e o do provedor.

No sistema (Ubuntu, com `ufw`):

```bash
sudo ufw allow OpenSSH      # nÃ£o se tranque para fora
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status              # confira antes de encerrar a sessÃ£o ssh
```

No painel do provedor (security group da AWS, firewall da Oracle Cloud, da
DigitalOcean, da Hetznerâ€¦), libere as mesmas portas â€” **22, 80 e 443**. Esse Ã© o
passo mais esquecido: o `ufw` fica certo, o painel bloqueia, e o certificado
nunca Ã© emitido.

### Deixar a VPS mais segura (opcional, recomendado)

Como a mÃ¡quina agora estÃ¡ exposta, vale endurecer o `ssh`:

```bash
# use chave em vez de senha, e desligue login por senha
sudo nano /etc/ssh/sshd_config
#   PasswordAuthentication no
#   PermitRootLogin no
sudo systemctl restart ssh
```

---

## Depois que subiu: o primeiro download

### 1. Instalar no celular

Abra `https://dlp.seudominio` no navegador do celular, digite a senha e use
**"Adicionar Ã  tela de inÃ­cio"**. Vira um Ã­cone como se fosse um app. A sessÃ£o
nÃ£o expira: vocÃª nÃ£o vai digitar a senha de novo.

### 2. Enviar os cookies

Sem isso, o YouTube provavelmente vai recusar os downloads da sua VPS, porque o
IP Ã© de datacenter. No Edge ou no Chrome, **logado** no YouTube:

1. instale a extensÃ£o [**Get cookies.txt LOCALLY**](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc);
2. abra o YouTube, clique na extensÃ£o e exporte o `cookies.txt`;
3. no inemadlp, abra o painel **Cookies** e envie o arquivo.

O painel passa a mostrar a data do Ãºltimo envio.

### 3. Baixar

Cole a URL, escolha **VÃ­deo** (MP4, atÃ© 1080p) ou **Ãudio** (M4A), e envie. O
item aparece na lista como "na fila", vira "baixando NN%" e termina com o tÃ­tulo
do vÃ­deo e um botÃ£o **Baixar**.

Lembre: o arquivo fica disponÃ­vel por **6 horas**. Depois disso ele some da VPS e
vocÃª precisa enfileirar de novo. Se quiser mais tempo, mude `DLP_TTL_HOURS` no
`.env` e reinicie.

---

## Cookies

### Por que isso Ã© necessÃ¡rio

O YouTube (e Instagram, TikTokâ€¦) desconfia de conexÃµes vindas de datacenters. A
sua VPS Ã© um datacenter. Sem cookies, o download falha com um recado do tipo
*"Sign in to confirm you're not a bot"*.

Os cookies sÃ£o a prova de que existe uma conta logada por trÃ¡s. VocÃª tira eles do
**seu navegador**, onde vocÃª jÃ¡ estÃ¡ logado, e entrega para a VPS usar.

Cookies **nÃ£o sÃ£o a sua senha**. SÃ£o a sua sessÃ£o jÃ¡ aberta. Ainda assim, quem
tiver o arquivo entra na sua conta â€” por isso ele sÃ³ vive dentro da sua VPS,
protegido pela senha do inemadlp.

### Passo a passo (Windows, Edge ou Chrome)

**1. Instalar a extensÃ£o** â€” uma vez sÃ³.

**Get cookies.txt LOCALLY** â€” link direto:
https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc

Ã‰ open source e funciona offline: ela lÃª os cookies do prÃ³prio navegador e gera um
arquivo, sem mandar nada para lugar nenhum.

Serve para o Chrome e para o Edge â€” o Edge instala a partir dessa mesma loja
(ele pede uma confirmaÃ§Ã£o para permitir extensÃµes de outra loja, na primeira vez).

**2. Abrir o site e confirmar que estÃ¡ logado**

VÃ¡ em `https://www.youtube.com` e veja se a sua foto de perfil aparece no canto
superior direito. Se nÃ£o estiver logado, faÃ§a login. **Este passo Ã© o que
importa** â€” a extensÃ£o sÃ³ copia o que existe naquele momento.

**3. Exportar**

Ainda na aba do YouTube, clique no Ã­cone da extensÃ£o (talvez precise clicar na
peÃ§a de quebra-cabeÃ§a ðŸ§© para achÃ¡-la) e escolha **Export** (ou *Export as*
â†’ `cookies.txt`).

Um arquivo `cookies.txt` cai na sua pasta de Downloads. Ele comeÃ§a com a linha
`# Netscape HTTP Cookie File` â€” o inemadlp recusa o arquivo se essa linha nÃ£o
estiver lÃ¡, justamente porque o yt-dlp tambÃ©m recusaria.

O inemadlp tambÃ©m aceita o export em **JSON** que algumas versÃµes dessa (e de
outras) extensÃµes oferecem â€” se a pÃ¡gina de exportaÃ§Ã£o te der a escolher entre
`cookies.txt` e JSON, **prefira `cookies.txt`**: Ã© o formato que o yt-dlp lÃª
nativamente, entÃ£o hÃ¡ uma etapa de conversÃ£o a menos e menos chance de erro. Se
o painel recusar o arquivo, a mensagem agora explica o motivo (JSON malformado,
pÃ¡gina HTML salva por engano, campos separados por espaÃ§o em vez de TAB,
arquivo vazio etc.) e o que fazer para corrigir.

**4. Enviar para o inemadlp**

1. abra `https://seu-dominio` (no PC mesmo, Ã© mais fÃ¡cil que no celular);
2. entre com a senha, se pedir;
3. lÃ¡ embaixo, clique em **Cookies** para abrir o painel;
4. clique em escolher arquivo e selecione o `cookies.txt` que acabou de baixar;
5. o painel responde com a quantidade de cookies recebidos e passa a mostrar a
   data do envio.

Pronto. Agora Ã© sÃ³ enfileirar um link.

**5. Apagar o arquivo baixado**

O `cookies.txt` na sua pasta de Downloads dÃ¡ acesso Ã  sua conta. Depois de
enviar, apague.

### Quando refazer isso

SÃ³ quando um download falhar. A interface diz *"cookies expirados â€” envie um
cookies.txt novo"* quando reconhece o motivo, entÃ£o vocÃª nÃ£o fica adivinhando.

Na prÃ¡tica: YouTube dura semanas, Ã s vezes meses. Instagram e TikTok quebram bem
mais rÃ¡pido. Repita os passos 2 a 5 e reenfileire o link â€” nÃ£o existe
retentativa automÃ¡tica.

### Por que nÃ£o dÃ¡ para automatizar no Windows

O Edge e o Chrome passaram a cifrar os cookies com App-Bound Encryption a partir
da versÃ£o 127: sÃ³ o prÃ³prio processo do navegador consegue a chave. Nenhum script
externo lÃª aqueles cookies, por mais permissÃ£o que tenha. Por isso o caminho Ã©
exportar pela extensÃ£o.

### Atalho, sÃ³ se vocÃª usa Firefox no Linux

O Firefox guarda os cookies sem essa cifragem, entÃ£o dÃ¡ para automatizar. Na
mÃ¡quina Linux com o Firefox logado:

```bash
# pegue o token na VPS:  grep DLP_UPLOAD_TOKEN .env
DLP_URL=https://seu-dominio DLP_UPLOAD_TOKEN=<token> ./sync-cookies.sh
```

E, se quiser nunca mais ver o erro, um cron semanal:

```cron
0 9 * * 1 DLP_URL=https://seu-dominio DLP_UPLOAD_TOKEN=<token> /caminho/para/inemadlp/sync-cookies.sh
```

---

## TranscriÃ§Ã£o

Existe uma terceira opÃ§Ã£o, ao lado de VÃ­deo e Ãudio: **TranscriÃ§Ã£o**, usando o
Whisper (`whisper-large-v3-turbo`) na API da Groq. Dois jeitos de pedir:

- Direto: cole o link e escolha **TranscriÃ§Ã£o** no formulÃ¡rio â€” o job baixa o
  Ã¡udio e transcreve, do zero.
- A partir de um job jÃ¡ pronto: jobs de vÃ­deo ou Ã¡udio com status "pronto" ganham
  um botÃ£o **Transcrever**, que reaproveita o arquivo jÃ¡ baixado em vez de buscar
  a fonte de novo (sÃ³ volta a baixar se esse arquivo jÃ¡ tiver expirado).

O texto aparece na prÃ³pria tela, num bloco expansÃ­vel com botÃ£o de copiar â€” pensado
pra celular, onde abrir um `.txt` baixado Ã© chato â€” e tambÃ©m dÃ¡ pra baixar o `.txt`
pelo botÃ£o de sempre.

**LimitaÃ§Ã£o conhecida, por decisÃ£o de projeto: vÃ­deos longos ficam de fora.**

A API de transcriÃ§Ã£o da Groq aceita no mÃ¡ximo **25 MB por arquivo**. O Ã¡udio Ã©
reamostrado para 16 kHz mono â€” o formato que o Whisper usa internamente â€” e
comprimido em mp3 a 64 kbit/s (bitrate constante, entÃ£o o tamanho nÃ£o depende
do conteÃºdo). Medido: ~481 KB por minuto, ou seja os 25 MB batem em
aproximadamente **54 minutos de Ã¡udio**.

Acima disso o job Ã© **recusado com um aviso** dizendo a duraÃ§Ã£o do Ã¡udio e o limite,
sem gastar chamada Ã  API. NÃ£o hÃ¡ fatiamento automÃ¡tico: seria possÃ­vel cortar o
Ã¡udio em pedaÃ§os e emendar o texto, mas isso foi deliberadamente deixado de fora
para manter a coisa simples. Aula longa e podcast, portanto, nÃ£o sÃ£o transcritos
aqui â€” para esses, use o inemavox.

Exige `GROQ_API_KEY` no `.env`. Sem a chave, a opÃ§Ã£o nÃ£o aparece na interface.

Outros limites da Groq, para referÃªncia: 7.200 segundos de Ã¡udio e 2.000
requisiÃ§Ãµes por janela de cota. O modelo `whisper-large-v3-turbo` custa em torno de
US$ 0,04 por hora de Ã¡udio.

---

## Atualizar

Na VPS, um comando:

```bash
cd inemadlp
./atualizar.sh
```

Ele traz o cÃ³digo novo, reconstrÃ³i a imagem, recria os containers e confere se o
site voltou a responder. Antes de tudo isso, se houver ediÃ§Ãµes locais nÃ£o
commitadas, ele **avisa e pergunta** â€” e o que guardar vai para o `git stash`, de
onde vocÃª recupera com `git stash pop`. O `.env` nunca Ã© tocado: senha, chaves,
cookies e arquivos baixados continuam onde estÃ£o.

Se o histÃ³rico local tiver divergido do remoto, ele para em vez de criar um merge
silencioso, e diz como resolver.

Ã€ mÃ£o, se preferir:

```bash
git pull
docker compose up -d --build
```

O `--build` Ã© obrigatÃ³rio â€” sem ele o Docker reaproveita a imagem antiga e a
atualizaÃ§Ã£o nÃ£o tem efeito nenhum.

Depois de atualizar, a versÃ£o aparece discretamente no rodapÃ© da interface
(`inemadlp vX.Y.Z`) â€” basta abrir o site e conferir se bate com a versÃ£o nova
antes de considerar a VPS atualizada.

---

## OperaÃ§Ã£o do dia a dia

```bash
docker compose logs -f app     # ver o que estÃ¡ acontecendo
docker compose restart app     # reinicia e atualiza o yt-dlp
docker compose down            # desliga (os dados em ./data ficam)
docker compose up -d           # liga de novo
```

**Trocar a senha:** edite `DLP_PASSWORD` no `.env` e rode
`docker compose restart app`. As sessÃµes abertas continuam vÃ¡lidas.

**Derrubar todas as sessÃµes** (celular perdido): troque tambÃ©m o
`DLP_SECRET_KEY` e reinicie. AÃ­ todo mundo, vocÃª inclusive, precisa logar de novo.

**Atualizar o cÃ³digo:**

```bash
git pull && docker compose up --build -d
```

---

## Quando algo dÃ¡ errado

| Sintoma | Causa provÃ¡vel | O que fazer |
|---|---|---|
| O site nÃ£o abre, erro de certificado | DNS nÃ£o propagou, ou porta 80 fechada | `dig +short dlp.seudominio` e confira o firewall do provedor |
| Download falha com "cookies expirados" | os cookies venceram | exporte e envie de novo; depois reenfileire o link (nÃ£o hÃ¡ retry automÃ¡tico) |
| Download falha com "Requested format is not available" ou lista de formatos vazia | o YouTube exige runtime JS (Deno/EJS) e a imagem estÃ¡ desatualizada | `docker compose pull && docker compose up -d --build`; o boot jÃ¡ atualiza yt-dlp/deno sozinho, mas uma imagem muito antiga pode nÃ£o ter o suporte a EJS ainda |
| Download falha com outra mensagem | a fonte nÃ£o Ã© suportada, ou a URL estÃ¡ errada | a mensagem crua do yt-dlp aparece na lista |
| O arquivo sumiu | passou o TTL de 6 horas | enfileire de novo, ou aumente `DLP_TTL_HOURS` |
| Job travado em "baixando" apÃ³s reiniciar | o container reiniciou no meio | ele vira "erro" sozinho no boot; reenfileire |
| Disco enchendo | improvÃ¡vel (hÃ¡ limpeza automÃ¡tica) | `du -sh data/downloads` e `docker compose logs app` |

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe