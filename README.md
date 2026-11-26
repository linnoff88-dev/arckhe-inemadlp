ï»¿# ARCKHÉdlp

**ê°Å¸â¡Â§ê°Å¸â¡Â· [PortuguêÂªs](README.md) êÂ· ê°Å¸â¡Âºê°Å¸â¡Â¸ [English](README.en.md) êÂ· ê°Å¸â¡Âªê°Å¸â¡Â¸ [EspaêÂ±ol](README.es.md)**

Downloader pessoal de vêÂ­deo e êÂ¡udio (FastAPI + yt-dlp) que roda na sua VPS. VocêÂª
cola o link pelo celular, a VPS baixa e devolve o arquivo ê¢â¬â que se apaga sozinho
depois de 6 horas.

## ê°Å¸ââ Guia de uso

Guia completo (landing + passo a passo): **https://linnoff88-dev.github.io/arckhe-/guia/**

---

## êÂndice

1. [As trêÂªs chaves do `.env`](#as-trêÂªs-chaves-do-env)
2. [Antes de comeêÂ§ar](#antes-de-comeêÂ§ar)
3. [Passo a passo do deploy](#passo-a-passo-do-deploy)
4. [Acesso remoto e portas](#acesso-remoto-e-portas)
5. [Depois que subiu: o primeiro download](#depois-que-subiu-o-primeiro-download)
6. [Cookies](#cookies)
7. [TranscriêÂ§êÂ£o](#transcriêÂ§êÂ£o)
8. [Atualizar](#atualizar)
9. [OperaêÂ§êÂ£o do dia a dia](#operaêÂ§êÂ£o-do-dia-a-dia)
10. [Quando algo dêÂ¡ errado](#quando-algo-dêÂ¡-errado)

---

## As trêÂªs chaves do `.env`

O `.env` tem trêÂªs segredos com papêÂ©is **diferentes**. Vale entender antes de
gerar, porque as consequêÂªncias de vazar ou trocar cada um nêÂ£o sêÂ£o as mesmas.

### `DLP_PASSWORD` ê¢â¬â a senha que **vocêÂª** digita

A êÂºnica que vocêÂª usa com as mêÂ£os. êâ° a que aparece na tela de login. VocêÂª digita
uma vez no celular e nunca mais, porque a sessêÂ£o nêÂ£o expira.

- **Trocar:** edite a linha e rode `docker compose restart app`. As sessêÂµes jêÂ¡
  abertas continuam valendo (isso êÂ© intencional ê¢â¬â trocar a senha nêÂ£o te desloga).
- **Cuidado:** ela êÂ© a êÂºnica coisa entre a internet e o seu `cookies.txt`, que
  contêÂ©m a sessêÂ£o do seu Google/YouTube/Instagram. NêÂ£o existe bloqueio por
  tentativas erradas, por design. EntêÂ£o ela **precisa ser longa e aleatêÂ³ria**,
  nêÂ£o uma senha que vocêÂª decora.

### `DLP_SECRET_KEY` ê¢â¬â a assinatura do cookie de sessêÂ£o

VocêÂª nunca digita isso. Quando vocêÂª acerta a senha, o servidor devolve um cookie
assinado com essa chave. A cada requisiêÂ§êÂ£o ele confere a assinatura para saber
que foi ele mesmo quem emitiu aquele cookie, e nêÂ£o alguêÂ©m que o inventou.

- **Se vazar:** qualquer um fabrica um cookie vêÂ¡lido e entra **sem saber a
  senha**. êâ° têÂ£o grave quanto vazar a senha.
- **Se vocêÂª trocar:** todo mundo desloga, inclusive vocêÂª. êâ° justamente o botêÂ£o de
  "derrubar todas as sessêÂµes" ê¢â¬â use se perder o celular.
- êâ° **independente da senha**, e êÂ© por isso que trocar a senha nêÂ£o desloga ninguêÂ©m.

### `DLP_UPLOAD_TOKEN` ê¢â¬â a credencial do script

Serve sêÂ³ para o `sync-cookies.sh`. Ele roda sozinho no cron, sem navegador e sem
sessêÂ£o, entêÂ£o precisa provar quem êÂ© de outro jeito: manda o token no cabeêÂ§alho
`X-Upload-Token`.

êâ° uma senha de mêÂ¡quina para **uma êÂºnica rota** (`POST /api/cookies`). NêÂ£o lista
jobs, nêÂ£o baixa arquivo, nêÂ£o cria download.

- **Se vazar:** o dano êÂ© limitado ê¢â¬â dêÂ¡ para substituir o seu `cookies.txt`, o que
  quebraria seus downloads, mas nêÂ£o expêÂµe nada (a rota sêÂ³ recebe, nunca devolve o
  arquivo).
- **Se vocêÂª trocar:** basta atualizar a linha do cron.

**Resumindo:** senha = vocêÂª entra êÂ· secret key = o servidor te reconhece depois êÂ·
upload token = o script entra sozinho por uma portinha sêÂ³.

---

## Antes de comeêÂ§ar

VocêÂª precisa de trêÂªs coisas prontas:

| O quêÂª | Por quêÂª |
|---|---|
| Uma **VPS Linux** com acesso `ssh` | êÂ© onde tudo roda |
| **Docker + Compose** instalados nela | êÂºnica dependêÂªncia; ffmpeg, yt-dlp e um runtime JS (Deno) vêÂªm na imagem |
| Um **subdomêÂ­nio** apontado para o IP da VPS | o Caddy emite o certificado HTTPS sozinho |

> O YouTube passou a exigir um runtime JavaScript para resolver seus desafios
> antes de liberar os formatos de vêÂ­deo. Por isso a imagem jêÂ¡ vem com Deno e o
> suporte a EJS do yt-dlp, atualizados a cada boot junto com o prêÂ³prio yt-dlp.

Instalar o Docker, se ainda nêÂ£o tiver:

```bash
curl -fsSL https://get.docker.com | sh
```

O DNS precisa estar apontado **antes** de subir. No painel do seu domêÂ­nio, crie
um registro do tipo A:

```
Tipo   Nome            Valor
A      dlp             203.0.113.10      ê¢â Â o IP da sua VPS
```

Confira que propagou (pode levar alguns minutos):

```bash
dig +short dlp.seudominio    # tem que responder o IP da VPS
```

---

## Passo a passo do deploy

Tudo abaixo roda **dentro da VPS**, via `ssh`.

### Caminho rêÂ¡pido: o instalador

Faz tudo ê¢â¬â gera os trêÂªs segredos, escreve o `.env` com a permissêÂ£o certa, sobe os
containers e imprime a senha no fim:

```bash
ssh usuario@203.0.113.10

git clone https://github.com/linnoff88-dev/ARCKHÉdlp.git
cd ARCKHÉdlp
./instalar.sh dlp.seudominio
```

Ele avisa se o DNS ainda nêÂ£o aponta para a mêÂ¡quina, normaliza o domêÂ­nio se vocêÂª
colar com `https://` por engano, e se jêÂ¡ existir um `.env` pergunta antes de
mexer (ajustando sêÂ³ o domêÂ­nio, sem trocar a sua senha). Se rodar de novo depois,
ele tambêÂ©m detecta e conserta linhas `DLP_DOMAIN` duplicadas.

Depois dele, pule para [Depois que subiu](#depois-que-subiu-o-primeiro-download).

### Caminho manual

Se preferir fazer êÂ  mêÂ£o, ou entender cada peêÂ§a:

### 1. Entrar na VPS e clonar

```bash
ssh usuario@203.0.113.10

git clone https://github.com/linnoff88-dev/ARCKHÉdlp.git
cd ARCKHÉdlp
```

### 2. Gerar os trêÂªs segredos

Rode os trêÂªs comandos e **guarde cada saêÂ­da** ê¢â¬â vocêÂª vai colar no `.env` no passo
seguinte:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(24))"   # ê¢â â DLP_PASSWORD
python3 -c "import secrets; print(secrets.token_urlsafe(32))"   # ê¢â â DLP_SECRET_KEY
python3 -c "import secrets; print(secrets.token_urlsafe(32))"   # ê¢â â DLP_UPLOAD_TOKEN
```

A senha do primeiro comando êÂ© a que vocêÂª vai digitar no celular ê¢â¬â anote no seu
gerenciador de senhas.

### 3. Preencher o `.env`

```bash
cp .env.example .env
nano .env          # ou vim
```

Deixe assim, com os valores que vocêÂª acabou de gerar:

```ini
DLP_PASSWORD=<saêÂ­da do 1êÂº comando>
DLP_SECRET_KEY=<saêÂ­da do 2êÂº comando>
DLP_UPLOAD_TOKEN=<saêÂ­da do 3êÂº comando>
DLP_TTL_HOURS=6
DLP_DATA_DIR=/data
DLP_DOMAIN=dlp.seudominio
```

> **Edite a linha `DLP_DOMAIN` que jêÂ¡ existe** ê¢â¬â nêÂ£o acrescente outra com
> `echo ... >> .env`. Com duas linhas iguais o Docker usa a êÂºltima, e o Caddy
> acaba pedindo certificado para o domêÂ­nio errado. O sintoma êÂ© o site nêÂ£o abrir
> por erro de certificado, com tudo o mais aparentemente certo.

Proteja o arquivo ê¢â¬â ele tem os trêÂªs segredos em texto puro:

```bash
chmod 600 .env
```

### 4. Subir

```bash
docker compose up --build -d
```

A primeira construêÂ§êÂ£o leva alguns minutos (baixa Python e ffmpeg). Acompanhe:

```bash
docker compose logs -f
```

VocêÂª quer ver o `uvicorn running on http://0.0.0.0:8000` do app e o Caddy
obtendo o certificado. `Ctrl-C` sai dos logs sem derrubar nada.

### 5. Confirmar que estêÂ¡ no ar

```bash
curl -I https://dlp.seudominio
```

Um `HTTP/2 200` significa que o Caddy jêÂ¡ pegou o certificado e o app responde.
Se der erro de certificado, quase sempre êÂ© DNS que ainda nêÂ£o propagou ou a porta
80 fechada ê¢â¬â veja a seêÂ§êÂ£o seguinte.

---

## Acesso remoto e portas

**VocêÂª nêÂ£o configura nada de acesso remoto alêÂ©m do DNS.** O acesso êÂ© o prêÂ³prio
site: vocêÂª abre `https://dlp.seudominio` de qualquer lugar, no celular ou no
desktop, e entra com a senha. NêÂ£o tem VPN, nêÂ£o tem têÂºnel, nêÂ£o tem porta
esquisita para lembrar.

### As portas envolvidas

| Porta | Quem usa | Precisa estar aberta? |
|---|---|---|
| **443** | HTTPS ê¢â¬â êÂ© por onde vocêÂª usa o ARCKHÉdlp | **sim** |
| **80** | O Caddy, para emitir/renovar o certificado e redirecionar para o 443 | **sim** (sem ela o HTTPS nêÂ£o êÂ© emitido) |
| **22** | Seu `ssh` para administrar a VPS | **sim** |
| 8000 | O app, **sêÂ³ dentro** da rede do Docker | **nêÂ£o** ê¢â¬â e nêÂ£o deve ser |

A porta 8000 merece uma nota: no `docker-compose.yml` o serviêÂ§o `app` usa
`expose`, nêÂ£o `ports`. Isso significa que ele êÂ© alcanêÂ§êÂ¡vel **apenas** pelo Caddy,
por dentro da rede interna do Docker. Da internet ele nêÂ£o existe. êâ° isso que
garante que ninguêÂ©m chegue no serviêÂ§o pulando o HTTPS e a senha.

### Abrir o firewall

Na maioria das VPS hêÂ¡ **dois** firewalls: o do sistema e o do provedor.

No sistema (Ubuntu, com `ufw`):

```bash
sudo ufw allow OpenSSH      # nêÂ£o se tranque para fora
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status              # confira antes de encerrar a sessêÂ£o ssh
```

No painel do provedor (security group da AWS, firewall da Oracle Cloud, da
DigitalOcean, da Hetznerê¢â¬Â¦), libere as mesmas portas ê¢â¬â **22, 80 e 443**. Esse êÂ© o
passo mais esquecido: o `ufw` fica certo, o painel bloqueia, e o certificado
nunca êÂ© emitido.

### Deixar a VPS mais segura (opcional, recomendado)

Como a mêÂ¡quina agora estêÂ¡ exposta, vale endurecer o `ssh`:

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
**"Adicionar êÂ  tela de inêÂ­cio"**. Vira um êÂ­cone como se fosse um app. A sessêÂ£o
nêÂ£o expira: vocêÂª nêÂ£o vai digitar a senha de novo.

### 2. Enviar os cookies

Sem isso, o YouTube provavelmente vai recusar os downloads da sua VPS, porque o
IP êÂ© de datacenter. No Edge ou no Chrome, **logado** no YouTube:

1. instale a extensêÂ£o [**Get cookies.txt LOCALLY**](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc);
2. abra o YouTube, clique na extensêÂ£o e exporte o `cookies.txt`;
3. no ARCKHÉdlp, abra o painel **Cookies** e envie o arquivo.

O painel passa a mostrar a data do êÂºltimo envio.

### 3. Baixar

Cole a URL, escolha **VêÂ­deo** (MP4, atêÂ© 1080p) ou **êÂudio** (M4A), e envie. O
item aparece na lista como "na fila", vira "baixando NN%" e termina com o têÂ­tulo
do vêÂ­deo e um botêÂ£o **Baixar**.

Lembre: o arquivo fica disponêÂ­vel por **6 horas**. Depois disso ele some da VPS e
vocêÂª precisa enfileirar de novo. Se quiser mais tempo, mude `DLP_TTL_HOURS` no
`.env` e reinicie.

---

## Cookies

### Por que isso êÂ© necessêÂ¡rio

O YouTube (e Instagram, TikTokê¢â¬Â¦) desconfia de conexêÂµes vindas de datacenters. A
sua VPS êÂ© um datacenter. Sem cookies, o download falha com um recado do tipo
*"Sign in to confirm you're not a bot"*.

Os cookies sêÂ£o a prova de que existe uma conta logada por trêÂ¡s. VocêÂª tira eles do
**seu navegador**, onde vocêÂª jêÂ¡ estêÂ¡ logado, e entrega para a VPS usar.

Cookies **nêÂ£o sêÂ£o a sua senha**. SêÂ£o a sua sessêÂ£o jêÂ¡ aberta. Ainda assim, quem
tiver o arquivo entra na sua conta ê¢â¬â por isso ele sêÂ³ vive dentro da sua VPS,
protegido pela senha do ARCKHÉdlp.

### Passo a passo (Windows, Edge ou Chrome)

**1. Instalar a extensêÂ£o** ê¢â¬â uma vez sêÂ³.

**Get cookies.txt LOCALLY** ê¢â¬â link direto:
https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc

êâ° open source e funciona offline: ela lêÂª os cookies do prêÂ³prio navegador e gera um
arquivo, sem mandar nada para lugar nenhum.

Serve para o Chrome e para o Edge ê¢â¬â o Edge instala a partir dessa mesma loja
(ele pede uma confirmaêÂ§êÂ£o para permitir extensêÂµes de outra loja, na primeira vez).

**2. Abrir o site e confirmar que estêÂ¡ logado**

VêÂ¡ em `https://www.youtube.com` e veja se a sua foto de perfil aparece no canto
superior direito. Se nêÂ£o estiver logado, faêÂ§a login. **Este passo êÂ© o que
importa** ê¢â¬â a extensêÂ£o sêÂ³ copia o que existe naquele momento.

**3. Exportar**

Ainda na aba do YouTube, clique no êÂ­cone da extensêÂ£o (talvez precise clicar na
peêÂ§a de quebra-cabeêÂ§a ê°Å¸Â§Â© para achêÂ¡-la) e escolha **Export** (ou *Export as*
ê¢â â `cookies.txt`).

Um arquivo `cookies.txt` cai na sua pasta de Downloads. Ele comeêÂ§a com a linha
`# Netscape HTTP Cookie File` ê¢â¬â o ARCKHÉdlp recusa o arquivo se essa linha nêÂ£o
estiver lêÂ¡, justamente porque o yt-dlp tambêÂ©m recusaria.

O ARCKHÉdlp tambêÂ©m aceita o export em **JSON** que algumas versêÂµes dessa (e de
outras) extensêÂµes oferecem ê¢â¬â se a pêÂ¡gina de exportaêÂ§êÂ£o te der a escolher entre
`cookies.txt` e JSON, **prefira `cookies.txt`**: êÂ© o formato que o yt-dlp lêÂª
nativamente, entêÂ£o hêÂ¡ uma etapa de conversêÂ£o a menos e menos chance de erro. Se
o painel recusar o arquivo, a mensagem agora explica o motivo (JSON malformado,
pêÂ¡gina HTML salva por engano, campos separados por espaêÂ§o em vez de TAB,
arquivo vazio etc.) e o que fazer para corrigir.

**4. Enviar para o ARCKHÉdlp**

1. abra `https://seu-dominio` (no PC mesmo, êÂ© mais fêÂ¡cil que no celular);
2. entre com a senha, se pedir;
3. lêÂ¡ embaixo, clique em **Cookies** para abrir o painel;
4. clique em escolher arquivo e selecione o `cookies.txt` que acabou de baixar;
5. o painel responde com a quantidade de cookies recebidos e passa a mostrar a
   data do envio.

Pronto. Agora êÂ© sêÂ³ enfileirar um link.

**5. Apagar o arquivo baixado**

O `cookies.txt` na sua pasta de Downloads dêÂ¡ acesso êÂ  sua conta. Depois de
enviar, apague.

### Quando refazer isso

SêÂ³ quando um download falhar. A interface diz *"cookies expirados ê¢â¬â envie um
cookies.txt novo"* quando reconhece o motivo, entêÂ£o vocêÂª nêÂ£o fica adivinhando.

Na prêÂ¡tica: YouTube dura semanas, êÂ s vezes meses. Instagram e TikTok quebram bem
mais rêÂ¡pido. Repita os passos 2 a 5 e reenfileire o link ê¢â¬â nêÂ£o existe
retentativa automêÂ¡tica.

### Por que nêÂ£o dêÂ¡ para automatizar no Windows

O Edge e o Chrome passaram a cifrar os cookies com App-Bound Encryption a partir
da versêÂ£o 127: sêÂ³ o prêÂ³prio processo do navegador consegue a chave. Nenhum script
externo lêÂª aqueles cookies, por mais permissêÂ£o que tenha. Por isso o caminho êÂ©
exportar pela extensêÂ£o.

### Atalho, sêÂ³ se vocêÂª usa Firefox no Linux

O Firefox guarda os cookies sem essa cifragem, entêÂ£o dêÂ¡ para automatizar. Na
mêÂ¡quina Linux com o Firefox logado:

```bash
# pegue o token na VPS:  grep DLP_UPLOAD_TOKEN .env
DLP_URL=https://seu-dominio DLP_UPLOAD_TOKEN=<token> ./sync-cookies.sh
```

E, se quiser nunca mais ver o erro, um cron semanal:

```cron
0 9 * * 1 DLP_URL=https://seu-dominio DLP_UPLOAD_TOKEN=<token> /caminho/para/ARCKHÉdlp/sync-cookies.sh
```

---

## TranscriêÂ§êÂ£o

Existe uma terceira opêÂ§êÂ£o, ao lado de VêÂ­deo e êÂudio: **TranscriêÂ§êÂ£o**, usando o
Whisper (`whisper-large-v3-turbo`) na API da Groq. Dois jeitos de pedir:

- Direto: cole o link e escolha **TranscriêÂ§êÂ£o** no formulêÂ¡rio ê¢â¬â o job baixa o
  êÂ¡udio e transcreve, do zero.
- A partir de um job jêÂ¡ pronto: jobs de vêÂ­deo ou êÂ¡udio com status "pronto" ganham
  um botêÂ£o **Transcrever**, que reaproveita o arquivo jêÂ¡ baixado em vez de buscar
  a fonte de novo (sêÂ³ volta a baixar se esse arquivo jêÂ¡ tiver expirado).

O texto aparece na prêÂ³pria tela, num bloco expansêÂ­vel com botêÂ£o de copiar ê¢â¬â pensado
pra celular, onde abrir um `.txt` baixado êÂ© chato ê¢â¬â e tambêÂ©m dêÂ¡ pra baixar o `.txt`
pelo botêÂ£o de sempre.

**LimitaêÂ§êÂ£o conhecida, por decisêÂ£o de projeto: vêÂ­deos longos ficam de fora.**

A API de transcriêÂ§êÂ£o da Groq aceita no mêÂ¡ximo **25 MB por arquivo**. O êÂ¡udio êÂ©
reamostrado para 16 kHz mono ê¢â¬â o formato que o Whisper usa internamente ê¢â¬â e
comprimido em mp3 a 64 kbit/s (bitrate constante, entêÂ£o o tamanho nêÂ£o depende
do conteêÂºdo). Medido: ~481 KB por minuto, ou seja os 25 MB batem em
aproximadamente **54 minutos de êÂ¡udio**.

Acima disso o job êÂ© **recusado com um aviso** dizendo a duraêÂ§êÂ£o do êÂ¡udio e o limite,
sem gastar chamada êÂ  API. NêÂ£o hêÂ¡ fatiamento automêÂ¡tico: seria possêÂ­vel cortar o
êÂ¡udio em pedaêÂ§os e emendar o texto, mas isso foi deliberadamente deixado de fora
para manter a coisa simples. Aula longa e podcast, portanto, nêÂ£o sêÂ£o transcritos
aqui ê¢â¬â para esses, use o ARCKHÉvox.

Exige `GROQ_API_KEY` no `.env`. Sem a chave, a opêÂ§êÂ£o nêÂ£o aparece na interface.

Outros limites da Groq, para referêÂªncia: 7.200 segundos de êÂ¡udio e 2.000
requisiêÂ§êÂµes por janela de cota. O modelo `whisper-large-v3-turbo` custa em torno de
US$ 0,04 por hora de êÂ¡udio.

---

## Atualizar

Na VPS, um comando:

```bash
cd ARCKHÉdlp
./atualizar.sh
```

Ele traz o cêÂ³digo novo, reconstrêÂ³i a imagem, recria os containers e confere se o
site voltou a responder. Antes de tudo isso, se houver ediêÂ§êÂµes locais nêÂ£o
commitadas, ele **avisa e pergunta** ê¢â¬â e o que guardar vai para o `git stash`, de
onde vocêÂª recupera com `git stash pop`. O `.env` nunca êÂ© tocado: senha, chaves,
cookies e arquivos baixados continuam onde estêÂ£o.

Se o histêÂ³rico local tiver divergido do remoto, ele para em vez de criar um merge
silencioso, e diz como resolver.

êâ¬ mêÂ£o, se preferir:

```bash
git pull
docker compose up -d --build
```

O `--build` êÂ© obrigatêÂ³rio ê¢â¬â sem ele o Docker reaproveita a imagem antiga e a
atualizaêÂ§êÂ£o nêÂ£o tem efeito nenhum.

Depois de atualizar, a versêÂ£o aparece discretamente no rodapêÂ© da interface
(`ARCKHÉdlp vX.Y.Z`) ê¢â¬â basta abrir o site e conferir se bate com a versêÂ£o nova
antes de considerar a VPS atualizada.

---

## OperaêÂ§êÂ£o do dia a dia

```bash
docker compose logs -f app     # ver o que estêÂ¡ acontecendo
docker compose restart app     # reinicia e atualiza o yt-dlp
docker compose down            # desliga (os dados em ./data ficam)
docker compose up -d           # liga de novo
```

**Trocar a senha:** edite `DLP_PASSWORD` no `.env` e rode
`docker compose restart app`. As sessêÂµes abertas continuam vêÂ¡lidas.

**Derrubar todas as sessêÂµes** (celular perdido): troque tambêÂ©m o
`DLP_SECRET_KEY` e reinicie. AêÂ­ todo mundo, vocêÂª inclusive, precisa logar de novo.

**Atualizar o cêÂ³digo:**

```bash
git pull && docker compose up --build -d
```

---

## Quando algo dêÂ¡ errado

| Sintoma | Causa provêÂ¡vel | O que fazer |
|---|---|---|
| O site nêÂ£o abre, erro de certificado | DNS nêÂ£o propagou, ou porta 80 fechada | `dig +short dlp.seudominio` e confira o firewall do provedor |
| Download falha com "cookies expirados" | os cookies venceram | exporte e envie de novo; depois reenfileire o link (nêÂ£o hêÂ¡ retry automêÂ¡tico) |
| Download falha com "Requested format is not available" ou lista de formatos vazia | o YouTube exige runtime JS (Deno/EJS) e a imagem estêÂ¡ desatualizada | `docker compose pull && docker compose up -d --build`; o boot jêÂ¡ atualiza yt-dlp/deno sozinho, mas uma imagem muito antiga pode nêÂ£o ter o suporte a EJS ainda |
| Download falha com outra mensagem | a fonte nêÂ£o êÂ© suportada, ou a URL estêÂ¡ errada | a mensagem crua do yt-dlp aparece na lista |
| O arquivo sumiu | passou o TTL de 6 horas | enfileire de novo, ou aumente `DLP_TTL_HOURS` |
| Job travado em "baixando" apêÂ³s reiniciar | o container reiniciou no meio | ele vira "erro" sozinho no boot; reenfileire |
| Disco enchendo | improvêÂ¡vel (hêÂ¡ limpeza automêÂ¡tica) | `du -sh data/downloads` e `docker compose logs app` |

---

Projeto  - ARCKHÉe