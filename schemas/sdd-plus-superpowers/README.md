# Schema sdd-plus-superpowers

Integra o fluxo de governança de artefatos do OpenSpec com as habilidades de execução do Superpowers em um único fluxo de trabalho.

## O que este Schema resolve

O OpenSpec gerencia "o que fazer" (proposal → specs → design → tasks), enquanto o Superpowers gerencia "como fazer" (brainstorming, writing-plans, subagent-driven-development). Ambos são excelentes individualmente, mas ao alternarem entre si no desenvolvimento real, surgem três problemas estruturais:

**Duplicação de produção** — o brainstorming gera documentos de design no diretório do Superpowers (`docs/superpowers/specs/`), enquanto o OpenSpec reescreve proposal/design no diretório change, com conteúdo muito semelhante.

**Fragmentação de Tasks** — o `tasks.md` do OpenSpec (checkboxes de granularidade grossa) e o plan do Superpowers (passos micro TDD) descrevem a mesma coisa, mas com formatos, localizações e rastreamento de estado independentes.

**Orquestração manual** — o usuário precisa decidir sozinho "qual habilidade usar agora", sem integração automática entre os dois sistemas.

**Incompatibilidade de entrada** — fontes externas como PRDs, tickets ou documentos de produto não têm um ponto de entrada padronizado no pipeline. O brainstorm espera uma ideia vaga; o proposal espera decisões tomadas. Sem uma camada de normalização, o usuário precisa adaptar manualmente o formato da entrada ao que cada artefato espera.

## Por que usar um Schema personalizado em vez de modificar as habilidades existentes

Duas alternativas foram consideradas:

- **Adicionar campos personalizados no `config.yaml`** (como `skill_bindings`): o CLI do OpenSpec não reconhece esses campos, sem validação, sem descoberta, e seria necessário modificar vários SKILL.md para lê-los.
- **Modificar diretamente os arquivos de habilidades do opsx**: invasivo, afeta todos os changes, e seria sobrescrito ao atualizar os SKILL.md.

O schema personalizado utiliza o mecanismo de schema em nível de projeto suportado nativamente pelo OpenSpec:

- O CLI valida a estrutura do schema
- `openspec schemas` lista automaticamente
- Cada change pode escolher seu schema de forma independente (`--schema spec-driven` ou `--schema sdd-plus-superpowers`)
- Nenhum SKILL.md ou arquivo de comando existente é alterado

## Visão geral do fluxo de trabalho

```
intent (opcional) ──→ brainstorm ──→ proposal ──→ specs ──→ tasks ──→ plan
        │                                                     ↑
        └───────────────────→ design ─────────────────────────┘
```
O `intent.md` funciona como normalizador de entrada: qualquer fonte externa (PRD, ticket, conversa, documento de produto) é destilada em um formato padronizado que o pipeline consegue consumir. Se todas as decisões já foram tomadas (campo "Questões em aberto" vazio), o brainstorm é pulado e o proposal consome o intent diretamente.

### Diferenças em relação ao spec-driven:

| | spec-driven | sdd-plus-superpowers |
|---|---|---|
| Ponto de entrada | proposal (escrito manualmente) | intent (opcional) → brainstorm (invoca a habilidade brainstorming) |
| Normalização de fontes externas | — | intent (aceita PRD, ticket, conversa) |
| Ponto de saída | tasks (granularidade grossa) | plan (passos micro TDD) |
| apply requer | tasks | plan |
| Método de apply | task-by-task padrão | worktree + subagent-driven-development |
| Novos artefatos | — | intent, brainstorm, plan |

## Habilidades Superpowers integradas

| Fase do Schema | Habilidade Superpowers invocada | Como é acionada |
|---|---|---|
| artefato brainstorm | `superpowers:brainstorming` | instrução do artefato |
| artefato plan | `superpowers:writing-plans` | instrução do artefato |
| fase apply | `superpowers:using-git-worktrees` | instrução do apply |
| fase apply | `superpowers:subagent-driven-development` | instrução do apply |
| após apply | `superpowers:finishing-a-development-branch` | instrução do apply |

