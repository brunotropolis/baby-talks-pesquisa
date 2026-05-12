# Pesquisa Pré-Evento — Manual do Recém-nascido

Sistema de pesquisa pré-lançamento de evento presencial para gestantes. Recebe respostas via formulário web, valida qualidade com Claude, gera token de acesso ao curso "O Dia do Parto" (11 aulas, 60 dias) e disponibiliza dashboard com insights pra organização.

> ⚠️ **Não usar "Baby Talks"** — nome do evento ainda não divulgado. Tudo na UI deve falar "Manual do Recém-nascido" ou "Pesquisa Pré-Evento".

## 🌐 URLs em produção

| O quê | URL |
|---|---|
| Formulário (gestante responde) | https://pesquisa.manualdorecemnascido.com.br |
| Área do curso "O Dia do Parto" | https://parto.manualdorecemnascido.com.br/?t=TOKEN |
| Dashboard admin (Bruno) | https://pesquisa.manualdorecemnascido.com.br/dashboard.html?t=mrn-dash-9k3xq2vnz8 |
| Relatório público (equipe do evento, sem PII) | https://pesquisa.manualdorecemnascido.com.br/relatorio.html |
| Planilha de respostas | https://docs.google.com/spreadsheets/d/1VqS_Ud1ho_FXVT7edr13XpQPNxtUOcKerCacbEtdFa8/edit |

**Token admin do dashboard:** `mrn-dash-9k3xq2vnz8` (na primeira visita vai pro localStorage e some da URL).

## 📅 Datas

- **Lançamento da pesquisa:** segunda 11/maio/2026
- **Expiração do acesso ao curso:** **10/julho/2026** (data fixa, mesma pra todo respondente — 60 dias após o lançamento)
  - Configurado em `build_pesquisa.py` → `EXPIRA_FIXA = "2026-07-10"`

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│  GESTANTE                                                    │
│    └─ pesquisa.manualdorecemnascido.com.br  (form)          │
│         ↓ POST /webhook/baby-talks-pesquisa                  │
│    ┌─────────────────────────────────────┐                  │
│    │  n8n: Pesquisa | Receber             │                  │
│    │  • Pré-valida local (regex anti-lixo)│                  │
│    │  • Claude analisa qualidade           │                  │
│    │  • Gera token + salva Sheets          │                  │
│    └─────────────────────────────────────┘                  │
│         ↓ aprovado → redireciona com ?t=TOKEN                │
│    parto.manualdorecemnascido.com.br/?t=TOKEN  (curso)      │
│         ↓ GET /webhook/baby-talks-validar?t=TOKEN            │
│    ┌─────────────────────────────────────┐                  │
│    │  n8n: Pesquisa | Validar Acesso       │                  │
│    │  • Checa token + expiração (10/jul)  │                  │
│    │  • Retorna lista de aulas + URLs Panda│                  │
│    └─────────────────────────────────────┘                  │
│         ↓ clica em uma aula → iframe Panda Video             │
└─────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────┐
│  BRUNO / EQUIPE                                              │
│    └─ /dashboard.html  (admin) ou /relatorio.html (público) │
│         ↓ GET /webhook/pesquisa-dashboard?token=... ou ?public=1
│    ┌─────────────────────────────────────┐                  │
│    │  n8n: Pesquisa | Dashboard Stats     │                  │
│    │  • Agrega stats das respostas         │                  │
│    │  • Detecta recusa_pagar via regex     │                  │
│    │  • Claude gera insights dos abertos   │                  │
│    │  • Anonimiza em modo público          │                  │
│    └─────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

**Camadas:**
- **Frontend:** HTML estático em 2 repos GitHub → GitHub Pages → Cloudflare (CNAME proxied, SSL flexible)
- **Backend:** n8n hospedado em easypanel (`n8n-n8n.xktssy.easypanel.host`)
- **Storage:** Google Sheets (planilha `Pesquisa Pré-Evento`)
- **Video:** Panda Video (player embed)
- **IA:** Claude Haiku 4.5 via Anthropic API (filtro qualidade + insights dashboard)

