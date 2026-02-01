# Especificação: Sistema de Créditos

## Visão Geral

Sistema de créditos pré-pagos para controlar o uso de gerações de imagem.

## Modelo de Precificação

| Pacote | Preço | Créditos | Custo/Crédito |
|--------|-------|----------|---------------|
| Padrão | $9.00 | 20 | $0.45 |

## Regras de Créditos

- **1 crédito = 1 geração** de imagem
- Créditos **nunca expiram**
- Créditos **não são reembolsáveis**
- Créditos **não são transferíveis** entre contas

## Operações com Créditos

### Débito (Geração)

```typescript
const debitCredit = async (userId: string): Promise<boolean> => {
  const result = await db
    .update(users)
    .set({
      credits: sql`credits - 1`,
      updatedAt: new Date()
    })
    .where(and(
      eq(users.id, userId),
      gt(users.credits, 0)
    ))
    .returning({ credits: users.credits });

  return result.length > 0;
};
```

### Crédito (Compra)

```typescript
const addCredits = async (userId: string, amount: number): Promise<number> => {
  const result = await db
    .update(users)
    .set({
      credits: sql`credits + ${amount}`,
      updatedAt: new Date()
    })
    .where(eq(users.id, userId))
    .returning({ credits: users.credits });

  return result[0].credits;
};
```

### Consulta de Saldo

```typescript
const getCredits = async (userId: string): Promise<number> => {
  const result = await db
    .select({ credits: users.credits })
    .from(users)
    .where(eq(users.id, userId));

  return result[0]?.credits ?? 0;
};
```

## Estados do Usuário

| Estado | Créditos | Ação Disponível |
|--------|----------|-----------------|
| Novo | 0 | Ver galeria, comprar créditos |
| Com créditos | > 0 | Gerar imagens |
| Sem créditos | 0 | Prompt para comprar |

## UI de Créditos

### Header (quando logado)
```
[UserButton] 🎨 12 credits
```

### Página /generate (sem créditos)
```
You're out of credits!
Get 20 credits for $9 to continue creating.
[Buy Credits →]
```

### Após geração
```
Image generated!
You have 11 credits remaining.
[Generate Another] [Download PDF]
```

## Webhook Flow (Compra)

```
Stripe Checkout → Webhook → Validar → Creditar → Confirmar
```

1. Usuário completa pagamento no Stripe
2. Stripe envia evento `checkout.session.completed`
3. Webhook valida assinatura
4. Busca `purchases` pelo `stripe_session_id`
5. Atualiza status para `completed`
6. Adiciona créditos ao usuário
7. Retorna 200 OK

## Proteções

### Condição de Corrida

Usar transação atômica para débito:

```sql
UPDATE users
SET credits = credits - 1
WHERE id = $1 AND credits > 0
RETURNING credits;
```

Se `RETURNING` vazio = créditos insuficientes (outro request consumiu).

### Idempotência

Webhook Stripe pode ser chamado múltiplas vezes. Verificar:

```typescript
const purchase = await db.query.purchases.findFirst({
  where: eq(purchases.stripeSessionId, sessionId)
});

if (purchase?.status === 'completed') {
  return; // Já processado
}
```

### Auditoria

Toda operação de crédito deve ser rastreável:
- Compras em `purchases` com `stripe_session_id`
- Uso em `images` (1 imagem = 1 crédito usado)

## Endpoint: Saldo

### GET /api/credits

**Response:**
```typescript
{
  credits: number,
  canGenerate: boolean
}
```

## Notificações

| Evento | Notificação |
|--------|-------------|
| Créditos zerados | Toast + banner na página |
| Compra completada | Toast de confirmação |
| Crédito usado | Atualização no header |

## Testes TDD

### Unit Tests - Cálculo de Créditos

| ID | Descrição | Entrada | Esperado | Status |
|----|-----------|---------| ---------|--------|
| UT-001 | Calcular saldo inicial (novo usuário) | userId: "user1" | credits: 0 | TODO |
| UT-002 | Calcular saldo após compra | userId: "user1", amount: 20 | credits: 20 | TODO |
| UT-003 | Calcular saldo após múltiplas compras | userId: "user1", [20, 20, 10] | credits: 50 | TODO |
| UT-004 | Calcular consumo após geração | currentCredits: 20, debit: 1 | credits: 19 | TODO |
| UT-005 | Custo por crédito (Pacote Padrão) | preço: 9.00, credits: 20 | costPerCredit: 0.45 | TODO |
| UT-006 | Validar créditos nunca negativos | currentCredits: 5, debit: 10 | blocked: true | TODO |
| UT-007 | Validar créditos não expiram | createdAt: -365 dias | credits: válidos | TODO |

### Unit Tests - Débito de Créditos

