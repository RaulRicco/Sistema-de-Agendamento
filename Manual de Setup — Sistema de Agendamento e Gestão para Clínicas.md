# Manual de Setup — Sistema de Agendamento e Gestão para Clínicas

Oct 1, 2026 · @Raul

## 1. Visão geral

Um setup completo para uma nova clínica leva de 2 a 4 horas de trabalho, mais a espera da verificação de DNS (de minutos a algumas horas). Este manual segue a ordem exata que funcionou na Ello Clinic e registra os erros que já aconteceram, para não se repetirem.

**O que cada clínica recebe:**

| Parte | Quem usa | Endereço | O que faz |
| --- | --- | --- | --- |
| Site de agendamento | Clientes | `/` | Agendamento em 5 etapas, sem login |
| Gerenciar agendamento | Cliente (link do e-mail) | `/agendamento/<código>` | Ver, remarcar, cancelar, adicionar à agenda |
| Área do cliente | Cliente (link por e-mail, sem senha) | `/minha-conta` | Próximos atendimentos, histórico, pacotes, saldo de cashback |
| Painel | Equipe da clínica | `/admin` | Agenda, clientes, serviços, equipe, marketing, financeiro, relatórios, configurações |

**Como o sistema é feito:** React + Vite no navegador; um Cloudflare Worker (Hono) serve o site e a API; banco D1 (SQLite); arquivos no R2; tarefas automáticas a cada 15 minutos (lembretes, recorrências, validade do cashback); e-mails pelo Resend. Tudo roda no plano gratuito da Cloudflare para uma clínica pequena ou média.

**Onde está o modelo:** o projeto-base é o repositório da Ello (`github.com/elloclinicdev/ello-clinic`, privado). Cada clínica nova recebe uma cópia própria, com contas próprias. Nunca publique duas clínicas no mesmo Worker ou banco.

**Duas regras que valem para o manual inteiro:**

- Cada clínica tem suas próprias contas (Cloudflare, GitHub, Resend). Antes de qualquer comando, confira em qual conta você está logado.
- Chaves e senhas nunca vão para o GitHub, para o chat nem para a linha de comando: sempre no campo de valor de um *secret* ou num prompt que esconde o que é digitado.

## 2. Pré-requisitos

Antes de começar, a clínica precisa ter as contas abaixo, todas em nome dela (de preferência num Gmail criado só para isso, como `elloclinicdev@gmail.com`).

| Conta / acesso | Para quê | Obrigatório? | Observação |
| --- | --- | --- | --- |
| Cloudflare | Hospedar o sistema, banco D1, arquivos R2 | Sim | Ativar o R2 no painel exige cartão, mas o uso fica no gratuito |
| GitHub | Guardar o código (repositório privado) | Sim | Uma conta por clínica |
| Domínio próprio | Remetente dos e-mails e, opcionalmente, endereço do site | Sim, para e-mail | Ideal: o DNS do domínio na mesma conta Cloudflare |
| Resend | Enviar os e-mails | Sim | Grátis até 3.000 e-mails/mês |
| Meta (Gerenciador de Eventos) | Pixel + API de Conversões | Se anunciar no Meta | Pixel ID + token |
| Google Ads | Tag de conversão | Se anunciar no Google | ID `AW-` + rótulo |

**No seu Mac:**

- Node 20 ou mais novo (`node -v`), npm, Git.
- GitHub CLI (`gh`), instalado via Homebrew.
- O Wrangler vem dentro do projeto; use sempre `npx wrangler`.

**Onde guardar o projeto:** numa pasta fora do iCloud, como `~/Developer/<nome-da-clinica>`. Nunca em `Documentos` ou `Mesa`: o iCloud remove arquivos do disco para liberar espaço, e o sistema quebra (isso aconteceu no setup da Ello).

## 3. Questionário para o cliente

Envie estas perguntas antes do setup; com as respostas em mãos, o setup não para no meio. A coluna da direita mostra a resposta da Ello como exemplo.

