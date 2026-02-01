# Especificação: Biblioteca Pessoal

## Visão Geral

Área privada onde usuários gerenciam suas imagens geradas.

## Acesso

- **Requer autenticação**
- Usuário vê apenas suas próprias imagens
- Rota: `/library`

## Funcionalidades

### Listagem

- Grid de imagens do usuário
- Ordenação: mais recentes primeiro
- Filtros: Todas, Públicas, Privadas
- Contador total de imagens

### Ações por Imagem

| Ação | Descrição |
|------|-----------|
| Download PDF | Gera e baixa PDF A4 |
| Toggle Público | Alterna visibilidade na galeria |
| Deletar | Remove imagem permanentemente |
| Ver prompt | Exibe prompt usado |

## Layout

### Header da Página
```
My Library                    [Generate New →]
24 images
[All] [Public (5)] [Private (19)]
```

### Grid (similar à galeria)
```
┌─────────┐ ┌─────────┐ ┌─────────┐
│  Image  │ │  Image  │ │  Image  │
│         │ │   🌐    │ │         │
├─────────┤ ├─────────┤ ├─────────┤
│ prompt..│ │ prompt..│ │ prompt..│
│ [⬇][🌐][🗑]│ │ [⬇][🔒][🗑]│ │ [⬇][🌐][🗑]│
└─────────┘ └─────────┘ └─────────┘

🌐 = Público  🔒 = Privado  ⬇ = Download  🗑 = Deletar
```

## API Endpoints

### GET /api/images

Lista imagens do usuário autenticado.

**Query Parameters:**
| Param | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| visibility | string | 'all' | 'all', 'public', 'private' |
| cursor | string | null | Paginação |
| limit | number | 20 | Max 50 |

**Response:**
```typescript
{
  images: Array<{
    id: string,
    prompt: string,
    imageUrl: string,
    isPublic: boolean,
    createdAt: string
  }>,
  total: number,
  nextCursor: string | null
}
```

### PATCH /api/images/[id]

Atualiza visibilidade da imagem.

**Request:**
```typescript
{
  isPublic: boolean
}
```

**Response:**
```typescript
{
  success: true,
  image: {
    id: string,
    isPublic: boolean
  }
}
```

**Validações:**
- Usuário deve ser dono da imagem
- Retorna 404 se imagem não existe ou não pertence ao usuário

### DELETE /api/images/[id]

Remove imagem permanentemente.

**Response:**
```typescript
{
  success: true
}
```

**Ações:**
1. Verificar ownership
2. Deletar registro do banco
3. Deletar arquivo do R2

## Query SQL

### Listar imagens do usuário
```sql
SELECT * FROM images
WHERE user_id = $userId
  AND ($visibility = 'all'
    OR ($visibility = 'public' AND is_public = true)
    OR ($visibility = 'private' AND is_public = false))
ORDER BY created_at DESC
LIMIT $limit OFFSET $offset;
```

### Contar por visibilidade
```sql
SELECT
  COUNT(*) as total,
  COUNT(*) FILTER (WHERE is_public = true) as public_count,
  COUNT(*) FILTER (WHERE is_public = false) as private_count
FROM images
WHERE user_id = $userId;
```

## Componentes UI

### LibraryHeader
```typescript
interface LibraryHeaderProps {
  totalImages: number;
  publicCount: number;
  privateCount: number;
  activeFilter: 'all' | 'public' | 'private';
  onFilterChange: (filter: string) => void;
}
```

### LibraryCard
```typescript
interface LibraryCardProps {
  image: LibraryImage;
  onTogglePublic: () => void;
  onDelete: () => void;
  onDownload: () => void;
}
```

### DeleteConfirmModal
```typescript
interface DeleteConfirmModalProps {
  isOpen: boolean;
  imageName: string;
  onConfirm: () => void;
  onCancel: () => void;
}
```

## Estados da UI

### Empty State (sem imagens)
```
┌────────────────────────────────┐
│                                │
│     🎨 No images yet          │
│                                │
│  Create your first coloring   │
│  page and it will appear here │
│                                │
│     [Create Your First →]     │
│                                │
└────────────────────────────────┘
```

### Loading State
- Skeleton cards durante carregamento
- Spinner em ações (toggle, delete)

### Error State
- Toast para erros de ação
- Retry button para falha de listagem

## Confirmações

### Deletar Imagem
```
Delete this image?

This action cannot be undone. The image will be
permanently removed from your library and the
public gallery (if shared).

[Cancel] [Delete]
```

### Tornar Pública
```
Share to Gallery?

This image will be visible to everyone in the
public gallery.

[Cancel] [Share]
```

