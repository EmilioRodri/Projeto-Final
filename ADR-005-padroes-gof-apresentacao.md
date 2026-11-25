# ADR-005 — Padrões GoF (Criacionais, Estruturais e Comportamentais) e Padrões de Apresentação MVC/MVVM

| Campo | Valor |
|---|---|
| **Status** | Aceito |
| **Data** | 2026-11-25 (Sprint 4) |
| **Autor** | Leonardo Hamilton — Executor & QA / Fitness Functions |
| **Revisores** | Emilio Rodrigues (Arquiteto Principal), Gabriel |
| **Substitui** | ADRs de Padrões Criacionais, Estruturais e Comportamentais (Sprint 3) |
| **Relacionadas** | ADR-002 (resiliência), ADR-003 (DIP/auditoria) |

---

## 1. Contexto e Problema

As ADRs anteriores definiram a arquitetura macro (contêineres, camadas, DIP). No nível tático, o código ainda corria riscos de **erosão arquitetural**:

- `switch/case` por linguagem para criar executores (viola OCP);
- lógica de timeout/retry/circuit breaker misturada à regra de negócio;
- objeto `ResultadoCorrecao` com construtor telescópico e sem validação de invariantes (nota entre 0 e 10);
- interface web precisando refletir em tempo real estados assíncronos (`EM_PROCESSAMENTO`, `PENDENTE_PUBLICACAO`, similaridade calculada).

A Sprint 3 catalogou os 23 padrões GoF. Esta ADR registra **quais são efetivamente adotados**, onde, e quais foram **adiados** para evitar abstração especulativa.

## 2. Opções Consideradas

1. Não adotar padrões formais (código procedural nos casos de uso).
2. Aplicar os 23 padrões GoF de forma ampla.
3. **Adotar um subconjunto justificado por problema real, com padrões de apresentação distintos para servidor e SPA.**

## 3. Decisão

### 3.1 Padrões Criacionais

| Padrão | Onde | Problema resolvido |
|---|---|---|
| **Factory Method** | `ExecutorFactory.criar(linguagem)` | Remove o `switch` espalhado; nova linguagem = nova classe |
| **Builder** | `ResultadoCorrecaoBuilder` | Valida invariantes (nota 0–10, status obrigatório) antes de instanciar |
| **Singleton (via DI)** | Estado do Circuit Breaker e configuração global | Instância única **gerenciada pelo container de DI**, não por `static getInstance()`, preservando testabilidade |
| Abstract Factory | *Adiado* | Só se justifica quando houver mais de um LMS a integrar |
| Prototype | *Adiado* | Clonagem de atividades entre semestres — fora do escopo do MVP |

### 3.2 Padrões Estruturais

| Padrão | Onde | Problema resolvido |
|---|---|---|
| **Adapter** | `PublicadorNotaLMS` | Implementa a ACL (ADR-001) traduzindo para o contrato do mainframe |
| **Decorator** | `PublicadorNotaResilienteDecorator` | Envolve o publicador com Timeout → Retry → Circuit Breaker → Fallback, sem poluir o domínio |
| **Facade** | `CorrigirSubmissaoUseCase` | Ponto único de orquestração para o Controller |
| **Proxy** | Stub gRPC do Sandbox | Representa o executor remoto e controla acesso |
| **Composite** | Árvore `Atividade → Questão → GrupoDeTestes → CasoDeTeste` (`Avaliavel`) | Cálculo de nota uniforme em qualquer nível |
| Bridge, Flyweight | *Adiados* | Ativar quando houver métricas de memória ou múltiplas tecnologias de isolamento |

### 3.3 Padrões Comportamentais

