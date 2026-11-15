# PARTE 1 — PRD

## 1. Visão e Problema

**Problema:** Pessoas abandonam o controle financeiro porque apps atuais exigem formulários complexos, planilhas e categorização manual. Além disso, os apps existentes falham ao separar "data da compra" de "data do pagamento da fatura", gerando saldos mentirosos.

**Solução:** Um **assistente financeiro conversacional** que combina:
- Registro por **chat em linguagem natural** (interface principal)
- **Entrada manual** como fallback claro
- **Classificação automática** de transações
- **Metas financeiras** com acompanhamento real
- **Dicas personalizadas** de um agente financeiro (v1)
- **Relatórios simples** adaptados ao perfil
- **Design Universal** desde o protótipo (acessível, inclusivo, intuitivo)

**Persona:** João (28 anos), iniciante, quer organizar a vida financeira sem planilhas nem formulários.
**Público:** Pessoas físicas iniciantes, incluindo pessoas com limitações cognitivas, visuais ou baixa alfabetização digital.

---

## 2. Escopo

**Dentro (MVP):**
- Chat como interface principal (registro por texto)
- Entrada manual como fallback
- Classificação automática de transações
- Metas financeiras com progresso real
- Relatório simples (gastos por categoria, progresso de metas)
- Motor financeiro completo (conta, cartão, ciclo de fatura, pagamento com três cenários, saldo materializado)
- Estorno e edição de transações (sem excluir registros)
- Reconciliação financeira (auditoria)
- Design Universal: contraste, leitor de tela, fonte ajustável, feedback auditivo

**Fora (v1/v2):**
- Captura de imagem de documentos fiscais (decisão mantida)
- OCR, upload de comprovantes, processamento de imagem
- Dicas personalizadas avançadas (v1)
- Comandos por voz (v1)
- Multi-conta, compartilhamento, importação de extrato (v2)
- Open finance, investimentos, multi-moeda (v3+)

---

## 3. Modelo de Dados

| Entidade | Campos principais | Relações |
|----------|-------------------|----------|
| **Account** | id, nome, saldo_inicial, saldo_atual, ultima_reconciliacao | 1:N Transaction; 1:N ReconciliationLog |
| **Card** | id, nome, dia_fechamento, dia_vencimento, is_principal | 1:N Transaction; 1:N Invoice |
| **Transaction** | id, tipo (income/expense/transfer), valor, descrição, categoria, payment_method (account/credit), card_id, data_compra, parcela_atual, total_parcelas, status (active/reversed), origem (chat/manual), reversed_at, edited_at, original_valor, original_categoria | N:1 Account/Card; 1:N Installment |
| **Installment** | id, transaction_id, numero_parcela, valor, data_prevista, invoice_id, status (pending/paid), payment_id (nullable) | N:1 Transaction/Invoice/Payment |
| **Invoice** | id, card_id, mês_referência, data_fechamento, data_vencimento, rolled_balance, new_charges, valor_total, valor_pago, status (open/closed/paid), debt_balance | N:1 Card; 1:N Installment; 1:N Payment |
| **Payment** | id, invoice_id, valor_pago, data_pagamento, tipo (exact/over/under/zero), debt_balance_gerado, status (active/reversed), created_at, reversed_at | N:1 Invoice; 1:N Installment |
| **Goal** | id, nome, valor_alvo, prazo, categoria_vinculada, status | N:1 Account |
| **GoalProgress** | id, goal_id, valor_atual, data_calculo | N:1 Goal |
| **ChatMessage** | id, user_id, texto, resposta_ia, timestamp | N:1 Account |
| **ReconciliationLog** | id, account_id, data_execucao, gatilho, saldo_calculado, saldo_atual_antes, saldo_atual_depois, divergencia, status, detalhes (JSON), corrigido_por, created_at | N:1 Account |

> **Notas de modelagem:**
> - `Installment.payment_id` é **nullable** e só é preenchido quando a parcela é paga (R14).
> - `Transaction.status` **não** indica pagamento; indica se a transação foi estornada.
> - `Invoice.rolled_balance` guarda a dívida rolada da fatura anterior; `new_charges` guarda as compras do mês.
> - `ReconciliationLog` é **append-only** (nunca editado).

---

## 4. Regras de Negócio

### 4.1 Saldo

**R1 — Saldo Materializado**
```
Onboarding:        saldo_atual = saldo_inicial
Receita:           saldo_atual += valor
Despesa (account): saldo_atual -= valor
Pagamento:         saldo_atual -= valor_pago
Despesa (credit):  NÃO altera saldo_atual
```

**R12 — Reconciliação Financeira**
```
saldo_calculado = saldo_inicial 
                + Σ receitas (account, active) 
                − Σ despesas (account, active) 
                − Σ pagamentos (active)

se saldo_calculado == saldo_atual: status = 'sem_divergencia'
se saldo_calculado != saldo_atual: status = 'divergencia_detectada'
    (corrigir se autorizado)
```

