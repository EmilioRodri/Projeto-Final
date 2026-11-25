# ADR-001 — Estilo Arquitetural, Processamento Assíncrono e Camada Anticorrupção (ACL)

| Campo | Valor |
|---|---|
| **Status** | Aceito |
| **Data** | 2026-11-25 (Sprint 4) |
| **Autor** | Emilio Rodrigues — Líder & Arquiteto Principal |
| **Revisores** | Gabriel, João Nicolas, Leonardo Hamilton, Marcell |
| **Substitui** | ADR 001 (Sprint 1) e ADR "Estilo Arquitetural Adotado" (Sprint 2) |
| **Relacionadas** | ADR-002, ADR-003, ADR-004 |

---

## 1. Contexto e Problema

O **Check Your Work** é um sistema universitário de correção automática de trabalhos de programação, com detecção de similaridade (plágio), que precisa publicar notas em um **LMS Legado baseado em mainframe**. O mainframe **não pode sofrer grandes alterações**.

O desenho inicial apresentava problemas estruturais graves:

- regras de negócio acessando diretamente banco de dados e APIs;
- um único componente concentrando correção, execução e publicação de notas;
- lógica de correção acoplada ao contrato do LMS legado;
- código não confiável de alunos executando próximo ao núcleo da aplicação.

### Drivers arquiteturais (Mini-QAW, Sprint 1)

| Prioridade | Driver | Cenário de risco |
|---|---|---|
| 1 | **Segurança e Privacidade** (LGPD) | QAW-03 — código malicioso tenta acessar rede ou estourar memória |
| 2 | **Integridade e Auditabilidade** das notas | QAW-02 — nota alterada sem rastro |
| 3 | **Integração Confiável** com o LMS | QAW-01 — pico de submissões congestiona o sistema ou o LMS cai |

### Premissas de carga (BOTE)

- ~10.000 estudantes ativos × 6 trabalhos × 2 semestres ≈ **120.000 submissões no Ano 1**;
- throughput de pico projetado: **10 req/s** (30% do volume concentrado na janela de 2 h antes do prazo);
- meta: **99,9% das submissões aceitas em < 2 s**.

## 2. Opções Consideradas

1. **Monólito tradicional** (estado inicial).
2. **Microsserviços completos**, cada um com banco, deploy e governança próprios.
3. **Arquitetura em Camadas com DIP** dentro de um monólito único.
4. **Arquitetura distribuída em contêineres especializados**, com núcleo em camadas + DIP, processamento assíncrono por fila e **Camada Anticorrupção (ACL)** para o LMS.

## 3. Decisão

Adotar a **opção 4**:

1. **Núcleo em camadas** (`Presentation → Domain ← Infrastructure`) com Inversão de Dependência (detalhada na ADR-003).
2. **Contêineres especializados**: Web App, API REST, Sandbox de execução e Worker de Similaridade (detalhados na ADR-002).
3. **Processamento assíncrono**: a submissão é aceita, persistida e respondida rapidamente com status `RECEBIDA`; correção pesada, análise de similaridade e publicação ocorrem fora do caminho síncrono, via fila AMQP (RabbitMQ).
4. **Camada Anticorrupção (ACL)** entre o domínio e o LMS Legado: o domínio conhece apenas a interface `PublicadorNota`; um *Adapter* traduz o modelo interno (`Resultado`, `Nota`) para o contrato do mainframe e vice‑versa.

```
[Aluno/Professor] → Web App ──REST──► API REST ──gRPC──► Sandbox (isolado)
                                       │
                                       ├──AMQP──► Worker de Similaridade
                                       │
                                       └──► [ACL: PublicadorNotaLMS] ──REST/HTTPS──► LMS Legado (mainframe)
```

## 4. Justificativa

- **Absorção de picos (QAW-01):** a fila desacopla a taxa de chegada das submissões da taxa de processamento; nenhuma submissão é perdida se o Worker ou o LMS estiverem lentos.
- **Proteção do mainframe:** a ACL impede que conceitos, formatos e instabilidades do LMS "vazem" para o domínio. Mudanças no LMS impactam apenas o adapter.
- **Isolamento de risco:** o código do aluno roda em um contêiner próprio, fora do processo da API.
- **Regra de Ouro da Estabilidade:** dependências apontam sempre das camadas voláteis para o domínio estável.