## 📁 Repos GitHub

| Repo | Conteúdo | Domínio |
|---|---|---|
| `baby-talks-pesquisa` | form + dashboard.html + relatorio.html | pesquisa.manualdorecemnascido.com.br |
| `baby-talks-aulas` | área do curso + thumbs | parto.manualdorecemnascido.com.br |

⚠️ Nome dos repos ainda tem "baby-talks" (não dá pra mudar facilmente, são URLs de origem internas). **Não importa porque o usuário só vê o domínio custom.**

## ⚙️ Workflows n8n

| Workflow | ID | Webhook | Função |
|---|---|---|---|
| `Pesquisa \| Receber` | `rV4XgEfRcD8Jp2iY` (atualizar se mudar) | `POST /webhook/baby-talks-pesquisa` | Recebe form, valida, gera token, salva |
| `Pesquisa \| Validar Acesso` | `mDNt6CEiY0EqIUX6` | `GET /webhook/baby-talks-validar?t=TOKEN` | Valida token + retorna aulas |
| `Pesquisa \| Dashboard Stats` | `p3DWEIv3IO44f5vo` | `GET /webhook/pesquisa-dashboard?token=ADMIN_TOKEN` ou `?public=1` | Stats + insights Claude |

⚠️ **Os webhook paths ainda usam "baby-talks-"** — não muda pra evitar quebrar URLs em produção. Os **nomes dos workflows** foram trocados pra "Pesquisa | ..." (sem "Baby Talks").

## 📋 Planilha — schema

- **ID:** `1VqS_Ud1ho_FXVT7edr13XpQPNxtUOcKerCacbEtdFa8`
- **Nome:** `Pesquisa Pré-Evento`
- **Credential ID no n8n:** `6VV3tATljXfg5uvi`

### Aba `Respostas` (32 colunas)

```
id, criado_em, idade, trabalho, profissao, email, cidade,
fase, trimestre, filho,
gestacao_interesse, gestacao_temas, gestacao_outro,
parto_interesse, parto_temas, parto_outro,
bebe_interesse, bebe_temas, bebe_outro,
evento_extras, evento_extras_outro,
aprender_evento, tempo_ideal, periodo, conjuge_evento, investir,
score_qualidade, status_analise, feedback_analise,
token, user_agent, referrer
```

`status_analise` possíveis: `APROVADO`, `REJEITADO_LOCAL`, `REJEITADO_CLAUDE`. Dashboard só conta `APROVADO`.

### Aba `Acessos` (8 colunas)
`token, criado_em, expira_em, ultimo_acesso, contagem_visitas, idade, fase, status`

### Aba `Aulas` (5 colunas, 11 linhas)
`ordem, titulo, thumb_url, panda_url, ativa`

Aulas atuais: Boas-vindas, O que levar pra maternidade, Mala da mãe, Plano de parto, Chegada à maternidade, Parto normal, Parto cesárea, Golden hour, Fotos e visitas, A saída, O puerpério.

URLs Panda usam `video_external_id` (NÃO o `id` interno) no formato:
`https://player-vz-2c8b2b2b-65c.tv.pandavideo.com.br/embed/?v={video_external_id}`

## 📝 Formulário — 15 perguntas

1. Idade
2. Trabalha? (+ profissão se Sim)
3. **E-mail**
4. **Cidade**
5. Fase (Tentando / Grávida 1-3º tri / Bebê <90d / Bebê >90d)
6. Esse é seu 1º/2º/3º/4º+ filho
7. **GESTAÇÃO** importante? Sim/Não/Talvez → multi-temas (11 opções)
8. **PARTO** importante? → multi-temas (13 opções)
9. **PRIMEIROS MESES BEBÊ** importante? → multi-temas (14 opções)
10. O que esperam ter no evento (multi)
11. **O que mais gostaria de aprender** (texto aberto — analisado por Claude)
12. Tempo ideal (3h/6h/dia)
13. Período (sáb/dom × manhã/tarde)
14. Cônjuge participa? (Sim/Não/Talvez)
15. **Quanto investir** (texto aberto — analisado por Claude)

