# codex-run

Agent skill (Claude Code / agentes compatíveis) que dispara o **Codex CLI** (`codex exec`) **dentro de uma git
worktree dedicada** para implementar um prompt/spec já preparado — com edição autônoma, saída em streaming e
captura detalhada — depois **avalia a entrega com nota 0–10**, **escala em caso de falha** e **PARA para o gate**
(o merge é decisão do dono). Agnóstica: funciona em qualquer projeto.

## Pré-requisitos
- `codex` CLI no PATH (`codex --version`) + credenciais.
- Uma **git worktree** off main (NUNCA o checkout principal) com o prompt/spec a implementar.

## Instalação
```bash
npx skills add brunosps/codex-run -g -y     # global
npx skills add brunosps/codex-run -y        # local no repo atual
```
Ou copie `SKILL.md` para `~/.claude/skills/codex-run/SKILL.md`.

## Princípios
- **Regra hard:** só roda em worktree dedicada, nunca no thread/checkout principal.
- Claude **avalia a complexidade** e sugere/escolhe modelo + reasoning effort.
- **Escalonamento gradual** (effort → modelo) quando a nota fica baixa / o gate falha.
- **Nota 0–10 obrigatória** da entrega + **saída detalhada** para direcionar o follow-up.
- Não mergeia: STOP para o gate; merge é decisão do dono.

Ver `SKILL.md` para o protocolo completo.