| Padrão | Onde | Problema resolvido |
|---|---|---|
| **Strategy** | Executores por linguagem; algoritmos de similaridade (léxico, estrutural) | Algoritmos intercambiáveis |
| **Chain of Responsibility** | Pipeline de validação: Autenticação → `Idempotency-Key` → Schema → Tamanho/extensão | Novas validações sem alterar as existentes |
| **Template Method** | Fluxo fixo: validar → executar → pontuar → persistir → publicar | Ordem invariável; pontuação varia por subtipo |
| **State** | Circuit Breaker (Fechado/Aberto/Meio‑aberto) e ciclo de vida da submissão | Transições explícitas e testáveis |
| **Observer** | Evento `SubmissaoCorrigida` → Auditoria e Similaridade (via AMQP) | Desacoplamento publicador/assinante |
| **Command** | Pendência `PENDENTE_PUBLICACAO` persistida e reexecutada pelo relay | Operação enfileirável e reexecutável |
| **Memento** | Instantâneos de resultados anteriores em recorreções | Suporte à auditoria WORM (ADR-003) |
| **Iterator** | Paginação de submissões e comparações | Evita estouro de memória em lotes grandes |
| Visitor, Interpreter, Mediator | *Adiados* | Visitor para AST em análise estrutural avançada; Interpreter para fórmulas de nota definidas pelo professor |

### 3.4 Padrões de Apresentação

| Camada | Padrão | Aplicação |
|---|---|---|
| **API (servidor)** | **MVC** | `ControllerSubmissao` recebe HTTP → Model (Domain) → View = JSON conforme OpenAPI 3.0 |
| **Web App (SPA)** | **MVVM** | ViewModels reativos atualizam estado de correção, similaridade e publicação sem recarregar a página |
| Componentes isolados | MVP (opcional) | Usado onde for necessário testar a View passiva sem navegador |

## 4. Justificativa

- Cada padrão adotado resolve um **problema já observado** no código ou nas ADRs; os demais ficam documentados como adiados para não gerar complexidade sem retorno.
- **Decorator + State** concentram a resiliência (ADR-002) fora do domínio, mantendo `CorrigirSubmissao` limpo.
- **MVVM** é o padrão natural para exibir a **consistência eventual** introduzida pela fila: o usuário vê o status mudar sem *refresh*.
- **MVC no servidor** mantém o Controller fino, com apenas tradução HTTP ↔ caso de uso.

## 5. Consequências

### Positivas
- Adição de linguagens, validações ou algoritmos de similaridade sem modificar código estável (OCP).
- Resiliência testável isoladamente (testes unitários do Decorator e da máquina de estados).

### Negativas / Trade-offs aceitos
- Mais classes e indireção; a equipe precisa conhecer os padrões para navegar no código.
- Decorators aninhados dificultam a depuração — mitigado com logs estruturados por camada.
- Singleton deve **sempre** ser injetado; acesso global direto é proibido.

## 6. Critérios de Verificação (Fitness Functions)

| ID | Verificação | Meta |
|---|---|---|
| FF-005-1 | Regra de lint: nenhum `new Executor*` fora da `ExecutorFactory` | 0 violações |
| FF-005-2 | `ResultadoCorrecaoBuilder` com nota 11 ou −1 | Exceção lançada |
| FF-005-3 | Teste unitário da máquina de estados do Circuit Breaker (todas as transições) | 100% das transições cobertas |
| FF-005-4 | Regra de lint: proibido `getInstance()` estático | 0 ocorrências |
| FF-005-5 | Teste E2E: mudança de status no backend reflete na SPA | < 2 s, sem recarregar |

## 7. Referências
- GAMMA, E. et al. *Design Patterns*. Addison-Wesley, 1994.
- FREEMAN, E. et al. *Head First Design Patterns*. 2. ed. O'Reilly, 2020.
- HAMILTON, L. *O Nexo entre Design Patterns, Princípios SOLID e Arquitetura de Software* — Sprint 3.

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
| Emilio Rodrigues | Revisor | ⬜ Pendente | | |
| Gabriel | Revisor | ⬜ Pendente | | |
| João Nicolas | Revisor | ⬜ Pendente | | |
| Leonardo Hamilton | Autor | ✅ Aprovado (autor) | | |
| Marcell | Revisor | ⬜ Pendente | | |

> **Legenda:** ✅ Aprovado · ✏️ Aprovado com ajustes (descrever no comentário) · ❌ Reprovado (justificar) · ⬜ Pendente
