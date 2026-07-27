# Template de PRD na skill `ralvin-new-simple-project`

**Data:** 2026-07-25
**Status:** Implementado (commit `e63e0d1`)
**Versão do plugin:** `ralvin-skills` 0.1.1

## Objetivo

Fazer a skill gerar `docs/PRD.md` no projeto criado, contendo a especificação do
projeto, a arquitetura-alvo do backend (Clean Architecture + CQRS) e a estratégia
de testes — incluindo uma regra de ArchUnit que verifica se todo entrypoint tem
teste de integração.

## O desalinhamento entre as fontes e o destino

As informações de arquitetura vieram de duas skills existentes, ambas
incompatíveis com o projeto que esta skill gera:

| | Skills fonte | O que a skill gera |
|---|---|---|
| Linguagem | Kotlin | Java |
| Módulos | `api` + `core` (multi-módulo) | módulo único |
| Banco em teste | Oracle via Testcontainers | H2 em memória |
| Mensageria | `@KafkaListener` na regra ArchUnit | sem `spring-kafka` |

Fontes consultadas: `scaffold-kotlin-monorepo` (nomes de camada) e
`quality-setup-integration-test-module` (regra ArchUnit, `BaseIntegrationTest`,
`archunit.properties`).

Esse desalinhamento é a origem de todas as decisões abaixo. Copiar as fontes
literalmente produziria um PRD que descreve um projeto diferente do gerado.

## Decisões

### 1. O PRD é documento, não scaffold

A skill escreve `docs/PRD.md` e **não** cria os pacotes de camada, os testes nem
adiciona ArchUnit ao `build.gradle`. O documento é a especificação do que deve
ser construído.

O template declara isso na abertura, e o `SKILL.md` repete a instrução no passo
6. Sem essa marcação explícita, o PRD seria lido como descrição do que já existe
— e a primeira reação de quem abrisse o projeto seria procurar pacotes que não
estão lá.

### 2. Exemplos de código em Java

Os trechos das skills fonte são Kotlin e não compilariam no projeto gerado. A
regra ArchUnit e o `BaseIntegrationTest` foram traduzidos para Java.

Um detalhe da tradução: o original usa `allClasses.any { ... }`, mas em Java
`JavaClasses` implementa `Iterable` sem expor `stream()`. A versão Java usa um
laço `for` explícito.

### 3. Camadas como pacotes, não módulos

A `scaffold-kotlin-monorepo` prescreve módulos `api` e `core`. Prescrever isso
aqui contradiria o projeto de módulo único gerado pelo Initializr — quem lesse o
PRD teria de reestruturar antes de escrever a primeira linha.

As camadas viraram pacotes sob o pacote base:

```
domain/            regras de negócio puras, sem framework
application/       command/ e query/ (CQRS)
infrastructure/    jdbc/ e http/
buildingblocks/    config/ e exception/
entrypoint/        controller/, listener/, job/
```

O agrupamento `entrypoint/` é acréscimo em relação à fonte, que espalha
controllers no módulo `api`. Reunir os três tipos de ponto de entrada sob um
pacote deixa a regra de ArchUnit mais legível.

### 4. H2 primeiro, Testcontainers como evolução

O PRD parte do H2 que já vem no scaffold, de modo que o primeiro teste de
integração roda sem Docker. O bloco de migração para Testcontainers está
documentado como passo seguinte, contido no `BaseIntegrationTest`.

Concentrar a infraestrutura na classe base é o que torna essa troca um edit em um
arquivo, sem tocar em nenhum teste.

### 5. ArchUnit com escopo deliberadamente restrito

Uma única regra: todo entrypoint tem `<Nome>IT` estendendo `BaseIntegrationTest`.
O ArchUnit **não** valida as regras de dependência entre camadas — essas ficam
por revisão de código.

Uma suíte ampla de ArchUnit tende a virar ruído e ser desativada no primeiro
conflito. Uma regra só, que responde a uma pergunta objetiva, sobrevive.

Mantidos da fonte: `FreezingArchRule` (congela violações existentes, permitindo
ligar a regra sem correção em massa), a exclusão de `@RestControllerAdvice` e o
`archunit.properties`.

### 6. Kafka fica documentado, não na regra

A regra original inclui `@KafkaListener`, cujo import exige `spring-kafka` — que
não está nas dependências do scaffold. Aquela linha **não compilaria**.

A regra entregue cobre `@RestController` e `@Scheduled`. O trecho do Kafka está
no PRD logo abaixo da regra, com a instrução de incluí-lo quando a dependência
entrar no projeto.

Consequência a registrar: Listeners estão cobertos conceitualmente na seção de
testes, mas não na regra executável. Fechar essa lacuna exige que a skill passe a
adicionar `spring-kafka` — o que muda o escopo do "projeto simples".

### 7. A regra de marcação dos planos entra no PRD, não fica implícita

O `writing-plans` gera a sintaxe `- [ ]` e anuncia que ela serve de tracking,
mas as skills de execução (`executing-plans`, `subagent-driven-development`)
registram conclusão na lista de todos da sessão e num ledger — os dois somem
quando a sessão acaba. A única instrução do pacote para escrever `- [x]` de
volta no arquivo está em `using-superpowers/references/antigravity-tools.md`,
que existe porque aquele harness não tem ferramenta de todo, e não se aplica ao
Claude Code — não há `claude-code-tools.md`.

Resultado: nada na cadeia grava no artefato que sobrevive à sessão, e um plano
pode ser inteiramente executado com os checkboxes intactos. Um plano sem
marcação não distingue "não fiz" de "esqueci de marcar" — ambiguidade que
esconde step de verificação que nunca rodou.

Por isso a regra é do projeto e vai para a seção 5.1 do PRD, com o item
correspondente na definição de pronto da seção 5. O texto se declara aplicável
só a partir do primeiro plano em `docs/superpowers/plans/`, para não parecer
requisito de um scaffold que ainda não tem planos.

## Alterações no `SKILL.md`

- Novo **passo 6** cria `docs/` e escreve `docs/PRD.md` a partir do template,
  substituindo `{project-name}`, `{group}`, `{package}`, `{java-version}`,
  `{boot-version}` e `{dependencies}` pelos valores reais da geração
- Passos seguintes renumerados: Git (7), Verificar (8), Relatar (9)
- Diagrama de estrutura passa a mostrar `docs/PRD.md`
- `description` do frontmatter menciona o PRD
- Passo 6 explica a seção 5.1 do template (marcação dos planos) e instrui a
  mantê-la mesmo em projeto sem `docs/superpowers/plans/`

## Verificação

| Checagem | Resultado |
|---|---|
| Numeração dos passos | 1–9 sequencial, sem duplicata |
| Links relativos do `SKILL.md` | Os 5 resolvem, incluindo o novo template |
| Frontmatter YAML | Parseia |
| Placeholders do template | Os 6 são citados no passo 6 |
| `claude plugin validate` | Passa no marketplace e no plugin |

## Não verificado

O Java do PRD **não passou por compilador**. É documento de especificação e
nenhum projeto foi gerado para exercitá-lo. Pontos de risco: a assinatura de
`DescribedPredicate.describe` e o `ArchCondition` anônimo podem exigir ajuste na
primeira execução real.

A skill completa também nunca foi executada de ponta a ponta em pasta limpa.

## Fora de escopo

Gerar o código da arquitetura descrita, adicionar `spring-kafka`, trocar H2 por
Testcontainers no scaffold e validar as regras de dependência via ArchUnit.
