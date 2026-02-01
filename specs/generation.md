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

## Testes TDD

### 1. Testes Unitários

#### 1.1 buildPrompt Function

| Caso de Teste | Input | Expected Output | Status |
|---------------|-------|-----------------|--------|
| Prompt simples | "cat" | Retorna string com SYSTEM_PROMPT + "Subject: cat" | [ ] |
| Prompt com espaços | "sleeping dog" | Mantém espaços corretamente | [ ] |
| Prompt vazio | "" | Retorna prompt com "Subject: " | [ ] |
| Prompt com pontuação | "happy cat!" | Preserva pontuação no subject | [ ] |
| Prompt com quebra de linha | "cat\ndog" | Escapa ou remove quebras de linha | [ ] |

```typescript
describe('buildPrompt', () => {
  it('should build prompt with system prompt prefix', () => {
    const result = buildPrompt('cat');
    expect(result).toContain(SYSTEM_PROMPT);
    expect(result).toContain('Subject: cat');
  });

  it('should preserve spaces and punctuation', () => {
    const result = buildPrompt('sleeping happy dog!');
    expect(result).toContain('sleeping happy dog!');
  });

  it('should handle empty string', () => {
    const result = buildPrompt('');
    expect(result).toContain('Subject: ');
  });

  it('should handle special characters safely', () => {
    const result = buildPrompt('cat (sleeping)');
    expect(result).toContain('cat (sleeping)');
  });
});
```

#### 1.2 Validação de Prompt

| Caso de Teste | Input | Validação | Expected | Status |
|---------------|-------|-----------|----------|--------|
| Tamanho mínimo válido | "cat" | Deve ter ≥ 3 caracteres | ✓ Válido | [ ] |
| Tamanho abaixo do mínimo | "ab" | Deve rejeitar < 3 caracteres | ✗ Inválido | [ ] |
| Tamanho máximo válido | 500 caracteres | Deve aceitar ≤ 500 caracteres | ✓ Válido | [ ] |
| Tamanho acima do máximo | 501 caracteres | Deve rejeitar > 500 caracteres | ✗ Inválido | [ ] |
| Caracteres alfanuméricos | "cat123" | Deve aceitar letras e números | ✓ Válido | [ ] |
| Espaços permitidos | "sleeping cat" | Deve aceitar espaços | ✓ Válido | [ ] |
| Pontuação básica | "happy cat!" | Deve aceitar . ! ? , ; | ✓ Válido | [ ] |
| Caracteres especiais inválidos | "cat@#$" | Deve rejeitar @#$% etc | ✗ Inválido | [ ] |
| Apenas espaços | "   " | Deve rejeitar espaços vazios | ✗ Inválido | [ ] |
| Unicode válido | "gato" | Deve aceitar acentos | ✓ Válido | [ ] |

```typescript
describe('validatePrompt', () => {
  it('should accept prompt with 3 characters', () => {
    expect(validatePrompt('cat')).toBe(true);
  });

  it('should reject prompt with less than 3 characters', () => {
    expect(validatePrompt('ab')).toBe(false);
  });

  it('should accept prompt with up to 500 characters', () => {
    const prompt = 'a'.repeat(500);
    expect(validatePrompt(prompt)).toBe(true);
  });

  it('should reject prompt exceeding 500 characters', () => {
    const prompt = 'a'.repeat(501);
    expect(validatePrompt(prompt)).toBe(false);
  });

  it('should accept alphanumeric and basic punctuation', () => {
    expect(validatePrompt('happy cat 123!')).toBe(true);
  });

  it('should reject special characters', () => {
    expect(validatePrompt('cat@#$%')).toBe(false);
  });

  it('should reject whitespace-only strings', () => {
    expect(validatePrompt('   ')).toBe(false);
  });

  it('should accept accented characters', () => {
    expect(validatePrompt('gato')).toBe(true);
  });
});
```

#### 1.3 isPromptSafe (Content Filter)

| Caso de Teste | Input | Termo Bloqueado | Expected | Status |
|---------------|-------|-----------------|----------|--------|
| Prompt seguro | "happy cat" | Nenhum | ✓ Safe | [ ] |
| Conteúdo adulto | "xxx" | Sim | ✗ Unsafe | [ ] |
| Violência | "kill" | Sim | ✗ Unsafe | [ ] |
| Linguagem de ódio | "hate" | Sim | ✗ Unsafe | [ ] |
| Informação pessoal | "credit card" | Sim | ✗ Unsafe | [ ] |
| Case insensitive | "KILL" | Sim (lowercase) | ✗ Unsafe | [ ] |
| Termo bloqueado em contexto | "killua" | "kill" | ✗ Unsafe | [ ] |
| Múltiplos termos seguros | "sleeping dog cat" | Nenhum | ✓ Safe | [ ] |

