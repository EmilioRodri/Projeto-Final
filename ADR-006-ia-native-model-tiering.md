# ADR-006 — Arquitetura IA-Native: Model Tiering (SLM local vs. LLM em nuvem), RAG Vetorial e Guardrails LGPD/JSON Schema

| Campo | Valor |
|---|---|
| **Status** | Proposto → Aceito na Sprint 4 |
| **Data** | 2026-11-25 (Sprint 4) |
| **Autor** | Marcell — Executor & Especialista em IA-Native |
| **Revisores** | Emilio Rodrigues (Arquiteto Principal), João Nicolas (dados), Leonardo Hamilton (QA) |
| **Relacionadas** | ADR-002 (Worker/AMQP), ADR-003 (auditoria e revisão humana), ADR-004 (schemas) |

---

## 1. Contexto e Problema

A equipe deseja usar IA generativa para:

1. **Feedback pedagógico** ao aluno ("por que meu teste falhou?", sugestões de melhoria);
2. **Apoio ao professor** na leitura das evidências de similaridade (resumo explicável).

Restrições do domínio:

- **Código-fonte associado à matrícula é dado pessoal** (LGPD). Não há presunção genérica de consentimento para enviá-lo a terceiros.
- Na Sprint 1, a análise **semântica por IA foi rejeitada como motor de detecção de plágio**: alto custo, baixa explicabilidade e risco de viés. O motor oficial é **híbrido léxico + estrutural (AST)**.
- **A IA nunca decide nota nem acusação de plágio** (revisão humana — ADR-003).
- Saídas de LLM são não determinísticas e podem "alucinar" ou quebrar o formato esperado pela API.
- Custo variável por token precisa caber no orçamento com ~120.000 submissões/ano.

## 2. Opções Consideradas

1. **Não usar IA generativa.**
2. **Somente LLM em nuvem** para tudo.
3. **Somente SLM local** (modelo pequeno auto-hospedado).
4. **Model Tiering**: SLM local como padrão + LLM em nuvem apenas para casos complexos com dados anonimizados, ambos com RAG e guardrails.

## 3. Decisão

Adotar a **opção 4**, implementada como um módulo **`Assistente IA`** executado no Worker (assíncrono, via AMQP), atrás da interface de domínio `GeradorFeedback` (DIP — ADR-003).

### 3.1 Model Tiering

| Tier | Modelo | Quando usar | Dados enviados |
|---|---|---|---|
| **Tier 1 — SLM local** | Modelo pequeno aberto (~3–8 B parâmetros, quantizado), servido dentro da rede interna | **Padrão** para feedback de erros de compilação/teste e resumos curtos | Código e saída de testes — **nunca saem da infraestrutura** |
| **Tier 2 — LLM em nuvem** | Provedor comercial com contrato de tratamento de dados (DPA) e sem retenção para treino | Somente se o Tier 1 falhar na validação ou o professor solicitar análise aprofundada | **Apenas conteúdo anonimizado**: sem nome, matrícula, e‑mail ou comentários identificadores |

**Roteador de modelos** (padrão Strategy — ADR-005): decide o tier por complexidade, resultado da validação e orçamento mensal. Se o orçamento do Tier 2 se esgotar, o sistema permanece apenas no Tier 1.

### 3.2 RAG Vetorial
- Base de conhecimento: enunciados, critérios de correção, casos de teste públicos e material da disciplina.
- Embeddings armazenados com **pgvector** na tabela `schema_similaridade.embeddings_feedback` (ADR-004), sem dados identificáveis de alunos.
- **Separação de finalidade:** o índice RAG **não é usado para detectar plágio**; serve apenas para contextualizar o feedback.
- Cada resposta cita os trechos do material usados (rastreabilidade).

### 3.3 Guardrails

| Camada | Guardrail |
|---|---|
| **Entrada (LGPD)** | Anonimizador remove PII antes do Tier 2; detecção de *prompt injection* em comentários do código do aluno (o código é tratado como **dado**, nunca como instrução) |
| **Saída (JSON Schema)** | Resposta precisa validar contra o schema `FeedbackIA` (`resumo`, `problemas[]`, `sugestoes[]`, `referencias[]`, `confianca`); campos proibidos: `nota`, `veredito_plagio` |
| **Conteúdo** | Bloqueio de entrega de solução completa (feedback pedagógico, não resposta pronta) |
| **Falha** | 1 nova tentativa → escalonamento para Tier 2 → *fallback* sem IA ("feedback automático indisponível") |
| **Auditoria** | Cada geração registra modelo, versão, tier, hash do prompt e da resposta em `auditoria_eventos` (ADR-003) |

