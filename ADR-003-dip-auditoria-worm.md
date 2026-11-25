# ADR-003 — Refatoração com DIP, Rigidez do Domínio (A = 0,5) e Auditoria WORM

| Campo | Valor |
|---|---|
| **Status** | Aceito |
| **Data** | 2026-11-25 (Sprint 4) |
| **Autor** | Emilio Rodrigues — Líder & Arquiteto Principal |
| **Revisores** | Leonardo Hamilton (QA / Fitness Functions), João Nicolas |
| **Substitui** | ADR 002 (Sprint 1 — Auditoria imutável) e ADR 1 (Sprint 2 — Refatoração) |
| **Relacionadas** | ADR-001, ADR-004, ADR-005 |

---

## 1. Contexto e Problema

Duas questões estruturais precisam ser registradas juntas, pois ambas protegem o **núcleo de domínio** e a **confiança nas notas**:

### 1.1 Acoplamento do núcleo
O diagnóstico estrutural (Sprint 2) identificou:

| Risco estrutural | Mitigação |
|---|---|
| Regra de negócio acessa banco e APIs diretamente | Interfaces abstratas para encapsular I/O |
| Lógica de correção conhece o LMS antigo | Adapter específico para o LMS |
| Um componente centraliza correção, execução e publicação | Segregação em módulos |
| Código não confiável executando em produção | Delegação a Sandbox isolada |

### 1.2 Integridade das notas (QAW-02, Prioridade 2)
> "Um professor ou administrador altera uma nota durante o período de recurso."

Sem trilha imutável, não há como responder a uma contestação: quem alterou, quando, por quê, valor anterior e novo, e qual versão do algoritmo gerou a nota. Notas e metadados são dados pessoais protegidos pela LGPD.

## 2. Opções Consideradas

**Estrutura do núcleo**
1. Manter acoplamento direto (status quo).
2. DIP + extrair abstração de **todas** as entidades do domínio (busca de D = 0).
3. **DIP com 5 interfaces de porta + aceitar entidades concretas (A = 0,5).**

**Auditoria**
1. Coluna `updated_at`/`updated_by` na própria tabela de notas.
2. Log de aplicação em arquivo.
3. **Tabela append-only (WORM) em schema próprio, com encadeamento por hash e revisão humana obrigatória para plágio.**

## 3. Decisão

### 3.1 Inversão de Dependência
O caso de uso `CorrigirSubmissao` depende **apenas de abstrações** definidas no `Domain`:

| Interface (Domain) | Implementação (Infrastructure) |
|---|---|
| `RepositorioSubmissao` | `RepositorioSubmissaoPostgres` |
| `ExecutorDeCodigo` | `ExecutorDeCodigoSandbox` (gRPC) |
| `AnalisadorSimilaridade` | `AnalisadorSimilaridadeWorker` (AMQP) |
| `PublicadorNota` | `PublicadorNotaLMS` (Adapter/ACL) |
| `Auditoria` | `AuditoriaPostgres` (WORM) |

As implementações são injetadas na **raiz de composição** (container de DI).

### 3.2 Aceitação pragmática da rigidez do Domínio

Métricas de Robert C. Martin após a refatoração:

| Pacote | Ca | Ce | I | A | D | Zona |
|---|---|---|---|---|---|---|
| Presentation | 0 | 1 | 1,0 | 0,0 | **0,0** | Sequência Principal |
| Domain | 6 | 0 | 0,0 | 0,5 | **0,5** | Fora da reta (não é Zona de Dor) |
| Infrastructure | 0 | 5 | 1,0 | 0,0 | **0,0** | Sequência Principal |

**Decisão:** registrar D = 0,5 do `Domain` como **trade-off aceito**. As 4 entidades concretas (`Submissao`, `ResultadoExecucao`, `Evidencias`, `Resultado`) permanecem concretas.

### 3.3 Auditoria WORM (Write Once, Read Many)
- Tabela `schema_auditoria.auditoria_eventos`, **somente INSERT**:
  - a role da aplicação recebe apenas `INSERT` e `SELECT`;
  - *trigger* rejeita qualquer `UPDATE` ou `DELETE`;
  - cada registro guarda `hash_anterior` + `hash_atual` (SHA‑256), formando uma **cadeia verificável**.
- Campos obrigatórios: `usuario_id`, `data_hora`, `acao`, `justificativa`, `valor_anterior`, `valor_novo`, `versao_algoritmo`, `submissao_id`.
- **Revisão humana:** suspeita de plágio **nunca** gera punição ou nota zero automaticamente; o sistema produz evidências e o professor decide (registro também auditado).
- Instantâneos de resultados anteriores são preservados via padrão **Memento** (ADR-005).

## 4. Justificativa

- **DIP** torna o domínio testável com dublês e protege o núcleo contra mudanças em SQL, gRPC, AMQP e no LMS.
- **Por que não buscar D = 0:** entidades de domínio são concretas por natureza; criar interfaces como `IResultado` só para mover a métrica acrescentaria indireção sem ganho real de flexibilidade (abstração especulativa). O `Domain` **não está na Zona de Dor**, que exige I ≈ 0 **e** A ≈ 0 ao mesmo tempo.
- **WORM** atende LGPD e contestação acadêmica; colunas de "última alteração" perdem o histórico, e logs em arquivo podem ser editados ou rotacionados.

## 5. Consequências

### Positivas
- Dependências sempre apontam para o domínio estável (Regra de Ouro preservada).
- Toda alteração de nota é rastreável e verificável por hash.

### Negativas / Trade-offs aceitos
- Mais interfaces, adaptadores e **testes de contrato** para manter.
- `Domain` mais rígido que o ideal: cada nova entidade concreta precisa ser avaliada.
- **Armazenamento cresce continuamente** (contribui para a projeção de ~1 TB em 5 anos); exige política de retenção e arquivamento compatível com a LGPD.
- Correções de dados nunca são feitas por `UPDATE`: exigem um **evento compensatório**.

## 6. Critérios de Verificação (Fitness Functions)

| ID | Verificação | Meta |
|---|---|---|
| FF-003-1 | Script de métricas no CI calcula D por pacote | D(Domain) ≤ 0,5; D(Presentation, Infrastructure) ≤ 0,1 |
| FF-003-2 | Regra de arquitetura: `Domain` não importa `Infrastructure` nem `Presentation` | 0 violações |
| FF-003-3 | Tentativa de `UPDATE`/`DELETE` em `auditoria_eventos` | Rejeitada pelo banco |
| FF-003-4 | Verificador de cadeia de hash executado diariamente | 0 inconsistências |
| FF-003-5 | Alteração de nota sem justificativa | Rejeitada pela API |

## 7. Referências
- MARTIN, R. C. *Clean Architecture*. Prentice Hall, 2017.
- Relatório "Métricas Quantitativas de Acoplamento, Instabilidade e Abstracionalidade" — Sprint 2.
- Lei nº 13.709/2018 (LGPD).

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
| Emilio Rodrigues | Autor | ✅ Aprovado (autor) | 25/11/2026 | |
| Gabriel | Revisor | ✅ Aprovado | 25/11/2026 | |
| João Nicolas | Revisor | ✅ Aprovado | 25/11/2026 | |
| Leonardo Hamilton | Revisor | ✅ Aprovado | 25/11/2026 | |
| Marcell | Revisor | ✅ Aprovado | 25/11/2026 | |
> **Legenda:** ✅ Aprovado · ✏️ Aprovado com ajustes (descrever no comentário) · ❌ Reprovado (justificar) · ⬜ Pendente
