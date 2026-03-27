# Agibank-Performance

Teste técnico de performance (JMeter) para o cenário de compra de passagem aérea no BlazeDemo.

Obs: As execuções não atingiram o critério de aceite (`>=250 req/s` e `P90 < 2000 ms`). Após revisar e rodar diversas vezes, a configuração da automação está correta, possivelmente o ambiente é limitado para altas cargas.

## Cenário coberto

- URL alvo: `https://www.blazedemo.com`
- Jornada validada: busca de voo - escolha de voo - compra concluída com sucesso
- Critério funcional: resposta deve conter `Thank you for your purchase today!`

## Estrutura do projeto

- `jmeter/plans/blazedemo_load_test.jmx`: teste de carga
- `jmeter/plans/blazedemo_spike_test.jmx`: teste de pico
- `jmeter/plans/blazedemo_capacity_step_test.jmx`: step test formal de capacidade (20 -> 40 -> 60 -> ...)
- `jmeter/data/routes.csv`: massa de dados de origens e destinos

### 1) Teste de carga

- Arquivo: `jmeter/plans/blazedemo_load_test.jmx`
- Objetivo: sustentar vazão de **250 req/s**
- Configuração principal:
  - Thread Group: `700` usuários, ramp-up `60s`, duração `900s`
  - Constant Throughput Timer: `15000` amostras/min (aprox. `250 req/s`)
  - Fluxo completo encapsulado em Transaction Controller (`TRX - Compra de passagem`)

### 2) Teste de pico

- Arquivo: `jmeter/plans/blazedemo_spike_test.jmx`
- Objetivo: gerar pico acima do critério e verificar estabilidade
- Configuração principal:
  - Thread Group: `1200` usuários, ramp-up `5s`, duração `300s`
  - Constant Throughput Timer: `30000` amostras/min (aprox. `500 req/s`)
  - Fluxo completo encapsulado em Transaction Controller (`TRX - Compra de passagem`)
- Observação: o plano foi desenhado para pico acima de `250 req/s`, mas o resultado real depende da capacidade do SUT e do gerador de carga.

## Instruções para execução

## Pré-requisitos

- Java 11+ instalado
- Apache JMeter 5.6+ instalado e disponível no PATH (`jmeter`)

## Execução em modo não-GUI

### Teste de carga

```bash
jmeter -n -t jmeter/plans/blazedemo_load_test.jmx -l results/load_test.jtl -e -o results/load_dashboard
```

### Teste de pico

```bash
jmeter -n -t jmeter/plans/blazedemo_spike_test.jmx -l results/spike_test.jtl -e -o results/spike_dashboard
```

### Step test de capacidade (arquivo separado, sem impactar evidências >=200)

```bash
jmeter -n -t jmeter/plans/blazedemo_capacity_step_test.jmx -l results/capacity_step_test.jtl -Jcap.startRps=20 -Jcap.incrementRps=20 -Jcap.maxRps=200 -Jcap.stepDurationSec=120 -Jcap.totalDurationSec=1200 -Jcap.threads=800 -Jcap.ramp=30
```

Parâmetros principais do step test:

- `cap.startRps`: vazão inicial do primeiro degrau (default `20`)
- `cap.incrementRps`: incremento por degrau (default `20`)
- `cap.maxRps`: teto de vazão (default `200`)
- `cap.stepDurationSec`: duração de cada degrau em segundos (default `120`)
- `cap.totalDurationSec`: duração total do teste (default `1200`)
- `cap.threads`: concorrência máxima disponível para sustentar os degraus (default `800`)
- `cap.ramp`: ramp-up de threads (default `30`)

### Resultado de execução - Step test formal (20 -> 200 RPS)

Execução realizada em `27/03/2026` com:

- `cap.startRps=20`
- `cap.incrementRps=20`
- `cap.maxRps=200`
- `cap.stepDurationSec=120`
- `cap.totalDurationSec=1200`
- `cap.threads=800`
- `cap.ramp=30`

Resultados por degrau (sampler `TRX - Compra de passagem`):

| Degrau | Alvo (RPS) | RPS real | P90 (ms) | Erro (%) |
|---|---:|---:|---:|---:|
| 1 | 20  | 8.20  | 21620 | 0.00 |
| 2 | 40  | 10.75 | 16959 | 0.00 |
| 3 | 60  | 13.68 | 36133 | 17.22 |
| 4 | 80  | 18.35 | 30881 | 36.44 |
| 5 | 100 | 22.14 | 30823 | 58.82 |
| 6 | 120 | 28.69 | 12303 | 21.56 |
| 7 | 140 | 33.60 | 14211 | 3.08 |
| 8 | 160 | 36.75 | 17897 | 3.18 |
| 9 | 180 | 31.46 | 32161 | 27.51 |
| 10 | 200 | 35.01 | 25810 | 38.83 |

Conclusão do step test:

- Capacidade sustentável observada ficou muito abaixo da meta de `250 req/s`.
- O maior throughput observado por degrau foi `36.75 req/s` (degrau alvo `160 RPS`), com P90 ainda acima do SLA.
- O SLA de aceitação (`P90 < 2000 ms`) não foi atendido em nenhum degrau.

## Verificação dos critérios de aceitação

Avaliar no relatório HTML (dashboard) ou em listeners agregados:

- Vazão (`Throughput`) >= `250 req/s`
- Tempo de resposta no percentil 90 (`90th pct`) < `2000 ms`
- Taxa de erro idealmente `0%` para o sampler transacional (`TRX - Compra de passagem`)

## Relatório de execução dos testes

Execução realizada localmente em `27/03/2026` (Windows, JMeter 5.6.3, Java 21).  
Métricas calculadas sobre o sampler transacional `TRX - Compra de passagem`.

### Execução 1 - Carga (250 req/s)

- Data/hora: `27/03/2026 12:57 BRT`
- Duração total: `900 s` (janela planejada)
- Throughput médio (`req/s`): `52.52`
- Percentil 90 (`ms`): `14447`
- Taxa de erro (`%`): `8.50`
- Resultado frente ao critério: **Reprovado**

### Execução 2 - Pico (>=250 req/s, alvo 500 req/s)

- Data/hora: `27/03/2026 13:17 BRT`
- Duração total: `300 s`
- Throughput médio (`req/s`): `60.49`
- Percentil 90 (`ms`): `30246`
- Taxa de erro (`%`): `10.44`
- Resultado frente ao critério: **Reprovado**

## Calibração adicional (threads)

Foi executada uma bateria adicional de calibração no plano de carga, com janela curta e variação de concorrência:

- `load_t200_d120.jtl` -> Throughput `42.26 req/s`, P90 `24750 ms`, erro `17.84%`
- `load_t400_d120.jtl` -> Throughput `29.66 req/s`, P90 `31974 ms`, erro `46.73%`
- `load_t800_d120.jtl` -> Throughput `24.65 req/s`, P90 `35027 ms`, erro `71.12%`

Conclusão da calibração:

- O melhor resultado desta rodada foi com menor concorrência (`200` threads).
- Aumentar threads piorou throughput e estabilidade, reforçando saturação do ambiente alvo.