**RC-01 — Gatilhos de Reconciliação**
```
Executar APENAS em: manual | agendado | migracao | pos_correcao | pos_estorno_em_lote
NÃO executar em: lançamento, edição, estorno individual, pagamento, carregamento de dashboard
```

**RC-02 — Escopo:** 1 conta por vez (exceto migração, que reconcilia todas)
**RC-03 — Autorização:** correção automática só em migração/pos_correcao ou autorização explícita
**RC-04 — Preservação:** reconciliação nunca deleta ou altera transações; apenas lê, registra e corrige saldo
**RC-05 — Detalhamento:** log registra período, totais, top 5 transações, observações (JSON)
**RC-06 — Pós-estorno em lote:** 10+ estornos em <5min disparam reconciliação (sem correção automática)
**RC-07 — Pós-migração:** correção automática + log `corrigido`

### 4.2 Ciclo de Fatura

**R2 — Cálculo da Fatura**
```
se data_compra >= dia_fechamento: fatura = mês seguinte
senão: fatura = fechamento do mês corrente
```
Exemplo (fechamento dia 8): compra dia 7 → fatura do mês; dia 8 → seguinte; dia 9 → seguinte.

**R3 — Faturas no Mês**
```
Faturas_no_Mes = Σ parcelas cuja fatura tem data_vencimento 
                 dentro do mês corrente E status != 'paid'
```

**R4 — Próximas Faturas**
```
Proximas_Faturas = Σ faturas futuras (vencimento > mês corrente) com status != 'paid'
```

**R5 — Comprometido Futuro**
```
Comprometido_Futuro = Σ Installments futuras (não faturadas) 
                    + Σ faturas fechadas não pagas
```

### 4.3 Pagamento de Fatura

**R6 — Pagamento (revisada, cobre zero)**
```
se valor_pago == 0:       tipo='zero';  debt_balance = dívida; rolled_balance próxima += dívida;
                           parcelas → 'paid' com payment_id; saldo NÃO altera
se valor_pago == dívida:  tipo='exact'; saldo -= valor_pago; parcelas → 'paid' com payment_id
se valor_pago >  dívida:  tipo='over';  saldo -= valor_pago; SEM crédito; SEM rolled_balance
se 0 < valor_pago < dívida: tipo='under'; saldo -= valor_pago; 
                            debt_balance = dívida − valor_pago; 
                            rolled_balance próxima += debt_balance;
                            parcelas → 'paid' com payment_id
```

**R7 — Atalho "cc"**
```
"cc" → sempre credit (cartão de crédito)
Fallback: se "cc" sem valor → modal vazio pedindo valor
          se cartão não existir → pedir cadastro
```

**R8 — Anti-dupla contagem**
```
Pagamento registrado como Transaction tipo 'transfer', categoria 'Transferência/Fatura'.
NÃO entra em gráficos de categoria.
Dívida rolada NÃO é tratada como nova despesa.
```

**R14 — Rastreabilidade de Pagamento (crítica)**
```
Toda parcela 'paid' DEVE ter payment_id preenchido.
Nenhuma alteração para 'paid' sem registrar payment_id.
O conceito de "pago" pertence à dívida (Invoice/Installment), não à compra original.
Transaction.status NÃO indica pagamento.
```

**R15 — Data de Vencimento (informativa)**
```
Data de vencimento NÃO influencia o fluxo.
Pagamento antecipado, no dia ou atrasado → mesmo comportamento.
Pagamento a menor: alerta + confirmação + parcelas 'paid' + diferença para próxima fatura.
Pagamento a maior: alerta + confirmação + parcelas 'paid' + sem crédito.
```

**R16 — Transparência da Fatura**
```
Tela de fatura exibe separadamente:
- rolled_balance (saldo anterior)
- new_charges (novas compras)
- total_due (soma)
```

### 4.4 Estorno e Edição

**RE-01 — Nunca excluir**
```
Nenhum DELETE em Transaction, Installment, Payment, Invoice.
Estorno → status='reversed' + reversed_at
Edição  → status='active' + edited_at + original_*
```

**RE-02 — Estorno reverte apenas o impacto**
```
Receita (account):        saldo_atual -= valor
Despesa conta (account):  saldo_atual += valor
Despesa cartão (credit):  saldo NÃO altera; remove Installment da fatura;
                          Invoice.valor_total -= valor; se parcela paga → bloquear
Pagamento (transfer):     fluxo do Módulo 6 (estorno de pagamento)
```

**RE-03 — Edição aplica apenas o delta**
```
delta = impacto_novo − impacto_antigo
saldo_atual += delta
```

