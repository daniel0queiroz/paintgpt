# Especificação: Geração de Imagens

## Visão Geral

Sistema de geração de imagens de colorir usando AI via OpenRouter.

## Provedor AI

**OpenRouter** → `google/gemini-3-pro-image-preview`

## Fluxo de Geração

```
[User Prompt] → [Validação] → [Debita Crédito] → [OpenRouter API] → [Upload R2] → [Salva DB] → [Retorna URL]
```

### Passo a Passo

1. **Receber prompt** do usuário
2. **Validar** prompt (sanitização, tamanho, conteúdo)
3. **Verificar créditos** do usuário
4. **Debitar 1 crédito** (transação atômica)
5. **Montar prompt completo** com template
6. **Chamar OpenRouter API**
7. **Receber imagem** (base64 ou URL)
8. **Upload para R2**
9. **Salvar registro** em `images`
10. **Retornar** URL da imagem

## Template de Prompt

```typescript
const SYSTEM_PROMPT = `Generate a simple black and white coloring page for children ages 3-5.
The image should have:
- Clear, thick outlines
- No shading or gradients
- Simple shapes suitable for young children
- White background
- No text or letters
- Large, easy-to-color areas`;

const buildPrompt = (userPrompt: string) => `${SYSTEM_PROMPT}

Subject: ${userPrompt}`;
```

## API Request (OpenRouter)

```typescript
const response = await fetch('https://openrouter.ai/api/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${OPENROUTER_API_KEY}`,
    'Content-Type': 'application/json',
    'HTTP-Referer': 'https://paintgpt.com',
    'X-Title': 'PaintGPT'
  },
  body: JSON.stringify({
    model: 'google/gemini-3-pro-image-preview',
    messages: [
      {
        role: 'user',
        content: buildPrompt(userPrompt)
      }
    ],
    // Parâmetros específicos para geração de imagem
    response_format: { type: 'image' }
  })
});
```

## Validações

### Prompt do Usuário

| Regra | Valor |
|-------|-------|
| Tamanho mínimo | 3 caracteres |
| Tamanho máximo | 500 caracteres |
| Caracteres permitidos | Alfanuméricos, espaços, pontuação básica |

### Filtro de Conteúdo

Bloquear prompts que contenham:
- Conteúdo adulto/sexual
- Violência explícita
- Linguagem de ódio
- Informações pessoais

```typescript
const BLOCKED_TERMS = [
  // Lista de termos bloqueados
];

const isPromptSafe = (prompt: string): boolean => {
  const lower = prompt.toLowerCase();
  return !BLOCKED_TERMS.some(term => lower.includes(term));
};
```

## Tratamento de Erros

| Erro | Ação |
|------|------|
| Créditos insuficientes | Retornar 402, redirecionar para `/pricing` |
| Prompt inválido | Retornar 400 com mensagem específica |
| Timeout OpenRouter | Retornar 504, devolver crédito |
| Erro OpenRouter | Retornar 502, devolver crédito |
| Erro upload R2 | Retornar 500, devolver crédito |

### Devolução de Crédito

Se a geração falhar após o débito:

```typescript
const refundCredit = async (userId: string) => {
  await db.update(users)
    .set({ credits: sql`credits + 1` })
    .where(eq(users.id, userId));
};
```

## API Endpoint

### POST /api/generate

**Request:**
```typescript
{
  prompt: string
}
```

**Response (sucesso):**
```typescript
{
  success: true,
  image: {
    id: string,
    url: string,
    prompt: string
  },
  creditsRemaining: number
}
```

**Response (erro):**
```typescript
{
  success: false,
  error: {
    code: string,
    message: string
  }
}
```

## Configuração OpenRouter

```env
OPENROUTER_API_KEY=sk-or-...
OPENROUTER_MODEL=google/gemini-3-pro-image-preview
```

## Rate Limiting

| Limite | Valor |
|--------|-------|
| Por usuário | 10 requisições/minuto |
| Global | 100 requisições/minuto |

## Timeout

- **Geração:** 60 segundos
- **Upload R2:** 30 segundos

## Formato da Imagem

| Propriedade | Valor |
|-------------|-------|
| Formato | PNG |
| Resolução | 1024x1024 (ou aspect ratio similar) |
| Cor | Preto e branco (line art) |
| Background | Branco (#FFFFFF) |

## Métricas a Coletar

- Tempo de geração (ms)
- Taxa de sucesso/falha
- Prompts mais populares
- Distribuição de erros
