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

**Decisão deliberada — NÃO migrar:**

- **Titularidade do domínio `guaracare.com.br`**: por escolha da Dra.
  Alessandra, o registro no registro.br continua em nome do Igor. Ele
  continua sendo acionado pontualmente para mudanças de DNS quando
  necessário. Não tente sugerir/iniciar uma transferência de titularidade
  sem confirmar com ela primeiro — foi decisão consciente, não pendência.

**Ainda pendente (prioridade alta — envolve dados de pacientes):**

- **Formspree** (`formspree.io/f/mlgpyqry`) e o **Google Apps Script /
  planilha** (`script.google.com/macros/s/AKfycby.../exec`) que recebem os
  leads do quiz STOP-BANG (nome, telefone, e-mail, respostas de saúde) ainda
  estão em contas pessoais do Igor. Plano acordado: **recriar do zero** (novo
  formulário Formspree + nova planilha/Apps Script na conta da Dra.
  Alessandra) em vez de tentar transferir — mais simples e confiável. Depois
  de migrado, atualizar as constantes `FORMSPREE_ID`/`SHEETS_WEBHOOK`
  (`index.html`) e `FORMSPREE`/`SHEETS` (`teste.html`).
- **Acesso do Igor**: decisão tomada — desligar o acesso dele a tudo (GitHub,
  Vercel, Formspree, Apps Script), **exceto o domínio** (ver acima). Fazer
  isso só depois de confirmar que os novos Formspree/planilha estão
  funcionando em produção — não revogar acesso antes de validar.

## Arquitetura

Site estático, sem framework e sem build step — HTML/CSS/JS puro. Dois
arquivos de página:

- `index.html` — site principal (SEO com Schema.org/FAQPage, seção de
  produtos, formulário de lead, quiz STOP-BANG embutido, widget de WhatsApp)
- `teste.html` — página standalone do "Teste do Sono" (mesmo quiz STOP-BANG
  de `index.html`, extraído para funil de captação isolado)
- `vercel.json` — headers de segurança (X-Frame-Options, nosniff, etc.) e
  cache longo para assets estáticos; `cleanUrls: true`

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

### 4. Guard morto bloqueia o envio de leads em `index.html` (bug confirmado)

Em `index.html`, tanto o envio ao Formspree quanto ao Google Sheets estão
dentro de um `if` que compara a constante ao **próprio valor literal**:

```js
const FORMSPREE_ID = 'https://formspree.io/f/mlgpyqry';
if (FORMSPREE_ID !== 'https://formspree.io/f/mlgpyqry') { /* fetch nunca roda */ }
```

O mesmo padrão existe para `SHEETS_WEBHOOK`. Essa condição é **sempre falsa**
— na prática, o quiz STOP-BANG embutido na página principal (`index.html`,
diferente de `teste.html`) muito provavelmente **nunca enviou um lead
sequer** para Formspree ou para a planilha, silenciosamente, desde que o ID
real foi preenchido ali (o usuário só percebe o redirecionamento pro
WhatsApp, que funciona independente, mascarando a falha). `teste.html` não
tem esse guard e funciona normalmente. Corrigir removendo o `if` (ou
comparando contra um placeholder genérico de verdade) da próxima vez que
essas constantes forem atualizadas — está previsto para acontecer junto da
migração do Formspree/Sheets para a conta da Dra. Alessandra.

## Pendências conhecidas

- **Sem `sitemap.xml` nem `robots.txt`** no repositório — impacto de SEO/
  indexação a avaliar.
- O repositório de trabalho deve continuar fora de pastas sincronizadas por
  Google Drive/Dropbox/OneDrive (o cliente de sync injeta `desktop.ini` em
  toda subpasta do `.git`, corrompendo refs). Local atual de trabalho:
  `C:\Users\<usuário>\projetos\guaracare-site`.