**ED-01:** edição de transação 'reversed' → bloquear
**ED-02:** edição de despesa em cartão com parcela paga → bloquear
**ED-03:** edição de pagamento → bloquear; orientar estornar + criar novo
**ED-04:** edição de tipo (income↔expense↔transfer) → bloquear
**ED-05:** edição de payment_method (account↔credit) → bloquear
**ED-06:** edição de descrição/categoria → permitir sem impacto
**ED-07:** edição de data que muda fatura → mover Installment, ajustar 2 faturas, saldo NÃO altera

### 4.5 Metas

**R11 — Progresso de Meta**
```
Progresso = Σ receitas no período − Σ despesas no período
Se progresso >= valor_alvo → meta atingida
Se ritmo de gastos projetar não atingir → alerta visual
```

**R12-M — Dicas personalizadas (v1)**
```
Basear em: top 3 categorias, comparação com mês anterior, metas em risco.
Nunca recomendar produto financeiro específico.
```

### 4.6 Acessibilidade

**R13-A — Acessibilidade**
```
Todas as telas: ARIA labels, contraste WCAG AA, navegação por teclado.
Fonte ajustável (3 tamanhos).
Feedback auditivo em ações críticas (salvar, pagar, meta atingida).
```

---

## 5. Requisitos Funcionais por Módulo

| ID | Módulo | Requisito | Regra | Prioridade |
|----|--------|-----------|-------|------------|
| RF-01 | Onboarding | Cadastrar Conta Principal com saldo inicial | R1 (saldo_atual = saldo_inicial) | Must |
| RF-02 | Onboarding | Cadastrar cartões (nome, fechamento, vencimento, principal) | 1..N; 1 = sem seletor; 1 principal | Must |
| RF-03 | Dashboard | Cards: Saldo em Conta, Faturas no Mês, Próximas Faturas, Comprometido Futuro | R1, R3, R4, R5 | Must |
| RF-04 | Dashboard | Feed de transações com ações "Estornar" e "Editar" | RE-01, RE-03 | Must |
| RF-05 | Chat | Interface principal de registro por texto livre | R7 | Must |
| RF-06 | Chat | IA extrai valor, descrição, meio, tipo, categoria | R7 | Must |
| RF-07 | Chat | Modal de confirmação pré-preenchido (HITL) | R7 | Must |
| RF-08 | Lançamento Manual | Botão "+" → Receita/Gasto (fallback do chat) | R1 | Must |
| RF-09 | Lançamento | Gasto em Conta altera Saldo Atual | R1 | Must |
| RF-10 | Lançamento | Gasto em Cartão não altera Saldo Atual | R1 | Must |
| RF-11 | Lançamento | Cálculo de ciclo de fatura | R2 | Must |
| RF-12 | Lançamento | Despesa parcelada | R2 | Must |
| RF-13 | Pagamento | Pagamento de fatura (exato/a maior/a menor/zero) | R6 | Must |
| RF-14 | Pagamento | Anti-dupla contagem | R8 | Must |
| RF-15 | Pagamento | Rastreabilidade de parcelas pagas | R14 | Must |
| RF-16 | Metas | Criar meta com valor e prazo | R11 | Must |
| RF-17 | Metas | Progresso calculado a partir de transações reais | R11 | Must |
| RF-18 | Metas | Alerta visual quando meta em risco | R11 | Should |
| RF-19 | Relatórios | Gastos por categoria | — | Must |
| RF-20 | Relatórios | Progresso de metas | R11 | Must |
| RF-21 | Dicas | Sugestões baseadas em hábitos reais | R12-M | Should |
| RF-22 | Acessibilidade | Leitor de tela em todas as telas | R13-A | Must |
| RF-23 | Acessibilidade | Fonte ajustável e feedback auditivo | R13-A | Must |
| RF-24 | Acessibilidade | Comandos por voz (v1) | R13-A | Could |
| RF-25 | Estorno | Estornar transações sem excluir | RE-01, RE-02 | Must |
| RF-26 | Edição | Editar transações com delta | RE-03, ED-01 a ED-07 | Must |
| RF-27 | Reconciliação | Verificar integridade do saldo | R12, RC-01 a RC-07 | Must |
| RF-28 | Reconciliação | Histórico de reconciliações | RC-05 | Must |

---

## 6. Requisitos Não Funcionais

- **Mobile-first**, com layout adaptável a tablet e desktop.
- **Acessibilidade:** WCAG 2.1 nível AA; leitor de tela; fonte ajustável; feedback auditivo.
- **Desempenho:** resposta do chat em até 5 segundos; fallback manual se a IA falhar; reconciliação em background para >100k transações.
- **Privacidade:** dados isolados por usuário (Supabase RLS).
- **Confiabilidade:** cálculo de ciclo de fatura determinístico; rollback transacional em estorno/edição.
- **Escalabilidade:** modelo preparado para multi-conta e compartilhamento (v2).

---

## 7. Métricas de Sucesso