| # | Pergunta | Exemplo (Ello Clinic) |
| --- | --- | --- |
| 1 | Nome da clínica e slogan | Ello Clinic · “um elo entre você e seu bem estar” |
| 2 | Logo (SVG ou PNG em alta) e cores (principal, fundo claro, texto escuro, em hex) | Símbolo de dois elos · `#EA7F64`, `#F1EEED`, `#3A2E2B` |
| 3 | Fontes (títulos e texto, do Google Fonts) ou “sugerir pelo logo” | Cormorant Garamond · Nunito Sans |
| 4 | Telefone, WhatsApp (com 55), e-mail de contato, endereço, Instagram | (61) 99306-2431 · Planaltina-GO · @ello.clinic |
| 5 | Domínio para os e-mails (e, se quiser, para o site) e quem administra o DNS | elloclinicgo.com.br (DNS na Cloudflare) |
| 6 | Serviços: categoria, nome, duração, preço, intervalo de higienização, adicionais, se é em grupo | Limpeza de pele, 60 min, R$ 180, +15 min |
| 7 | Profissionais: nome, cargo, e-mail, serviços que faz, jornada semanal com almoço, comissão % | Ana, seg–sex 9h–18h, almoço 12h–13h, 40% |
| 8 | Pacotes de sessões (serviço, nº de sessões, preço, validade) | Drenagem, 10 sessões, R$ 1.300, 180 dias |
| 9 | Confirmação dos agendamentos online: automática ou manual? | Automática |
| 10 | Antecedência mínima, até quantos dias à frente, prazo para a cliente cancelar | 2 h · 60 dias · 24 h |
| 11 | Perguntas extras do formulário | Alergias? Gestante? Como nos conheceu? |
| 12 | Contas financeiras e saldo inicial; formas de pagamento e taxas | Banco, Caixa, Maquininha · PIX 0%, débito 1,49%, crédito 3,49% |
| 13 | Categorias de despesa | Aluguel, salários, insumos, marketing, impostos… |
| 14 | Anúncios: Pixel ID e token da API de Conversões; Google Ads ID e rótulo | Pixel 420378614399742 |
| 15 | Cashback: desligado, percentual (qual %) ou por atendimentos (a cada N, quanto em R$); validade | 5% de volta, 180 dias |
| 16 | E-mail que recebe os avisos da recepção | contato@… |

As respostas 1 a 5 e 9 a 10 vão para o arquivo de respostas do setup (passo 1). As respostas 6 a 8 e 11 a 13 vão para os dados iniciais (passo 2). O restante é configurado pelo painel depois de publicar.

## 4. Passo 1 — Copiar o modelo e configurar a marca

O comando `npm run setup` troca nome, cores, fontes, contatos e regras sem editar código. O resto do passo é copiar o modelo limpo e trocar a logo.

1. Copie o modelo para uma pasta nova (troque `nova-clinica` pelo nome curto da clínica, sem acentos):

   ```bash
   git clone https://github.com/elloclinicdev/ello-clinic.git ~/Developer/nova-clinica
   cd ~/Developer/nova-clinica && rm -rf .git .wrangler dist && git init -b main
   ```
2. Instale as dependências com `npm ci`. Se aparecer `EACCES` no cache do npm, rode antes `sudo chown -R $(id -u):$(id -g) ~/.npm`.
3. Copie o arquivo de respostas e preencha com as respostas 1 a 5, 9 e 10 do questionário:

   ```bash
   cp setup.answers.example.json setup.answers.json
   ```

   Campos: `name`, `tagline`, `domain`, `phone`, `whatsapp`, `email`, `mailFrom`, `address`, `instagram`, `timezone`, `primary`, `surface`, `text`, `headingFont`, `bodyFont`, `minNoticeHours`, `maxAdvanceDays`, `cancelWindowHours`, `autoApprove`. O `mailFrom` precisa usar o domínio que será verificado no Resend (ex.: `agenda@elloclinicgo.com.br`).
4. Rode o setup:

   ```bash
   npm run setup -- --answers setup.answers.json
   ```

   Ele gera `clinic.config.ts`, o tema (`web/src/theme/_tokens.scss`), o favicon e troca no `wrangler.jsonc` o nome do Worker, do banco e do bucket.
5. Troque a logo: o símbolo fica em `web/src/components/Logo.tsx` (SVG desenhado em código) e o favicon em `scripts/gen-theme.mjs`. Se o cliente mandar só PNG, peça o vetor ou vetorize antes.
6. Confira se o setup **não** apagou os dados da Ello no `wrangler.jsonc`: `account_id`, `database_id` e `APP_URL` ainda são os da Ello e precisam ser trocados no passo 3. Publicar sem trocar enviaria o sistema para a conta da Ello.

O arquivo `setup.answers.json` fica fora do GitHub (está no `.gitignore`).

## 5. Passo 2 — Dados iniciais e teste local

