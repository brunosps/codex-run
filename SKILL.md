---
name: codex-run
description: "Dispara o Codex CLI (codex exec) DENTRO de uma git worktree para implementar um prompt/spec já preparado, com edição autônoma, saída em streaming e captura estruturada; depois AVALIA a entrega (nota 0–10) e PARA para o gate. Use SEMPRE que o usuário disser 'roda o codex na worktree X', 'manda o codex implementar Y', 'executa o codex-prompt/o prompt', 'dispara o Codex' — em QUALQUER projeto. NÃO use para planejar nem para mergear (decisão do dono, após o gate)."
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

# codex-run — disparar o Codex na worktree certa, avaliar a entrega, parar para o gate

Skill **agnóstica** (qualquer repo/projeto) para o passo "rodar o Codex" de um fluxo de delegação: existe uma
**git worktree** com um **prompt/spec** já preparado → esta skill dispara `codex exec` lá dentro (streaming,
não-bloqueante) → **avalia a entrega com nota 0–10** → escala em caso de falha → **para para o gate** (o merge é
decisão do dono). Existe porque rodar o Codex à mão é repetitivo e arrisca: rodar na worktree errada, travar sem
TTY, e esquecer de aferir a qualidade antes de mergear.

> **Dual-use:** executa **edição autônoma** (`--sandbox workspace-write --full-auto`). Confirme a **worktree certa**
> e o **escopo** antes de disparar.

## REGRA (hard) — nunca no thread/checkout principal
`codex-run` **só** roda numa **git worktree dedicada** (off main). **NUNCA** dispare o `codex exec` no
**checkout principal** do repo (a árvore raiz no branch primário, ex.: `~/code/<projeto>` em `main`) — o Codex
edita de forma autônoma e isso corromperia o trabalho ativo / a árvore que o dono usa. Se o diretório alvo for o
checkout principal **ABORTE** (`BLOCKED`) e oriente: crie/uso uma worktree primeiro
(`git worktree add ../<projeto>-<slug> -b <branch> main`). Verificação obrigatória antes de rodar:
`git worktree list` → o alvo é uma worktree secundária (não a marcada como principal) **E** o `cwd` do `codex exec`
é essa worktree.

## Pré-flight (falhar cedo > rodar no lugar errado)
1. `codex --version` responde (CLI + credenciais). Se faltar → `BLOCKED`.
2. `git worktree list` — confirme a worktree alvo e que **não** é o **checkout principal** (a árvore do branch
   primário, ex. `~/code/<projeto>`). O Codex edita ali; rodar no principal corromperia o trabalho ativo.
3. Existe um **prompt/spec** (o que o Codex receberá). Convenções comuns: `.dw/spec/<slug>/codex-prompt.md`
   (dev-workflow), `PROMPT.md`, `TASK.md`, ou um caminho que o dono indicar. **Leia-o** — é o escopo/fence/gate.

## Onde rodar — PREFERIR Workflow (visível em /workflows)
Quando houver harness de **Workflow** (multi-agente), **dispare o `codex exec` DENTRO de um workflow** — um agente
de workflow consegue rodar o codex (rede OK, comprovado). Vantagens: a execução aparece em `/workflows`, e as
fases **Implementar → Avaliar (nota 0–10) → Escalar → Gate (fan-out)** orquestram juntas. **Fallback:** se não
houver harness de workflow, rode via Bash em background (precisa de rede → sandbox do Bash desabilitado p/ o codex
alcançar a API). Em ambos, vale a REGRA hard (worktree dedicada) + a captura detalhada + STOP-para-gate.

## Protocolo
1. **Resolver alvo.** Worktree + caminho do prompt (do usuário ou inferidos).
2. **Modelo + effort — o parent AVALIA a complexidade e SUGERE/escolhe.** Leia o prompt e dimensione; **proponha**
   modelo+effort com um one-liner de racional; o dono confirma ou sobrescreve; se ele disser "escolhe você",
   **decida** pela heurística.
   - Modelos (forte→leve): `gpt-5.5` · `gpt-5.3-codex` · `gpt-5.4` · `gpt-5.3-codex-spark` · `gpt-5.4-mini`.
     Effort: `low`·`medium`·`high`·`xhigh`.
   - Sinais: nº de tasks; amplitude do fence; superfície sensível (auth/segurança/tenant/migration/financeiro);
     "fundação/redesign/arquitetural" × "fix mecânico/typo/docs"; E2E/secure-audit no gate.
   - Heurística: **Trivial** → leve + `low/medium`; **Padrão** (<10 tasks, 1 área) → `gpt-5.3-codex`/`gpt-5.4` +
     `medium/high`; **Complexa/sensível** (fundação, multi-pacote, segurança) → `gpt-5.5` + `high/xhigh`.
     Comece **um degrau abaixo do teto** e escale se falhar (ver Escalonamento).
