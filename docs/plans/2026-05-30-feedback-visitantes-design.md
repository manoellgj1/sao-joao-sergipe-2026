# Design — Feedback dos visitantes (São João SE 2026)

**Data:** 2026-05-30
**Status:** Aprovado (Manoel) — pronto para plano de implementação

## Objetivo

Permitir que visitantes deixem feedback (erro, sugestão ou elogio) no site, e que
Manoel + equipe recebam isso **por email e numa planilha**, para saber o que melhorar.

## Restrições

- Grátis e ilimitado
- Sem cookies (LGPD-friendly)
- Mobile-first (público de festa), baixa fricção
- Formulário **nativo** no site (sem iframe), mantendo o visual laranja/creme
- Site é estático (HTML único) na Vercel — sem backend próprio

## Abordagem escolhida: Google Apps Script

Endpoint grátis do Google que recebe o POST do formulário e faz duas coisas:
1. Acrescenta uma linha numa **Planilha Google**
2. Envia um **email** de notificação (`MailApp.sendEmail`)

Único caminho que entrega email + planilha 100% grátis, ilimitado, com dados
sob controle do Manoel e formulário nativo.

**Alternativas descartadas:**
- Web3Forms — planilha só no plano pago; grátis apaga dados após 30 dias.
- Tally — bom free tier, mas formulário em iframe (não nativo) + outra conta.
- Web3Forms + Zapier — duas ferramentas, Zapier grátis limitado a 100/mês.

## UX

- **Gatilho:** botão flutuante fixo (ícone 💬) no canto inferior direito, acima
  da bottom-nav. Abre uma **gaveta** (reusa o componente `.bottom-sheet` + `.sheet-overlay`).
- **Formulário (na gaveta):**
  - Tipo: 3 pílulas → 🐞 Erro · 💡 Sugestão · ❤️ Elogio (default: Sugestão)
  - Mensagem: textarea (obrigatório)
  - Contato: input opcional ("email ou WhatsApp, só se quiser resposta")
  - Botão Enviar + microtexto LGPD
- **Estados:** loading → sucesso ("Valeu! Recebemos 🙌", fecha sozinho) → erro (retry).

## Dados capturados

Por feedback: `Data/Hora` (timestamp do servidor) · `Tipo` · `Mensagem` ·
`Contato` (opcional) · `Página` (`location.hash`, ex: `/cidade/aracaju`) ·
`Dispositivo` (userAgent). Não identifica a pessoa além do contato voluntário.

## Entregáveis ao Manoel

- **Planilha:** colunas Data/Hora · Tipo · Mensagem · Contato · Página · Dispositivo
- **Email:** assunto `[Feedback São João] {emoji} {Tipo}`, corpo formatado;
  pode ir para mais de um destinatário (equipe).

## Telemetria

Eventos Umami: `feedback_aberto` e `feedback_enviado` (com propriedade `tipo`).

## Segurança / conformidade

- Sem cookies. Contato é opcional e explícito.
- Anti-spam: campo honeypot escondido (ignora envios preenchidos por bots).
- CSP (`vercel.json`): liberar `connect-src` para `https://script.google.com` e
  `https://script.googleusercontent.com` (Apps Script redireciona entre os dois).
- A URL de deploy do Apps Script é pública por design (só aceita POST e grava/envia;
  não expõe segredos). Diferente de chaves de API — aqui não há segredo no cliente.

## Detalhe técnico a resolver na implementação

Apps Script web app não devolve headers CORS para POST cross-origin. Padrão
consolidado: enviar com `fetch` usando corpo `application/x-www-form-urlencoded`
ou `FormData` (requisição "simples", sem preflight) e/ou `mode: 'no-cors'` com
sucesso otimista. Validar no navegador real antes de concluir.

## O que Manoel precisa fornecer (pré-requisito da implementação)

1. Criar uma Planilha Google e abrir o editor de Apps Script (código + passo a passo
   serão entregues no plano).
2. Publicar como Web App ("Executar como: eu", "Quem tem acesso: qualquer pessoa") e
   autorizar.
3. Enviar: (a) a URL de deploy do Web App e (b) o(s) email(s) que recebem as notificações.

## Fora de escopo (YAGNI)

Sem nota/estrelas, sem captcha (honeypot basta), sem painel próprio, sem sistema de
resposta. Planilha + email já dão o controle desejado.