Serviços, equipe e financeiro iniciais ficam em `seed/clinic-data.json`; o script `scripts/build-seed.mjs` transforma esse arquivo no SQL que vai para o banco. Tudo pode ser editado depois pelo painel, mas começar certo poupa retrabalho.

| Bloco do arquivo | O que preencher | Formato importante |
| --- | --- | --- |
| `location` | Nome e endereço da unidade | — |
| `serviceCategories` | `key`, `name`, `description` | `key` sem espaço, usado pelos serviços |
| `services` | `key`, `category`, `name`, `description`, `duration` (min), `price` (R$), `bufferAfter`, `color`, `extras`, `capacity` | Preço em reais (180), não em centavos; `capacity` > 1 = serviço em grupo |
| `packages` | `service` (key), `name`, `sessions`, `price`, `validityDays` | — |
| `staff` | `key`, `name`, `role`, `email`, `color`, `commission` (%), `services` (keys), `schedule`, `breaks` | Dias: 0 = domingo … 6 = sábado; `"1-5": [["09:00","18:00"]]` |
| `holidays` | `["MM-DD", "Nome"]` | Repetem todo ano; inclua feriados municipais |
| `customFields` | `label`, `type`, `options`, `required` | Tipos: text, textarea, select, radio, checkbox, date |
| `coupons` | `code`, `type` (percent/fixed), `value`, `oncePerCustomer` | — |
| `accounts` | `name`, `type`, `default`, `opening` | Exatamente uma com `default: true` |
| `paymentMethods` | `name`, `code`, `fee` (%), `account` | `account` = nome exato de uma conta |
| `incomeCategories` / `expenseCategories` | `name`, `color`, `system` | Não apague as que têm `system` (services, packages, card\_fees, commissions…) |

**Teste local antes de publicar:**

```bash
npm run db:migrate:local
ADMIN_EMAIL=voce@exemplo.com ADMIN_PASSWORD=senha-de-teste node scripts/build-seed.mjs --demo
npx wrangler d1 execute DB --local --file=seed/seed.sql
npm run dev   # abre em http://localhost:5173 e o painel em /admin
```

O `--demo` apaga e recria o banco **local** com clientes e agendamentos fictícios, úteis para ver o layout cheio. Em produção ele nunca deve ser usado (passo 3). No local, os e-mails ficam só registrados (modo `log`) e aparecem em Configurações → Notificações.

Rode também `npm test` (21 testes) e `npm run typecheck`; os dois precisam passar sem erro.

## 6. Passo 3 — Cloudflare: banco, arquivos e publicação

Todo comando deste passo age na conta em que o Wrangler estiver logado; por isso o primeiro item é conferir a conta, e o `account_id` no `wrangler.jsonc` trava o projeto nela.

1. **Crie a conta Cloudflare da clínica** e ative o R2: painel → *R2 Object Storage* → *Ativar* (pede cartão; o uso fica no gratuito).
2. **Faça login na conta certa**, pelo modo por código (o modo padrão, que abre `localhost:8976`, falha neste Mac):

   ```bash
   npx wrangler login --device
   npx wrangler whoami   # confira o e-mail e copie o Account ID
   ```

   Para quem cuida de várias clínicas, prefira um token só do projeto: crie um *API Token* (modelo “Editar Cloudflare Workers” + permissão *Conta → D1 → Editar*) e salve em `.env` como `CLOUDFLARE_API_TOKEN=` e `CLOUDFLARE_ACCOUNT_ID=`. Os comandos `npm run deploy` e `npm run cf -- <comando>` usam esse token, sem depender do login global.
3. **Trave a conta no projeto:** no `wrangler.jsonc`, troque `account_id` pelo Account ID da nova clínica.
4. **Crie o banco:**

   ```bash
   npx wrangler d1 create nova-clinica-db --location enam
   ```

   Não existe região na América do Sul; `enam` (leste dos EUA) é a mais próxima. Copie o `database_id` exibido para o `wrangler.jsonc`.
