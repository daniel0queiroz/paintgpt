# Especificação: Galeria Pública

## Visão Geral

Galeria de imagens compartilhadas pela comunidade, acessível sem autenticação.

## Acesso

- **Visualização:** Público (sem login)
- **Salvar imagem:** Requer login
- **Compartilhar própria imagem:** Requer login + ser dono da imagem

## Funcionalidades

### Listagem

- Grid responsivo de imagens
- Ordenação por data (mais recentes primeiro)
- Paginação infinita ou "Load More"
- Exibir: imagem thumbnail, prompt (truncado)

### Ações por Imagem

| Ação | Auth Required | Descrição |
|------|---------------|-----------|
| Ver ampliado | Não | Modal com imagem full size |
| Ver prompt | Não | Mostrar prompt completo |
| Salvar na biblioteca | Sim | Copia referência para biblioteca pessoal |
| Download PDF | Sim | Gera PDF para impressão |

## Layout

### Desktop (≥1024px)
```
[  ] [  ] [  ] [  ]
[  ] [  ] [  ] [  ]
[  ] [  ] [  ] [  ]
     [Load More]
```

### Tablet (768px - 1023px)
```
[  ] [  ] [  ]
[  ] [  ] [  ]
  [Load More]
```

### Mobile (<768px)
```
[    ] [    ]
[    ] [    ]
 [Load More]
```

## API Endpoint

### GET /api/gallery

**Query Parameters:**
| Param | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| cursor | string | null | ID da última imagem (paginação) |
| limit | number | 20 | Itens por página (max 50) |

**Response:**
```typescript
{
  images: Array<{
    id: string,
    prompt: string,
    imageUrl: string,
    createdAt: string
  }>,
  nextCursor: string | null,
  hasMore: boolean
}
```

## Query SQL

```sql
SELECT
  i.id,
  i.prompt,
  i.image_url,
  i.created_at
FROM images i
WHERE i.is_public = true
  AND (i.id < $cursor OR $cursor IS NULL)
ORDER BY i.created_at DESC, i.id DESC
LIMIT $limit + 1;
```

## Componentes UI

### GalleryGrid
```typescript
interface GalleryGridProps {
  images: GalleryImage[];
  onLoadMore: () => void;
  hasMore: boolean;
  isLoading: boolean;
}
```

### GalleryCard
```typescript
interface GalleryCardProps {
  image: GalleryImage;
  onSave?: () => void;
  onDownload?: () => void;
}
```

### ImageModal
```typescript
interface ImageModalProps {
  image: GalleryImage;
  isOpen: boolean;
  onClose: () => void;
  onSave?: () => void;
  onDownload?: () => void;
}
```

## Salvar na Biblioteca

Quando usuário clica "Save to Library":

1. Verificar autenticação
2. Criar cópia da referência na biblioteca do usuário
3. Não duplica imagem no R2 (usa mesma URL)
4. Marcar como `is_public: false` na cópia (biblioteca pessoal)

```typescript
// Não implementado no MVP - apenas referência
// Para MVP: usuário pode baixar PDF e re-gerar se quiser
```

**Decisão MVP:** No MVP, "salvar" apenas baixa o PDF. Implementar biblioteca de favoritos no Phase 2.

## Cache

- Cache de lista: 60 segundos (ISR ou SWR)
- Cache de imagem: CDN do R2 (long-lived)

## SEO

- Página indexável (`/gallery`)
- Meta tags dinâmicas para compartilhamento
- Structured data (ImageGallery schema)

## Performance

- Lazy loading de imagens
- Placeholder blur/skeleton durante carregamento
- Thumbnails otimizados (diferentes tamanhos via R2)
- Prefetch próxima página

## Moderação

- Imagens públicas passam por review automático (filtro de conteúdo)
- Flag para reportar imagem imprópria (Phase 2)
- Admin pode remover imagens (Phase 2)

## Métricas

- Imagens mais visualizadas
- Imagens mais baixadas
- Prompts populares
