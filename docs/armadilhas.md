# Armadilhas que custaram tempo

Cada linha aqui custou de minutos a uma tarde. Estão registradas para não
custarem de novo.

## Banco, testes e CI

| Sintoma | Causa | Como evitar |
|---|---|---|
| CI vermelho com "supabaseUrl is required" | `describe.skipIf` executa o corpo do bloco mesmo quando pula | nada de client criado no corpo sem chave; simular com `CI=true npm test` |
| teste de banco deixava agendamentos num lead do seed | o telefone fixo do teste era de um lead existente, e a função achou esse lead | teste usa telefone de ficção do Reino Unido (07700 900xxx) e lead novo |
| guardrail barrou o aviso interno | a faixa de faturamento informada pelo lead parecia preço | regra de preço só nas mensagens ao lead |
| Vitest não resolve `@/...` | não havia config com o atalho do tsconfig | `vitest.config.ts` próprio no app |
| `npm install` falha com `reading 'edgesOut'` | bug do npm 10 com workspaces | pacote novo com `npx -y npm@11 install`; o `npm ci` do CI funciona |
| PR empilhado em conflito depois do merge dos anteriores | os anteriores entraram por squash | trazer o main e conferir `git diff --stat origin/main` |

## Front-end e movimento

| Sintoma | Causa | Como evitar |
|---|---|---|
| imagem com cortina nunca aparece | `clip-path: inset(100%)` deixa o alvo com área zero para o IntersectionObserver | observar o elemento pai, não o alvo |
| `clip-path` não anima | não existe transição a partir de `none` | a base precisa ser `inset(0)` |
| erro #418 do React (hidratação) | script mexendo no texto de um elemento antes da hidratação | estrutura vem do servidor; o navegador só troca texto depois |
| degradê SVG some numa linha reta | `objectBoundingBox` com altura 0 | `gradientUnits="userSpaceOnUse"` |
| painel `sticky` esconde conteúdo | painel mais alto que a tela | `position: relative` abaixo de 1024 px |
| elemento absoluto fora do lugar dentro de grid | o bloco de contenção é a área da grade | `grid-area: auto` no filho |
| leitor de tela lê texto decorativo | `content` de pseudo-elemento é lido | `content: attr(data-texto) / ""` |
| e-mail longo estoura a largura em 375 px | maiúsculas com espaçamento não quebram | `overflow-wrap: anywhere` |
| animação "parada" durante o teste | aba oculta não roda `requestAnimationFrame` nem IntersectionObserver | medir com a aba visível |

## Ambiente

| Sintoma | Causa | Como evitar |
|---|---|---|
| `gh` responde "Bad credentials" com login válido | uma `GITHUB_TOKEN` inválida no ambiente tem prioridade | `env -u GITHUB_TOKEN gh ...` |
| arquivos com CRLF depois de script Python | `open(..., "w")` no Windows troca `\n` por `\r\n` | `newline=""` e Prettier depois |
| `next dev` não enxerga export novo de pacote | cache do servidor de desenvolvimento | reiniciar depois de mudar `exports` |
| imagem trocada continua a antiga | o cache de imagem do Next guarda pelo nome | trocar o **nome** do arquivo, não só o conteúdo |