5. **Crie o bucket de arquivos:** `npx wrangler r2 bucket create nova-clinica-files` (o nome precisa bater com `bucket_name` no `wrangler.jsonc`).
6. **Defina o endereço público:** em *Workers & Pages*, veja o subdomínio `workers.dev` da conta. Em `wrangler.jsonc → vars`, coloque `APP_URL` = `https://<nome-do-worker>.<subdominio>.workers.dev` e deixe `EMAIL_PROVIDER` = `resend`. Os links dos e-mails usam o `APP_URL`.
7. **Estrutura do banco de produção:** `npm run db:migrate:remote` (aplica 0001, 0002 e 0003).
8. **Dados iniciais de produção (só uma vez):**

   ```bash
   node scripts/build-seed.mjs
   npx wrangler d1 execute DB --remote --file=seed/seed.sql
   ```

   Sem `--demo`, sem `--reset` e sem `ADMIN_EMAIL`: produção começa sem clientes fictícios e sem usuário. Num banco que já tem dados, esse comando falha de propósito em vez de apagar.
9. **Publique:** `npm run deploy`. Nos primeiros 1–2 minutos algumas rotas podem responder `error code: 1042`; é a propagação, e passa sozinho.
10. **Crie o administrador imediatamente:** abra `/admin`. Enquanto não existir usuário, a primeira pessoa que abrir essa página cria o admin.
11. **Domínio próprio (opcional, recomendado):** com o domínio na mesma conta Cloudflare, vá em *Worker → Settings → Domains & Routes → Add Custom Domain* (ex.: `agenda.clinica.com.br`). Depois troque o `APP_URL`, publique de novo e adicione o domínio nas permissões de tráfego do Pixel (passo 7).

## 7. Passo 4 — GitHub

O código de cada clínica vai para um repositório **privado** na conta GitHub dela. Como o seu Mac tem várias contas, o repositório fica amarrado à conta da clínica e não interfere nas outras.

1. Adicione a conta da clínica ao GitHub CLI: `gh auth login --web` (copie o código exibido e autorize no navegador logado na conta da clínica).
2. Confira o que vai subir com `git status`. Nunca podem aparecer: `.env`, `.dev.vars`, `.wrangler/`, `node_modules/`, `dist/`, `seed/seed.sql`, `setup.answers.json`. Todos já estão no `.gitignore`; se algum aparecer, pare e corrija.
3. Faça o primeiro commit com o autor da conta da clínica:

   ```bash
   git config user.name "conta-da-clinica"
   git config user.email "ID+conta-da-clinica@users.noreply.github.com"
   git add -A && git commit -m "Setup inicial"
   gh repo create conta-da-clinica/nova-clinica --private --source=. --remote=origin --push
   ```
4. Amarre o repositório à conta da clínica e volte a conta ativa do `gh` para a sua de sempre:

   ```bash
   git config --local --add credential.https://github.com.helper ''
   git config --local --add credential.https://github.com.helper '!f() { test "$1" = get && echo username=conta-da-clinica && echo "password=$(gh auth token -h github.com -u conta-da-clinica)"; }; f'
   gh auth switch -h github.com -u sua-conta-de-sempre
   git fetch   # precisa funcionar mesmo com outra conta ativa
   ```

O ID numérico do e-mail `noreply` sai de `gh api user --jq .id` com a conta da clínica ativa.

## 8. Passo 5 — E-mails (Resend)

Sem a chave do Resend, o sistema só registra os e-mails no painel e não envia nada. O remetente precisa ser do domínio verificado no Resend, e a chave vai num *secret* chamado exatamente `RESEND_API_KEY`.

1. Crie a conta em **resend.com** → *Domains → Add Domain* → domínio da clínica, região **São Paulo (sa-east-1)**.
2. Cadastre no DNS do domínio os 3 registros que o Resend mostrar: um MX e um TXT no subdomínio `send` e um TXT em `resend._domainkey`. Eles não mexem no site nem nas caixas de e-mail existentes. Depois clique em *Verify*.
3. Adicione o DMARC no mesmo DNS: tipo `TXT`, nome `_dmarc`, conteúdo `v=DMARC1; p=none;`. Sem ele, Gmail e Outlook tendem a mandar para o spam.
4. Confirme que o remetente bate com o domínio: `clinic.config.ts → mail.from` (ex.: `agenda@elloclinicgo.com.br`). Se mudar, publique de novo.
5. No Resend: *API Keys → Create API Key*, permissão **Sending access**, restrita ao domínio. A chave começa com `re_` e aparece uma única vez.
6. Salve a chave como *secret*, por um destes dois caminhos:
   - Terminal: `npx wrangler secret put RESEND_API_KEY` e cole a chave **quando ele pedir** “Enter a secret value”.
   - Painel Cloudflare: *Workers & Pages → \<worker> → Settings → Variables and Secrets → Add*: **Type** `Secret`, **Variable name** `RESEND_API_KEY`, **Value** a chave; depois *Deploy*.