## 4. Justificativa

- **SLM local como padrão** resolve o maior risco jurídico (código pessoal enviado a terceiros) e mantém custo previsível.
- **LLM em nuvem como exceção** preserva qualidade em casos difíceis sem expor dados identificáveis.
- **RAG** ancora a resposta no material oficial da disciplina, reduzindo alucinação.
- **JSON Schema** torna a saída da IA um contrato verificável, como qualquer outra integração do sistema.
- Processar no **Worker assíncrono** evita que a latência da IA afete a meta de < 2 s na submissão.

### Opções rejeitadas
| Opção | Motivo |
|---|---|
| Sem IA | Perde ganho pedagógico relevante para alunos e professores. |
| Somente LLM em nuvem | Envio de código pessoal a terceiros (LGPD), custo variável alto e dependência de fornecedor. |
| Somente SLM local | Qualidade insuficiente em análises complexas; sem caminho de escalonamento. |

## 5. Consequências

### Positivas
- Feedback útil sem violar a privacidade dos alunos.
- Saídas validadas e auditáveis; a IA não interfere na nota nem no veredito de plágio.

### Negativas / Trade-offs aceitos
- **Infraestrutura adicional** para servir o SLM (CPU/GPU, memória).
- **Manutenção de modelos:** atualização de versões, avaliação periódica de qualidade e de viés.
- Anonimização é **redução de risco**, não garantia absoluta — exige testes contínuos.
- Mais um ponto de não determinismo: testes precisam de *datasets* de avaliação, não apenas asserções exatas.

## 6. Critérios de Verificação (Fitness Functions)

| ID | Verificação | Meta |
|---|---|---|
| FF-006-1 | Payloads enviados ao Tier 2 inspecionados contra detector de PII | 0 ocorrências de nome/matrícula/e‑mail |
| FF-006-2 | Respostas validadas contra JSON Schema `FeedbackIA` | 100% válidas após guardrail (inválidas → fallback) |
| FF-006-3 | Resposta contendo campo `nota` ou `veredito_plagio` | Rejeitada |
| FF-006-4 | Conjunto de *prompt injection* em comentários de código | 0 instruções obedecidas |
| FF-006-5 | Proporção de chamadas atendidas pelo Tier 1 | ≥ 80% |
| FF-006-6 | Custo mensal do Tier 2 | Dentro do orçamento definido pela coordenação |

## 7. Referências
- Lei nº 13.709/2018 (LGPD).
- LEWIS, P. et al. *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*. NeurIPS, 2020.
- OWASP. *Top 10 for Large Language Model Applications*.
- Relatório de Imersão Sprint 1 — "Trade-off Tecnológico: O Motor de Similaridade".

---

## 8. Validação da Equipe

Cada integrante deve ler este ADR, conferir o checklist abaixo e marcar sua linha na tabela. **Resultado: ADR aprovado pelos 5 integrantes — Aceito (validado) em 25/11/2026.**

**Checklist do revisor**
- [x] O contexto e o problema estão corretos e coerentes com o projeto Check Your Work.
- [x] A decisão é consistente com as demais ADRs (sem contradições de nomes, números ou componentes).
- [x] As opções rejeitadas e os trade-offs estão bem justificados.
- [x] Os critérios de verificação (Fitness Functions) são mensuráveis.
- [x] Concordo com a decisão registrada.

| Integrante | Papel | Status | Data | Comentário |
|---|---|---|---|---|
| Emilio Rodrigues | Revisor | ✅ Aprovado | 25/11/2026 | |
| Gabriel | Revisor | ✅ Aprovado | 25/11/2026 | |
| João Nicolas | Revisor | ✅ Aprovado | 25/11/2026 | |
| Leonardo Hamilton | Revisor | ✅ Aprovado | 25/11/2026 | |
| Marcell | Autor | ✅ Aprovado (autor) | 25/11/2026 | |
> **Legenda:** ✅ Aprovado · ✏️ Aprovado com ajustes (descrever no comentário) · ❌ Reprovado (justificar) · ⬜ Pendente
