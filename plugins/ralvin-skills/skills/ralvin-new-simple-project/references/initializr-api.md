# API do Spring Initializr

Referência para gerar o backend chamando `start.spring.io` diretamente, em vez
de manter templates estáticos de `build.gradle`.

## Por que a API em vez de templates

O Spring Boot renomeia artefatos entre majors (ver `pitfalls.md`). Um
`build.gradle` congelado na skill envelhece em silêncio e passa a gerar projetos
que não compilam ou que usam starters descontinuados. A API sempre devolve o que
é correto para a versão escolhida.

O custo é a dependência de rede. Se não houver rede, **pare e avise** — não
improvise um `build.gradle` à mão.

## Passo 1 — Validar o metadata antes de gerar

Sempre consulte o metadata primeiro. Ele diz quais versões existem **hoje**;
não assuma as de ontem.

```bash
curl -s -H 'Accept: application/vnd.initializr.v2.2+json' \
  https://start.spring.io/metadata/client -o /tmp/initializr-meta.json
```

Campos relevantes: `bootVersion`, `javaVersion`, `type`, `packaging`,
`language`, e `dependencies`. Cada um traz `default` e `values`.

Confira que a `bootVersion` e a `javaVersion` pretendidas estão em `values`. Se
não estiverem, use o `default` do metadata e diga ao usuário o que mudou.

## Passo 2 — Gerar

```bash
curl -sS --max-time 90 -o produtos.zip -G https://start.spring.io/starter.zip \
  --data-urlencode 'type=gradle-project' \
  --data-urlencode 'language=java' \
  --data-urlencode 'bootVersion={boot-version}' \
  --data-urlencode 'groupId={group}' \
  --data-urlencode 'artifactId={artifact}' \
  --data-urlencode 'name={artifact}' \
  --data-urlencode 'packageName={package}' \
  --data-urlencode 'packaging=jar' \
  --data-urlencode 'javaVersion={java-version}' \
  --data-urlencode 'dependencies={deps}'
```

Use `-G` com `--data-urlencode` em vez de montar a query à mão: o `groupId` e o
`packageName` contêm pontos e o `import-alias` do frontend contém `*`.

### Valores de `type`

| Valor | Corresponde a |
|---|---|
| `gradle-project` | Gradle - Groovy |
| `gradle-project-kotlin` | Gradle - Kotlin |
| `maven-project` | Maven |

### IDs de dependência usados por padrão

| ID | Nome na UI | Grupo |
|---|---|---|
| `web` | Spring Web | Web |
| `actuator` | Spring Boot Actuator | Ops |
| `h2` | H2 Database | SQL |
| `data-jdbc` | Spring Data JDBC | SQL |

Outros IDs saem do próprio metadata, em `dependencies.values[].values[].id`.

## Passo 3 — Inspecionar antes de extrair

Nunca extraia um zip baixado sem olhar. Verifique path traversal e caminhos
absolutos:

```bash
python3 -c "
import zipfile
names = zipfile.ZipFile('produtos.zip').namelist()
bad = [n for n in names if n.startswith('/') or '..' in n.split('/')]
print('unsafe:', bad if bad else 'NONE')
print('roots:', {n.split('/')[0] for n in names})
"
```

O zip **não tem diretório raiz** — os arquivos vêm no nível superior. Extraia
direto no destino, sem achatar nada.

## Parâmetros que não existem na API

A UI do start.spring.io tem um seletor **Configuration: Properties / YAML** que
não é parâmetro da API e não aparece no metadata. `application.properties` é o
comportamento padrão. Para YAML, renomeie o arquivo depois da geração.
