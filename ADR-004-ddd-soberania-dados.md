# ADR-004 — Contextos Delimitados (DDD), Distribuição de Dados e Soberania por Schemas no PostgreSQL

| Campo | Valor |
|---|---|
| **Status** | Aceito |
| **Data** | 2026-11-25 (Sprint 4) |
| **Autor** | João Nicolas — Pesquisador & Modelagem DDD |
| **Revisores** | Emilio Rodrigues (Arquiteto Principal), Gabriel |
| **Relacionadas** | ADR-001, ADR-002, ADR-003, ADR-006 |

---

## 1. Contexto e Problema

Com API, Sandbox e Worker em contêineres separados (ADR-002), surge a pergunta: **quem é dono de qual dado?**

Um banco único em que qualquer componente lê e escreve qualquer tabela cria **acoplamento oculto pelo banco**: uma alteração de coluna feita pelo Worker pode quebrar a API sem que ninguém perceba. Por outro lado, um banco físico por serviço traria transações distribuídas e custo operacional incompatíveis com a carga (10 req/s) e o tamanho da equipe.

Além disso, o domínio mistura linguagens diferentes: "submissão" para a correção não é o mesmo que "submissão" para o acervo de similaridade, e "nota" para o domínio não é o mesmo que "lançamento" para o LMS.

## 2. Opções Consideradas

1. **Banco compartilhado sem fronteiras** (todas as tabelas no schema `public`).
2. **Database-per-service** (instâncias PostgreSQL físicas separadas).
3. **PostgreSQL único com schemas por contexto delimitado e roles segregadas.**

## 3. Decisão

### 3.1 Contextos Delimitados

| Contexto | Linguagem ubíqua | Dono | Tipo |
|---|---|---|---|
| **Correção** (Core Domain) | Submissão, Execução, Resultado, Nota | API REST | Núcleo |
| **Similaridade** | Acervo, Fingerprint, Evidência, Suspeita | Worker de Similaridade | Suporte |
| **Auditoria** | Evento, Justificativa, Versão do algoritmo | API (módulo Auditoria) | Genérico / Compliance |
| **Execução** | Job, Limites, Saída | Sandbox | Suporte (sem persistência) |
| **Acadêmico Legado** | Matrícula, Turma, Lançamento de nota | LMS (mainframe) | Externo — acessado via ACL |

**Mapa de contextos:**
- Correção → LMS: **Anticorruption Layer** (ADR-001).
- Correção → Similaridade: **Publisher/Subscriber** via eventos AMQP (`SubmissaoRecebida`, `SimilaridadeCalculada`).
- Correção → Execução: **Customer/Supplier** via contrato gRPC.
- Correção → Auditoria: **Conformist** (Auditoria aceita o modelo de eventos do Core).

### 3.2 Soberania por schemas

| Schema | Tabelas | Role com escrita | Quem pode ler |
|---|---|---|---|
| `schema_core` | `submissoes`, `resultados`, `outbox_publicacao_lms` | `role_api` | `role_api` |
| `schema_auditoria` | `auditoria_eventos` (WORM — ADR-003) | `role_api` (só INSERT) | `role_api`, `role_auditor` (somente leitura) |
| `schema_similaridade` | `acervo_submissoes`, `evidencias_similaridade`, `embeddings_feedback`* | `role_worker` | `role_worker` |
| Sandbox | — (armazenamento efêmero) | **nenhuma credencial** | — |

\* Tabela vetorial prevista na ADR-006.

**Regras de soberania:**
1. Um componente **só acessa o próprio schema**. Não existem *joins* entre schemas.
2. Dados de outro contexto são obtidos **por API ou por evento**, nunca por consulta direta.
3. Cada schema tem suas **migrações versionadas** e independentes.
4. **Minimização (LGPD):** o acervo de similaridade guarda o código associado a um identificador pseudonimizado, não ao nome ou matrícula do aluno.

## 4. Justificativa

- Mantém **uma única instância** para operar, fazer backup e monitorar, coerente com a projeção de ~1 TB em 5 anos.
- As roles fazem o **banco impor** as fronteiras de domínio — não depende só da disciplina dos desenvolvedores.
- A separação lógica **facilita** uma futura migração do `schema_similaridade` para instância própria, caso o Worker precise escalar.

### Opções rejeitadas
| Opção | Motivo |
|---|---|
| Banco sem fronteiras | Acoplamento oculto; qualquer componente pode corromper dados de outro; viola minimização da LGPD. |
| Database-per-service | Custo operacional e de consistência desproporcional à carga atual; sem ganho real para 10 req/s. |

## 5. Consequências

### Positivas
- Fronteiras explícitas e verificáveis; vazamentos entre contextos são bloqueados pelo banco.
- Modelos de domínio independentes por contexto (sem "entidade gigante" compartilhada).

### Negativas / Trade-offs aceitos
- **Ponto único de falha físico:** a instância PostgreSQL é compartilhada — exige réplica e backup testado.
- **Contenção de recursos:** uma consulta pesada no acervo pode afetar o Core (mitigar com limites de conexão por role).
- **Dados duplicados entre contextos** (ex.: código no Core e no acervo) mantidos por eventos, com consistência eventual.
- Migração física futura é **facilitada, não gratuita**: exige plano de sincronização, corte e rollback.

## 6. Critérios de Verificação (Fitness Functions)

| ID | Verificação | Meta |
|---|---|---|
| FF-004-1 | Teste de permissão: `role_worker` tenta `SELECT` em `schema_core` | Negado |
| FF-004-2 | Análise estática das migrações / queries: nenhum *join* entre schemas | 0 ocorrências |
| FF-004-3 | Sandbox tenta conexão TCP na porta do PostgreSQL | Bloqueada pela política de rede |
| FF-004-4 | Varredura do `schema_similaridade` por nome/matrícula em texto claro | 0 ocorrências |

## 7. Referências
- EVANS, E. *Domain-Driven Design*. Addison-Wesley, 2003.
- VERNON, V. *Implementing Domain-Driven Design*. Addison-Wesley, 2013.
- Documento "Distribuição de Dados e Soberania dos Módulos" — Equipe Check Your Work.

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
| João Nicolas | Autor | ✅ Aprovado (autor) | 25/11/2026 | |
| Leonardo Hamilton | Revisor | ✅ Aprovado | 25/11/2026 | |
| Marcell | Revisor | ✅ Aprovado | 25/11/2026 | |
> **Legenda:** ✅ Aprovado · ✏️ Aprovado com ajustes (descrever no comentário) · ❌ Reprovado (justificar) · ⬜ Pendente