```typescript
describe('isPromptSafe', () => {
  const BLOCKED_TERMS = [
    'kill', 'hate', 'xxx', 'credit card', 'ssn',
    'violence', 'adult', 'porn'
  ];

  it('should allow safe prompts', () => {
    expect(isPromptSafe('happy cat', BLOCKED_TERMS)).toBe(true);
  });

  it('should block explicit adult content', () => {
    expect(isPromptSafe('xxx', BLOCKED_TERMS)).toBe(false);
  });

  it('should block violence terms', () => {
    expect(isPromptSafe('kill the dragon', BLOCKED_TERMS)).toBe(false);
  });

  it('should block hate speech', () => {
    expect(isPromptSafe('i hate', BLOCKED_TERMS)).toBe(false);
  });

  it('should block personal information', () => {
    expect(isPromptSafe('my credit card is', BLOCKED_TERMS)).toBe(false);
  });

  it('should be case insensitive', () => {
    expect(isPromptSafe('KILL THE MONSTER', BLOCKED_TERMS)).toBe(false);
  });

  it('should block terms within words', () => {
    expect(isPromptSafe('killua from anime', BLOCKED_TERMS)).toBe(false);
  });
});
```

### 2. Testes de Integração

#### 2.1 Débito de Crédito Atômico

| Cenário | Setup | Ação | Expected | Status |
|---------|-------|------|----------|--------|
| Débito bem-sucedido | Usuário com 10 créditos | Debita 1 crédito | Saldo final = 9 créditos | [ ] |
| Débito com saldo exato | Usuário com 1 crédito | Debita 1 crédito | Saldo final = 0 créditos | [ ] |
| Débito insuficiente | Usuário com 0 créditos | Tenta debitar 1 crédito | Erro 402, saldo = 0 | [ ] |
| Transação atômica - sucesso | Múltiplas requisições simultâneas | Débito sincronizado | Nenhuma race condition | [ ] |
| Transação atômica - falha | Erro durante débito | Débito não aplicado | Saldo permanece intacto | [ ] |
| Verificação antes do débito | Usuário com 0 créditos | Verifica antes de debitar | Rejeita ANTES da operação | [ ] |

```typescript
describe('Credit Debit Integration', () => {
  beforeEach(async () => {
    await db.users.create({ id: 'user1', credits: 10 });
  });

  it('should debit 1 credit successfully', async () => {
    await debitCredit('user1', 1);
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(9);
  });

  it('should reject debit when insufficient credits', async () => {
    await db.users.update('user1', { credits: 0 });
    await expect(debitCredit('user1', 1)).rejects.toThrow('INSUFFICIENT_CREDITS');
  });

  it('should be atomic - no partial debit on failure', async () => {
    const debitPromise = debitCredit('user1', 1);
    const rollbackPromise = debitPromise.then(() => {
      throw new Error('Simulated failure');
    });

    await expect(rollbackPromise).rejects.toThrow();
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(10); // Unchanged
  });

  it('should handle concurrent requests safely', async () => {
    await db.users.update('user1', { credits: 5 });
    const promises = [
      debitCredit('user1', 1),
      debitCredit('user1', 1),
      debitCredit('user1', 1),
      debitCredit('user1', 1),
      debitCredit('user1', 1),
    ];

    await Promise.all(promises);
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(0);
  });

  it('should check credits before debiting', async () => {
    await db.users.update('user1', { credits: 0 });
    await expect(debitCredit('user1', 1)).rejects.toThrow('INSUFFICIENT_CREDITS');
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(0);
  });
});
```

#### 2.2 Refund em Caso de Falha

| Cenário | Trigger | Expected Refund | Expected Status | Status |
|---------|---------|-----------------|-----------------|--------|
| Timeout OpenRouter | 60s sem resposta | Devolver 1 crédito | 504 Gateway Timeout | [ ] |
| Erro OpenRouter 5xx | Erro 500 da API | Devolver 1 crédito | 502 Bad Gateway | [ ] |
| Erro Upload R2 | Falha ao fazer upload | Devolver 1 crédito | 500 Internal Server Error | [ ] |
| Falha DB ao salvar | Erro ao inserir imagem | Devolver 1 crédito | 500 Internal Server Error | [ ] |
| Sucesso parcial (sem refund) | Imagem gerada OK, upload OK | Nenhum refund | 200 Success | [ ] |

