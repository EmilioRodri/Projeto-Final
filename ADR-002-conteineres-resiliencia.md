# ADR-002 — Contêineres Separados, Protocolos REST/gRPC/AMQP e Resiliência com Circuit Breaker + Outbox

| Campo | Valor |
|---|---|
| **Status** | Aceito |
| **Data** | 2026-11-25 (Sprint 4) |
| **Autor** | Gabriel — Pesquisador & Resiliência / Integração |
| **Revisores** | Emilio Rodrigues (Arquiteto Principal) |
| **Relacionadas** | ADR-001 (estilo), ADR-004 (dados), ADR-005 (Decorator/State) |

---

## 1. Contexto e Problema

A ADR-001 definiu uma arquitetura distribuída em contêineres. Falta decidir **como os contêineres se comunicam** e **como o sistema se comporta quando algo falha** — especialmente o LMS Legado, que é o elo mais fraco da cadeia.

Fluxos com perfis muito diferentes coexistem:

| Fluxo | Natureza | Risco principal |
|---|---|---|
| Web App ↔ API | Interativo, síncrono | Latência percebida pelo usuário |
| API ↔ Sandbox | Execução de código não confiável | Fuga de isolamento, consumo de recursos |
| API → Worker de Similaridade | Processamento pesado, pode demorar | Bloquear o fluxo interativo |
| API → LMS Legado | Integração externa instável | Indisponibilidade, nota duplicada ou perdida |

## 2. Opções Consideradas

**Comunicação**
1. REST em todos os fluxos.
2. Tudo por mensageria assíncrona.
3. **Comunicação híbrida**: cada protocolo no fluxo em que faz mais sentido.

**Resiliência na publicação de notas**
1. Chamada direta ao LMS com tratamento de exceção simples.
2. Transação distribuída (2PC) entre banco e LMS.
3. **Timeout + Retry com backoff + Circuit Breaker + Transactional Outbox + estado `PENDENTE_PUBLICACAO`.**

## 3. Decisão

### 3.1 Contêineres

| Contêiner | Responsabilidade | Acesso ao banco |
|---|---|---|
| **Web App (SPA)** | Interface de alunos e professores | Não |
| **API REST** | Coordenação, validação, regras de negócio | Sim (`schema_core`, `schema_auditoria`) |
| **Sandbox** | Execução isolada do código do aluno | **Não** (sem credenciais, sem rota para o PostgreSQL) |
| **Worker de Similaridade** (Python) | Análise léxica + estrutural assíncrona | Sim (`schema_similaridade`) |
| **Broker AMQP** (RabbitMQ) | Transporte de eventos | — |

### 3.2 Protocolos

| Fluxo | Protocolo | Motivo |
|---|---|---|
| Web App ↔ API | **REST/HTTPS** (OpenAPI 3.0) | Interoperabilidade, fácil de testar e observar |
| API ↔ Sandbox | **gRPC** (Protobuf) | Contrato tipado, deadline e cancelamento nativos |
| API → Worker | **AMQP** | Desacoplamento produtor/consumidor, absorção de picos |
| API → LMS | **REST/HTTPS** via Adapter (ACL) | Único contrato aceito pelo mainframe |

### 3.3 Política de resiliência (somente na integração com o LMS)

| Mecanismo | Parâmetro |
|---|---|
| **Timeout** | 1 s por chamada ao LMS |
| **Retry** | 2 tentativas adicionais (3 no total), backoff exponencial 200 ms → 400 ms com *jitter* (±20%); somente para erros transitórios (timeout, 5xx, 429) |
| **Circuit Breaker** | Janela **deslizante de 20 chamadas**; mínimo de 10 chamadas antes de avaliar; abre com **≥ 50% de falhas**; permanece **aberto por 30 s**; estado **meio‑aberto** permite 3 chamadas de sondagem |
| **Fallback** | Nota persistida como `PENDENTE_PUBLICACAO` |
| **Transactional Outbox** | Nota e registro em `outbox_publicacao_lms` gravados **na mesma transação**; um *relay* reenvia em background |
| **Idempotência** | Chave `Idempotency-Key = submissaoId + versaoNota` enviada ao LMS e verificada no consumidor |
| **DLQ** | Mensagens que falham após N tentativas vão para fila de mensagens mortas, com alerta para intervenção manual |

> ⚠️ Os parâmetros de timeout de 1 s e 2 retentativas **valem apenas para o LMS**. A chamada ao Sandbox usa deadline gRPC próprio (padrão 10 s, configurável por atividade).

