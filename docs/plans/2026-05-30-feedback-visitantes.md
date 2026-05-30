# Feedback dos Visitantes — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Visitantes deixam feedback (erro/sugestão/elogio) numa gaveta nativa do site; cada envio vira uma linha numa Planilha Google e dispara um email para a equipe.

**Architecture:** Site estático (Vercel) com formulário nativo numa bottom-sheet. O `fetch` envia (POST `FormData`, `mode: 'no-cors'`, sucesso otimista) para um Web App do Google Apps Script, que grava na planilha (`appendRow`) e notifica por email (`MailApp.sendEmail`). Sem backend próprio, sem cookies.

**Tech Stack:** HTML/CSS/JS vanilla (single file `index.html`), Google Apps Script, Vercel (`vercel.json` headers/CSP), Umami (eventos).

**Nota sobre testes:** o projeto é um HTML único sem framework de testes. "Verificação" aqui = `node --check` de sintaxe dos `<script>` inline, `curl` contra o endpoint, e teste manual no navegador real com resultado esperado explícito.

---

## Pré-requisito (Manoel) — Backend no Google Apps Script

Sem isto a implementação não conecta. Entrego o código + passo a passo; Manoel executa uma vez e devolve a **URL de deploy** + **email(s)**.

### Passo a passo

1. Criar uma Planilha Google nova (nome ex.: `Feedback São João SE 2026`).
2. Menu **Extensões → Apps Script**.
3. Apagar o conteúdo de `Code.gs` e colar o código abaixo. Ajustar `NOTIFY_EMAILS`.
4. **Implantar → Nova implantação → Tipo: App da Web.**
   - Descrição: `feedback`
   - Executar como: **Eu**
   - Quem tem acesso: **Qualquer pessoa**
5. Autorizar (vai aparecer aviso de "app não verificado" — é o seu próprio script; clicar em *Avançado → Acessar (não seguro)*).
6. Copiar a **URL do app da Web** (`https://script.google.com/macros/s/.../exec`).
7. Enviar para o Claude: (a) essa URL e (b) o(s) email(s) que recebem (já no `NOTIFY_EMAILS`).

### `Code.gs`

```javascript
// São João SE 2026 — endpoint de feedback
const NOTIFY_EMAILS = 'manoellgj1@gmail.com'; // vírgula para vários: 'a@x.com,b@y.com'
const SHEET_NAME = 'Feedback';

function doPost(e) {
  try {
    const p = (e && e.parameter) ? e.parameter : {};
    if (p.website) return _json({ ok: true });           // honeypot: bot -> ignora
    const tipo        = (p.tipo || 'sugestao').toString().slice(0, 40);
    const mensagem    = (p.mensagem || '').toString().slice(0, 4000);
    const contato     = (p.contato || '').toString().slice(0, 200);
    const pagina      = (p.pagina || '').toString().slice(0, 200);
    const dispositivo = (p.dispositivo || '').toString().slice(0, 300);
    if (!mensagem.trim()) return _json({ ok: false, error: 'mensagem vazia' });

    const ss = SpreadsheetApp.getActiveSpreadsheet();
    let sheet = ss.getSheetByName(SHEET_NAME);
    if (!sheet) {
      sheet = ss.insertSheet(SHEET_NAME);
      sheet.appendRow(['Data/Hora', 'Tipo', 'Mensagem', 'Contato', 'Página', 'Dispositivo']);
    }
    const agora = new Date();
    sheet.appendRow([agora, tipo, mensagem, contato, pagina, dispositivo]);

    const emoji = tipo === 'erro' ? '🐞' : (tipo === 'elogio' ? '❤️' : '💡');
    MailApp.sendEmail(NOTIFY_EMAILS, '[Feedback São João] ' + emoji + ' ' + tipo,
      'Tipo: ' + tipo + '\n' +
      'Mensagem: ' + mensagem + '\n' +
      'Contato: ' + (contato || '—') + '\n' +
      'Página: ' + (pagina || '—') + '\n' +
      'Dispositivo: ' + (dispositivo || '—') + '\n' +
      'Quando: ' + agora.toLocaleString('pt-BR'));

    return _json({ ok: true });
  } catch (err) {
    return _json({ ok: false, error: String(err) });
  }
}

function doGet() { return _json({ ok: true, msg: 'feedback endpoint up' }); }

function _json(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**Verificação do backend (Claude, quando tiver a URL):**
```bash
curl -s -L -X POST "<URL>" \
  --data-urlencode "tipo=sugestao" \
  --data-urlencode "mensagem=teste de backend via curl" \
  --data-urlencode "pagina=/teste" --data-urlencode "dispositivo=curl"