```typescript
describe('Credit Refund Integration', () => {
  beforeEach(async () => {
    await db.users.create({ id: 'user1', credits: 10 });
  });

  it('should refund credit on OpenRouter timeout', async () => {
    mockOpenRouterAPI.timeout();

    await expect(generateImage('user1', 'cat')).rejects.toThrow('TIMEOUT');
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(10); // Refunded
  });

  it('should refund credit on OpenRouter 5xx error', async () => {
    mockOpenRouterAPI.error500();

    await expect(generateImage('user1', 'cat')).rejects.toThrow('API_ERROR');
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(10); // Refunded
  });

  it('should refund credit on R2 upload failure', async () => {
    mockR2.uploadError();

    await expect(generateImage('user1', 'cat')).rejects.toThrow('UPLOAD_ERROR');
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(10); // Refunded
  });

  it('should refund credit on database save failure', async () => {
    mockDB.insertError();

    await expect(generateImage('user1', 'cat')).rejects.toThrow('DB_ERROR');
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(10); // Refunded
  });

  it('should not refund on success', async () => {
    mockOpenRouterAPI.success();
    mockR2.success();

    const result = await generateImage('user1', 'cat');
    const user = await db.users.findById('user1');
    expect(user.credits).toBe(9); // Not refunded
    expect(result.success).toBe(true);
  });
});
```

#### 2.3 Upload para R2

| Cenário | Input | Expected Behavior | Expected Output | Status |
|---------|-------|-------------------|-----------------|--------|
| Upload bem-sucedido | Imagem base64 válida | Salva em R2 com nome único | URL pública de R2 | [ ] |
| Arquivo vazio | Buffer vazio | Rejeita upload | Erro EMPTY_FILE | [ ] |
| Formato inválido | JPEG ao invés de PNG | Converte ou rejeita | Error ou convertido | [ ] |
| Timeout upload | Upload > 30s | Abort e refund | Erro UPLOAD_TIMEOUT | [ ] |
| Falha de conexão R2 | R2 offline | Retry e refund | Erro R2_UNAVAILABLE | [ ] |
| Sucesso com retry | Primeira falha, segunda OK | Faz upload na segunda | URL de R2 | [ ] |
| Nome de arquivo único | Múltiplos uploads | Gera nomes únicos | URLs diferentes | [ ] |

```typescript
describe('R2 Upload Integration', () => {
  it('should upload image successfully to R2', async () => {
    const imageBuffer = Buffer.from('fake-png-data');
    const url = await uploadToR2(imageBuffer, 'user1');

    expect(url).toMatch(/^https:\/\/.*\.paintgpt\.com\//);
    expect(url).toContain('user1');
  });

  it('should reject empty file', async () => {
    const emptyBuffer = Buffer.alloc(0);

    await expect(uploadToR2(emptyBuffer, 'user1')).rejects.toThrow('EMPTY_FILE');
  });

  it('should timeout after 30 seconds', async () => {
    mockR2.delay(31000);

    await expect(uploadToR2(imageBuffer, 'user1')).rejects.toThrow('UPLOAD_TIMEOUT');
  }, 35000);

  it('should retry on connection failure', async () => {
    mockR2.failOnce().thenSucceed();

    const url = await uploadToR2(imageBuffer, 'user1');
    expect(url).toBeDefined();
    expect(mockR2.retryCount).toBe(1);
  });

  it('should generate unique filenames for concurrent uploads', async () => {
    const promises = Array(5).fill(null).map(() =>
      uploadToR2(imageBuffer, 'user1')
    );

    const urls = await Promise.all(promises);
    const uniqueUrls = new Set(urls);
    expect(uniqueUrls.size).toBe(5);
  });
});
```

#### 2.4 Salvamento no DB