3. **Disparar (implementação — padrão).** Streaming (`--json`) p/ acompanhar sem travar; `-o` captura o relatório
   final; rode **em background** (não bloquear o parent):
   ```bash
   cd <WORKTREE> && codex exec --skip-git-repo-check \
     -m <MODEL> --config model_reasoning_effort="<EFFORT>" \
     --sandbox workspace-write --full-auto --json \
     -o <OUT_DIR>/codex-last-message.md \
     "$(cat <PROMPT_PATH>)" </dev/null
   ```
   - `</dev/null` SEMPRE (sem TTY trava). `--json` = eventos parseáveis em streaming; **não** suprima tudo com
     `2>/dev/null` por padrão. `<OUT_DIR>`: dir de spec do projeto (ex.: `.dw/spec/<slug>/QA/`) ou um temp.
   - **GATE PRECISA DE REDE?** O `--sandbox workspace-write` **bloqueia rede** dos comandos shell do codex — então
     `pnpm install`/build/test/E2E falham com `EAI_AGAIN`. Se o codex-prompt manda rodar o gate (install/test/etc.),
     troque por **`--dangerously-bypass-approvals-and-sandbox`** (sem sandbox + sem approvals = full access, com
     rede) para o codex rodar o gate e **se auto-validar**. Justificável porque a **REGRA hard** garante worktree
     isolada (off main). Sem rede no gate, o codex só implementa e reporta `blockers` (aí o gate fica com o parent).
   - **Saída DETALHADA por padrão** (para o parent direcionar o follow-up): passe `--output-schema <schema.json>`
     exigindo um relatório rico — não basta "ok". Schema mínimo:
     ```jsonc
     { "type":"object","required":["summary","tasks","filesChanged","gate","fenceViolations","uncommitted","blockers","nextSteps"],
       "properties":{
         "summary":{"type":"string"},
         "tasks":{"type":"array","items":{"type":"object","required":["id","status","notes"],
           "properties":{"id":{"type":"string"},"status":{"enum":["done","partial","skipped","failed"]},"notes":{"type":"string"}}}},
         "filesChanged":{"type":"array","items":{"type":"string"}},
         "gate":{"type":"object","properties":{"lint":{"type":"string"},"test":{"type":"string"},"build":{"type":"string"},"e2e":{"type":"string"}}},
         "fenceViolations":{"type":"array","items":{"type":"string"}},
         "uncommitted":{"type":"array","items":{"type":"string"}},
         "blockers":{"type":"array","items":{"type":"string"}},
         "nextSteps":{"type":"array","items":{"type":"string"}} } }
     ```
     (Garanta que o `codex-prompt.md` peça "STOP com relatório DETALHADO por task" — schema + prompt se reforçam.)
4. **Modo análise/review (read-only):** sem editar → `--sandbox read-only`, sem `--full-auto`.
5. **Resume** (continuar a sessão anterior na mesma worktree):
   `cd <WORKTREE> && echo "<prompt>" | codex exec --skip-git-repo-check resume --last </dev/null`.

## Avaliação obrigatória — nota 0–10 (SEMPRE)
Ao terminar, o parent **AVALIA a entrega do Codex e dá uma nota de 0 a 10** (10 = conformidade total + entrega
completa). Não pule isto — é o sinal que decide gate × escalonamento. Pontue contra o prompt:
- **Conformidade ao escopo/fence** (fez o pedido; ficou dentro do allowlist).
- **Gate técnico** (lint/test/build/E2E conforme o prompt — verde?).
- **Completude** (todas as tasks/itens entregues).
- **Higiene** (commitado, sem lixo, nada fora do fence, sem deps proibidas).
- **Qualidade/sem regressão**.
**Bar de aceite: nota ≥9** (decisão do dono). Faixas: **≥9** = aceitável (pronto pro gate humano); **6–8** =
`FINDINGS` → **escalar** p/ chegar a ≥9; **<6** = falha → **escalar**. `BLOCKED` se a escada esgotar sem atingir 9.
Sempre mostre a nota + o racional curto por critério.