```
Esperado: `{"ok":true}` + uma linha nova na planilha + 1 email. (Honeypot: repetir com `--data-urlencode "website=x"` → `{"ok":true}` mas **sem** linha nova.)

---

## Task 1: Liberar o endpoint no CSP

**Files:** Modify `sao-joao-sergipe-2026/vercel.json`

**Step 1:** No valor da `Content-Security-Policy`, na diretiva `connect-src`, acrescentar `https://script.google.com https://script.googleusercontent.com` (o Apps Script redireciona entre os dois).

**Step 2:** Validar JSON:
```bash
python3 -c "import json;json.load(open('sao-joao-sergipe-2026/vercel.json'));print('ok')"
```
Esperado: `ok`

**Step 3:** Commit `chore: libera endpoint de feedback no CSP`.

---

## Task 2: Botão flutuante + gaveta (HTML + CSS)

**Files:** Modify `sao-joao-sergipe-2026/index.html`

**Step 1 — CSS** (perto dos estilos de `.bottom-sheet`): adicionar
```css
.fab-feedback{position:fixed;right:14px;bottom:84px;z-index:60;width:48px;height:48px;border-radius:999px;border:none;background:var(--primary);color:#fff;box-shadow:0 6px 18px rgba(0,0,0,.28);font-size:20px;cursor:pointer;display:flex;align-items:center;justify-content:center}
.fb-form{display:flex;flex-direction:column;gap:.75rem;padding:0 1rem 1rem}
.fb-tipos{display:flex;gap:.5rem}
.fb-tipo{flex:1;padding:.5rem;border-radius:12px;border:1.5px solid rgba(0,0,0,.12);background:#fff;cursor:pointer;font-weight:700;font-size:.85rem}
.fb-tipo.sel{border-color:var(--primary);background:rgba(234,88,12,.08);color:var(--primary)}
.fb-text{width:100%;min-height:96px;padding:.6rem;border-radius:12px;border:1.5px solid rgba(0,0,0,.15);font:inherit;resize:vertical}
.fb-input{width:100%;padding:.6rem;border-radius:12px;border:1.5px solid rgba(0,0,0,.15);font:inherit}
.fb-hp{position:absolute;left:-9999px;width:1px;height:1px;opacity:0}
.fb-send{width:100%;padding:.8rem;border:none;border-radius:12px;background:var(--primary);color:#fff;font-weight:800;cursor:pointer}
.fb-note{font-size:.7rem;color:var(--muted-fg);text-align:center}
.fb-msg{text-align:center;padding:1.5rem 1rem;font-weight:700}
```

**Step 2 — FAB:** no template de `render()` (`root.innerHTML = \`...\``), adicionar logo após `<div id="citySheet" ...></div>` (junto dos outros sheets):
```html
<button class="fab-feedback" id="fabFeedback" aria-label="Enviar feedback">💬</button>
<div id="fbSheet" class="bottom-sheet" role="dialog" aria-label="Enviar feedback"></div>
```

**Step 3 — conteúdo da gaveta:** criar `renderFbSheetContent()` (perto de `renderCitySheetContent`) retornando o cabeçalho da sheet (mesmo padrão das outras: título + botão fechar `id="fbSheetClose"`) e:
```html
<form class="fb-form" id="fbForm" autocomplete="off">
  <div class="fb-tipos" id="fbTipos">
    <button type="button" class="fb-tipo" data-tipo="erro">🐞 Erro</button>
    <button type="button" class="fb-tipo sel" data-tipo="sugestao">💡 Sugestão</button>
    <button type="button" class="fb-tipo" data-tipo="elogio">❤️ Elogio</button>
  </div>
  <textarea class="fb-text" id="fbMensagem" placeholder="Conta pra gente..." required></textarea>
  <input class="fb-input" id="fbContato" placeholder="Email ou WhatsApp (só se quiser resposta)">
  <input class="fb-hp" id="fbWebsite" tabindex="-1" aria-hidden="true" placeholder="Não preencha">
  <button type="submit" class="fb-send" id="fbSend">Enviar</button>
  <p class="fb-note">Usamos seus dados só pra responder o feedback.</p>
</form>
```

**Step 4:** `node --check` dos scripts inline (script de verificação já usado). Esperado: `✅ ... OK`.

**Step 5:** Commit `feat: botao flutuante e gaveta de feedback (markup)`.

---

## Task 3: Abrir/fechar a gaveta + seleção de tipo + Umami

