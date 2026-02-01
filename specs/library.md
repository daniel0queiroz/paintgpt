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
