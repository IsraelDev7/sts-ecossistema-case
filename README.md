# Ecossistema STS — estudo de caso

Infraestrutura digital de uma empresa de educação comercial: site
institucional, diagnóstico comercial online, captura de leads com
consentimento e régua automática de follow-up por e-mail e WhatsApp — num
monorepo com banco versionado, testes e CI.

**Este repositório é a documentação de arquitetura.** O código de produção é
privado, porque é do cliente. O que está aqui são as decisões, os erros que
elas corrigem e as armadilhas que custaram tempo.

**No ar:** <https://sts-ecossistema.vercel.app>

**Stack:** Next.js 16 (App Router) · TypeScript estrito · Supabase (Postgres +
RLS) · n8n · Fastify · Vitest · GitHub Actions · Vercel

---

## O problema

A empresa só faturava quando o fundador estava presente: palestra, mentoria,
treinamento. O pedido era uma estrutura que rodasse em paralelo — que
captasse, respondesse, educasse e **qualificasse** sem consumir horas dele.

A tese que governa toda decisão técnica:

> O ecossistema digital não é o motor de receita. É o sistema de captação e
> **qualificação** do cliente de alto valor. Entre vender mais coisa barata e
> identificar melhor quem pode contratar o serviço grande, escolhe-se o
> segundo.

---

## O sistema em uma passada

```
                   ┌──────────────── site (Next.js) ────────────────┐
  visitante ──────►│ páginas · /avaliacao (diagnóstico) · /contato  │
                   └───────────────┬────────────────────────────────┘
                                   │ POST /api/diagnostico · /api/contato
                                   ▼
                   ┌──── Postgres (Supabase, RLS) ──────────────────┐
                   │ função transacional: lead + pedido +           │
                   │ agendamentos numa única transação, idempotente │
                   └───────────────┬────────────────────────────────┘
                                   │ agendamentos vencidos
       cron 5 min                  ▼
  n8n ──────────► POST /api/regua/pendentes ──► site trava, monta, aplica
   │                                             guardrail, devolve o que
   │                                             pode sair
   ├──► e-mail · WhatsApp
   └──► POST /api/regua/resultado ──► grava cada tentativa em `envios`
```

---

## Estrutura

```
packages/
  brand/     tokens de marca, 40 componentes, sistema de movimento, logo
  db/        migrations versionadas, tipos gerados, client
  core/      regras de domínio: diagnóstico, normalização de telefone
apps/
  site/      Next.js — páginas, diagnóstico, contato, rotas da régua
  agentes/   Fastify — atendimento conversacional (próxima onda)
flows/       workflows do n8n versionados em JSON
docs/        decisões por onda e armadilhas conhecidas
```

**Em números:** 33 entregas por pull request · 17 tabelas com RLS ·
6 migrations · 21 arquivos de teste · ~29 mil linhas de TypeScript, SQL e CSS.

Construído em ondas, cada uma fechada com um documento de decisões antes da
próxima começar:

| Onda | Escopo | Estado |
|---|---|---|
| 1 · Fundação | monorepo, marca, banco com RLS, CI, deploy | concluída |
| 2 · Site | home e 5 páginas (sobre, consultoria, conteúdo, palestras, cases em vídeo) | concluída |
| 3 · Captação | diagnóstico, contato, gravação, régua no n8n, SEO para IA | em andamento |
| 4 · Agentes | atendimento conversacional com handoff humano | planejada |

---

## As decisões que definem a arquitetura

### 1. Cada peça tem uma responsabilidade — e não invade a outra

| Peça | Faz | Nunca faz |
|---|---|---|
| Rotas de API do Next.js | recebem, validam, gravam, enfileiram | lógica longa |
| `apps/agentes` | conversa com estado, fila, contexto, handoff | régua de e-mail |
| n8n | régua e integração: fluxo com começo e fim | conversa ou webhook crítico |

A regra mais importante saiu daqui: **as chaves de envio ficam no n8n; a
escrita no banco fica no site.** O n8n pede ao site o que está pendente e
devolve o resultado. Assim toda regra de banco tem teste automatizado, e o
n8n guarda só credencial de envio.

### 2. Gravação numa função do Postgres, não em três chamadas

Lead, pedido e agendamentos entram numa **transação só**
(`registrar_pedido_contato`, `registrar_diagnostico`). Ou entra tudo, ou nada.