### Por que as outras opções foram rejeitadas

| Opção | Motivo da rejeição |
|---|---|
| Monólito tradicional | Mistura regras de negócio e infraestrutura; dificulta testes; mantém o acoplamento direto ao legado. |
| Microsserviços completos | Complexidade operacional (bancos separados, transações distribuídas, governança) desproporcional ao tamanho da equipe e à carga de 10 req/s. |
| Camadas + DIP em processo único | Resolve o acoplamento, mas **não isola** a execução de código não confiável nem a análise pesada de similaridade; um pico derrubaria toda a aplicação. |

> **Nota de nomenclatura:** a arquitetura **não** é chamada de "monólito modular" (há contêineres separados) **nem** de "microsserviços autônomos" (há um PostgreSQL compartilhado com schemas segregados — ver ADR-004). O termo correto é **arquitetura distribuída em contêineres especializados com núcleo em camadas**.

## 5. Consequências

### Positivas
- Picos de entrega absorvidos pela fila sem perda de submissões.
- Domínio testável sem LMS, banco ou Sandbox reais (uso de dublês das interfaces).
- Substituição do LMS no futuro exige apenas um novo adapter.

### Negativas / Trade-offs aceitos
- **Consistência eventual:** a nota não aparece instantaneamente no LMS; a interface precisa exibir estados como `EM_PROCESSAMENTO` e `PENDENTE_PUBLICACAO`.
- **Operação de fila:** é necessário monitorar profundidade da fila, DLQ e consumidores.
- **Custo de tradução:** a ACL adiciona uma camada de indireção e testes de contrato com o LMS.

## 6. Critérios de Verificação (Fitness Functions)

| ID | Verificação | Meta |
|---|---|---|
| FF-001-1 | Teste de carga com 10 req/s sustentados por 2 h | 99,9% das submissões aceitas em < 2 s |
| FF-001-2 | Teste de arquitetura (ArchUnit / dependency-cruiser): nenhuma classe do `Domain` importa pacote do LMS | 0 violações |
| FF-001-3 | LMS indisponível por 10 min durante teste | 0 submissões perdidas; notas ficam `PENDENTE_PUBLICACAO` |

## 7. Referências
- RICHARDS, M.; FORD, N. *Fundamentals of Software Architecture*. O'Reilly, 2020.
- EVANS, E. *Domain-Driven Design* — padrão Anticorruption Layer. Addison-Wesley, 2003.
- Relatório de Imersão Sprint 1 (Mini-QAW e BOTE) — Equipe Check Your Work.

---

## 8. Validação da Equipe

Cada integrante deve ler este ADR, conferir o checklist abaixo e marcar sua linha na tabela. A ADR só passa para **Aceito (validado)** quando os 5 integrantes aprovarem.

**Checklist do revisor**
- [✅ ] O contexto e o problema estão corretos e coerentes com o projeto Check Your Work.
- [✅ ] A decisão é consistente com as demais ADRs (sem contradições de nomes, números ou componentes).
- [✅ ] As opções rejeitadas e os trade-offs estão bem justificados.
- [✅ ] Os critérios de verificação (Fitness Functions) são mensuráveis.
- [ ✅] Concordo com a decisão registrada.

| Integrante | Papel | Status | Data | Comentário |
|---|---|---|---|---|
| Emilio Rodrigues | Autor | ✅ Aprovado (autor) | | |
| Gabriel | Revisor | ⬜ Pendente | | |
| João Nicolas | Revisor | ⬜ Pendente | | |
| Leonardo Hamilton | Revisor | ⬜ Pendente | | |
| Marcell | Revisor | ⬜ Pendente | | |

> **Legenda:** ✅ Aprovado · ✏️ Aprovado com ajustes (descrever no comentário) · ❌ Reprovado (justificar) · ⬜ Pendente