UX: multi-step, 1 pergunta por tela, barra de progresso, rascunho salvo em `localStorage['pesquisa_draft']`, sub-blocos abrem só com Sim/Talvez.

## 🧠 Filtro de qualidade

**Camada 1 — local (JS sem IA, instantâneo):**
- `aprender_evento`: rejeita se <30 chars, <4 palavras reais, keyboard mash (`asdf`, `qwerty`), frase-isca (`sla`, `tanto faz`, `não sei`), ou repetição (`lalala`)
- `investir`:
  - Se contém número (ex: "R$ 300", "uns 250") → aceita
  - Se "gratuito/grátis/de graça/R$ 0/zero/nada" → **aceita** (não rejeita) mas marca como `recusa_pagar` (detectado depois)
  - Lixo evidente (`asdf`, `kkk`) → rejeita

**Camada 2 — Claude Haiku 4.5 (api.anthropic.com):**
- Prompt rigoroso, instrui "na dúvida, rejeite"
- Scores 0-10 pra cada campo
- Aprova só se `aprender_evento ≥ 6` E `investir ≥ 5`
- Trata "gratuito" como score ≥ 7 (não rejeita)

Quando rejeita, retorna feedback amigável especificando o campo problemático. Front leva pessoa de volta pra aquela pergunta.

## 📊 Dashboard

**2 versões do mesmo workflow:**
- `dashboard.html?t=mrn-dash-9k3xq2vnz8` (admin) — vê tudo: e-mail, cidade exata
- `relatorio.html` (público) — anonimizado: sem e-mail/cidade individual, cidades com <2 respondentes viram "Outras (N)"

**5 abas em ambos:**
1. **Visão Geral** — KPIs, insights da Claude (resumo + temas + sugestões), interesse por bloco
2. **Demografia** — idade, fase, trimestre, filho, trabalho, cônjuge, cidades, profissões
3. **Temas de Interesse** — TODOS os temas de gestação/parto/bebê com % sobre total de respondentes
4. **Logística do Evento** — extras do evento, tempo, período, cônjuge
5. **Respostas Abertas** — tabela com texto integral

**KPI especial "Querem gratuito":** quantos respondentes indicaram "gratuito/R$ 0/de graça" no campo investir. Esses são **excluídos do cálculo do ticket** (faixa de preço sugerida pelo público) — Claude e médias filtram `r._recusa_pagar`.

**Detecção `recusa_pagar`:** feita em tempo real no workflow do dashboard, regex no campo `investir` (não tem coluna separada na planilha). Permite mudar a regra sem migrar dados.

**Médias inteligentes:** descartam maior e menor valor (idade, investir) antes de calcular — outliers não distorcem números. Banner azul no topo explica.

## 🛠️ Scripts de deploy (em `D:/CLAUDE/baby-talks-workflows/`)

```bash
cd D:/CLAUDE/baby-talks-workflows
python build_pesquisa.py     # workflow Pesquisa | Receber
python build_validar.py      # workflow Pesquisa | Validar Acesso
python build_dashboard.py    # workflow Pesquisa | Dashboard Stats
python setup_github.py       # cria/atualiza repos GitHub e Pages
```

Todos são **idempotentes** — re-rodam o build deactivate → PUT → activate via API n8n. Se mudar o nome do workflow (`WF_NAME` no script), o script cria um workflow NOVO em vez de atualizar — cuidado com duplicação.

**Token n8n API** está hardcoded nos scripts (`N8N_KEY`).

## 🌐 DNS (Cloudflare)