7. Confira: `npx wrangler secret list` deve mostrar `RESEND_API_KEY`. Se aparecer um nome começando com `re_`, a chave entrou no lugar do nome: apague esse segredo, gere uma chave nova no Resend e repita o passo 6.
8. Teste: agende pelo site com um e-mail seu e veja em *Configurações → Notificações → Últimos envios*. O status certo é **enviado**; “registrado” significa que a chave não está sendo lida.

**Quem recebe o quê:**

| Movimentação | Cliente | Profissional | Clínica |
| --- | --- | --- | --- |
| Agendou pelo site | Confirmação (ou “pedido recebido”, se a aprovação for manual) | Novo agendamento | Novo agendamento online |
| Remarcou pelo site | Remarcado | Remarcado | Remarcado pela cliente |
| Cancelou pelo site | Cancelado | Cancelado | Cancelado pela cliente |
| Recepção agendou, remarcou ou cancelou no painel | Sim (opção “avisar por e-mail”) | Sim | Não |
| 24 h antes (ajustável) | Lembrete | — | — |

O e-mail da profissional é o do cadastro dela na Equipe. O da clínica é o de *Configurações → Notificações → Avisos para a clínica* (vazio = e-mail de contato da clínica; aceita vários separados por vírgula).

## 9. Passo 6 — Primeiro acesso e ajustes no painel

Depois de publicado, quase tudo se ajusta pelo painel em `/admin`, sem mexer em código. Faça nesta ordem:

1. **Administrador:** abra `/admin` e preencha nome, e-mail e senha forte (mínimo 8 caracteres). Essa tela só aparece enquanto não existe nenhum usuário.
2. **Configurações → Geral:** nome, telefone, WhatsApp (só números, com 55), e-mail, endereço e Instagram. Esses dados aparecem no rodapé do site, nos botões de WhatsApp, nos e-mails e no arquivo de agenda.
3. **Configurações → Agendamento:** confirmação automática ou manual, intervalo entre horários, antecedência mínima e máxima, prazo de cancelamento, hora do lembrete, opções do site (sem preferência de profissional, repetição, lista de espera, cupom, mostrar preços, e-mail e nascimento obrigatórios) e os textos do site e da LGPD. O link de agendamento está nessa tela, com botão de copiar.
4. **Serviços e Equipe:** confira cada serviço (preço, duração, intervalo, profissionais que fazem) e cada profissional (jornada, almoço, serviços, comissão, **e-mail real**). Cadastre folgas, férias e feriados locais.
5. **Configurações → Formulário:** perguntas extras do agendamento.
6. **Configurações → Notificações:** e-mail que recebe os avisos da clínica; revise os modelos de e-mail e use o botão de teste em cada um.
7. **Configurações → Usuários:** crie os logins da equipe e vincule cada profissional ao seu cadastro.
8. **Financeiro:** confira contas, saldos iniciais, formas de pagamento e taxas, categorias.

| Perfil de usuário | Pode |
| --- | --- |
| Administrador | Tudo, inclusive relatórios, configurações, usuários, anúncios e ajuste de cashback |
| Recepção | Agenda, clientes, serviços, cupons, lançamentos financeiros (sem excluir), comissões (ver) |
| Profissional | Só a própria agenda e seus clientes; não vê financeiro |

## 10. Passo 7 — Anúncios: Meta e Google Ads

Quando a cliente confirma um agendamento no site, o sistema envia o evento **AgendamentoFeito** duas vezes, com o mesmo ID: pelo navegador (Pixel/tag) e pelo servidor (API de Conversões). A Meta conta uma vez só. Tudo se configura em *Marketing → Anúncios e conversões* (só administrador).

**Meta (Pixel + API de Conversões)**

1. No Gerenciador de Eventos, copie o **ID do Pixel** e gere o **token da API de Conversões** (*Configurações → API de Conversões → Gerar token*).
2. No painel, preencha os dois campos, **ligue a chave “Ativo”** do cartão da Meta e salve. Com a chave desligada, nada funciona (aconteceu na Ello).
3. Em *Configurações → Permissões de tráfego* do Pixel, adicione à **lista de permissões** o endereço do sistema (ex.: `nova-clinica.conta.workers.dev`) e o domínio próprio, se houver. Sem isso, a Meta bloqueia o Pixel no navegador; o aviso aparece no console como “unavailable on this website due to its traffic permission settings”. Endereços `workers.dev` não podem ser verificados no Business Manager, então o domínio próprio evita esse problema.
4. Valide: copie o código da aba *Testar eventos* para o campo “Código de teste”, salve e clique em **Enviar evento de teste**. O evento deve aparecer na aba em segundos.
5. **Apague o código de teste depois.** Com ele preenchido, os agendamentos reais também vão só para a aba de teste.
6. Crie uma **conversão personalizada** com o evento `AgendamentoFeito` para otimizar as campanhas.

