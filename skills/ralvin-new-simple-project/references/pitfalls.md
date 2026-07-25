# Armadilhas conhecidas

Cada item aqui custou tempo de depuração real. Leia antes de "corrigir" algo que
parece estranho no scaffold gerado.

## Makefile

### `sed` engole a saída dos serviços

Prefixar log com `| sed 's/^/[backend] /'` funciona num terminal e falha em
qualquer outro lugar: o `sed` passa a usar buffer de bloco quando a saída não é
um TTY. Com `make run > log.txt`, o arquivo fica **sem nenhuma linha** dos
serviços, mesmo com tudo rodando normalmente.

Use `awk` com `fflush()`:

```make
| awk '{ print "[backend]  " $$0; fflush() }'
```

O `fflush()` do awk é portável entre BSD (macOS) e GNU. O modo unbuffered do sed
não é: `-l` no BSD, `-u` no GNU.

Dentro do Makefile o `$0` do awk vira `$$0`, senão o make interpreta como
variável dele.

### `trap` com `echo` imprime várias vezes

Todo subshell do grupo herda o `trap` e executa o corpo dele. Um
`trap 'echo "encerrando..."; kill 0' INT` imprimiu a mensagem **7 vezes** num
Ctrl+C. Mantenha o trap silencioso: `trap 'kill 0' INT TERM`.

### `make run` sai com código diferente de zero

`kill 0` derruba o grupo de processos inteiro, incluindo o próprio make. O
Ctrl+C portanto encerra com código ≠ 0 (144 no teste). É esperado para um alvo
que sobe servidor. Para CI, use `run-backend` / `run-frontend` com controle
próprio de ciclo de vida.

### Receitas precisam de TAB

Indentação de receita no Makefile é TAB, não espaços. Ao gerar o arquivo,
confirme:

```bash
grep -Pc '^\t' Makefile
```

## Spring Boot 4.x

### Os starters foram renomeados

Memória muscular do Boot 3.x gera dependências inexistentes:

| Boot 3.x | Boot 4.x |
|---|---|
| `spring-boot-starter-web` | `spring-boot-starter-webmvc` |
| (console embutido) | `spring-boot-h2console` |
| `spring-boot-starter-test` | granulares: `-actuator-test`, `-data-jdbc-test`, `-webmvc-test` |

Este é o motivo de a skill chamar a API do Initializr em vez de manter um
`build.gradle` estático.

### `HELP.md` já vem ignorado

O `.gitignore` gerado pelo Initializr tem `HELP.md` na primeira linha. Um
`git mv HELP.md` falha com "not under version control". Use `mv` comum.

## Frontend

### Nunca rode `npm audit fix --force`

O `npm audit` acusa vulnerabilidades high em dependências transitivas do Next
(`sharp`, `postcss`). A correção proposta instala `next@9.3.3` — sete majors
para trás, quebrando o projeto inteiro. A remediação é pior que o problema.

Reporte as vulnerabilidades ao usuário e **não faça nada**. A resolução real
depende do upstream do Next atualizar o range do `sharp`.

### `create-next-app` e repositório aninhado

Ele detecta um repositório Git existente na raiz e não cria outro. Confirme
mesmo assim, porque um `.git` aninhado quebra o versionamento do monorepo:

```bash
[ -d {project-name}-frontend/.git ] && echo "ATENÇÃO: .git aninhado"
```

### Avisos de install script do npm 11

O npm 11 bloqueia postinstall de `sharp` e `unrs-resolver` pelo gate
`allow-scripts`. O build de produção passa mesmo assim. Só libere com
`npm approve-scripts sharp` se a otimização de imagens der problema.

## Verificação

### `-q` do Gradle esconde o resultado

`./gradlew build -q | tail` suprime o "BUILD SUCCESSFUL", e num pipe o `$?`
passa a ser o da última etapa do pipeline, não o do gradle. Redirecione para
arquivo e capture o código explicitamente:

```bash
./gradlew build > /tmp/be.log 2>&1; echo "EXIT=$?"
```

### Jar gerado não prova que funciona

O build produz o jar antes de rodar os testes. Confirme o resultado real no XML:

```bash
python3 -c "
import glob, xml.etree.ElementTree as ET
for f in glob.glob('build/test-results/test/*.xml'):
    r = ET.parse(f).getroot()
    print(r.get('name'), 'tests=', r.get('tests'), 'failures=', r.get('failures'))
"
```

No smoke test do `make run`, o log do teste de contexto deve mostrar o
HikariPool inicializando e encerrando — é o que prova que o datasource foi
realmente conectado, e não apenas compilado.