- Tempo médio de lançamento (chat vs. manual)
- % de lançamentos por chat vs. manual
- % de edições de categoria após sugestão da IA (proxy de acerto)
- % de metas criadas e atingidas
- % de usuários que retornam na 2ª semana (retenção)
- Frequência de erros de ciclo de fatura reportados
- Frequência de divergências detectadas na reconciliação
- NPS após 30 dias de uso

---

## 8. Riscos e Questões em Aberto

**Riscos:**
- **R-01:** erro de ciclo de fatura por 1 dia quebra o saldo futuro.
- **R-02:** parcelamento + pagamento a menor = interação complexa.
- **R-03:** pagamento parcial marca todas as despesas como pagas — exige aviso explícito na UI.
- **R-04:** atalho "cc" sem fallback gera lançamento incompleto.
- **R-05:** "Faturas no Mês" usa vencimento como referência — validar.
- **R-06:** chat como interface principal pode gerar fadiga conversacional.
- **R-07:** estorno de despesa em cartão com parcela paga é bloqueado — usuário precisa estornar pagamento primeiro.
- **R-08:** edição de pagamento é bloqueada — usuário precisa estornar + criar novo.

**Questões em aberto [VERIFICAR]:**
1. "Faturas no Mês": a fórmula R3 (por vencimento) é a correta, ou por data de fechamento?
2. Metas têm categoria vinculada ou são genéricas?
3. Dicas personalizadas entram no MVP ou na v1?
4. Comandos por voz entram no MVP ou na v1?
5. Reconciliação agendada: qual frequência padrão (semanal/mensal)?

---

## 9. Roadmap por Fases

| Fase | Escopo | Critério de saída |
|------|--------|-------------------|
| **MVP** | RF-01 a RF-20, RF-22, RF-23, RF-25 a RF-28 | Usuário registra, vê progresso, atinge meta, estorna, edita, reconcilia |
| **v1** | RF-21 (dicas), RF-24 (voz), alertas de meta em risco | Agente financeiro ativo |
| **v2** | Multi-conta, compartilhamento, importação de extrato | Escalabilidade |
| **v3+** | Open finance, investimentos, multi-moeda | Expansão |

---

# PARTE 2 — Prompts de Vibe Coding

> **Ordem de execução:** rode um por vez. Teste. Corrija. Avance. Cada prompt é autocontido.

---

## MÓDULO 1 — Setup Inicial e Onboarding

**[OBJETIVO]** Onboarding para cadastrar Conta Principal (saldo inicial) e Cartões (nome, fechamento, vencimento, principal).

**[CONTEXTO]** Entidades Account e Card. Apenas 1 cartão principal por usuário. 1 cartão = sem seletor; N = seletor com principal pré-selecionado. `saldo_atual = saldo_inicial` na criação.

**[ENTREGÁVEL]** Tela de onboarding, tabelas Supabase (accounts, cards), lógica de seletor condicional.

**[RESTRIÇÕES]** Não criar funcionalidades fora do escopo. Não permitir cartão sem fechamento/vencimento. Não permitir 2 principais. Não permitir `saldo_atual` diferente de `saldo_inicial` na criação.

**[CRITÉRIOS DE ACEITE]**
- CA-ONB-01: conta com saldo_atual = saldo_inicial
- CA-ONB-02: cartões persistidos com datas
- CA-ONB-03: seletor oculto com 1 cartão
- CA-ONB-04: seletor visível com N cartões, principal pré-selecionado
- CA-ONB-05: principal editável, apenas 1 por usuário

---

## MÓDULO 2 — Dashboard

**[OBJETIVO]** Dashboard com Saldo em Conta, Faturas no Mês, Próximas Faturas, Comprometido Futuro + feed + botão "+" + ações Estornar/Editar.

**[CONTEXTO]** Fórmulas R1, R3, R4, R5. Saldo Atual é lido de `Account.saldo_atual` (materializado, não recalculado).

**[ENTREGÁVEL]** Tela principal com 4 cards, feed (verde/vermelho), botão flutuante, ações por transação.

**[RESTRIÇÕES]** Máximo 4 números grandes. Não contar pagamento de fatura como despesa no feed de categorias. Não recalcular saldo por leitura. Usar `data_vencimento` para "Faturas no Mês".

**[CRITÉRIOS DE ACEITE]**
- CA-DASH-01: saldo bate com `Account.saldo_atual`
- CA-DASH-02: Faturas no Mês = vencimento no mês corrente
- CA-DASH-03: Próximas Faturas = vencimento futuro
- CA-DASH-04: Comprometido Futuro inclui parcelas não faturadas
- CA-DASH-05: feed ordenado; ações "Estornar" e "Editar" visíveis
- CA-DASH-06: botão "+" abre modal

---

## MÓDULO 3 — Lançamento Simples

**[OBJETIVO]** Modal Receita/Gasto com forma de pagamento (Conta/Cartão), valor, descrição, categoria.

**[CONTEXTO]** Gasto em Conta → `saldo_atual -= valor`. Gasto em Cartão → `Installment` na fatura. Receita → `saldo_atual += valor`. `Transaction.status = 'active'`.