Todas as integrações são realizadas pelo campo `instruction` do `schema.yaml` — instruindo a IA a invocar a habilidade correspondente no momento adequado. Nenhum arquivo de habilidade do Superpowers é modificado.

## Redirecionamento de saída (Output Redirection)

As habilidades do Superpowers têm caminhos de saída padrão (ex.: brainstorming escreve em `docs/superpowers/specs/`). Neste schema, a instrução do artefato inclui diretrizes de redirecionamento, informando à habilidade invocada para gravar no diretório do change:

- brainstorming → `openspec/changes/<name>/brainstorm.md` (+ `design.md` opcional)
- writing-plans → `openspec/changes/<name>/plan.md`

Isso é feito por injeção de contexto (anexando instruções ao invocar a habilidade), sem modificar o código das habilidades.

## Como usar

### Fluxo rápido (recomendado)

```bash
/opsx/ff my-feature    # tudo em um: cria diretório + intent + brainstorm + proposal + design + specs + tasks + plan
/opsx/apply            # worktree + subagent-driven-development
/opsx/archive          # arquivar
```

### Fluxo com fonte externa (PRD, ticket, etc.)

```bash
/opsx/new my-feature --schema sdd-plus-superpowers
/opsx/continue         # → intent (cole ou referencie o PRD/ticket como fonte)
# Se intent tem questões em aberto:
/opsx/continue         # → brainstorm (resolve as questões)
# Se intent não tem questões em aberto:
# brainstorm é pulado automaticamente
/opsx/continue         # → proposal
/opsx/continue         # → design
/opsx/continue         # → specs
/opsx/continue         # → tasks
/opsx/continue         # → plan
/opsx/apply
/opsx/archive
```

### Fluxo passo a passo (sem fonte externa)

```bash
/opsx/new my-feature --schema sdd-plus-superpowers
# intent é opcional — pule se a ideia ainda é vaga
/opsx/continue         # → brainstorm (conversa interativa)
/opsx/continue         # → proposal
/opsx/continue         # → design
/opsx/continue         # → specs
/opsx/continue         # → tasks
/opsx/continue         # → plan
/opsx/apply
/opsx/archive
```

### Voltar para spec-driven

```bash
# Usar um schema diferente em um único change
/opsx/new my-simple-fix --schema spec-driven

# Ou alterar o padrão do projeto
# openspec/config.yaml: schema: spec-driven
```

## Registro de decisões de design

### Por que intent é um artefato opcional e não obrigatório

O intent.md resolve um problema real (normalização de fontes externas), mas nem todo change tem uma fonte externa. Quando o desenvolvedor começa com uma ideia vaga, o brainstorm já cumpre o papel de exploração — forçar um intent antes seria burocracia sem valor. Marcá-lo como `optional: true` permite três caminhos de entrada:

- **Fonte externa madura (PRD, ticket detalhado)** → intent (sem questões abertas) → proposal (brainstorm pulado)
- **Fonte externa incompleta** → intent (com questões abertas) → brainstorm (resolve questões) → proposal
- **Ideia vaga, sem documento** → brainstorm direto → proposal

### Por que brainstorm é um artefato e não um hook

O brainstorming é uma conversa interativa de múltiplas rodadas que requer participação do usuário. Tratá-lo como o primeiro artefato, em vez de um hook no nível do schema, traz dois benefícios:

- **Pode ser pulado** — se o usuário já sabe o que quer fazer (via intent.md sem questões abertas ou criando o `brainstorm.md` diretamente sem invocar a habilidade)
- **Rastreável** — `openspec status` consegue exibir se o brainstorm foi concluído, e os artefatos seguintes têm dependências claras

### Por que plan é independente de tasks

O `tasks.md` é composto por checkboxes de granularidade grossa ("adicionar PdfServiceTest"), enquanto o `plan.md` contém passos micro ("criar esqueleto do teste → escrever teste downloadPdf → executar → commit"). Granularidade e propósito são diferentes:

- `tasks.md` → rastreia o progresso geral (o campo `tracks` da fase apply analisa os checkboxes)
- `plan.md` → guia o subagente na implementação passo a passo (entrada do executor)

