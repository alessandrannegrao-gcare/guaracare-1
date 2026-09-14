# GuaraCare — site institucional

## O negócio

GuaraCare é uma empresa de produtos médicos em Guarapuava/PR, especializada em
terapia do sono: CPAP, BiPAP e máscaras (marca ResMed, entre outras). Atende
diagnóstico, locação e venda de equipamentos, com acompanhamento clínico
(ajuste de pressão, adaptação, suporte via WhatsApp e app myAir).

- Contato principal: WhatsApp Business próprio da empresa — (42) 98866-5313
  (em breve integrado via Evolution API para automações)
- Site: landing page única (`index.html`) + funil standalone de captação de
  lead via quiz STOP-BANG (`teste.html`)

## Estado da migração de conta (quase concluída)

Migração da infraestrutura digital da conta pessoal do Igor Lachowski
(desenvolvedor original) para a conta da Dra. Alessandra Negrão (dona do
negócio), feita em setembro/2026.

**Já migrado e confirmado funcionando:**

- **GitHub**: repositório transferido de `igorbigao/guaracare` para
  `alessandrannegrao-gcare/guaracare-1` (o GitHub adicionou o sufixo `-1`
  porque já existia outro repositório vazio chamado `guaracare` — sem relação
  com o site — na conta dela). Remote local já atualizado.
- **Vercel**: em vez de tentar transferir o projeto antigo (a busca de
  destino da Vercel não localizava a conta/time de destino de forma
  confiável), foi criado um **projeto novo do zero**, no time **"Guaracare"**
  da conta dela, importado diretamente do repositório já transferido. Sem
  variáveis de ambiente (o site não usa nenhuma).
- **Domínio `guaracare.com.br`**: migrado do projeto antigo pro projeto novo
  via verificação TXT (`_vercel.guaracare.com.br`) direto no painel do
  registro.br — sem alterar os registros A/CNAME existentes (já apontavam
  para a Vercel) e **sem downtime**. `www.guaracare.com.br` é o canônico
  (confirmado via `<link rel="canonical">` no `index.html`); o domínio raiz
  redireciona (308) para o `www`.
- **WhatsApp Business**: já é próprio da empresa, não precisou migrar.
- **Formspree**: recriado do zero na conta da Dra. Alessandra
  (`formspree.io/f/mqpkvygg`), substituindo o antigo (`.../f/mlgpyqry`, do
  Igor). Constantes `FORMSPREE_ID` (`index.html`) e `FORMSPREE` (`teste.html`)
  já atualizadas.
- **Google Sheets / Apps Script**: recriado do zero na conta da Dra.
  Alessandra (novo deployment terminado em `.../exec`), substituindo o antigo
  do Igor. Constantes `SHEETS_WEBHOOK` (`index.html`) e `SHEETS`
  (`teste.html`) já atualizadas.

**Decisão deliberada — NÃO migrar:**

- **Titularidade do domínio `guaracare.com.br`**: por escolha da Dra.
  Alessandra, o registro no registro.br continua em nome do Igor. Ele
  continua sendo acionado pontualmente para mudanças de DNS quando
  necessário. Não tente sugerir/iniciar uma transferência de titularidade
  sem confirmar com ela primeiro — foi decisão consciente, não pendência.

**Ainda pendente:**

- **Acesso do Igor**: decisão tomada — desligar o acesso dele a tudo (GitHub,
  Vercel, Formspree antigo, Apps Script antigo), **exceto o domínio** (ver
  acima). Fazer isso só depois de confirmar em produção, por alguns dias,
  que os novos Formspree/planilha estão recebendo os leads corretamente —
  não revogar acesso antes de validar.
- Exportar/copiar o histórico de leads da planilha antiga do Igor para a
  nova, se ainda não foi feito, antes de revogar o acesso dele a ela.

## Arquitetura

Site estático, sem framework e sem build step — HTML/CSS/JS puro. Dois
arquivos de página:

- `index.html` — site principal (SEO com Schema.org/FAQPage, seção de
  produtos, formulário de lead, quiz STOP-BANG embutido, widget de WhatsApp)
- `teste.html` — página standalone do "Teste do Sono" (mesmo quiz STOP-BANG
  de `index.html`, extraído para funil de captação isolado)
- `vercel.json` — headers de segurança (X-Frame-Options, nosniff, etc.) e
  cache longo para assets estáticos; `cleanUrls: true`
- `sitemap.xml` e `robots.txt` — servidos como arquivos estáticos na raiz
  (sem rewrite necessário; mesmo esquema do `logo.png`)

Integrações de captação de lead (Formspree para e-mail, Google Apps Script
para Google Sheets) são chamadas via `fetch` direto do client, sem backend
próprio. As constantes variam de nome entre os dois arquivos (mesmos valores,
lógica duplicada, não compartilhada):