| Cenário | Input | Expected Behavior | Expected DB State | Status |
|---------|-------|-------------------|-------------------|--------|
| Salvamento bem-sucedido | Dados completos | Insere em `images` | Registro com id, url, prompt | [ ] |
| Dados incompletos | prompt vazio | Rejeita | Nenhum registro | [ ] |
| URL inválida | URL malformada | Rejeita ou valida | Erro ou validação | [ ] |
| Duplicata de URL | URL já existe | Rejeita ou gera nova | Erro ou nova URL | [ ] |
| Campo timestamp | Insert automático | Salva timestamp gerado | created_at = now() | [ ] |
| Constraint violated | user_id inválido | Erro FK | Nenhum registro | [ ] |
| Transação com crédito | Débito + Insert imagem | Ambos succedem ou ambos falham | Saldo + imagem ou nada | [ ] |

```typescript
describe('Database Save Integration', () => {
  beforeEach(async () => {
    await db.users.create({ id: 'user1', credits: 10 });
  });

  it('should save image record successfully', async () => {
    const imageData = {
      userId: 'user1',
      url: 'https://r2.example.com/image.png',
      prompt: 'cat'
    };

    const result = await saveImage(imageData);
    expect(result.id).toBeDefined();
    expect(result.prompt).toBe('cat');
    expect(result.createdAt).toBeDefined();
  });

  it('should reject incomplete data', async () => {
    const invalidData = {
      userId: 'user1',
      prompt: '' // Empty prompt
    };

    await expect(saveImage(invalidData)).rejects.toThrow();
  });

  it('should validate URL format', async () => {
    const invalidUrl = {
      userId: 'user1',
      url: 'not-a-url',
      prompt: 'cat'
    };

    await expect(saveImage(invalidUrl)).rejects.toThrow('INVALID_URL');
  });

  it('should enforce foreign key constraint', async () => {
    const invalidUserId = {
      userId: 'nonexistent-user',
      url: 'https://r2.example.com/image.png',
      prompt: 'cat'
    };

    await expect(saveImage(invalidUserId)).rejects.toThrow('FK_CONSTRAINT');
  });

  it('should auto-generate timestamp', async () => {
    const imageData = {
      userId: 'user1',
      url: 'https://r2.example.com/image.png',
      prompt: 'cat'
    };

    const before = new Date();
    const result = await saveImage(imageData);
    const after = new Date();

    expect(result.createdAt.getTime()).toBeGreaterThanOrEqual(before.getTime());
    expect(result.createdAt.getTime()).toBeLessThanOrEqual(after.getTime());
  });

  it('should be part of atomic transaction with credit debit', async () => {
    const generatePromise = generateImageFullFlow('user1', 'cat');

    await generatePromise;

    const user = await db.users.findById('user1');
    const images = await db.images.findByUserId('user1');

    expect(user.credits).toBe(9); // Deducted
    expect(images.length).toBe(1); // Saved
  });
});
```

### 3. Mock Strategies para OpenRouter API

#### 3.1 Mock Tiers

| Tier | Complexidade | Caso de Uso | Setup |
|------|-------------|-----------|-------|
| Level 1: Jest Mock | Básico | Testes unitários rápidos | `jest.mock()` simples |
| Level 2: MSW (Mock Service Worker) | Médio | Testes de integração com HTTP | Intercepta requests HTTP |
| Level 3: Docker Container | Avançado | Testes E2E realistas | Container mock da API |
| Level 4: Testcontainers | Produção-like | Staging tests | Containers dinâmicos |

#### 3.2 Implementação de Mocks

**Level 1: Jest Mock (Rápido)**

```typescript
// __mocks__/openrouter.ts
export const mockOpenRouterAPI = {
  success: jest.fn().mockResolvedValue({
    choices: [{ message: { content: 'base64-image-data' } }]
  }),
  timeout: jest.fn().mockRejectedValue(new Error('TIMEOUT')),
  error: jest.fn().mockRejectedValue(new Error('API_ERROR')),
  invalidModel: jest.fn().mockRejectedValue(new Error('INVALID_MODEL'))
};

jest.mock('openrouter', () => ({
  generateImage: mockOpenRouterAPI.success
}));

// test file
describe('OpenRouter Mock Level 1', () => {
  it('should call OpenRouter with correct params', async () => {
    await generateImage('user1', 'cat');

    expect(mockOpenRouterAPI.success).toHaveBeenCalledWith(
      expect.objectContaining({
        model: 'google/gemini-3-pro-image-preview',
        messages: expect.any(Array)
      })
    );
  });

  it('should handle API timeout', async () => {
    mockOpenRouterAPI.success.mockRejectedValueOnce(new Error('TIMEOUT'));

    await expect(generateImage('user1', 'cat')).rejects.toThrow('TIMEOUT');
  });
});
```