**Google Ads**

1. Em *Metas → Conversões → Nova ação → Site*, crie a ação “Agendamento feito” (valor: usar valores diferentes; contagem: uma). Ative as conversões otimizadas.
2. No painel, preencha o **ID `AW-`** e o **rótulo**, ligue “Ativo” e salve. Valide com a extensão Tag Assistant.
3. Envio pelo servidor (opcional): exige *developer token* aprovado pelo Google, credenciais OAuth (client ID, secret, refresh token), ID da conta e ID de uma ação de conversão do tipo importação de cliques.

**LGPD e origem dos agendamentos**

- Com “Pedir consentimento” ligado (padrão), o Pixel e a tag só carregam depois que a visitante aceita os cookies; se ela recusar, nada é enviado, nem pelo servidor.
- A aba *Origem dos agendamentos* mostra de onde veio cada agendamento (Meta Ads, Google Ads, UTM, Instagram orgânico, direto). Use links com UTM nos anúncios, ex.: `?utm_source=instagram&utm_campaign=botox-outubro`. Links diretos para um serviço: `?servico=ID&profissional=ID`.
- A aba *Eventos enviados* mostra cada envio com status (enviado, falhou, não enviado, teste) e a resposta da plataforma.

## 11. Passo 8 — Cashback (fidelidade)

Em produção o cashback começa **desligado**. Ligue pelo card *Cashback* do Dashboard (botão *Configurar*) ou em *Marketing → Cashback*.

| Configuração | O que faz | Exemplo |
| --- | --- | --- |
| Modo percentual | X% do valor efetivamente pago vira crédito | 5% de R$ 200 = R$ 10 |
| Modo por atendimentos | A cada N atendimentos pagos, crédito fixo | A cada 10, R$ 50 |
| Validade (dias) | Crédito expira depois do prazo; 0 = não expira | 180 |
| Saldo mínimo para usar | Abaixo disso, o saldo não pode ser usado | R$ 0 |
| Pagar até (% da conta) | Limite do valor de cada atendimento que pode ser pago com cashback | 100% |

**Como funciona no dia a dia:**

- O crédito nasce quando a recepção registra o pagamento de um atendimento (*Concluir e receber*).
- O uso acontece só na recepção: no registro do pagamento aparece “Usar cashback” com o saldo disponível. O valor usado entra como **desconto**, não como receita.
- O saldo é consumido do crédito que vence primeiro. Créditos vencidos expiram sozinhos (tarefa automática).
- Se a receita do atendimento for excluída, o crédito gerado é estornado e o cashback usado volta para a cliente.
- A cliente vê o saldo na área do cliente e nos e-mails de confirmação e lembrete. A ficha da cliente no painel tem a aba *Cashback*, com o extrato e ajuste manual (só administrador).

## 12. Checklist antes de entregar

Marque cada item em produção, no endereço final, antes de passar o sistema ao cliente.

- [ ] `npx wrangler whoami` mostra a conta da clínica, e o `account_id` do `wrangler.jsonc` é dela
- [ ] `npm test` e `npm run typecheck` passam
- [ ] O administrador foi criado e a senha está com o cliente
- [ ] Rodapé do site e botão de WhatsApp mostram os dados da clínica (não os da Ello nem os de exemplo)
- [ ] Logo, cores e fontes estão certas no site, no painel e no e-mail
- [ ] Serviços, preços, durações e profissionais conferidos; e-mails reais nas profissionais
- [ ] Jornadas, almoços, folgas e feriados locais cadastrados; os horários do site batem com a realidade
- [ ] Agendamento de teste pelo celular (tela de 390 px) chega até “Tudo certo!”
- [ ] Os 3 e-mails do agendamento aparecem como **enviado** e chegaram na caixa de entrada (não no spam)
- [ ] Remarcar e cancelar pelo link do e-mail funcionam; dentro do prazo mínimo o sistema recusa
- [ ] No painel: concluir e receber gera receita, taxa do cartão e comissão; o saldo da conta muda
- [ ] DRE e fluxo de caixa mostram o lançamento
- [ ] Pixel: o evento de teste chegou; o código de teste foi apagado; o domínio está nas permissões de tráfego
- [ ] Cashback configurado (ou desligado de propósito)
- [ ] Agendamentos de teste cancelados; clientes de teste excluídos
- [ ] Código no GitHub privado da clínica; `git status` limpo