- `index.html`: `FORMSPREE_ID` (linha ~1347) e `SHEETS_WEBHOOK` (linha ~1368)
- `teste.html`: `FORMSPREE` e `SHEETS` (linhas ~406–407)

Ao alterar uma integração, lembre de checar **os dois arquivos**.

## Armadilhas técnicas já mapeadas

### 1. Cloudflare Rocket Loader quebra JS inline

Se a zona Cloudflare de um domínio tiver o Rocket Loader ativo, ele
adia/altera a execução de `<script>` inline por padrão, o que quebra
qualquer lógica que dependa de rodar imediatamente (event listeners,
inicialização do quiz, etc). A defesa é o atributo `data-cfasync="false"` na
tag `<script>`.

- `index.html` tem a proteção: `<script data-cfasync="false">` (linha 1233).
- **`teste.html` NÃO tem** — o `<script>` da linha 404 está desprotegido.

**Atualização (set/2026):** confirmado que `guaracare.com.br` usa DNS direto
do registro.br para a Vercel — **não passa por Cloudflare hoje**, então este
risco está dormente na configuração atual. Mesmo assim, mantenha a proteção
em `index.html` e considere aplicá-la em `teste.html` — se o domínio um dia
for colocado atrás de Cloudflare (ex: para WAF/cache), essa inconsistência
volta a ser um risco real e silencioso.

### 2. `pointer-events` no widget de WhatsApp

O widget flutuante (`.wa-bubble` / `.wa-trigger`) usa `pointer-events: none`
no balão fechado e `pointer-events: all` quando `.show` está aplicado, para
permitir cliques através da área do balão sem bloquear o conteúdo por trás
quando ele está fechado. Se o widget parecer "não clicável" ou "bloqueando a
página", checar primeiro se essas classes/estados estão sincronizados
corretamente — é a causa mais provável, não um problema de z-index.

### 3. Sanitizar antes de embutir em HTML (XSS não tratado)

Em ambos os arquivos, o resultado do quiz é montado via
`resultContent.innerHTML = \`...\`` interpolando diretamente
`${name.split(' ')[0]}`, onde `name` vem de
`document.getElementById('leadName').value` sem qualquer escape ou
sanitização:

- `teste.html`, função `showResult` (linhas ~558–580)
- `index.html`, função equivalente (linhas ~1420–1458)

Isso é uma vulnerabilidade de XSS real: HTML/script digitado no campo "nome"
do formulário é executado na própria página do visitante. Qualquer alteração
nesse fluxo deve escapar o valor (ex: `textContent` em vez de `innerHTML`
para partes com dado do usuário, ou uma função de escape de HTML) antes de
interpolar em template strings destinadas a `innerHTML`.

### 4. Guard morto bloqueava o envio de leads em `index.html` (corrigido em set/2026)

Até a migração do Formspree/Sheets, `index.html` tinha o envio ao Formspree e
ao Google Sheets dentro de um `if` que comparava a constante ao **próprio
valor literal** — condição sempre falsa, então o `fetch` nunca rodava:

```js
const FORMSPREE_ID = 'https://formspree.io/f/mlgpyqry';
if (FORMSPREE_ID !== 'https://formspree.io/f/mlgpyqry') { /* nunca executava */ }
```

Na prática, o quiz STOP-BANG embutido na página principal (diferente de
`teste.html`, que não tinha esse guard) muito provavelmente **nunca enviou
um lead sequer** para Formspree ou para a planilha, silenciosamente — o
usuário só percebia o redirecionamento pro WhatsApp, que funciona
independente e mascarava a falha. Havia ainda um segundo bug junto: o
`fetch` do Formspree remontava a URL como
`` `https://formspree.io/f/${FORMSPREE_ID}` ``, mas `FORMSPREE_ID` já era a
URL completa — o resultado ficaria duplicado
(`.../f/https://formspree.io/f/...`) e nunca teria funcionado mesmo sem o
guard.

**Ambos corrigidos** na troca para os novos endpoints (Formspree e Sheets da
Dra. Alessandra): o `if` foi removido e o `fetch` do Formspree agora usa a
constante diretamente como URL. Se `index.html` for tocado novamente nessa
área, confirme que nenhum guard equivalente foi reintroduzido.

## Pendências conhecidas

- O repositório de trabalho deve continuar fora de pastas sincronizadas por
  Google Drive/Dropbox/OneDrive (o cliente de sync injeta `desktop.ini` em
  toda subpasta do `.git`, corrompendo refs). Local atual de trabalho:
  `C:\Users\<usuário>\projetos\guaracare-site`.