## Segurança

- Todas as operações validam `user_id`
- Não expor IDs de outros usuários
- Rate limit em delete (prevenir spam)

## Performance

- Prefetch de próxima página
- Optimistic updates para toggle
- Debounce em filtros

## Testes TDD

### Unit Tests

#### Componentes de Biblioteca Pessoal

| Componente | Teste | Entrada | Saída Esperada |
|------------|-------|---------|----------------|
| LibraryHeader | Renderiza contador total | `totalImages: 24` | "24 images" exibido |
| LibraryHeader | Exibe contadores por filtro | `publicCount: 5, privateCount: 19` | "[All] [Public (5)] [Private (19)]" |
| LibraryHeader | Callback ao mudar filtro | Clique em "Public" | `onFilterChange('public')` chamado |
| LibraryCard | Renderiza imagem | `imageUrl, prompt` | Imagem e prompt exibidos |
| LibraryCard | Exibe ícone correto | `isPublic: true` | Ícone 🌐 exibido |
| LibraryCard | Exibe ícone correto | `isPublic: false` | Ícone 🔒 exibido |
| LibraryCard | Callbacks de ação | Cliques em botões | `onTogglePublic()`, `onDelete()`, `onDownload()` chamados |
| DeleteConfirmModal | Modal fecha ao cancelar | Clique "Cancel" | `onCancel()` chamado, modal fechado |
| DeleteConfirmModal | Modal confirma exclusão | Clique "Delete" | `onConfirm()` chamado |
| EmptyState | Renderiza quando lista vazia | `images: []` | "No images yet" com botão de criar |

#### Filtros e Ordenação

| Teste | Entrada | Saída Esperada |
|-------|---------|----------------|
| Ordenação padrão | GET `/api/images` | Imagens ordenadas por `createdAt DESC` |
| Filtro "all" | `visibility: 'all'` | Retorna públicas e privadas |
| Filtro "public" | `visibility: 'public'` | Retorna apenas `isPublic: true` |
| Filtro "private" | `visibility: 'private'` | Retorna apenas `isPublic: false` |
| Debounce em filtro | 3 mudanças em 100ms | API chamada apenas 1 vez |
| Paginação cursor | `cursor: 'abc123', limit: 20` | Retorna próximas 20 imagens após cursor |
| Validação limit | `limit: 100` | Máximo 50 retornado |
| Contador por visibilidade | Database com mix de público/privado | `public_count` e `private_count` corretos |

### Integration Tests

#### GET /api/images do Usuário

| Teste | Setup | Ação | Verificação |
|-------|-------|------|-------------|
| Retorna imagens do usuário | User A com 5 imagens, User B com 3 | GET `/api/images` com User A | Apenas 5 imagens de User A retornadas |
| Autenticação obrigatória | Sem token | GET `/api/images` | Status 401 Unauthorized |
| Paginação funciona | 25 imagens no DB | GET `/api/images?limit=20` | 20 imagens + `nextCursor` não nulo |
| Cursor válido | `nextCursor` da resposta anterior | GET `/api/images?cursor=xyz` | Próximas 20 imagens retornadas |
| Sem duplicatas | Pagination com cursor | GET página 1, depois página 2 | Nenhuma imagem duplicada |
| Total count correto | 24 imagens do usuário | GET `/api/images` | `total: 24` retornado |
| Timestamp correto | Imagem criada em `2024-01-15T10:30:00Z` | GET `/api/images` | `createdAt: '2024-01-15T10:30:00Z'` |

#### DELETE /api/images/[id]

| Teste | Setup | Ação | Verificação |
|-------|-------|------|-------------|
| Delete sucesso | Imagem de User A no DB e R2 | DELETE `/api/images/img-123` | Status 200, `success: true` |
| Ownership check | Imagem pertence a User B | DELETE `/api/images/img-123` como User A | Status 404 Unauthorized |
| Imagem não existe | ID inválido no DB | DELETE `/api/images/fake-id` | Status 404 Not Found |
| Arquivo R2 deletado | Imagem em R2 | DELETE após sucesso | Arquivo removido de R2 |
| DB limpo | Imagem em DB | DELETE após sucesso | Registro removido do banco |
| Rate limit | 10 deletes em 1 minuto | 11º delete | Status 429 Too Many Requests |
| Retorna resposta correta | Imagem deletada | Verifica response | `{ success: true }` |

#### PATCH /api/images/[id] - Visibilidade

