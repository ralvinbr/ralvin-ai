---
name: ralvin-new-simple-project
description: >-
  Cria do zero um monorepo simples com backend Java Spring Boot (gerado pela API
  do Spring Initializr) e frontend Next.js, mais um Makefile na raiz para buildar
  e rodar os dois juntos. Use quando o usuário pedir "novo projeto simples",
  "criar projeto Java com Next.js", "monorepo Java + Next", "projeto do zero
  Spring Boot e frontend", ou invocar /ralvin-new-simple-project. Gera
  {project-name}-backend, {project-name}-frontend, Makefile, .gitignore da raiz e
  docs/PRD.md com a arquitetura-alvo (Clean Architecture + CQRS) e a estratégia
  de testes. Para projetos Kotlin multi-módulo com CQRS, use setup-kotlin-gradle.
argument-hint: "Sem argumentos — o nome vem da pasta raiz; group, pacote e dependências são perguntados com defaults"
---

# Novo projeto simples: Spring Boot + Next.js

Monorepo de dois módulos independentes, sem ferramenta de orquestração:

```
{project-name}/
├── {project-name}-backend/    Spring Boot (gradlew próprio)
├── {project-name}-frontend/   Next.js
├── docs/PRD.md                especificação e arquitetura-alvo
├── Makefile
└── .gitignore
```

## Quando usar

- Projeto novo, do zero, com backend Java e frontend web
- Quer o equivalente ao Spring Initializr somado a um frontend já plugado
- **Não** use para Kotlin multi-módulo com CQRS/Oracle — essa é a
  `setup-kotlin-gradle`

## Entradas

| Entrada | Origem | Default |
|---|---|---|
| `project-name` | Nome da pasta raiz do workspace | — |
| `group` | Perguntar | `br.com.ralvin` |
| `package` | Perguntar | `{group}.{project-name}` |
| `java-version` | Perguntar | `25` |
| `dependencies` | Perguntar | `web,actuator,h2,data-jdbc` |
| `boot-version` | Do metadata do Initializr | `default` do metadata |

Não invente a `boot-version`: leia do metadata (passo 2).

## Procedimento

### Passo 1 — Verificar o ambiente

```bash
java -version; node -v; npm -v
curl -s -o /dev/null -w "%{http_code}" --max-time 5 https://start.spring.io/metadata/client
```

Precisa de JDK compatível com a `java-version` escolhida, Node com npm, e a API
respondendo `200`. **Sem rede, pare e avise** — não escreva `build.gradle` à mão
(ver [referência do Initializr](./references/initializr-api.md)).

Se a pasta não estiver vazia, mostre o conteúdo e confirme com o usuário antes
de escrever qualquer coisa.

### Passo 2 — Validar o metadata

Siga [referência do Initializr](./references/initializr-api.md), seção
"Validar o metadata". Confirme que a `java-version` escolhida e a
`boot-version` pretendida existem em `values`. Se não existirem, use o `default`
do metadata e informe a mudança ao usuário.

### Passo 3 — Gerar o backend

Baixe o `starter.zip` com os parâmetros do usuário, **inspecione antes de
extrair** e extraia em `{project-name}-backend/`. Comandos completos na
referência do Initializr.

`artifactId` e `name` são o `{project-name}`. `rootProject.name` fica como o
`{project-name}`, sem o sufixo `-backend`: a pasta indica localização, o
artefato é o produto, e o jar não deve carregar o sufixo.

### Passo 4 — Gerar o frontend

```bash
npx --yes create-next-app@latest {project-name}-frontend \
  --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm --yes
```

Depois confirme que não nasceu repositório Git aninhado
(ver [armadilhas](./references/pitfalls.md)).

### Passo 5 — Makefile e .gitignore

Copie [Makefile.template](./templates/Makefile.template) para `Makefile` na
raiz, substituindo `{project-name}`. Copie
[gitignore-root.template](./templates/gitignore-root.template) para
`.gitignore`.

O `.gitignore` da raiz é mínimo de propósito: backend e frontend trazem os seus
próprios, e é assim que devem permanecer.

Confirme que a indentação das receitas ficou com TAB.

### Passo 6 — docs/PRD.md

Crie a pasta `docs/` e escreva `docs/PRD.md` a partir de
[PRD.md.template](./templates/PRD.md.template), substituindo todos os
placeholders: `{project-name}`, `{group}`, `{package}`, `{java-version}`,
`{boot-version}` e `{dependencies}`.

Use os valores **reais** da geração — a `boot-version` é a que voltou do
metadata, não a que a skill imaginou.