**[ENTREGÁVEL]** Modal, validação, persistência em Transaction + Account atualizado.

**[RESTRIÇÕES]** Gasto em cartão **não** altera `saldo_atual`. Não permitir valor zero/negativo. Atualizar saldo incrementalmente (não recalcular).

**[CRITÉRIOS DE ACEITE]**
- CA-LAN-01: gasto em conta reduz saldo imediatamente
- CA-LAN-02: gasto em cartão aumenta fatura sem reduzir saldo
- CA-LAN-03: receita aumenta saldo
- CA-LAN-04: `Transaction.status = 'active'` na criação

---

## MÓDULO 4 — Despesa Parcelada

**[OBJETIVO]** Compra parcelada com N parcelas distribuídas nas N faturas seguintes.

**[CONTEXTO]** `Transaction` com `parcela_atual`/`total_parcelas` + N `Installment` com `payment_id = NULL`.

**[ENTREGÁVEL]** Campo de parcelas, geração de Installments, exibição por fatura.

**[RESTRIÇÕES]** Não somar valor total na fatura atual. Não duplicar em gráficos. `Installment.payment_id = NULL` na criação.

**[CRITÉRIOS DE ACEITE]**
- CA-PARC-01: R$ 1.000 em 10x = 10 parcelas de R$ 100
- CA-PARC-02: cada parcela na fatura correta
- CA-PARC-03: saldo atual intacto
- CA-PARC-04: `payment_id = NULL` na criação

---

## MÓDULO 5 — Compra no Cartão e Ciclo de Fatura

**[OBJETIVO]** Calcular a fatura correta com base na data da compra e no fechamento.

**[CONTEXTO]** R2: `data_compra >= dia_fechamento` → fatura seguinte; senão → fatura do mês.

**[ENTREGÁVEL]** Função determinística, testes unitários, vínculo `Transaction → Invoice`.

**[RESTRIÇÕES]** Não usar data de pagamento. Não alterar `saldo_atual`. Não duplicar em edição.

**[CRITÉRIOS DE ACEITE]**
- CA-CIC-01: compra dia 7 com fechamento 8 → fatura do mês
- CA-CIC-02: compra dia 8 → fatura seguinte
- CA-CIC-03: compra dia 9 → fatura seguinte
- CA-CIC-04: testes cobrem os 3 casos

---

## MÓDULO 6 — Pagamento de Fatura

**[OBJETIVO]** Fluxo de pagamento com 4 cenários (exato, a maior, a menor, zero), rastreabilidade de parcelas (R14), estorno seguro, rolagem de dívida (R6).

**[CONTEXTO]** R6 revisada, R8, R14, R15. `Installment.payment_id` preenchido ao pagar. `Payment.status = 'active'`. `Invoice.rolled_balance` atualizado em under/zero.

**[ENTREGÁVEL]**
1. Tela de pagamento (seletor cartão/fatura, valor, data)
2. Modal de confirmação contextual (exact/over/under/zero)
3. Função transacional de pagamento (tudo ou nada): INSERT Payment; UPDATE Installments → 'paid' com payment_id; UPDATE Invoice; UPDATE Account; UPDATE próxima Invoice (se under/zero); INSERT Transaction (exceto zero)
4. Tela de histórico com botão "Estornar"
5. Função transacional de estorno: SELECT Installments WHERE payment_id; UPDATE → 'pending', NULL; UPDATE Payment → 'reversed'; UPDATE Invoice → 'open'; UPDATE próxima Invoice (se under/zero); UPDATE Account += valor_pago (exceto zero); UPDATE Transaction → 'reversed'

**[RESTRIÇÕES]** Nunca duplicar em gráficos. Não criar crédito em pagamento a maior. Não permitir parcela 'paid' sem payment_id. Não recalcular saldo (apenas incremental). Estorno atômico. Data de vencimento não altera comportamento. Cancelar modal = rollback.

**[CRITÉRIOS DE ACEITE]**
- CA-PAG-01: exato zera fatura, parcelas 'paid' com payment_id
- CA-PAG-02: a maior não cria crédito
- CA-PAG-03: a menor gera rolled_balance na próxima fatura
- CA-PAG-04: zero rola dívida integral sem mexer no saldo
- CA-PAG-05: fatura 'paid' bloqueia novo pagamento
- CA-PAG-06: pagamento > saldo exige confirmação
- CA-R14-01, CA-R14-02: rastreabilidade
- CA-EST-01 a CA-EST-05: estorno (exato, a menor, zero, já estornado, duplicado)
- CA-ROL-01 a CA-ROL-03: rolagem de dívida
- CA-DUP-01, CA-DUP-02: anti-dupla contagem
- CA-VENC-01, CA-VENC-02: data de vencimento informativa

---

## MÓDULO 7 — Chat (Lançamento Rápido)

