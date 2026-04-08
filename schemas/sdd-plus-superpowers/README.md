# Schema sdd-plus-superpowers

Integra o fluxo de governança de artefatos do OpenSpec com as habilidades de execução do Superpowers em um único fluxo de trabalho.

---

## O que este Schema resolve

O OpenSpec gerencia **"o que fazer"** (proposal → specs → design → tasks), enquanto o Superpowers gerencia **"como fazer"** (brainstorming, writing-plans, subagent-driven-development). Ambos são excelentes individualmente, mas ao alternarem entre si no desenvolvimento real, surgem três problemas estruturais:

- **Duplicação de produção** — o brainstorming gera documentos de design no diretório do Superpowers (`docs/superpowers/specs/`), enquanto o OpenSpec reescreve proposal/design no diretório change, com conteúdo muito semelhante.

- **Fragmentação de Tasks** — o `tasks.md` do OpenSpec (checkboxes de granularidade grossa) e o `plan` do Superpowers (passos micro TDD) descrevem a mesma coisa, mas com formatos, localizações e rastreamento de estado independentes.

- **Orquestração manual** — o usuário precisa decidir sozinho "qual habilidade usar agora", sem integração automática entre os dois sistemas.

---

## Por que usar um Schema personalizado em vez de modificar as habilidades existentes

Duas alternativas foram consideradas:

- **Adicionar campos personalizados no `config.yaml`** (como `skill_bindings`): o CLI do OpenSpec não reconhece esses campos, sem validação, sem descoberta, e seria necessário modificar vários `SKILL.md` para lê-los.
- **Modificar diretamente os arquivos de habilidades do opsx**: invasivo, afeta todos os changes, e seria sobrescrito ao atualizar os `SKILL.md`.

O schema personalizado utiliza o mecanismo de schema em nível de projeto suportado nativamente pelo OpenSpec:

- O CLI valida a estrutura do schema
- `openspec schemas` lista automaticamente
- Cada change pode escolher seu schema de forma independente (`--schema spec-driven` ou `--schema sdd-plus-superpowers`)
- Nenhum `SKILL.md` ou arquivo de comando existente é alterado

---

## Visão geral do fluxo de trabalho

```
brainstorm ──→ proposal ──→ specs ──→ tasks ──→ plan
                  │                     ↑
                  └──→ design ──────────┘
```

**Diferenças em relação ao spec-driven:**

| | spec-driven | sdd-plus-superpowers |
|---|---|---|
| Ponto de entrada | proposal (escrito manualmente) | brainstorm (invoca a habilidade brainstorming) |
| Ponto de saída | tasks (granularidade grossa) | plan (passos micro TDD) |
| apply requer | tasks | plan |
| Método de apply | task-by-task padrão | worktree + subagent-driven-development |
| Novos artefatos | — | brainstorm, plan |

---

## Habilidades Superpowers integradas

| Fase do Schema | Habilidade Superpowers invocada | Como é acionada |
|---|---|---|
| artefato brainstorm | `superpowers:brainstorming` | instrução do artefato |
| artefato plan | `superpowers:writing-plans` | instrução do artefato |
| fase apply | `superpowers:using-git-worktrees` | instrução do apply |
| fase apply | `superpowers:subagent-driven-development` | instrução do apply |
| após apply | `superpowers:finishing-a-development-branch` | instrução do apply |

Todas as integrações são realizadas pelo campo `instruction` do `schema.yaml` — instruindo a IA a invocar a habilidade correspondente no momento adequado. Nenhum arquivo de habilidade do Superpowers é modificado.

---

## Redirecionamento de saída (Output Redirection)

As habilidades do Superpowers têm caminhos de saída padrão (ex.: brainstorming escreve em `docs/superpowers/specs/`). Neste schema, a instrução do artefato inclui diretrizes de redirecionamento, informando à habilidade invocada para gravar no diretório do change:

- **brainstorming** → `openspec/changes/<name>/brainstorm.md` (+ `design.md` opcional)
- **writing-plans** → `openspec/changes/<name>/plan.md`

Isso é feito por injeção de contexto (anexando instruções ao invocar a habilidade), sem modificar o código das habilidades.

---

## Como usar

### Fluxo rápido (recomendado)

```bash
/opsx:ff my-feature    # tudo em um: cria diretório + brainstorm + proposal + design + specs + tasks + plan
/opsx:apply            # worktree + subagent-driven-development
/opsx:archive          # arquivar
```

### Fluxo passo a passo

```bash
/opsx:new my-feature --schema sdd-plus-superpowers
/opsx:continue         # → brainstorm (conversa interativa)
/opsx:continue         # → proposal
/opsx:continue         # → design
/opsx:continue         # → specs
/opsx:continue         # → tasks
/opsx:continue         # → plan
/opsx:apply
/opsx:archive
```

### Voltar para spec-driven

```bash
# Usar um schema diferente em um único change
/opsx:new my-simple-fix --schema spec-driven

# Ou alterar o padrão do projeto
# openspec/config.yaml: schema: spec-driven
```

---

## Registro de decisões de design

### Por que brainstorm é um artefato e não um hook

O brainstorming é uma conversa interativa de múltiplas rodadas que requer participação do usuário. Tratá-lo como o primeiro artefato, em vez de um hook no nível do schema, traz dois benefícios:

- **Pode ser pulado** — se o usuário já sabe o que quer fazer, pode criar o `brainstorm.md` diretamente sem invocar a habilidade
- **Rastreável** — `openspec status` consegue exibir se o brainstorm foi concluído, e os artefatos seguintes têm dependências claras

### Por que plan é independente de tasks

O `tasks.md` é composto por checkboxes de granularidade grossa ("adicionar PdfServiceTest"), enquanto o `plan.md` contém passos micro ("criar esqueleto do teste → escrever teste downloadPdf → executar → commit"). Granularidade e propósito são diferentes:

- **tasks.md** → rastreia o progresso geral (o campo `tracks` da fase apply analisa os checkboxes)
- **plan.md** → guia o subagente na implementação passo a passo (entrada do executor)

A fase apply requer `plan` em vez de `tasks`, pois o executor precisa de passos micro para trabalhar com eficiência. Porém, `tracks: tasks.md` garante que o progresso ainda seja rastreado pelos checkboxes de granularidade grossa.

### Estratégia de fallback

Se as habilidades do Superpowers não estiverem disponíveis (não instaladas, versão incompatível, etc.), cada instrução inclui um caminho de fallback:

- **brainstorm** → escrever `brainstorm.md` manualmente
- **plan** → escrever `plan.md` manualmente
- **apply** → implementação manual task-by-task padrão
