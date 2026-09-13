# GuaraCare — site institucional

## O negócio

GuaraCare é uma empresa de produtos médicos em Guarapuava/PR, especializada em
terapia do sono: CPAP, BiPAP e máscaras (marca ResMed, entre outras). Atende
diagnóstico, locação e venda de equipamentos, com acompanhamento clínico
(ajuste de pressão, adaptação, suporte via WhatsApp e app myAir).

- Contato principal: WhatsApp (42) 98866-5313
- Site: landing page única (`index.html`) + funil standalone de captação de
  lead via quiz STOP-BANG (`teste.html`)

## Estado da migração de conta (em andamento)

O repositório está hospedado em `github.com/igorbigao/guaracare`, na conta do
Igor Lachowski (desenvolvedor original). **Está em andamento a migração para
a conta da Dra. Alessandra Negrão** (dona do negócio). Enquanto a migração
não for concluída:

- Não assuma que o remote `origin` aponta para a conta final.
- Não assuma que os domínios/projetos na Vercel já estão na conta da Dra.
  Alessandra — confirme antes de qualquer deploy ou mudança de DNS.
- Qualquer ação que dependa de permissões de admin do repo/Vercel pode
  precisar ser feita pelo Igor até a transferência ser concluída.

## Arquitetura

Site estático, sem framework e sem build step — HTML/CSS/JS puro, hospedado
na Vercel. Dois arquivos de página:

- `index.html` — site principal (SEO com Schema.org/FAQPage, seção de
  produtos, formulário de lead, quiz STOP-BANG embutido, widget de WhatsApp)
- `teste.html` — página standalone do "Teste do Sono" (mesmo quiz STOP-BANG
  de `index.html`, extraído para funil de captação isolado)
- `vercel.json` — headers de segurança (X-Frame-Options, nosniff, etc.) e
  cache longo para assets estáticos; `cleanUrls: true`

Integrações de captação de lead (Formspree para e-mail, Google Apps Script
para Google Sheets) são chamadas via `fetch` direto do client, sem backend
próprio. As constantes variam de nome entre os dois arquivos:

- `index.html`: `FORMSPREE_ID` (linha ~1347)
- `teste.html`: `FORMSPREE` e `SHEETS` (linhas ~406–407)

Ao alterar uma integração, lembre de checar **os dois arquivos** — a lógica
foi duplicada, não compartilhada.

## Armadilhas técnicas já mapeadas

### 1. Cloudflare Rocket Loader quebra JS inline

Se a zona Cloudflare do domínio tiver o Rocket Loader ativo, ele adia/altera
a execução de `<script>` inline por padrão, o que quebra qualquer lógica que
dependa de rodar imediatamente (event listeners, inicialização do quiz, etc).
A defesa é o atributo `data-cfasync="false"` na tag `<script>`.

- `index.html` tem a proteção: `<script data-cfasync="false">` (linha 1233).
- **`teste.html` NÃO tem** — o `<script>` da linha 404 está desprotegido.
  Isso é uma inconsistência real: se o Rocket Loader estiver ligado, o quiz
  de `teste.html` pode falhar silenciosamente em produção mesmo com
  `index.html` funcionando normalmente. Ao mexer em `teste.html`, aplicar a
  mesma proteção antes de investigar "por que só essa página quebra".

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

## Pendências conhecidas

- **Sem `sitemap.xml` nem `robots.txt`** no repositório — impacto de SEO/
  indexação a avaliar.
- `index.html` tem um trecho de dead code aparente: a constante
  `FORMSPREE_ID` é comparada contra o próprio valor literal
  (`if (FORMSPREE_ID !== 'https://formspree.io/f/mlgpyqry')`), condição que
  nunca é verdadeira — investigar se é resquício de placeholder antes do
  cadastro real no Formspree.
- O clone de trabalho **não deve ficar dentro de uma pasta sincronizada por
  Google Drive/Dropbox/OneDrive**: o cliente de sync injeta `desktop.ini` em
  toda subpasta, incluindo dentro de `.git/refs/`, `.git/objects/` etc., o
  que corrompe refs do Git (`git branch -a` e `git log --all` já quebraram
  por esse motivo neste projeto). Manter o repositório em uma pasta local
  comum (ex: `C:\Users\<usuário>\projetos\`) e usar o Drive apenas para
  documentos administrativos, não para o código.