**[OBJETIVO]** Campo de texto livre com GPT-4o-mini + modal de confirmação.

**[CONTEXTO]** R7: "cc" = sempre cartão de crédito. Ignorar "reais", "pix", "débito". "recebi"/"ganhei" → income.

**[ENTREGÁVEL]** Campo no topo, integração OpenAI, modal pré-preenchido, seletor condicional.

**[RESTRIÇÕES]** Não salvar sem confirmação (HITL). Não inventar valor se não houver número. Fallback: se "cc" sem valor → modal vazio pedindo valor; se cartão não existir → pedir cadastro. Fallback manual se a IA falhar.

**[CRITÉRIOS DE ACEITE]**
- CA-CHAT-01: "almoço 30" → expense/account/R$ 30
- CA-CHAT-02: "uber 15 cc" → expense/credit/R$ 15
- CA-CHAT-03: "recebi 2000 salário" → income/R$ 2.000
- CA-CHAT-04: "cc 30" sem descrição → modal com valor 30 e descrição vazia
- CA-CHAT-05: N cartões → principal pré-selecionado
- CA-CHAT-06: usuário edita antes de salvar

---

## MÓDULO 8 — Categorização com HITL

**[OBJETIVO]** IA sugere categoria; usuário edita antes/depois de salvar.

**[CONTEXTO]** Categorias padrão. Pagamento de fatura = "Transferência/Fatura". Edição de categoria passa pelo Módulo 9 (ED-06).

**[ENTREGÁVEL]** Sugestão no modal, campo editável, persistência.

**[RESTRIÇÕES]** Não sobrescrever edição manual. Não categorizar pagamento como consumo. Não usar `Transaction.status = 'paid'` (não existe mais).

**[CRITÉRIOS DE ACEITE]**
- CA-CAT-01: sugestão no modal
- CA-CAT-02: edição persistida
- CA-CAT-03: pagamento em "Transferência/Fatura"
- CA-CAT-04: nenhuma referência a `status = 'paid'` em Transaction

---

## MÓDULO 9 — Estorno e Edição

**[OBJETIVO]** Estorno e edição de transações sem excluir (RE-01), revertendo apenas o impacto (RE-02) e aplicando apenas o delta (RE-03).

**[CONTEXTO]** RE-01 a RE-03, ED-01 a ED-07. Campos `Transaction.reversed_at`, `edited_at`, `original_valor`, `original_categoria`.

**[ENTREGÁVEL]**
1. Botão "Estornar" com modal contextual
2. Botão "Editar" com modal contextual
3. Função transacional de estorno:
   - receita (account): `saldo_atual -= valor`; `status='reversed'`
   - despesa conta (account): `saldo_atual += valor`; `status='reversed'`
   - despesa cartão (credit): valida parcelas pending; remove Installments; ajusta `Invoice.valor_total`; `status='reversed'`; saldo NÃO altera
   - pagamento: delega ao Módulo 6
4. Função transacional de edição:
   - calcula delta; atualiza saldo (se aplicável); atualiza Installment/Invoice (se cartão); guarda `original_*`; atualiza `edited_at`
5. Bloqueios: transação 'reversed', parcela 'paid', pagamento, tipo, payment_method
6. Histórico de edições (opcional)

**[RESTRIÇÕES]** Nunca DELETE. Nunca recalcular saldo. Nunca estornar despesa em cartão com parcela paga. Nunca editar transação 'reversed'. Nunca editar pagamento, tipo ou payment_method. Estorno/edição atômicos.

**[CRITÉRIOS DE ACEITE]**
- CA-EST-REC-01, CA-EST-REC-02
- CA-EST-DES-01, CA-EST-DES-02
- CA-EST-CAR-01, CA-EST-CAR-02, CA-EST-CAR-03
- CA-ED-REC-01, CA-ED-REC-02, CA-ED-REC-03
- CA-ED-DES-01, CA-ED-DES-02
- CA-ED-CAR-01, CA-ED-CAR-02, CA-ED-CAR-03
- CA-ED-NF-01
- CA-ED-BLOQ-01, CA-ED-BLOQ-02, CA-ED-BLOQ-03
- CA-INT-01, CA-INT-02, CA-INT-03

---

## MÓDULO 10 — Reconciliação

**[OBJETIVO]** Auditar integridade do saldo materializado contra saldo teórico, registrando divergências em `ReconciliationLog`.

**[CONTEXTO]** R12, RC-01 a RC-07. Entidade `ReconciliationLog` (append-only). `Account.ultima_reconciliacao`.

**[ENTREGÁVEL]**
1. Tela Configurações → Integridade (botão "Verificar", `ultima_reconciliacao`, histórico de logs)
2. Função de reconciliação (R12 + decisão de status)
3. Modal de divergência (Corrigir/Ignorar)
4. Job agendado (opcional)
5. Hook pós-migração (correção automática)
6. Hook pós-estorno em lote (10+ em <5min; sem correção automática)
7. Tela de detalhes do log (JSON, top 5 transações, antes/depois)
8. Notificação em divergência agendada