### Máquina de estados da publicação

```
CALCULADA ──► PUBLICANDO ──► PUBLICADA
                 │
                 └─(falha / CB aberto)─► PENDENTE_PUBLICACAO ──(relay)──► PUBLICANDO
                                                │
                                                └─(N falhas)─► FALHA_PUBLICACAO (DLQ + intervenção manual)
```

## 4. Justificativa

- **Separar o Sandbox** reduz a superfície de ataque e limita o impacto de código malicioso. Isolamento de contêiner sozinho **não elimina** todos os riscos; por isso somam‑se usuário sem privilégios, rede de saída bloqueada, limites de CPU/memória/tempo/processos e limpeza após cada execução.
- **gRPC** fornece deadline e cancelamento, essenciais para matar execuções infinitas.
- **Outbox** resolve o problema de "gravou no banco mas não enviou o evento" sem precisar de 2PC, que o mainframe não suporta.
- **Circuit Breaker** evita martelar um LMS degradado e libera threads da API rapidamente.

### Opções rejeitadas
- **REST em tudo:** acoplaria o tempo de resposta do usuário à análise de similaridade.
- **Tudo assíncrono:** tornaria a execução no Sandbox difícil de acompanhar e cancelar.
- **2PC:** não suportado pelo LMS legado e frágil sob falhas.

## 5. Consequências

### Positivas
- Uma queda do LMS não derruba a correção: as notas aguardam e são publicadas depois.
- Sandbox e Worker escalam de forma independente nos picos de entrega.

### Negativas / Trade-offs aceitos
- **Entrega pelo menos uma vez**, não "exatamente uma vez": todos os consumidores precisam ser **idempotentes**.
- Mais componentes para operar: broker, relay da Outbox, DLQ, políticas de rede.
- "Nunca duplicar notas" é um **critério de aceite a demonstrar em teste**, não uma garantia automática do padrão.
- **Correção registrada:** a implementação da Sprint 3 abria o circuito após 5 falhas consecutivas; deve ser alinhada à janela deslizante de 20 chamadas / 50% definida aqui.

## 6. Critérios de Verificação (Fitness Functions)

| ID | Teste | Resultado esperado |
|---|---|---|
| FF-002-1 | *Chaos test*: LMS responde 500 em 60% das chamadas | Circuito abre; notas vão para `PENDENTE_PUBLICACAO` |
| FF-002-2 | Reprocessar a mesma mensagem 5 vezes | Exatamente 1 nota publicada no LMS |
| FF-002-3 | Código do aluno tenta `curl` externo / *fork bomb* | Execução bloqueada e encerrada; API continua saudável |
| FF-002-4 | Derrubar o broker por 5 min | Nenhum evento perdido após o retorno (Outbox) |

## 7. Referências
- RICHARDSON, C. *Microservices Patterns* — Transactional Outbox, Idempotent Consumer. Manning, 2018.
- NYGARD, M. *Release It!* — Circuit Breaker, Timeouts. 2. ed. Pragmatic Bookshelf, 2018.
- Relatório Técnico "Arquitetura do CheckYouWork" — Equipe Sysops, nov/2026.

---

## 8. Validação da Equipe

Cada integrante deve ler este ADR, conferir o checklist abaixo e marcar sua linha na tabela. A ADR só passa para **Aceito (validado)** quando os 5 integrantes aprovarem.

**Checklist do revisor**
- [ ] O contexto e o problema estão corretos e coerentes com o projeto Check Your Work.
- [ ] A decisão é consistente com as demais ADRs (sem contradições de nomes, números ou componentes).
- [ ] As opções rejeitadas e os trade-offs estão bem justificados.
- [ ] Os critérios de verificação (Fitness Functions) são mensuráveis.
- [ ] Concordo com a decisão registrada.

| Integrante | Papel | Status | Data | Comentário |
|---|---|---|---|---|
| Emilio Rodrigues | Revisor | ✅ Aprovado | | |
| Gabriel | Autor | ⬜ Pendente (autor) | | |
| João Nicolas | Revisor | ✅ Aprovado | | |
| Leonardo Hamilton | Revisor | ✅ Aprovado | | |
| Marcell | Revisor | ✅ Aprovado | | |

> **Legenda:** ✅ Aprovado · ✏️ Aprovado com ajustes (descrever no comentário) · ❌ Reprovado (justificar) · ⬜ Pendente