**Level 2: MSW (HTTP Mocking)**

| Endpoint | Method | Mock Response | Status | Setup |
|----------|--------|---------------|--------|-------|
| /api/v1/chat/completions | POST | { choices: [{ message: { content } }] } | 200 | `rest.post()` |
| /api/v1/chat/completions (timeout) | POST | - | 504 | `mockDelay(70000)` |
| /api/v1/chat/completions (error) | POST | { error: 'Invalid request' } | 400 | `res(ctx.status(400))` |

```typescript
// mocks/handlers.ts
import { rest } from 'msw';

export const handlers = [
  rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
    const authHeader = req.headers.get('authorization');

    if (!authHeader) {
      return res(ctx.status(401), ctx.json({ error: 'Unauthorized' }));
    }

    return res(
      ctx.status(200),
      ctx.json({
        choices: [{
          message: {
            content: 'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg=='
          }
        }]
      })
    );
  }),

  // Timeout simulation
  rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
    if (req.body?.includes('timeout')) {
      return res(ctx.delay(70000)); // Exceeds 60s timeout
    }
  }),

  // Error simulation
  rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
    if (req.body?.includes('error')) {
      return res(ctx.status(500), ctx.json({ error: 'Internal Server Error' }));
    }
  })
];

// Setup in test
import { setupServer } from 'msw/node';

const server = setupServer(...handlers);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('OpenRouter Mock Level 2 (MSW)', () => {
  it('should generate image successfully', async () => {
    const result = await generateImage('user1', 'cat');

    expect(result).toHaveProperty('success', true);
    expect(result).toHaveProperty('image.url');
  });

  it('should timeout after 60 seconds', async () => {
    server.use(
      rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
        return res(ctx.delay(70000));
      })
    );

    await expect(generateImage('user1', 'timeout')).rejects.toThrow('TIMEOUT');
  }, 75000);

  it('should handle API errors', async () => {
    server.use(
      rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ error: 'Server Error' }));
      })
    );

    await expect(generateImage('user1', 'error')).rejects.toThrow('API_ERROR');
  });

  it('should validate authorization header', async () => {
    server.use(
      rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
        const auth = req.headers.get('authorization');
        if (!auth) {
          return res(ctx.status(401), ctx.json({ error: 'Unauthorized' }));
        }
        return res(ctx.status(200), ctx.json({ choices: [{ message: { content: '...' } }] }));
      })
    );

    await expect(generateImageWithoutAuth()).rejects.toThrow('UNAUTHORIZED');
  });
});
```

**Level 3: Docker Mock Container**

| Componente | Imagem | Porta | Healthcheck |
|-----------|--------|-------|-------------|
| Mock API | node:18-alpine | 3001 | GET /health |
| Redis (cache) | redis:7-alpine | 6379 | PING |
| PostgreSQL (test DB) | postgres:15 | 5432 | pg_isready |

```typescript
// docker-compose.test.yml
version: '3.8'
services:
  openrouter-mock:
    image: node:18-alpine
    ports:
      - '3001:3001'
    environment:
      - NODE_ENV=test
    volumes:
      - ./mocks/api-server.js:/app/server.js
    command: node /app/server.js
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:3001/health']
      interval: 5s
      timeout: 3s
      retries: 5

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=test
      - POSTGRES_PASSWORD=test
      - POSTGRES_DB=paintgpt_test
    ports:
      - '5433:5432'
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U test']
      interval: 5s
      timeout: 3s
      retries: 5

// Mock API Server
// mocks/api-server.js
const express = require('express');
const app = express();

app.post('/api/v1/chat/completions', (req, res) => {
  const { messages } = req.body;

  if (!messages || messages.length === 0) {
    return res.status(400).json({ error: 'Invalid request' });
  }

  // Simulate processing time (1-2 seconds)
  setTimeout(() => {
    res.json({
      choices: [{
        message: {
          content: Buffer.from('fake-coloring-page-image').toString('base64')
        }
      }],
      usage: { prompt_tokens: 10, completion_tokens: 50 }
    });
  }, Math.random() * 1000 + 1000);
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.listen(3001, () => console.log('Mock API running on port 3001'));

// Test using Docker
describe('OpenRouter Mock Level 3 (Docker)', () => {
  beforeAll(async () => {
    // Start containers
    await exec('docker-compose -f docker-compose.test.yml up -d');
    await waitForService('http://localhost:3001/health');
  });

  afterAll(async () => {
    await exec('docker-compose -f docker-compose.test.yml down');
  });

  it('should call mock OpenRouter API', async () => {
    const response = await fetch('http://localhost:3001/api/v1/chat/completions', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        messages: [{ role: 'user', content: 'cat' }]
      })
    });

    const data = await response.json();
    expect(data.choices[0].message.content).toBeDefined();
  });
});
```

