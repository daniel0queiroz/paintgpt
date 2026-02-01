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

## Futuro (Pós-MVP)

- [ ] Múltiplos pacotes de créditos
- [ ] Desconto por volume
- [ ] Créditos de trial para novos usuários
- [ ] Programa de referência
- [ ] Assinatura mensal com créditos recorrentes