| ID | Descrição | Entrada | Esperado | Status |
|----|-----------|---------| ---------|--------|
| UT-008 | Débito com saldo suficiente | userId: "user1", credits: 10 | success: true, credits: 9 | TODO |
| UT-009 | Débito com saldo insuficiente | userId: "user1", credits: 0 | success: false, credits: 0 | TODO |
| UT-010 | Débito com saldo exato | userId: "user1", credits: 1 | success: true, credits: 0 | TODO |
| UT-011 | Múltiplos débitos sequenciais | userId: "user1", credits: 5, debits: 3 | success: true, final: 2 | TODO |
| UT-012 | Débito atualiza updatedAt | userId: "user1" | updatedAt: agora | TODO |
| UT-013 | Débito não permite valor negativo | userId: "user1", debit: -5 | rejected: true | TODO |
| UT-014 | Débito com usuário inexistente | userId: "nonexistent" | success: false | TODO |

### Unit Tests - Adição de Créditos

| ID | Descrição | Entrada | Esperado | Status |
|----|-----------|---------| ---------|--------|
| UT-015 | Adicionar créditos (novo usuário) | userId: "user1", amount: 20 | credits: 20 | TODO |
| UT-016 | Adicionar créditos (existente) | userId: "user1", amount: 15, current: 5 | credits: 20 | TODO |
| UT-017 | Adicionar zero créditos | userId: "user1", amount: 0, current: 10 | credits: 10 | TODO |
| UT-018 | Adicionar valor decimal | userId: "user1", amount: 0.5 | rejected: true | TODO |
| UT-019 | Adicionar créditos atualiza updatedAt | userId: "user1", amount: 10 | updatedAt: agora | TODO |
| UT-020 | Adicionar não permite valor negativo | userId: "user1", amount: -5 | rejected: true | TODO |
| UT-021 | Adicionar com usuário inexistente | userId: "nonexistent", amount: 10 | success: true, user: created | TODO |
| UT-022 | Limite máximo de créditos | userId: "user1", amount: 999999999 | success: true | TODO |

### Unit Tests - Consulta de Saldo

| ID | Descrição | Entrada | Esperado | Status |
|----|-----------|---------| ---------|--------|
| UT-023 | Consultar saldo (novo usuário) | userId: "user1" | credits: 0 | TODO |
| UT-024 | Consultar saldo após compra | userId: "user1", after: addCredits(20) | credits: 20 | TODO |
| UT-025 | Consultar saldo (usuário inexistente) | userId: "nonexistent" | credits: 0 | TODO |
| UT-026 | Consultar saldo (múltiplas operações) | userId: "user1", ops: [add(20), debit(5), add(10)] | credits: 25 | TODO |

### Integration Tests - Fluxo de Créditos

| ID | Descrição | Cenário | Passos | Esperado | Status |
|----|-----------| ---------|--------|---------|---------|
| IT-001 | Novo usuário compra créditos | Fluxo completo de compra | 1. Login 2. Acessa compra 3. Completa pagamento 4. Webhook processa | credits: 20, status: "completed" | TODO |
| IT-002 | Geração com créditos suficientes | Usuário gera imagem | 1. Usuário tem 10 créditos 2. Clica gerar 3. Imagem gerada | credits: 9, image: saved | TODO |
| IT-003 | Geração sem créditos | Usuário sem saldo | 1. Usuário tem 0 créditos 2. Tenta gerar 3. Banner aparece | generation: blocked, banner: shown | TODO |
| IT-004 | Múltiplas compras idempotentes | Webhook recebido 2x | 1. Webhook 1 processa 2. Webhook 2 chega | credits: +20 (não duplicado) | TODO |
| IT-005 | Condição de corrida (débito) | Dois requests simultâneos | 1. Usuário tem 1 crédito 2. Requests simultâneos | success: 1 aprovado, 1 rejeitado | TODO |
| IT-006 | Auditoria de compra | Rastreamento de transação | 1. Compra realizada 2. Verificar purchases table | stripe_session_id: logged, status: tracked | TODO |
| IT-007 | Auditoria de uso | Rastreamento de geração | 1. Imagem gerada 2. Verificar images table | credit: deducted, image: linked | TODO |

### Integration Tests - Webhook Stripe

| ID | Descrição | Evento | Validações | Esperado | Status |
|----|-----------| ---------|-----------|---------|---------|
| IT-008 | Webhook com sessão válida | checkout.session.completed | signature, sessionId | credits: added, status: "completed" | TODO |
| IT-009 | Webhook com assinatura inválida | checkout.session.completed | signature: invalid | rejected: true, credits: unchanged | TODO |
| IT-010 | Webhook com sessão inexistente | checkout.session.completed | sessionId: nonexistent | error: not found, credits: unchanged | TODO |
| IT-011 | Webhook idempotente | checkout.session.completed | sessionId: duplicate | idempotent: true, credits: +20 (uma vez) | TODO |
| IT-012 | Webhook com usuário removido | checkout.session.completed | userId: deleted | handled: gracefully | TODO |