#### 3.3 Mock Scenarios Matrix

| Cenário | Trigger | Mock Response | Assert |
|---------|---------|---------------|--------|
| Success | prompt: "cat" | 200 + base64 image | image.url válida |
| Timeout | delay > 60s | No response (timeout) | error TIMEOUT |
| Rate Limit | 11º request/min | 429 Too Many Requests | error RATE_LIMITED |
| Invalid Model | model="invalid" | 400 Bad Request | error INVALID_MODEL |
| Invalid Auth | sem Bearer token | 401 Unauthorized | error UNAUTHORIZED |
| Quota Exceeded | créditos insuficientes na API | 402 Payment Required | error QUOTA_EXCEEDED |
| Malformed Response | JSON inválido | 200 + JSON quebrado | error INVALID_RESPONSE |
| Circuit Breaker | 5 erros seguidos | fallback response | use fallback |

```typescript
describe('OpenRouter Mock Scenarios', () => {
  beforeEach(() => {
    server.listen();
  });

  afterEach(() => {
    server.resetHandlers();
    server.close();
  });

  it('should handle 429 rate limit', async () => {
    server.use(
      rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
        return res(
          ctx.status(429),
          ctx.json({ error: 'Too Many Requests' }),
          ctx.set('Retry-After', '60')
        );
      })
    );

    await expect(generateImage('user1', 'cat')).rejects.toThrow('RATE_LIMITED');
  });

  it('should handle 402 quota exceeded', async () => {
    server.use(
      rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
        return res(
          ctx.status(402),
          ctx.json({ error: 'Insufficient credits' })
        );
      })
    );

    await expect(generateImage('user1', 'cat')).rejects.toThrow('QUOTA_EXCEEDED');
  });

  it('should implement circuit breaker after 5 failures', async () => {
    let failureCount = 0;

    server.use(
      rest.post('https://openrouter.ai/api/v1/chat/completions', (req, res, ctx) => {
        failureCount++;
        if (failureCount <= 5) {
          return res(ctx.status(500), ctx.json({ error: 'Server Error' }));
        }
        return res(ctx.status(503), ctx.json({ error: 'Circuit breaker open' }));
      })
    );

    for (let i = 0; i < 5; i++) {
      await expect(generateImage('user1', 'cat')).rejects.toThrow();
    }

    // Circuit breaker should now reject immediately
    await expect(generateImage('user1', 'cat')).rejects.toThrow('CIRCUIT_BREAKER_OPEN');
  });
});
```

### 4. Cobertura de Testes Esperada

| Componente | Cobertura Esperada | Métrica |
|-----------|------------------|---------|
| buildPrompt | 100% | Todos os casos de prompt |
| validatePrompt | 100% | Validações: tamanho, caracteres |
| isPromptSafe | 100% | Terms bloqueados + case sensitivity |
| debitCredit | 100% | Débito bem-sucedido, insuficiente, transação |
| refundCredit | 100% | Refund em cada erro possível |
| uploadToR2 | 95% | Happy path + erros comuns |
| saveImage | 100% | Insert bem-sucedido + constraints |
| generateImage (end-to-end) | 85% | Fluxo completo com diferentes cenários |
| OpenRouter API integration | 90% | Mock em todos os níveis |

### 5. Comandos de Teste

```bash
# Rodar todos os testes
bun run test

# Rodar apenas testes unitários
bun run test:unit

# Rodar apenas testes de integração
bun run test:integration

# Rodar com cobertura
bun run test:coverage

# Rodar testes de E2E com Docker
bun run test:e2e

# Watch mode
bun run test:watch

# Rodar teste específico
bun run test -- buildPrompt

# Gerar relatório de cobertura
bun run test:coverage -- --coverage-reporters=html
```

### 6. CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: paintgpt_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3
      - uses: oven-sh/setup-bun@v1

      - name: Install dependencies
        run: bun install

      - name: Run linter
        run: bun run lint

      - name: Run tests
        run: bun run test:coverage
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/paintgpt_test

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
```