**[RESTRIÇÕES]** Nunca executar em operações normais. Nunca corrigir automaticamente sem autorização (exceto migração/pos_correcao). Nunca deletar/alterar transações. Nunca editar log (append-only). Rodar em transação isolada. Detalhes com top 5 transações. Cancelar modal = nada registrado.

**[CRITÉRIOS DE ACEITE]**
- CA-REC-01 a CA-REC-04 (manual)
- CA-REC-05, CA-REC-06 (agendada)
- CA-REC-07 (migração)
- CA-REC-08, CA-REC-09 (pós-estorno em lote)
- CA-REC-10, CA-REC-11 (append-only, rastreabilidade)
- CA-REC-12, CA-REC-13, CA-REC-14 (não-impacto)
- CA-REC-15 (performance)

---

## MÓDULO 11 — Acessibilidade (transversal)

**[OBJETIVO]** Garantir Design Universal em todas as telas.

**[CONTEXTO]** R13-A. WCAG 2.1 AA. Leitor de tela, fonte ajustável, feedback auditivo.

**[ENTREGÁVEL]**
- ARIA labels em todos os elementos interativos
- Contraste mínimo WCAG AA
- Navegação por teclado
- Fonte ajustável (3 tamanhos)
- Feedback auditivo em ações críticas (salvar, pagar, meta atingida)
- Alvos de toque ≥ 44x44px
- Cores nunca sozinhas para indicar estado (usar ícone + texto)

**[RESTRIÇÕES]** Não sacrificar acessibilidade por velocidade. Testar com NVDA/VoiceOver.

**[CRITÉRIOS DE ACEITE]**
- CA-AC-01: leitor de tela navega por todas as telas
- CA-AC-02: fonte ajustável não corta texto
- CA-AC-03: feedback auditivo funciona
- CA-AC-04: contraste WCAG AA em todos os textos
- CA-AC-05: navegação por teclado funciona

---

# PARTE 3 — Critérios de Aceitação Consolidados

## Onboarding
- CA-ONB-01 a CA-ONB-05

## Dashboard
- CA-DASH-01 a CA-DASH-06

## Lançamento
- CA-LAN-01 a CA-LAN-04

## Parcelamento
- CA-PARC-01 a CA-PARC-04

## Ciclo de Fatura
- CA-CIC-01 a CA-CIC-04

## Pagamento
- CA-PAG-01 a CA-PAG-06
- CA-R14-01, CA-R14-02
- CA-EST-01 a CA-EST-05
- CA-ROL-01 a CA-ROL-03
- CA-DUP-01, CA-DUP-02
- CA-VENC-01, CA-VENC-02

## Chat
- CA-CHAT-01 a CA-CHAT-06

## Categorização
- CA-CAT-01 a CA-CAT-04

## Estorno e Edição
- CA-EST-REC-01, CA-EST-REC-02
- CA-EST-DES-01, CA-EST-DES-02
- CA-EST-CAR-01, CA-EST-CAR-02, CA-EST-CAR-03
- CA-ED-REC-01, CA-ED-REC-02, CA-ED-REC-03
- CA-ED-DES-01, CA-ED-DES-02
- CA-ED-CAR-01, CA-ED-CAR-02, CA-ED-CAR-03
- CA-ED-NF-01
- CA-ED-BLOQ-01, CA-ED-BLOQ-02, CA-ED-BLOQ-03
- CA-INT-01, CA-INT-02, CA-INT-03

## Reconciliação
- CA-REC-01 a CA-REC-15

## Acessibilidade
- CA-AC-01 a CA-AC-05

---

# PARTE 4 — Gabaritos de Validação (resumo)

## Pagamento (Módulo 6)
Cenário base: saldo R$ 2.000; fatura R$ 1.500 (5 parcelas); próxima fatura zerada.

| Teste | Ação | Resultado |
|-------|------|-----------|
| Exato | Pagar R$ 1.500 | saldo 500; fatura 'paid'; 5 parcelas 'paid' com payment_id |
| A maior | Pagar R$ 1.800 | saldo 200; fatura 'paid'; sem crédito |
| A menor | Pagar R$ 1.200 | saldo 800; fatura 'closed'; rolled_balance 300 em novembro |
| Zero | Pagar R$ 0 | saldo 2.000; fatura 'closed'; rolled_balance 1.500 em novembro |
| Estorno exato | Estornar R$ 1.500 | saldo 2.000; fatura 'open'; parcelas 'pending' |
| Estorno a menor | Estornar R$ 1.200 | saldo 2.000; rolled_balance 300 removido |
| Estorno zero | Estornar R$ 0 | saldo inalterado; rolled_balance 1.500 removido |