Zone: `manualdorecemnascido.com.br` (zone ID `42a253a5f2baf2b06c822e3ce9d8389d`)

Records:
- `pesquisa` → CNAME → `brunotropolis.github.io` (proxied: true)
- `parto` → CNAME → `brunotropolis.github.io` (proxied: true)

**SSL mode da zona:** `Flexible` (CF entrega HTTPS pro user, fala HTTP com GitHub Pages — bypass do cert do GH). Se voltar pra `Full`, GitHub Pages precisa ter cert ativo (o que demora ~30min).

## 🎬 Panda Video

**Whitelist de domínios** (em `dashboard.pandavideo.com.br/#/domains`):
- `*.manualdorecemnascido.com.br` (cobre pesquisa, parto, qualquer sub futuro)
- `*.greenn.club` (cobre `manualdorecemnascido.greenn.club` — área de membros existente)
- `brunotropolis.github.io`
- `pesquisa.manualdorecemnascido.com.br` (explícito, redundância)
- `parto.manualdorecemnascido.com.br` (explícito)
- `greenn.club`

⚠️ **Toggle "Permitir somente domínios selecionados" está ATIVO.** Sem whitelist, vídeos do curso (e da Greenn) param de tocar.

## 🐛 Bugs corrigidos / aprendizados

1. **Whitelist Panda zerou videos da Greenn** — quando ativamos a whitelist pra `*.manualdorecemnascido.com.br`, os videos do `manualdorecemnascido.greenn.club` pararam. Fix: adicionar `*.greenn.club` na lista.

2. **Player Panda retornava erro com URL embed** — usamos `id` interno (`aded102f-...`) em vez do `video_external_id` (`2a2a8eae-...`) que é o ID público. URL correta: `player-vz-{prefix}.tv.pandavideo.com.br/embed/?v={video_external_id}`.

3. **Workflows duplicados ao renomear** — quando muda `WF_NAME` no script, ele cria workflow novo em vez de atualizar (porque busca por nome). Os 2 ficam ativos no mesmo webhook path → n8n roteia pro mais antigo. Fix: deletar workflows antigos via API.

4. **Sheets node "Column names were updated after the node's setup"** — quando adicionamos coluna nova no schema do mapping mas o cache do n8n tinha o schema antigo da planilha. Resolução: deletar e recriar workflow OU manter coluna fixa e detectar lógica em runtime (que foi o caso do `recusa_pagar`).

5. **GitHub Pages 301 → http** — antes do SSL ser gerado pelo GitHub (15-60min), o subdomínio redireciona pra HTTP. Resolvido ativando proxy Cloudflare em modo Flexible.

6. **Aviso "Site perigoso" do Chrome no GitHub Pages** — `brunotropolis.github.io` é subdomínio compartilhado, qualquer outro site flagged contamina todos. Fix: usar domínio custom (`pesquisa.manualdorecemnascido.com.br`).

7. **Claude aprovando lixo** — primeiro prompt era lenient demais. Endureci com instrução "na dúvida, REJEITE", listei exemplos de lixo e thresholds altos.

8. **Form intro empurrado pro fim no mobile** — `flex: 1` no `.card` esticava o conteúdo. Fix: classe `.is-intro` com `flex: 0 0 auto` na intro.

9. **Validação inconsistente "gratuito"** — primeira versão rejeitava, depois Bruno pediu pra aceitar + filtrar do dashboard. Solução: aceitar no submit, detectar via regex no dashboard, excluir dos cálculos de ticket. Sem coluna nova na planilha.

## 🔄 Como atualizar coisas comuns

**Mudar pergunta:** editar `baby-talks-pesquisa/index.html` (steps), depois `build_pesquisa.py` (schema). Atualizar schema também na planilha (header da aba Respostas) se mudou coluna.

**Adicionar nova aula:** editar aba `Aulas` na planilha (nova linha com `ordem`, `titulo`, `thumb_url`, `panda_url`, `ativa=SIM`). Não precisa redeploy.