A fase apply requer `plan` em vez de `tasks`, pois o executor precisa de passos micro para trabalhar com eficiência. Porém, `tracks: tasks.md` garante que o progresso ainda seja rastreado pelos checkboxes de granularidade grossa.

## Estratégia de fallback

Se as habilidades do Superpowers não estiverem disponíveis (não instaladas, versão incompatível, etc.), cada instrução inclui um caminho de fallback:

- intent → escrever `intent.md` manualmente (sempre disponível, não depende de habilidade externa)
- brainstorm → escrever `brainstorm.md` manualmente
- plan → escrever `plan.md` manualmente
- apply → implementação manual task-by-task padrão


## Instalação
 
### Pré-requisitos
 
- OpenSpec CLI versão 1.0.0 ou superior (`openspec --version`)
- OpenSpec inicializado no projeto (`openspec init`)
- Superpowers instalado no Claude Code (para as habilidades integradas)
 
### Passo a passo
 
**1. Crie a estrutura no projeto (se ainda não existir):**
 
```bash
mkdir -p openspec/schemas/sdd-plus-superpowers/templates
```
 
**2. Copie os arquivos do schema:**

#### Opção 1 — Clone parcial (recomendado)
 
```bash
# Clone apenas o commit mais recente
git clone --depth 1 https://github.com/barnadsv/OpenSpec.git /tmp/openspec-schemas
 
# Copie o schema para o projeto
mkdir -p openspec/schemas/sdd-plus-superpowers
cp -R /tmp/openspec-schemas/schemas/sdd-plus-superpowers/* openspec/schemas/sdd-plus-superpowers/
 
# Limpe o temporário
rm -rf /tmp/openspec-schemas
```
 
#### Opção 2 — Download direto via Gitea raw (sem git)
 
```bash
# Crie a estrutura local
mkdir -p /tmp/sdd-plus-superpowers/templates
 
# Baixe schema e README
curl -o /tmp/sdd-plus-superpowers/schema.yaml \
  "https://git.senado.leg.br/policia/OpenSpec/raw/branch/main/schemas/sdd-plus-superpowers/schema.yaml"
 
curl -o /tmp/sdd-plus-superpowers/README.md \
  "https://git.senado.leg.br/policia/OpenSpec/raw/branch/main/schemas/sdd-plus-superpowers/README.md"
 
# Baixe todos os templates
for f in intent.md brainstorm.md proposal.md design.md spec.md tasks.md plan.md; do
  curl -o "/tmp/sdd-plus-superpowers/templates/$f" \
    "https://git.senado.leg.br/policia/OpenSpec/raw/branch/main/schemas/sdd-plus-superpowers/templates/$f"
done
 
# Copie para o projeto
mkdir -p openspec/schemas/sdd-plus-superpowers
cp -R /tmp/sdd-plus-superpowers/* openspec/schemas/sdd-plus-superpowers/
 
# Limpe o temporário
rm -rf /tmp/sdd-plus-superpowers
```
 
A estrutura resultante deve ser:
 
```
openspec/schemas/sdd-plus-superpowers/
├── schema.yaml
├── README.md
├── commands/
│   ├── ff.md
│   ├── new.md
│   └── continue.md
└── templates/
    ├── intent.md
    ├── brainstorm.md
    ├── proposal.md
    ├── design.md
    ├── spec.md
    ├── tasks.md
    └── plan.md
```
 
**3. Copie os comandos extras do schema:**

```bash
cp openspec/schemas/sdd-plus-superpowers/commands/*.md .claude/commands/opsx/
```

Isso instala `/opsx:ff`, `/opsx:new` e `/opsx:continue`. Para atualizar após upgrade do schema, rode o mesmo comando novamente.

**4. Valide o schema:**
 
```bash
openspec schema validate sdd-plus-superpowers
```
 
**5. Confirme que está visível:**
 
```bash
openspec schema which sdd-plus-superpowers
```
 
Deve retornar `Source: project`.
 
**6. (Opcional) Ative como schema padrão do projeto:**
 
```yaml
# openspec/config.yaml
schema: sdd-plus-superpowers
```
 
Se preferir usar por change sem alterar o padrão global:
 
```bash
/opsx/new my-feature --schema sdd-plus-superpowers
```
 