| Teste | Setup | Ação | Verificação |
|-------|-------|------|-------------|
| Toggle para público | `isPublic: false` | PATCH com `{ isPublic: true }` | Resposta: `isPublic: true` |
| Toggle para privado | `isPublic: true` | PATCH com `{ isPublic: false }` | Resposta: `isPublic: false` |
| DB atualizado | Imagem privada | PATCH para público | Database mostra `is_public = true` |
| Ownership check | Imagem de User B | PATCH como User A | Status 404 |
| Imagem não existe | ID inválido | PATCH com novo status | Status 404 Not Found |
| Validação payload | Sem campo `isPublic` | PATCH sem corpo | Status 400 Bad Request |
| Timestamp não muda | `updatedAt: 2024-01-15` | PATCH visibilidade | `updatedAt` permanece igual |
| Resposta contém ID | Imagem id: `img-456` | PATCH sucesso | Response inclui `id: 'img-456'` |

### E2E Tests

#### Visualizar Biblioteca Pessoal

| Cenário | Passos | Resultado Esperado |
|---------|--------|-------------------|
| Acessar biblioteca autenticado | 1. Login como User A 2. Navigate to `/library` | Página carrega, exibe header "My Library" |
| Exibir contador total | 1. User A com 24 imagens 2. Acessa `/library` | "24 images" visível abaixo do título |
| Grid renderiza | 1. Biblioteca acessada 2. Aguarda carregamento | Grid com imagens em colunas responsivas |
| Skeleton loading | 1. Acessa `/library` 2. Observa carregamento | Skeleton cards aparecem, depois imagens |
| Filtros funcionam | 1. Clica "Public (5)" 2. Aguarda | Apenas imagens públicas exibidas |
| Filtro "Private" | 1. Clica "Private (19)" 2. Aguarda | Apenas imagens privadas exibidas, contador certo |
| Filtro "All" reset | 1. Clica "Private" 2. Clica "All" | Todas imagens retornam, contador atualiza |
| Prompt exibido | 1. Paira sobre card de imagem | Prompt usado na geração visível |
| Acesso negado anônimo | 1. Sem login 2. Acessa `/library` | Redireciona para login |
| Empty state | 1. User com 0 imagens 2. Acessa `/library` | "No images yet" com botão [Create Your First →] |

#### Deletar Imagem

| Cenário | Passos | Resultado Esperado |
|---------|--------|-------------------|
| Delete com confirmação | 1. Biblioteca aberta 2. Clica 🗑 em imagem 3. Modal aparece 4. Clica "Delete" | Imagem removida da grid, toast "Deleted" |
| Cancel delete | 1. Clica 🗑 2. Modal abre 3. Clica "Cancel" | Modal fecha, imagem permanece |
| Contador atualiza | 1. 24 imagens, deleta 1 2. Vê "23 images" | Contador decrementado corretamente |
| Filtro atualiza | 1. 5 públicas, deleta pública 2. Vê "Public (4)" | Contadores corretos por visibilidade |
| Optimistic UI | 1. Clica delete 2. Imagem desaparece imediatamente | UI responde instantaneamente |
| Delete fallback | 1. API falha durante delete 2. Imagem reaparece | Imagem volta se requisição falhar |
| R2 limpo | 1. Delete bem-sucedido 2. Verifica R2 | Arquivo não existe mais em R2 |
| Erro de permissão | 1. Tenta deletar imagem de outro usuário (hack) | Delete falha, toast de erro |

#### Mudar Visibilidade

| Cenário | Passos | Resultado Esperado |
|---------|--------|-------------------|
| Tornar pública | 1. Imagem privada (🔒) 2. Clica ícone 3. Confirma share | Ícone muda para 🌐, toast "Shared" |
| Modal confirmação | 1. Clica ícone público em privado | "Share to Gallery?" modal aparece |
| Tornar privada | 1. Imagem pública (🌐) 2. Clica ícone 3. Confirma | Ícone muda para 🔒, toast "Made private" |
| Optimistic update | 1. Clica toggle 2. Ícone muda imediatamente | UI reflete mudança antes da resposta |
| Fallback otimista | 1. Toggle falha na API 2. Ícone volta | Ícone retorna ao estado anterior |
| Filtro reflete mudança | 1. Privada, torna pública, private count (-1) | Contadores atualizam em tempo real |
| Sem recarregar página | 1. Altera visibilidade 2. Grid permanece | Apenas card atualizado, sem reload |
| Múltiplas mudanças rápidas | 1. Toggle público/privado 3x rapidamente | Estado final correto, sem race conditions |
| Persiste no DB | 1. Muda visibilidade 2. Recarrega página | Estado persiste, ícone correto |
| Timeout request | 1. Toggle com timeout de rede 2. Aguarda | Retry button aparece ou toast de erro |