O PRD descreve a arquitetura-alvo do backend (Clean Architecture + CQRS, camadas
como pacotes) e a estratégia de testes: unitários no domínio, integrados em todo
entrypoint, e uma regra de ArchUnit restrita a verificar que entrypoint tem
teste de integração.

A seção 6.1 do template registra a regra de marcação dos planos: marcar
`- [x]` no commit de cada step, e deixar em aberto o que não foi verificado de
fato. Ela entra no scaffold porque nada na cadeia do superpowers escreve a
marcação de volta no arquivo do plano — as skills de execução registram
conclusão na lista de todos da sessão e num ledger, que não sobrevivem à sessão.
Mantenha a seção mesmo que o projeto ainda não tenha `docs/superpowers/plans/`;
o próprio texto diz que ela passa a valer quando o primeiro plano existir.

**Seção 5 — autenticação — é condicional.** O scaffold **não** gera login,
token nem Security, e isso não muda. A seção existe para que, no dia em que a
autenticação entrar, ela siga um desenho já decidido em vez de ser improvisada:
JWT em cookie `httpOnly` com header aceito em paralelo e tendo precedência,
`Max-Age` derivado da expiração do token, `Secure` por variável de ambiente,
endpoint de logout, CSRF sustentado por `SameSite=Lax` mais origem exata no
CORS, middleware em `src/middleware.ts` conferindo só presença, e 401
redirecionando nas duas pontas.

Copie a seção **exatamente como está no template**, sem reescrever nem resumir
— cada item ali corrige um erro observado em projeto real. Não a adapte ao
projeto na geração: ela é condicional por definição e o próprio texto se
apresenta assim.

Se o usuário disser que o projeto **não** terá autenticação, apague a seção 5
inteira e renumere as seguintes (Definição de pronto vira 5, com 5.1, e Fora de
escopo vira 6), ajustando também a referência à seção 5 no fim de "Fora de
escopo". Na dúvida, **mantenha** a seção: um projeto sem login apenas ignora um
trecho de documento, enquanto um projeto com login sem essa orientação erra de
formas que o build não pega.

**Não crie os pacotes de camada nem os testes descritos no PRD.** O documento é
a especificação do que deve ser construído; o scaffold entrega apenas a
estrutura mínima. O próprio template diz isso na abertura, e essa distinção
precisa continuar clara.

### Passo 7 — Git

Se ainda não for repositório, `git init`. Antes de commitar, audite o que entrou:

```bash
git add -A
git diff --cached --name-only | grep -cE 'node_modules|\.next/|build/'
```

O resultado precisa ser `0`. Faça um commit para o scaffold.

### Passo 8 — Verificar

Obrigatório. Sem isso não há como afirmar que o scaffold funciona.

```bash
make build > /tmp/build.log 2>&1; echo "EXIT=$?"
```

Exija `EXIT=0` e confirme o resultado dos testes no XML, não pela ausência de
erro (ver [armadilhas](./references/pitfalls.md), seção "Verificação").

Smoke test dos dois serviços juntos:

```bash
make run > /tmp/run.log 2>&1 &
until (curl -sf http://localhost:8080/actuator/health >/dev/null 2>&1 \
   && curl -sf http://localhost:3000 >/dev/null 2>&1); do sleep 2; done
curl -s http://localhost:8080/actuator/health   # espera status UP
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000   # espera 200
```

Encerre com SIGINT no grupo de processos do make e confirme que as portas 8080 e
3000 ficaram livres e que não sobrou processo órfão.

### Passo 9 — Relatar

Informe ao usuário:

- Versões reais geradas (Boot, Java, Next, React, Tailwind) — leia dos arquivos,
  não repita as desta skill
- Resultado da verificação com os códigos de saída observados
- As vulnerabilidades do `npm audit`, deixando explícito que **não** foram
  corrigidas e por quê (ver armadilhas)
- Que o banco não tem schema: com Spring Data JDBC e H2 sem entidades, a
  aplicação sobe limpa e não haverá tabela até a primeira migration. É o
  esperado, não defeito

## Fora de escopo

Integração entre frontend e backend (CORS, proxy de dev), Docker, CI/CD e código
de domínio. Não adicione controller nem entidade de exemplo: a skill entrega
apenas a criação inicial.

## Notas

- Sem Turborepo, Nx ou npm workspaces. Os dois projetos compilam independentes,
  e workspaces npm não abrangeriam o backend Java de qualquer forma.
- O `create-next-app` também gera `AGENTS.md` e `CLAUDE.md` dentro do frontend.
- `test-frontend` roda lint, não testes: o scaffold do Next não traz framework
  de teste. Troque quando houver um.