## Escalonamento gradual em falha
Se a nota ficou baixa / o gate falhou / o Codex não concluiu, **re-rode a MESMA tarefa subindo um degrau** —
gradual, sem desistir no 1º tropeço nem pular pro topo.
- **Escada (1 por vez):** primeiro **effort** `low→medium→high→xhigh`; esgotado, suba o **modelo** um tier e
  volte o effort a `high`. Re-rode e **reavalie a nota**.
- **Continuar × recomeçar:** edições parciais coerentes → `resume --last` (mantém contexto). Worktree
  quebrada/suja → **resete antes** (`git -C <worktree> reset --hard && git clean -fd`) e rode fresco no degrau
  maior — não empilhe erro sobre erro.
- **Parada:** pare ao atingir **nota ≥9** (pronto pro gate) OU ao **esgotar** (modelo mais forte + `xhigh` ainda
  abaixo) → `BLOCKED` com evidência. Anuncie cada degrau; com autonomia, escale sozinho até o teto.

## Saída detalhada → direcionar o trabalho depois
A entrega do Codex tem que ser **detalhada o bastante para o parent (Claude) decidir o próximo passo** — não um
"pronto" opaco. Ao terminar:
- **Leia o `-o` por inteiro** (o relatório estruturado) + varra o stream `--json`; complemente com `git -C
  <worktree> status` e `git diff --stat` para ver o que realmente mudou (não confie só no que o Codex disse).
- **Produza um DIRECIONAMENTO** (handoff para o follow-up com o parent): o que o **gate** deve focar, o que
  **ajustar/refazer**, se **escalar** (nota baixa), o que ficou **fora do fence/uncommitted**, e os **blockers**.
  Esse direcionamento é o que guia o trabalho seguinte — registre-o no Structured Return (Evidence/Artifacts/
  Next Step) e na resposta ao dono.
- Se o relatório do Codex vier raso (sem o detalhe do schema), trate como `FINDINGS` e peça o detalhe (resume) ou
  reconstrua o detalhe a partir do diff antes de seguir.

## Disciplina (ao terminar)
- **STOP — não foi mergeado.** O Codex implementou na worktree; **rode o gate** (testes/lint/build + revisão;
  se o projeto usa dev-workflow: `/dw-review` + `/dw-qa` + `/dw-secure-audit`) **antes** de qualquer merge.
- **Merge = decisão explícita do dono.** Nunca automático aqui.
- Sinalize trabalho **uncommitted** ou edição **fora do fence**.

## Anti-patterns
- Rodar no checkout principal / sem confirmar a worktree — corrompe o trabalho ativo. *(pré-flight 2)*
- Mergear/empurrar dentro da skill — o merge vive fora, após o gate.
- Pular a nota 0–10 — perde-se o sinal de qualidade/escalonamento.
- Suprimir todo o output por padrão — perde-se a auditoria do que o Codex fez.
- `--dangerously-bypass-approvals-and-sandbox` — só com pedido explícito + ambiente já isolado.

## Structured Return
- **Status:** `PASS` (rodou, nota ≥9, parou limpo na worktree certa, pronto pro gate) · `FINDINGS` (rodou, nota
  6–8 ou ressalvas: uncommitted/fora-do-fence/parcial) · `BLOCKED` (faltou CLI/credencial/worktree/prompt, ou
  escalonamento esgotado sem atingir 9) · `NOT_APPLICABLE` (sem worktree+prompt → ainda em planejamento).
- **Score:** a nota 0–10 + racional por critério (conformidade/gate/completude/higiene/qualidade).
- **Scope:** worktree + branch + prompt usado; modelo + effort + sandbox; degraus de escalonamento percorridos.
- **Evidence:** caminho do `-o`, trecho do stream JSONL, `git status`/diff da worktree pós-run.
- **Artifacts:** arquivos criados/alterados (resumo do diff).
- **Decisions:** modelo/effort e por quê; implementação × read-only; continuar × resetar no retry.
- **Risks:** fora do fence, uncommitted, gate não rodado, dual-use (edição autônoma).
- **Next Step:** "rodar o gate na worktree; merge é decisão do dono".

## Notas
- `codex exec` flags-chave: `-m`, `--config model_reasoning_effort=`, `-s/--sandbox {read-only,workspace-write,
  danger-full-access}`, `--full-auto`, `--skip-git-repo-check`, `--json`, `-o/--output-last-message`,
  `--output-schema`, `resume --last`. Testado: codex-cli 0.141.0. Sem dependência de projeto específico.