### E2E Tests - Fluxo do Usuário

| ID | Descrição | Ações do Usuário | Validações | Esperado | Status |
|----|-----------| ---------|-----------|---------|---------|
| E2E-001 | Novo usuário recebe header sem créditos | 1. Login 2. Observar header | "0 credits" visível | header: 0 credits | TODO |
| E2E-002 | Fluxo completo: Compra → Geração → Download | 1. Compra 20 créditos 2. Gera imagem 3. Download PDF | Cada etapa funciona | final: 19 credits, PDF: downloaded | TODO |
| E2E-003 | Notificação ao usar último crédito | 1. Usuário com 1 crédito 2. Gera imagem | Toast + banner | notification: shown, banner: displayed | TODO |
| E2E-004 | Galeria visível mesmo sem créditos | 1. Sem créditos 2. Acessa /gallery | Galeria carrega | gallery: visible, prompt: "buy credits" | TODO |
| E2E-005 | CTA para comprar aparece quando zerado | 1. Usuário com 10 créditos 2. Gera 10 imagens 3. Tenta gerar 11ª | CTA visível | button: "Buy Credits →", page: /purchase | TODO |
| E2E-006 | Header atualiza em tempo real | 1. Compra 20 créditos 2. Gera 5 imagens | Header sincronizado | credits: atualizado dinamicamente | TODO |
| E2E-007 | Créditos persistem entre sessões | 1. Usuário com 15 créditos 2. Logout 3. Login | Saldo mantido | credits: 15 | TODO |
| E2E-008 | Erro de pagamento não credita | 1. Falha no pagamento Stripe 2. Usuário sem créditos | Status: failed | credits: 0, error: handled | TODO |

### E2E Tests - Proteções

| ID | Descrição | Cenário | Ações | Validações | Esperado | Status |
|----|-----------| ---------| ------|-----------|---------|---------|
| E2E-009 | Proteção contra débito duplo | Requisição simultânea (race condition) | 2 requests: debit | Apenas 1 sucesso | final: -1 crédito | TODO |
| E2E-010 | Proteção contra débito negativo | Usuário com 5 créditos | Tenta debit 10 | Transação rejeitada | credits: 5 (unchanged) | TODO |
| E2E-011 | Webhook não duplica créditos | Mesmo evento 3x | Processa 3x | Idempotência validada | credits: +20 (uma vez) | TODO |
| E2E-012 | Validação de transferência (bloqueada) | Tenta transferir créditos | API call: POST /api/transfer | Endpoint: não existe | error: 404 | TODO |

### Performance Tests

| ID | Descrição | Operação | Limite | Métrica | Esperado | Status |
|----|-----------| ---------|--------|---------|---------|---------|
| PERF-001 | Latência getCredits | Consulta saldo | 1 usuário | < 50ms | passed | TODO |
| PERF-002 | Latência debitCredit | Débito crédito | 1 usuário | < 100ms | passed | TODO |
| PERF-003 | Latência addCredits | Adiciona créditos | 1 usuário | < 100ms | passed | TODO |
| PERF-004 | Load test múltiplos débitos | 100 débitos simultâneos | 1 usuário | < 2s total | all succeed/fail correctly | TODO |
| PERF-005 | Load test múltiplos usuários | 1000 usuários consultam saldo | concurrent | < 1s | all succeed | TODO |

### Security Tests

| ID | Descrição | Ataque | Teste | Esperado | Status |
|----|-----------| ---------|-------|---------|---------|
| SEC-001 | SQL Injection na consulta de saldo | userId: `" OR "1"="1` | Parametrized query | injected: false, user: own data | TODO |
| SEC-002 | Débito sem autenticação | GET /api/debits?userId=other&amount=10 | Sem token JWT | rejected: 401 | TODO |
| SEC-003 | Crédito sem validação de webhook | POST /webhook body falsificado | Assinatura inválida | rejected: 403 | TODO |
| SEC-004 | Acesso de usuário A aos créditos de B | Tenta consultar userId="userB" | Token de userA | rejected: 403 | TODO |
| SEC-005 | Webhook com payload modificado | Altera stripe_session_id | Verifica assinatura | integrity: valid | TODO |

## Futuro (Pós-MVP)

- [ ] Múltiplos pacotes de créditos
- [ ] Desconto por volume
- [ ] Créditos de trial para novos usuários
- [ ] Programa de referência
- [ ] Assinatura mensal com créditos recorrentes