**Mudar texto de aviso:** editar HTML, commit, push. GitHub Pages atualiza em ~1min.

**Bloquear acesso de um token:** mudar `status` pra `BLOQUEADO` na aba `Acessos`.

**Estender prazo do curso:** mudar `EXPIRA_FIXA` em `build_pesquisa.py` e re-deployar. Tokens novos terão data nova. Tokens antigos mantêm a data original (na coluna `expira_em` da aba Acessos).

**Revogar dashboard public:** trocar `ADMIN_TOKEN` em `build_dashboard.py` (também atualizar nas URLs compartilhadas).

## 🧪 Smoke tests

```bash
# Form aprovado
curl -X POST https://n8n-n8n.xktssy.easypanel.host/webhook/baby-talks-pesquisa \
  -H "Content-Type: application/json" -d '{...payload completo...}'

# Validar token
curl https://n8n-n8n.xktssy.easypanel.host/webhook/baby-talks-validar?t=TOKEN

# Dashboard admin
curl https://n8n-n8n.xktssy.easypanel.host/webhook/pesquisa-dashboard?token=mrn-dash-9k3xq2vnz8

# Dashboard público
curl https://n8n-n8n.xktssy.easypanel.host/webhook/pesquisa-dashboard?public=1
```

---

## 📩 Envio de e-mail (link do curso)

**Problema resolvido:** o token de acesso ficava só no `localStorage`. Se a pessoa fechasse a aba, limpasse cache ou trocasse de dispositivo, perdia acesso. Solução: enviar o link do curso por e-mail logo após cada resposta aprovada.

### Resend.com
- **Domínio:** `manualdorecemnascido.com.br` (Verified, com DKIM + SPF no Cloudflare)
- **Remetente:** `Manual do Recém-Nascido <no-reply@manualdorecemnascido.com.br>`
- **Sem caixa real** — replies dão bounce (aceitável)
- **API Key (restricted, send-only):** `re_aLpFtVbp_JFvutW1GBvwtXvxTBvKJ7trq`
- **Free tier:** 100/dia, 3000/mês
- **Painel:** https://resend.com/domains

### DNS no Cloudflare (já configurado)
- `TXT resend._domainkey` — DKIM
- `MX send` (prio 10) `feedback-smtp.sa-east-1.amazonses.com`
- `TXT send` — SPF `v=spf1 include:amazonses.com ~all`
- Região: `São Paulo (sa-east-1)`
- ⚠️ MX do `@` (root) deixado de fora **propositalmente** pra não conflitar caso queira criar caixa real no Turbocloud depois.

### Workflow n8n — Pesquisa | Despachar Email
- **ID:** `SDZJC3dl4qtJihEy`
- **Build script:** `D:/CLAUDE/baby-talks-workflows/build_dispatch_email.py`
- **Trigger 1 — Cron:** a cada **10 minutos**
- **Trigger 2 — Webhook manual:** `GET /webhook/pesquisa-despachar-email?token=mrn-dash-9k3xq2vnz8` (backfill on-demand)
- **Limite:** 50 envios por execução (margem vs 100/dia do Resend)

### Fluxo
```
CRON 10M + WEBHOOK MANUAL → MERGE → CHECK TOKEN → IF AUTORIZADO
  ↓
LER RESPOSTAS (Sheets) → FILTRAR PENDENTES (Code) → MONTAR EMAIL (Code, ForEach)
  → ENVIAR RESEND (HTTP POST) → TRATAR RESPOSTA (Code, ForEach)
  → MARCAR ENVIADO (Sheets update por row_number) → RESPONDER OK
```

### Filtro de pendentes
- `status_analise === 'APROVADO'`
- `email_enviado_em` vazio
- `email` com formato válido (regex)

