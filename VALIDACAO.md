# Matriz de Validação Cruzada dos ADRs — Sprint 4

Cada integrante valida **todos os ADRs dos colegas**. O autor já conta como aprovado no próprio ADR.
São **24 revisões** no total (6 ADRs × 5 integrantes − 6 autorias).

| ADR | Autor | Emilio | Gabriel | João Nicolas | Leonardo | Marcell |
|---|---|:-:|:-:|:-:|:-:|:-:|
| ADR-001 — Estilo Arquitetural e ACL | Emilio | ✍️ autor | ⬜ | ⬜ | ⬜ | ⬜ |
| ADR-002 — Contêineres e Resiliência | Gabriel | ⬜ | ✍️ autor | ⬜ | ⬜ | ⬜ |
| ADR-003 — DIP e Auditoria WORM | Emilio | ✍️ autor | ⬜ | ⬜ | ⬜ | ⬜ |
| ADR-004 — DDD e Soberania de Dados | João Nicolas | ⬜ | ⬜ | ✍️ autor | ⬜ | ⬜ |
| ADR-005 — Padrões GoF e MVC/MVVM | Leonardo | ⬜ | ⬜ | ⬜ | ✍️ autor | ⬜ |
| ADR-006 — IA-Native e Model Tiering | Marcell | ⬜ | ⬜ | ⬜ | ⬜ | ✍️ autor |

**Legenda:** ✅ Aprovado · ✏️ Aprovado com ajustes · ❌ Reprovado · ⬜ Pendente · ✍️ Autor

## O que cada um precisa validar

| Integrante | ADRs para revisar |
|---|---|
| Emilio Rodrigues | 002, 004, 005, 006 |
| Gabriel | 001, 003, 004, 005, 006 |
| João Nicolas | 001, 002, 003, 005, 006 |
| Leonardo Hamilton | 001, 002, 003, 004, 006 |
| Marcell | 001, 002, 003, 004, 005 |

## Como validar pelo GitHub (pelo site, sem terminal)

1. Abra o ADR no repositório e clique no **lápis (Edit this file)**.
2. Vá até a seção **8. Validação da Equipe** no final do arquivo.
3. Na sua linha, troque `⬜ Pendente` por `✅ Aprovado` (ou ✏️ / ❌), preencha a **data** e um **comentário curto**.
4. Em **Commit changes**, use a mensagem: `validação: ADR-00X aprovada por <seu nome>`.
5. Atualize também a sua coluna nesta matriz (`VALIDACAO.md`).

O histórico de commits do GitHub registra **quem** validou e **quando**, servindo como evidência para a entrega.

## Regras
- Se alguém marcar ✏️ ou ❌, o **autor** ajusta o ADR e pede nova validação de quem apontou o problema.
- Um ADR só muda de status para **Aceito (validado)** quando as 5 linhas estiverem ✅.
- Prazo sugerido: todas as validações concluídas antes da apresentação da Sprint 4.
