# Observability: Jenkins → InfluxDB → Grafana

Estudo de observabilidade de pipelines CI/CD. Este repositório documenta a arquitetura e os dashboards Grafana construídos para monitorar pipelines Jenkins de microsserviços.

---

## Sumário

- [Visão Geral da Arquitetura](#visão-geral-da-arquitetura)
- [As Ferramentas](#as-ferramentas)
  - [Jenkins](#jenkins)
  - [InfluxDB](#influxdb)
  - [Flux](#flux)
  - [Grafana](#grafana)
- [O Fluxo Completo dos Dados](#o-fluxo-completo-dos-dados)
- [Estrutura dos Dados no InfluxDB](#estrutura-dos-dados-no-influxdb)
- [Os Dashboards](#os-dashboards)
  - [Dashboard 1: Microservices (por namespace)](#dashboard-1-microservices-por-namespace)
  - [Dashboard 2: All Environments (visão geral)](#dashboard-2-all-environments-visão-geral)
- [Explicação de Cada Painel](#explicação-de-cada-painel)
  - [Global - Success Rate](#global---success-rate)
  - [Global - Job Count](#global---job-count)
  - [Production - Success Rate](#production---success-rate)
  - [Production - Job Count](#production---job-count)
  - [Production - Delivery Errors](#production---delivery-errors)
  - [Production - Mean Time to Delivery & Deploy](#production---mean-time-to-delivery--deploy)
- [Estrutura do JSON do Grafana](#estrutura-do-json-do-grafana)
- [Como Importar os Dashboards](#como-importar-os-dashboards)
- [Convenções de Nomenclatura](#convenções-de-nomenclatura)

---

## Visão Geral da Arquitetura

```
┌─────────────┐     envia métricas     ┌─────────────┐     consulta Flux     ┌─────────────┐
│   Jenkins   │ ─────────────────────▶ │   InfluxDB  │ ◀──────────────────── │   Grafana   │
│  (CI/CD)    │                        │  (bucket:   │                        │  (painéis)  │
│             │                        │  jenkinsci) │                        │             │
└─────────────┘                        └─────────────┘                        └─────────────┘
   executa                               armazena                               visualiza
   pipelines                             série temporal                         dashboards
```

Cada vez que um pipeline Jenkins termina, ele envia os dados para o InfluxDB. O Grafana consulta o InfluxDB a cada 1 minuto e atualiza os painéis automaticamente.

---

## As Ferramentas

### Jenkins

Jenkins é uma ferramenta de **automação de CI/CD** (Continuous Integration / Continuous Delivery). Ele executa pipelines automaticamente quando há um push de código ou quando acionado manualmente.

Cada execução de pipeline é chamada de **build** ou **job** e produz:
- Um **resultado**: `SUCCESS`, `FAILURE`, `UNSTABLE` ou `ABORTED`
- Uma **duração**: quanto tempo levou (em milissegundos)
- Um **número de build**: identificador sequencial da execução

Os jobs são organizados em **namespaces**:
| Namespace | O que contém |
|---|---|
| `microservices` | Pipelines dos microsserviços da aplicação |
| `frontend` | Pipelines do frontend |
| `infrastructure-terraform` | Pipelines de infraestrutura com Terraform |

---

### InfluxDB

InfluxDB é um **banco de dados de séries temporais** — otimizado para armazenar eventos que acontecem ao longo do tempo (métricas, logs, eventos de sistema).

Diferente de um banco relacional (MySQL, PostgreSQL), o InfluxDB é organizado em:
- **Bucket**: equivalente ao "banco de dados". O bucket usado aqui é `jenkinsci`.
- **Measurement**: equivalente à "tabela". O measurement é `jenkins_data`.
- **Tags**: campos indexados usados para filtrar (ex: `project_namespace`, `build_result`).
- **Fields**: valores numéricos medidos (ex: `build_number`, `build_time`).
- **Timestamp**: cada registro tem um tempo associado.

---

### Flux

Flux é a **linguagem de consulta do InfluxDB** — como o SQL é para bancos relacionais.

O operador `|>` é o "pipe" — passa o resultado de uma etapa para a próxima, como uma linha de montagem:

```flux
from(bucket: "jenkinsci")           // 1. busca os dados
  |> range(start: -24h)             // 2. filtra pelo período
  |> filter(fn: (r) => ...)         // 3. aplica condições
  |> group(columns: ["campo"])      // 4. agrupa os dados
  |> aggregateWindow(fn: mean)      // 5. calcula a agregação
```

Cada `|>` transforma a tabela de dados antes de passar para o próximo passo.

---

### Grafana

Grafana é a **camada de visualização**. Ele não armazena dados — apenas consulta fontes externas (como o InfluxDB) e renderiza os resultados em painéis visuais.

Cada painel tem:
- Uma **query Flux** que busca os dados
- Um **tipo de visualização**: número (stat), barras (bargauge), linha (timeseries)
- **Thresholds**: limiares que definem as cores conforme o valor
- **Variáveis**: filtros dinâmicos como o seletor de namespace

A datasource do InfluxDB é identificada pelo UID: `bfh788kcgnpc0a`.

---

## O Fluxo Completo dos Dados

```
1. Desenvolvedor faz push de código
            ↓
2. Jenkins detecta e inicia o pipeline
            ↓
3. Jenkins executa os steps (build, test, deploy...)
            ↓
4. Pipeline termina → Jenkins registra:
   - project_namespace: "microservices"
   - project_name: "delivery-payments-production"
   - project_path: "microservices/delivery-payments-production"
   - build_result: "SUCCESS"
   - build_number: 142
   - build_time: 185000 ms
            ↓
5. Jenkins envia os dados para o InfluxDB (bucket: jenkinsci)
            ↓
6. InfluxDB armazena com timestamp
            ↓
7. Grafana (a cada 1 min) executa as queries Flux
            ↓
8. Grafana renderiza os painéis com cores e valores atualizados
```

---

## Estrutura dos Dados no InfluxDB

Cada build Jenkins gera **dois registros** no InfluxDB:

| Registro | Campo (`_field`) | Valor (`_value`) | Para que serve |
|---|---|---|---|
| 1 | `build_number` | número do build (ex: 142) | contar execuções |
| 2 | `build_time` | duração em ms (ex: 185000) | medir tempo de execução |

Ambos os registros carregam as mesmas **tags**:

| Tag | Exemplo | Descrição |
|---|---|---|
| `project_namespace` | `microservices` | Namespace/grupo do job |
| `project_name` | `delivery-payments-production` | Nome do job |
| `project_path` | `microservices/delivery-payments-production` | Caminho completo |
| `build_result` | `SUCCESS` | Resultado da execução |

> **Por que dois registros?** Porque `build_number` e `build_time` são tipos de dado diferentes (identidade vs. duração). As queries filtram por `_field` para pegar apenas o dado relevante para cada cálculo.

---

## Os Dashboards

### Dashboard 1: Microservices (por namespace)

**Arquivo:** `dashboards/jenkins-pipelines-microservices.json`

**Propósito:** Drill-down por namespace específico. Permite filtrar por `frontend`, `infrastructure-terraform` ou `microservices` e ver apenas os dados daquele grupo.

**Painéis:** 6 (Global Success Rate, Global Job Count, Production Success Rate, Production Job Count, Production Delivery Errors, Production Mean Time)

**Filtro de namespace:**
```json
"templating": {
  "list": [{
    "name": "namespace",
    "type": "custom",
    "query": "frontend,infrastructure-terraform,microservices",
    "includeAll": false
  }]
}
```

As queries usam `== "${namespace}"` para filtrar exatamente o namespace selecionado.

---

### Dashboard 2: All Environments (visão geral)

**Arquivo:** `dashboards/jenkins-pipelines-all-environments.json`

**Propósito:** Visão executiva de todos os namespaces e ambientes simultaneamente. Sem filtro — exibe tudo.

**Painéis:** 14 organizados em 4 seções (rows):

| Seção | Filtro aplicado | Painéis |
|---|---|---|
| Global | Nenhum — todos os dados | Success Rate, Job Count |
| Production | `project_name =~ /production/` | Success Rate, Job Count, Delivery Errors, Mean Time |
| HML / Homologação | `project_name =~ /homolog/` | Success Rate, Job Count, Delivery Errors, Mean Time |
| Sandbox | `project_name =~ /sandbox/` | Success Rate, Job Count, Delivery Errors, Mean Time |

> **Por que dois dashboards separados?** O Grafana não consegue mostrar/esconder painéis dinamicamente com base em variáveis (sem plugins). A solução foi ter um dashboard específico por namespace e um segundo com a visão global de todos os ambientes.

---

## Explicação de Cada Painel

### Global - Success Rate

**Tipo:** Stat (número grande com fundo colorido)

**O que mostra:** Percentual de builds com resultado `SUCCESS` no período selecionado.

**Query:**
```flux
from(bucket: "jenkinsci")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_data")
  |> filter(fn: (r) => r._field == "build_number")
  |> filter(fn: (r) => r.project_namespace == "${namespace}")
  |> keep(columns: ["_time", "build_result"])
  |> map(fn: (r) => ({ _time: r._time, _value: if r.build_result == "SUCCESS" then 1.0 else 0.0 }))
  |> group()
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> map(fn: (r) => ({ r with _value: r._value * 100.0 }))
```

**Como funciona:**
1. Busca todos os builds do namespace no período
2. Converte cada build: `SUCCESS = 1.0`, qualquer outro resultado `= 0.0`
3. Calcula a média de todos os valores → isso já é a taxa de sucesso (ex: `0.8 = 80%`)
4. Multiplica por 100 para exibir como percentual

**Thresholds (cores):**
| Faixa | Cor | Significado |
|---|---|---|
| 0% – 50% | Vermelho | Crítico |
| 50% – 80% | Laranja | Atenção |
| 80% – 95% | Verde | Bom |
| 95% – 100% | Verde escuro | Excelente |

---

### Global - Job Count

**Tipo:** Bar Gauge (barras horizontais)

**O que mostra:** Contagem total de builds por resultado (`SUCCESS`, `FAILURE`, `UNSTABLE`, `ABORTED`).

**Query:**
```flux
from(bucket: "jenkinsci")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_data")
  |> filter(fn: (r) => r._field == "build_number")
  |> filter(fn: (r) => r.project_namespace == "${namespace}")
  |> group(columns: ["build_result"])
  |> count()
  |> group()
  |> keep(columns: ["build_result", "_value"])
```

**Como funciona:**
1. Agrupa todos os builds pelo campo `build_result` — cria 4 grupos
2. Conta quantos registros existem em cada grupo
3. Retorna uma linha por resultado com a contagem

A transformação `rowsToFields` no JSON converte as linhas em colunas separadas — necessário para o Grafana aplicar cores individuais por nome (SUCCESS = verde, FAILURE = vermelho, etc.).

---

### Production - Success Rate

Idêntico ao **Global - Success Rate**, com um filtro adicional:

```flux
|> filter(fn: (r) => r.project_name =~ /production/)
```

O operador `=~` significa "faz match com expressão regular". `/production/` seleciona qualquer job que contenha a palavra `production` no nome — ex: `delivery-payments-production`, `deploy-auth-production`.

---

### Production - Job Count

Idêntico ao **Global - Job Count**, com o mesmo filtro de `/production/`.

---

### Production - Delivery Errors

**Tipo:** Time Series (gráfico de barras empilhadas ao longo do tempo)

**O que mostra:** Quantidade de falhas (`FAILURE`) dos jobs `delivery-*-production` ao longo do tempo, separadas por serviço.

> **"No data" = boa notícia** — significa que não houve nenhuma falha no período selecionado.

**Query:**
```flux
import "strings"

from(bucket: "jenkinsci")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_data")
  |> filter(fn: (r) => r._field == "build_number")
  |> filter(fn: (r) => r.project_namespace == "${namespace}")
  |> filter(fn: (r) => r.project_name =~ /^delivery-/)
  |> filter(fn: (r) => r.project_name =~ /production/)
  |> filter(fn: (r) => r.build_result == "FAILURE")
  |> map(fn: (r) => ({ r with service: strings.split(v: r.project_path, t: "/")[1] }))
  |> keep(columns: ["_time", "service"])
  |> map(fn: (r) => ({ _time: r._time, service: r.service, _value: 1.0 }))
  |> group(columns: ["service"])
  |> aggregateWindow(every: v.windowPeriod, fn: sum, createEmpty: false)
```

**Passo a passo:**

| Etapa | O que faz |
|---|---|
| `filter(/^delivery-/)` | Só jobs que **começam** com `delivery-` (o `^` = início da string) |
| `filter(/production/)` | Só jobs de produção |
| `filter(build_result == "FAILURE")` | Só as falhas |
| `strings.split(v: project_path, t: "/")[1]` | Extrai o nome do serviço do caminho. Ex: `microservices/delivery-payments-production` → divide por `/` → pega posição `[1]` → `"delivery-payments-production"` |
| `group(columns: ["service"])` | Cria uma linha no gráfico por serviço |
| `aggregateWindow(fn: sum)` | Soma as falhas em cada janela de tempo |

---

### Production - Mean Time to Delivery & Deploy

**Tipo:** Time Series (linhas ao longo do tempo)

**O que mostra:** Duração média dos jobs `delivery-*-production` e `deploy-*-production`, separados por resultado (`SUCCESS` / `FAILURE`). Útil para identificar se builds que falham demoram mais ou menos que os que passam.

**Query:**
```flux
from(bucket: "jenkinsci")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "jenkins_data")
  |> filter(fn: (r) => r._field == "build_time")
  |> filter(fn: (r) => r.project_namespace == "${namespace}")
  |> filter(fn: (r) => r.project_name =~ /production/)
  |> filter(fn: (r) => r.project_name =~ /^(delivery-|deploy-)/)
  |> filter(fn: (r) => r.build_result == "SUCCESS" or r.build_result == "FAILURE")
  |> map(fn: (r) => ({
      _time: r._time,
      _value: float(v: r._value),
      category: (if r.project_name =~ /^delivery-/ then "delivery" else "deploy") + " - " + r.build_result
  }))
  |> group(columns: ["category"])
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: true)
```

**Destaques:**

| Linha | O que faz |
|---|---|
| `filter(_field == "build_time")` | Aqui queremos **duração**, não contagem — usa `build_time` em vez de `build_number` |
| `filter(/^(delivery-\|deploy-)/)` | Aceita jobs que começam com `delivery-` **ou** `deploy-` |
| `float(v: r._value)` | Converte o valor para decimal (necessário para calcular média) |
| `category: ... + " - " + build_result` | Cria rótulos como `"delivery - SUCCESS"` ou `"deploy - FAILURE"` — vira o nome da linha |
| `aggregateWindow(fn: mean)` | Calcula a média da duração em cada janela de tempo |
| `createEmpty: true` | Mantém pontos vazios quando não há dados — evita lacunas no gráfico |

**Linhas no gráfico:**
| Linha | Cor | Descrição |
|---|---|---|
| `delivery - SUCCESS` | Verde | Tempo médio de deliveries bem-sucedidas |
| `delivery - FAILURE` | Vermelho | Tempo médio de deliveries que falharam |
| `deploy - SUCCESS` | Azul | Tempo médio de deploys bem-sucedidos |
| `deploy - FAILURE` | Laranja | Tempo médio de deploys que falharam |

**Unidade:** milissegundos (`ms`) — o Grafana converte automaticamente para minutos/segundos na exibição.

---

## Estrutura do JSON do Grafana

Todo dashboard Grafana é um arquivo JSON. As seções principais:

```json
{
  "title": "Nome do dashboard",
  "uid": "identificador-unico",
  "schemaVersion": 42,
  "refresh": "1m",
  "timezone": "browser",

  "templating": { "list": [ /* variáveis/filtros */ ] },

  "panels": [
    {
      "id": 1,
      "type": "stat",
      "title": "Nome do painel",
      "gridPos": { "h": 8, "w": 6, "x": 0, "y": 0 },
      "datasource": { "uid": "bfh788kcgnpc0a" },
      "targets": [ { "query": "/* query Flux */" } ],
      "fieldConfig": { "defaults": { "thresholds": { /* cores */ } } }
    }
  ]
}
```

**`gridPos`** — O Grafana usa uma grade de 24 colunas:
- `w: 6` = ocupa 1/4 da largura
- `w: 18` = ocupa 3/4 da largura
- `w: 24` = ocupa a largura inteira
- `x` = coluna onde começa (0 = esquerda)
- `y` = linha onde começa (0 = topo)

**`uid`** — Identificador único do dashboard. Se importar o mesmo UID duas vezes, o Grafana sobrescreve o existente.

---

## Como Importar os Dashboards

### Via Interface do Grafana

1. Acesse seu Grafana
2. Menu lateral → **Dashboards** → **Import**
3. Clique em **Upload JSON file**
4. Selecione o arquivo desejado (`jenkins-pipelines-microservices.json` ou `jenkins-pipelines-all-environments.json`)
5. Confirme a datasource (UID: `bfh788kcgnpc0a`)
6. Clique em **Import**

### Via API

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer SEU_TOKEN_GRAFANA" \
  https://SEU_GRAFANA/api/dashboards/db \
  -d "{\"dashboard\": $(cat dashboards/jenkins-pipelines-microservices.json), \"overwrite\": true}"
```

---

## Convenções de Nomenclatura

Os jobs Jenkins seguem o padrão:

```
{tipo}-{servico}-{ambiente}
```

| Tipo | Ambiente | Exemplo |
|---|---|---|
| `delivery` | `production` | `delivery-payments-production` |
| `deploy` | `production` | `deploy-payments-production` |
| `delivery` | `homolog` | `delivery-payments-homolog` |
| `deploy` | `homolog` | `deploy-payments-homolog` |
| `delivery` | `sandbox` | `delivery-payments-sandbox` |
| `deploy` | `sandbox` | `deploy-payments-sandbox` |

> **Importante:** O ambiente de homologação usa `-homolog` (não `-hml`). As queries do dashboard HML filtram por `/homolog/`.

O `project_path` segue o padrão: `{namespace}/{project_name}`

Exemplo: `microservices/delivery-payments-production`

---

## Estrutura do Repositório

```
observability-jenkins-grafana/
├── README.md
└── dashboards/
    ├── jenkins-pipelines-microservices.json      # Dashboard por namespace (6 painéis)
    └── jenkins-pipelines-all-environments.json   # Dashboard todos os ambientes (14 painéis)
```