## 13. Referência: todas as funções do sistema

Use esta seção para conferir que nada ficou de fora e para responder dúvidas do cliente. A especificação técnica completa fica em `docs/SPEC.md` no repositório.

**Site de agendamento e área do cliente**

| Função | Detalhe |
| --- | --- |
| Agendamento em 5 etapas | Procedimento → profissional → data e hora → dados → confirmação |
| Filtro por categoria | Pílulas no topo da lista de serviços |
| Adicionais e grupo | Extras somam tempo e valor; serviços em grupo aceitam várias pessoas no mesmo horário |
| Sem preferência de profissional | Mostra todos os horários; escolhe a profissional com menos agendamentos no dia |
| Calendário inteligente | Só destaca dias com vaga; pula para o primeiro mês com horário; horários por manhã/tarde/noite |
| Repetir horário | Semanal, quinzenal ou mensal, até 24 vezes; datas sem vaga são puladas e avisadas |
| Lista de espera | Quando não há horário bom |
| Formulário | Nome, WhatsApp, e-mail, nascimento (opcional), perguntas extras por serviço, cupom, aceite LGPD |
| Confirmação | Resumo, “adicionar à agenda” (.ics), link para gerenciar |
| Gerenciar pelo link | Remarcar e cancelar até o prazo; depois disso mostra o WhatsApp |
| Área do cliente | Acesso por link no e-mail (sem senha, vale 30 min): próximos, histórico, pacotes, cashback |
| Aviso de cookies | Pixel e tag só carregam com aceite |

**Painel**

| Área | Funções |
| --- | --- |
| Dashboard | Agendamentos hoje e 7 dias, faturamento e ticket médio, resultado do mês, gráfico de receitas × despesas (12 meses), faltas, serviços mais agendados, próximos atendimentos, card de cashback |
| Agenda | Dia, semana, mês e lista; cores por profissional; arrastar para remarcar; clicar no vazio para agendar; mostrar cancelados |
| Agendamento (janela) | Detalhes e respostas, observação interna, WhatsApp, remarcar, encaixe, confirmar, recusar, concluir e receber, falta, cancelar, reenviar e-mail |
| Agendamentos | Filtros, ações em lote, exportação CSV |
| Clientes | Busca, CSV, ficha com histórico, pacotes, financeiro, cashback, vender pacote |
| Serviços | Categorias, preço e duração por profissional, adicionais, intervalos, capacidade, antecedências, pacotes |
| Equipe | Jornada com vários períodos e intervalos por dia, folgas e férias, feriados, comissão, foto |
| Marketing | Anúncios e conversões, cashback, cupons, lista de espera |
| Financeiro | Visão geral, receitas, despesas (a pagar e a receber, recorrência, anexo), transferências, comissões (pagamento em lote), contas, fornecedores, categorias, formas de pagamento com taxa, conciliação |
| Relatórios | DRE (caixa ou competência), fluxo de caixa, por serviço, por profissional, por categoria, melhores clientes; CSV e impressão |
| Configurações | Geral, agendamento, formulário, notificações (modelos editáveis, teste, histórico de envios), usuários, meu perfil |

**Regras que o sistema aplica sozinho**

- Um horário só aparece se cabe inteiro na jornada, não cruza almoço, respeita os intervalos antes e depois de outros atendimentos, não é feriado nem folga, e está entre a antecedência mínima e a máxima.
- Duas pessoas nunca conseguem o mesmo horário: o servidor confere de novo no momento de gravar.
- Concluir e receber gera a receita, a despesa da taxa da maquininha e a comissão da profissional; atendimento por pacote consome uma sessão sem gerar receita nova.
- Tarefas automáticas a cada 15 minutos: lembretes, despesas e receitas recorrentes, validade do cashback, limpeza de sessões.
- Senhas criptografadas; tokens de anúncios e do Resend nunca aparecem no painel nem no site.

## 14. Erros que já aconteceram e como resolver