**Files:** Modify `sao-joao-sergipe-2026/index.html`

**Step 1 — estado:** adicionar `fbOpen:false, fbTipo:'sugestao'` ao objeto de estado `S`.

**Step 2 — render:** em `updateSheets()` (que controla `.open`/overlay das outras sheets), incluir `fbSheet` seguindo o mesmo padrão de `calSheet`/`citySheet`, e preencher `fbSheet.innerHTML = renderFbSheetContent()`.

**Step 3 — handlers** (em `attachAll`/onde os outros sheets são ligados):
```javascript
q('#fabFeedback')?.addEventListener('click',()=>{ S.fbOpen=true; updateSheets(); track('feedback_aberto',{pagina:location.hash||'/'}); });
q('#fbSheetClose')?.addEventListener('click',()=>{ S.fbOpen=false; updateSheets(); });
qq('#fbTipos .fb-tipo').forEach(b=>b.addEventListener('click',()=>{ S.fbTipo=b.dataset.tipo; qq('#fbTipos .fb-tipo').forEach(x=>x.classList.toggle('sel',x===b)); }));
```
(Fechar pela overlay: incluir `S.fbOpen=false` no handler de `#sheetOverlay`.)

**Step 4:** `node --check`. **Step 5:** Commit `feat: abrir/fechar gaveta de feedback + tipo`.

---

## Task 4: Envio (fetch) + estados + honeypot + Umami

**Files:** Modify `sao-joao-sergipe-2026/index.html`

**Step 1 — constante** (perto do topo do script):
```javascript
const FEEDBACK_ENDPOINT = '<<URL_DO_APPS_SCRIPT>>'; // Manoel fornece
```

**Step 2 — submit handler** (ligado junto dos demais quando a sheet existe):
```javascript
q('#fbForm')?.addEventListener('submit', async (ev)=>{
  ev.preventDefault();
  if(q('#fbWebsite')?.value) return;                 // honeypot
  const msg = q('#fbMensagem').value.trim();
  if(!msg){ q('#fbMensagem').focus(); return; }
  const btn = q('#fbSend'); btn.disabled = true; btn.textContent = 'Enviando...';
  const fd = new FormData();
  fd.append('tipo', S.fbTipo);
  fd.append('mensagem', msg);
  fd.append('contato', q('#fbContato').value.trim());
  fd.append('pagina', location.hash || '/');
  fd.append('dispositivo', navigator.userAgent);
  try{
    await fetch(FEEDBACK_ENDPOINT, { method:'POST', mode:'no-cors', body: fd });
    track('feedback_enviado', { tipo: S.fbTipo });
    q('#fbSheet').innerHTML = `<div class="fb-msg">Valeu! Recebemos seu feedback 🙌</div>`;
    setTimeout(()=>{ S.fbOpen=false; updateSheets(); }, 1800);
  }catch(_){
    btn.disabled=false; btn.textContent='Enviar';
    alert('Ops, não consegui enviar. Tenta de novo?');
  }
});
```

**Step 3:** `node --check`. Esperado: OK.

**Step 4:** Commit `feat: envio de feedback para Apps Script + telemetria`.

---

## Task 5: Teste integrado + deploy

**Step 1:** Substituir `<<URL_DO_APPS_SCRIPT>>` pela URL real do Manoel.

**Step 2:** `node --check` + `python3 -c "import json;json.load(open('sao-joao-sergipe-2026/vercel.json'))"`.

**Step 3:** Commit + push (`feat: feedback dos visitantes ao vivo`).

**Step 4 — verificação na Vercel:** após deploy, `curl -sI <site>/ | grep -i content-security-policy` deve conter `script.google.com`.

**Step 5 — teste manual no navegador (Manoel ou Claude):**
- Abrir o site (hard refresh), tocar no 💬, escolher um tipo, escrever, Enviar.
- Esperado: "Valeu! Recebemos 🙌" → gaveta fecha.
- Conferir: 1 linha nova na planilha + 1 email + evento `feedback_enviado` no Umami.

**Step 6:** Marcar plano como concluído.

---

## Checklist final
- [ ] Backend Apps Script publicado e testado via curl
- [ ] CSP libera script.google.com / script.googleusercontent.com
- [ ] FAB + gaveta nativos, com cara do site
- [ ] Tipo/mensagem/contato + honeypot
- [ ] Envio no-cors com estados (loading/sucesso/erro)
- [ ] Umami: feedback_aberto / feedback_enviado
- [ ] Planilha recebendo linhas + email chegando
- [ ] Deploy verificado no navegador real
