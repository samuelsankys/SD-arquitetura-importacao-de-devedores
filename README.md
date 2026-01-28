# SD-arquitetura-importacao-de-devedores

## O problema

Você está entrando em uma empresa de cobrança que processa milhões de interações por mês. Um dos sistemas mais críticos é o de importação de devedores, que alimenta toda a operação.

A empresa recebe dados de clientes (credores) via SFTP em dois fluxos:

**Carga semanal (base completa)**
- Disponibilizado todo domingo às 02h
- Base completa de devedores: ~10 milhões de registros
- Formato CSV
- Precisa estar pronto antes das 08h de segunda

**Atualizações horárias (baixas de pagamento)**
- Arquivos de hora em hora, das 08h às 20h
- Apenas atualizações: pagamentos, acordos, baixas
- ~5.000 registros por arquivo
- Precisa processar em até 15 minutos

## Algumas coisas que você precisa saber

- O SFTP do cliente é instável. Cai, dá timeout, a conexão trava.
- O cliente já mandou o mesmo arquivo duas vezes por engano. Mais de uma vez.
- Segunda-feira às 08h, se a base não estiver atualizada, o time de operações não consegue trabalhar.



# Decisão

### Caso 1

Adotar uma arquitetura orientada a eventos com processamento distribuído e assincrono.

---

<img width="816" height="777" alt="image" src="https://github.com/user-attachments/assets/1a7d1bd7-9128-41f5-b44b-49a730d27bc0" />


## Componentes e Responsabilidades

### 1. Input Service

**Responsabilidade**: Download seguro de arquivos do SFTP com resiliência.

| Aspecto | Decisão | O que resolve |
| --- | --- | --- |
| **Agendamento** | Cron: Domingo 02h  | Executa no horário definido |
| **Circuit Breaker** | Threshold: 3 falhas, Reset: 60s, Timeout: 5min | Evita várias tentativas quando tiver instabilidade |
| **Retry Policy** | Exponential backoff: 20min inicial, até 10 tentativas (~4h total) | Recuperação automática de falhas |
| **Watchdog** | monitora avanço de bytes; se não houver progresso por 3**0s**, aborta stream; heartbeat a cada **10s**; retry entra após aborto | Detecta e encerra conexões travadas. |
| **Timeout** | Ler timeout de operação | Impede que haja bloqueio em chamadas |
| **Storage** | Upload imediato via stream para S3 (imutabilidade garantida) | Garante persistência do arquivo |
| **Evento** | Publica `file_uploaded` com metadata e checksum | Desacopla os serviço |
|  |  |  |

### 2. Orchestrator Service

**Responsabilidade**: Coordenação do processamento, deduplicação e distribuição de trabalho.

| Aspecto | Decisão |  |
| --- | --- | --- |
| **Deduplicação** | Checksum SHA-256 ou MD5 do arquivo | Garante idempotência, evita reprocessamento acidental |
| **Chunking** | Divisão em lotes configuráveis | Proteger memória e banco (Será preciso validar como está o consumo do banco e quanto ele suporta) |
| **Fila Principal** | BullMQ com filas distribuidas | distribuição de carga e concorrência  |
| **Fila de Falhas** | DLQ para jobs que falharam após 3 retries |  |
| **State Machine** | Tracking de estado | Gerenciar os estados das operações. |
| **Checkpointing** | Persiste o ultimo chunk processado | Auxilia na retomada em caso de crach ou restart |

### 3. Worker

**Responsabilidade**: Processamento paralelo dos chunks com persistência no banco.

| Aspecto | Decisão |
| --- | --- |
| **Concorrência** | 10 workers paralelos  |
| **Batch Insert** | Lotes de 1.000 registros via  |
| **Upsert Strategy** | `INSERT ON DUPLICATE KEY UPDATE` |

### Caso 2

Adotar uma arquitetura simples para processamento sincrono.

---

<img width="498" height="424" alt="image" src="https://github.com/user-attachments/assets/a8b26ab1-ada4-42ac-9cb5-f4feacb8c9b6" />


## Componentes e Responsabilidades

### 1. Process Service

**Responsabilidade**: Realizar a ingestão, validação e processamento de arquivos de baixas de pagamento provenientes do SFTP, garantindo integridade, idempotência e resiliência a falhas

| Aspecto | Decisão |
| --- | --- |
| **Agendamento** | Hora em hora 08h-20h (updates) |
| **Circuit Breaker** | Threshold: 3 falhas, Reset: 60s, Timeout: 5min |
| **Retry Policy** | Exponential backoff: 5min inicial, até 5 tentativas (25min total) |
| **Timeout** | Ler timeout de operação |
| **Deduplicação** | Checksum SHA-256 ou MD5 do arquivo |
| **State Machine** | Tracking de estado |

---

## Decisões Técnicas Detalhadas

---

### Como você pensa?

Começo entendendo o contexto do problema e os riscos envolvidos, antes de pensar em solução técnica.

Meu primeiro passo é identificar:

- Quais são os requisitos funcionais
- Quais são os requisitos não funcionais, principalmente SLA, volume, confiabilidade e impacto no negócio
- Onde estão os pontos de falha mais prováveis

A partir disso, penso em técnicas e padrões arquiteturais que permitam resolver o problema respeitando essas restrições, sempre priorizando previsibilidade operacional, resiliência a falhas e simplicidade onde possível.

### O que você considerou e descartou?

**Considerei:**

Volume de dados, principalmente no caso da carga semanal

Instabilidade do SFTP, assumindo quedas, timeouts e travamentos como cenário normal

Idempotência, para lidar com arquivos duplicados e reprocessamentos

Distribuição de carga, para evitar sobrecarregar banco e infraestrutura

Capacidade de retomada, em caso de falhas no meio do processamento

**Descartei:** 

Processar diretamente do SFTP para o banco, devido ao alto acoplamento com um sistema instável e à dificuldade de retomar processamento em caso de falha

Processamento síncrono para a carga semanal, pois aumentaria muito o risco de não cumprir o SLA

Trazer todo o arquivo em memória

Usar a mesma arquitetura para os dois fluxos, já que os SLAs, volumes e impactos no negócio são diferentes

### Por que você escolheu o que escolheu ?

Eu parti principalmente do impacto de negócio e dos SLAs envolvidos, e não apenas nos volumes. A maior preocupação é garantir que, mesmo sob falhas o processo consiga terminar.

Para o caso 1 que o SLA era maior e com dados maiores de serem operados, o foco foi processar eventualmente e garantir a resiliência do sistema. 

Para o caso 2 que o SLA era menor e com dados menores, e o impacto muito menor que o primeiro caso, o processamento sincrono e mais simples fez mais sentido para resolver o problema.

Logo a ideia principal foi escalar onde precisa e simplificar onde for necessário para resolver o problema.

### Quais problemas antecipou?

- Arquivo travado sem gerar erro (watchdog)
- Checkpoint para retomada de processamento caso o worker morra no meio do processo
- Dead letter Queue(DLQ) para erros de processos
- Estouro de memória: utilização de stream e chunks controlados
- Tamanho do paralelismo definido para evitar sobrecarga no banco

**Como melhoraria** 

- Ajustaria o batch size de acordo com o comportamento do banco depois de medir o comportamento real.
- Retomada automática de processamento em caso a aplicação tenha sido reiniciada a partir do ultimo do checkpoint