Todos os itens abaixo aconteceram no setup da Ello. A maioria vem de conta errada ou de campo trocado, não do código.

| Sintoma | Causa | Como resolver |
| --- | --- | --- |
| Arquivos com 0 bytes, testes quebram do nada | Projeto dentro de `Documentos` (iCloud tirou os arquivos do disco) | Projeto em `~/Developer`; `npm ci` de novo |
| `npm install` falha com `EACCES` | Cache do npm com arquivos de root | `sudo chown -R $(id -u):$(id -g) ~/.npm` |
| Login do Wrangler termina em “Não é possível acessar localhost” | O retorno em `localhost:8976` não chega ao processo | `npx wrangler login --device` (código no navegador) |
| `Authentication error [code: 10000]` no deploy | Wrangler logado em outra conta (o `account_id` travado impediu publicar na conta errada) | `npx wrangler whoami`; login na conta da clínica ou token em `.env` |
| `Please enable R2` (código 10042) | R2 não ativado na conta | Ativar o R2 no painel e repetir |
| `d1 create` recusa `--location sam` | Não existe região na América do Sul | Usar `--location enam` |
| `error code: 1042` logo após publicar | Propagação da nova versão | Esperar 1–2 minutos |
| Banco local vazio depois de configurar produção | O banco local é separado por `database_id` | Recriar os dados locais (passo 2) |
| Site mostra contatos antigos | Cache do navegador | `Cmd+Shift+R`; os dados vêm de *Configurações → Geral* |
| Pixel não dispara | Chave “Ativo” da Meta desligada | Ligar e salvar (o painel agora avisa em vermelho) |
| Pixel carrega mas não envia, console fala em “traffic permission” | Endereço fora da lista de permissões do Pixel | Adicionar o domínio em *Permissões de tráfego*; esperar até 20 min |
| E-mails ficam como “registrado” | Falta o secret `RESEND_API_KEY` | Passo 5, item 6 |
| `secret list` mostra um nome começando com `re_` | Chave digitada no campo de nome (aconteceu duas vezes) | Apagar o segredo, gerar chave nova no Resend, salvar com o nome `RESEND_API_KEY` |
| Resend recusa o envio | Remetente de outro domínio (`elloclinic.com.br` em vez do verificado `elloclinicgo.com.br`) | Ajustar `mail.from` no `clinic.config.ts` e publicar |
| `secret put` termina em `fetch failed` | Falha momentânea de rede ou VPN | Repetir; desligar VPN; usar o Terminal do Mac |
| Cliente não consegue cancelar | Está dentro do prazo mínimo (padrão 24 h) | Comportamento esperado; a recepção cancela pelo painel |

## 15. Manutenção e comandos úteis

Toda alteração segue a mesma sequência: testar no local, publicar e salvar no GitHub. Antes de publicar, confira a conta com `npx wrangler whoami`.

| Para quê | Comando (na pasta do projeto) |
| --- | --- |
| Rodar no computador | `npm run dev` |
| Testes e verificação de tipos | `npm test` · `npm run typecheck` |
| Publicar nova versão | `npm run deploy` |
| Aplicar uma migration nova em produção | `npm run db:migrate:remote` |
| Ver os segredos (só nomes) | `npx wrangler secret list` |
| Trocar a chave do Resend | `npx wrangler secret put RESEND_API_KEY` |
| Backup do banco | `npx wrangler d1 export <nome>-db --remote --output=backup-AAAA-MM-DD.sql` |
| Ver logs ao vivo | `npx wrangler tail` |
| Salvar no GitHub | `git add -A && git commit -m "…" && git push` |
| Atualizar o Wrangler | `npm i -D wrangler@latest` |

- **Backup:** o D1 guarda um histórico de 30 dias (*Time Travel*), mas faça um export mensal e guarde fora da Cloudflare.
- **Trocar chaves:** gere a chave nova na plataforma (Resend, Meta), salve no sistema e só depois apague a antiga, para os envios não pararem.
- **Melhorias no modelo:** faça a melhoria no repositório-modelo e leve para cada clínica com `git pull` de um remoto `modelo`, testando antes de publicar. Migrations novas sempre ganham um número novo (`0004_…sql`); nunca edite uma que já foi aplicada.
- **Manual técnico:** `README.md`, `docs/SPEC.md`, `docs/DEPLOY.md` e `SETUP_PROMPT.md` (prompt para refazer o setup com um agente de IA) ficam no repositório.