## Estorno e Edição (Módulo 9)
Cenário base: saldo R$ 1.000; fatura R$ 500.

| Teste | Ação | Resultado |
|-------|------|-----------|
| Estorno receita | Receita R$ 500 → estornar | saldo volta a 1.000 |
| Estorno despesa conta | Despesa R$ 300 → estornar | saldo volta a 1.000 |
| Estorno despesa cartão | Despesa R$ 200 → estornar | fatura -200; saldo inalterado |
| Estorno despesa cartão paga | Tentar estornar | bloqueado |
| Edição receita | 500 → 700 | saldo +200 |
| Edição despesa conta | 100 → 150 | saldo -50 |
| Edição despesa cartão | 200 → 300 | fatura +100; saldo inalterado |
| Edição de data | 05/10 → 10/10 | move parcela entre faturas; saldo inalterado |
| Edição descrição | alterar | saldo inalterado |
| Bloqueios | pagamento/tipo/meio/reversed | bloqueado |

## Reconciliação (Módulo 10)
Cenário base: saldo_inicial 1.000; receita 500; despesa 200; pagamento 300; saldo_calculado 1.000.

| Teste | Ação | Resultado |
|-------|------|-----------|
| Íntegro | Verificar | "Saldo íntegro"; log `sem_divergencia` |
| Divergência + corrigir | Corromper saldo para 900 → corrigir | saldo 1.000; log `corrigido` |
| Divergência + ignorar | Corromper → ignorar | saldo 900; log `ignorado` |
| Cancelar | Cancelar modal | nada registrado |
| Agendada | Job executa | log; notifica se divergência |
| Migração | Importar com divergência | corrige automaticamente |
| Pós-lote | 10+ estornos | reconciliação sem correção |
| Append-only | 3 reconciliações | 3 logs, nenhum editado |
| Não-impacto | lançar/estornar/pagar | nenhum log criado |

---

# PARTE 5 — Como Usar Este PRD na Prática

1. **Não cole o documento inteiro no Lovable.** Rode o Módulo 1, teste, corrija.
2. **Valide com o gabarito** de cada módulo antes de avançar.
3. **Se a IA errar a matemática**, volte com: *"Corrija o Módulo X: [descreva o erro]. A regra correta é [cole a regra R/RE/RC]."*
4. **Documente cada correção** — o PRD é vivo.
5. **Ordem recomendada de execução:**
   - Módulo 1 (Onboarding)
   - Módulo 5 (Ciclo de Fatura) — testar os 3 casos
   - Módulo 3 (Lançamento Simples)
   - Módulo 4 (Parcelamento)
   - Módulo 6 (Pagamento) — testar os 4 cenários + estorno
   - Módulo 7 (Chat)
   - Módulo 8 (Categorização)
   - Módulo 9 (Estorno e Edição)
   - Módulo 2 (Dashboard) — depende dos anteriores
   - Módulo 10 (Reconciliação)
   - Módulo 11 (Acessibilidade) — transversal, testar ao final
   - Módulos de Metas, Relatórios, Dicas (v1)

---

# PARTE 6 — Recomendações Finais

## O que priorizar

1. **R14 (Rastreabilidade)** — sem ela, estorno de pagamento é impossível.
2. **R1/R12 (Saldo materializado + reconciliação)** — performance e integridade.
3. **R6 revisada (rolagem de dívida)** — comportamento real de cartão.
4. **RE-01 a RE-03 (Estorno e Edição)** — funcionalidade ausente no v5.
5. **R16 (Transparência da fatura)** — confiança do usuário.
6. **R15 (Data de vencimento informativa)** — esclarecimento.

## O que NÃO fazer no MVP

- Não reintroduza captura de imagem sem validar se chat + manual resolvem.
- Não adicione open finance, investimentos ou multi-moeda.
- Não tente comandos por voz antes do chat estar sólido.
- Não construa dicas personalizadas sem dados reais.
- Não sacrifique acessibilidade por velocidade.
- Não execute reconciliação em operações normais.
- Não edite pagamento, tipo ou meio de pagamento.

## Próximos passos

| # | Passo | Prazo sugerido |
|---|-------|----------------|
| 1 | Fechar as 5 questões em aberto (seção 8) | Semana 1 |
| 2 | Validar modelo de dados no Supabase | Semana 1 |
| 3 | Rodar Módulo 1 e 5 no Lovable | Semana 2 |
| 4 | Rodar Módulos 3, 4, 6 | Semana 3 |
| 5 | Rodar Módulos 7, 8, 9 | Semana 4 |
| 6 | Rodar Módulos 2, 10, 11 | Semana 5 |
| 7 | Teste de usabilidade (5-8 usuários) | Semana 6 |
| 8 | Teste de acessibilidade (3-5 usuários) | Semana 6 |
| 9 | Iterar com feedback | Semana 7 |
| 10 | Preparar v1 (dicas, voz, metas em risco) | Semana 8 |

---