A chave do pedido é gerada no navegador e é `UNIQUE`: se a pessoa clicar duas
vezes, ou a rede reenviar, a função devolve o registro existente e **não
agenda mensagem de novo**. Mensagem duplicada para o lead destrói a impressão
de uma equipe humana do outro lado.

### 3. Régua sem `wait` longo

Workflow que "espera 7 dias" morre quando o container reinicia. Aqui não
existe espera: existe uma tabela de agendamentos e um cron.

1. A cada 5 minutos, o n8n chama `POST /api/regua/pendentes` com um
   cabeçalho secreto.
2. O site **trava** os agendamentos vencidos com `FOR UPDATE SKIP LOCKED` —
   duas execuções simultâneas nunca pegam o mesmo item.
3. Cada mensagem passa por um **guardrail no código**, não no prompt.
4. O n8n envia e devolve o resultado; o site grava cada tentativa em `envios`.
5. Falhou: nova tentativa em 10 e em 20 minutos. Na terceira, `falhou`.
   Item travado há mais de 15 minutos volta para a fila.

Quando alguém perguntar por que fulano não recebeu, a resposta está numa
tabela — não numa suposição.

### 4. Guardrail no código, não só no prompt

Nenhuma mensagem ao lead sai com promessa de resultado ou preço fora da
tabela. A mensagem barrada **não sai** e fica registrada como `cancelado`,
com o motivo.

O teste pegou um caso que nenhum prompt pegaria: o aviso interno para a
equipe foi barrado porque a faixa de faturamento informada pelo lead
("R$ 200 mil") parecia preço. A regra de preço passou a valer só para
mensagens ao lead.

### 5. Idempotência e consentimento no banco, não na aplicação

- `UNIQUE` em `provider_id`, `gateway_id`, `event_id` e
  `(provedor, id_externo)`: plataformas reenviam eventos, e o banco recusa o
  duplicado sozinho.
- Consentimento **por canal** (`consentimento_canais`). Dado uma vez,
  continua dado; revogar é pelo descadastro (`opt_out`), que funciona de
  verdade. LGPD desde a primeira migration.
- RLS em todas as tabelas: uma consulta que esquece o filtro não vaza nada.

### 6. Diagnóstico: a régua calcula, o modelo (se houver) redige

O diagnóstico comercial grava os sinais (faturamento, urgência, tamanho da
equipe, cargo), mas **a pontuação fica nula** até o especialista definir os
pesos. A regra mora em `packages/core`, com teste — não em um prompt.

Decisão determinística, auditável linha a linha. Linguagem natural, se
entrar, entra só na apresentação.

### 7. Uma marca, um lugar

Nenhuma aplicação define a própria cor. Tokens, componentes e movimento vêm
de `packages/brand`. A paleta padrão do Tailwind foi **removida**: só existem
as cores do cliente, então é impossível usar a errada.

Um único arquivo de movimento roda no navegador. Os componentes declaram a
intenção por atributo (`data-revelar`, `data-cortina`, `data-parallax`…) e
ele executa — o resto é componente de servidor.

### 8. SEO para buscadores e para IA

- Dados estruturados (Organization, ProfessionalService, Person, FAQ,
  Article) gerados do mesmo conteúdo das páginas
- `/llms.txt` e `/llms-full.txt` no padrão llmstxt.org
- `robots.txt` fechado até a liberação, e aberto pelo nome aos robôs de busca
  das IAs
- GA4 com Consent Mode: nada é gravado antes do aceite; chegadas vindas de
  ChatGPT, Perplexity e Gemini viram evento próprio

---

## Como se verifica antes de dizer "pronto"

```
npm run lint && npm run typecheck && npm test && npm run build
```

Mais, no navegador: rolagem horizontal **medida no DOM** em 375 e 768 px,
alvos de toque ≥ 44 px, console limpo. Depois do push, o PR só entra com CI e
deploy de preview verdes.

**Teste como contrato** em quatro pontos: normalização de telefone, cálculo do
diagnóstico, validação de saída e idempotência de webhook.

As armadilhas que custaram tempo estão em [`docs/armadilhas.md`](docs/armadilhas.md).

---

## Sobre este repositório

Documentação extraída de um projeto privado em produção. Identificadores de
conta, dados do cliente, regras comerciais e credenciais foram omitidos — o
que permanece são as decisões e as lições.

**Israel Passos** — AI Engineer · Smart LABS
[github.com/IsraelDev7](https://github.com/IsraelDev7)