### Colunas adicionadas na aba `Respostas`
- `email_enviado_em` (AH) — timestamp BR quando envio deu sucesso
- `email_erro` (AI) — mensagem do Resend se falhou (rate limit, validation, etc)

### Template do e-mail (texto final aprovado pelo Bruno)
- Assunto: `🎁 Seu acesso ao curso "O Dia do Parto"`
- Header rosa (#C2185B) com "Manual do Recém-Nascido"
- "Olá! Como você preencheu nossa pesquisa do evento, segue o curso **O Dia do Parto Completo** para que possa conferir."
- Botão CTA `▶ Acessar o curso agora` → `https://parto.manualdorecemnascido.com.br/?t=TOKEN`
- "Lembrando que ele ficará disponível até o dia **10 de julho de 2026** e depois sairá do ar."
- "Aproveite bem cada aula. Preparamos esse conteúdo com muito carinho..."
- Footer: "Esse link é pessoal e intransferível."
- **Sem** "responda esse e-mail" (porque é no-reply de verdade)

### Avisos no formulário (index.html)
- **Caixa do agradecimento** (única, unificada): "🎁 Como agradecimento, quem responder de maneira válida ganha: Curso 'O Dia do Parto' — 11 aulas" + divisor + "📩 Vamos enviar o link do curso por e-mail também, então use um endereço válido para que possa receber o link com tranquilidade."
- **Tela de redirect (loading pós-submit):** "Tudo certo! Redirecionando para o curso..." + "Já estamos liberando seu acesso 🎁 / 📩 Também enviamos o link por e-mail (pode demorar até 10 min)"
- Tempo do redirect aumentado de 1.2s → 3.5s pra dar tempo da pessoa ler o aviso

### Bugs corrigidos durante implementação
1. **`$input.first()` em runOnceForEachItem** — erro "Can't use .first() here". Fix: usar `$json` ou `$input.item.json` em modo ForEach.
2. **`return [{json}]` em runOnceForEachItem** — erro "An array was returned". Fix: retornar `{json}` (objeto único) nesse modo.
3. **Colunas `email_enviado_em` / `email_erro` não existiam na planilha** — Sheets node retornava 0 items silenciosamente. Fix: criar via Sheets API `batchUpdate` (appendDimension + updateCells). Script: `D:/CLAUDE/baby-talks-workflows/add_email_cols.py`.
4. **Matching por `id` falhava em ~75% das atualizações** — race condition / problema do batch update do n8n. Fix: usar `row_number` como matchingColumns (endereçamento direto da linha).
5. **Backfill mandou 2-3 emails pras primeiras 20 aprovadas** — durante o debug, marcações não rodavam, então cada retrigger reenviava. Daqui pra frente é idempotente (1 email por pessoa).

### Cuidados / próximos passos
- **Sem opção de "marcar lido":** se a pessoa não receber, vai reclamar. Por isso o cron de 10min e o aviso na tela do redirect.
- **Sem retry inteligente:** se Resend retornar 429 (rate limit), o item fica com `email_erro` preenchido e **não** tenta de novo até a próxima execução. Hoje isso volta a tentar porque o `email_enviado_em` continua vazio, então funciona como retry implícito.
- **Sem dispatch fora do horário comercial:** o workflow não tem checagem de hora — roda 24/7 a cada 10min. Se quiser limitar, adicionar um IF de horário antes do LER RESPOSTAS (igual ao dispatcher do Buscador Geek v2).

---

## 📌 Outras notas

- `localStorage` keys usadas: `pesquisa_draft` (form rascunho), `curso_token` (área de aulas), `dashboard_admin_token` (dashboard admin)
- Layout dark mode no dashboard, layout rosé no form (gestantes), área de aulas com header rosa e cards grid 6 colunas em desktop
- Análise de qualidade chama Claude **em todas as submissões** que passam do filtro local — gera custo mínimo (Haiku é barato), mas se ficar caro pode cachear ou pular Claude pra respostas claramente boas.
