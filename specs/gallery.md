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

## Testes TDD

### Unit Tests - Componentes UI

| Componente | Caso de Teste | Asserção | Status |
|-----------|---------------|----------|--------|
| **ImageCard** | Renderizar com imagem válida | Imagem carregada com alt text correto | ⬜ |
| **ImageCard** | Truncar prompt longo | Prompt com "..." após 100 caracteres | ⬜ |
| **ImageCard** | Clique no card | Abre modal com imagem full size | ⬜ |
| **ImageCard** | Hover efeito visual | Escala e shadow aplicados | ⬜ |
| **GalleryGrid** | Renderizar lista vazia | Mensagem "Nenhuma imagem encontrada" | ⬜ |
| **GalleryGrid** | Grid responsivo | 4 colunas desktop, 3 tablet, 2 mobile | ⬜ |
| **GalleryGrid** | Botão "Load More" | Visível quando hasMore = true | ⬜ |
| **GalleryGrid** | Botão "Load More" desabilitado | isLoading = true durante carregamento | ⬜ |
| **ImageModal** | Exibir imagem ampliada | Imagem full size visível | ⬜ |
| **ImageModal** | Exibir prompt completo | Prompt sem truncagem | ⬜ |
| **ImageModal** | Botão fechar | Modal fecha ao clicar X ou backdrop | ⬜ |
| **ImageModal** | Autenticado | Botões "Save" e "Download" visíveis | ⬜ |
| **ImageModal** | Não autenticado | Botões "Save" e "Download" ocultos | ⬜ |

### Unit Tests - Paginação Cursor-Based

| Funcionalidade | Caso de Teste | Asserção | Status |
|---------------|---------------|----------|--------|
| **Cursor Generation** | Gerar cursor válido | cursor = lastImageId | ⬜ |
| **Cursor Parsing** | Parsear cursor do query param | cursor extraído corretamente | ⬜ |
| **Cursor Null Handling** | Primeira página sem cursor | Retorna primeiras 20 imagens | ⬜ |
| **hasMore Flag** | Mais de 20 imagens | hasMore = true quando limit+1 retornado | ⬜ |
| **hasMore Flag** | Menos de 20 imagens | hasMore = false quando limite atingido | ⬜ |
| **nextCursor Value** | Cursor da próxima página | nextCursor = 20º item ID | ⬜ |
| **Limit Param** | Limite padrão | limit = 20 | ⬜ |
| **Limit Param** | Limite customizado | limit = 50 (máximo) | ⬜ |
| **Limit Param** | Limite excedido | Clamp a 50 | ⬜ |
| **Duplicates** | Mesma imagem em páginas | Sem repetição entre cursores | ⬜ |

### Integration Tests - API GET /api/gallery

| Endpoint | Caso de Teste | Asserção | Status |
|----------|---------------|----------|--------|
| **GET /api/gallery** | Sem parâmetros | Retorna 20 imagens públicas | ⬜ |
| **GET /api/gallery** | ?cursor=abc | Retorna imagens após ID "abc" | ⬜ |
| **GET /api/gallery** | ?limit=10 | Retorna até 10 imagens | ⬜ |
| **GET /api/gallery** | ?cursor=abc&limit=10 | Ambos parâmetros aplicados | ⬜ |
| **Response Structure** | Status 200 | Response matches TypeScript interface | ⬜ |
| **Response Structure** | Campos obrigatórios | images, nextCursor, hasMore presentes | ⬜ |
| **Response Structure** | Image Object | id, prompt, imageUrl, createdAt presentes | ⬜ |
| **Ordenação** | createdAt descending | Imagens mais recentes primeiro | ⬜ |
| **Cache** | Cache-Control header | max-age=60 presente | ⬜ |
| **Error Handling** | cursor inválido | Status 400 com mensagem de erro | ⬜ |

### Integration Tests - Filtros de Visibilidade

| Filtro | Caso de Teste | Asserção | Status |
|--------|---------------|----------|--------|
| **is_public = true** | Imagens públicas | Somente públicas retornadas | ⬜ |
| **is_public = false** | Imagens privadas | Privadas não aparecem na galeria | ⬜ |
| **Moderação** | Imagem flagged | Imagem não retornada no /api/gallery | ⬜ |
| **Usuário não autenticado** | Acesso a /api/gallery | Recebe apenas públicas | ⬜ |
| **Usuário autenticado** | Acesso a /api/gallery | Ainda recebe apenas públicas | ⬜ |
| **Admin access** | GET /api/admin/gallery | Pode ver privadas/flagged (Phase 2) | ⬜ |

### Integration Tests - Toggle Público/Privado

| Ação | Caso de Teste | Asserção | Status |
|-----|---------------|----------|--------|
| **POST /api/images/:id/toggle-visibility** | User is owner | Visibilidade alterada | ⬜ |
| **POST /api/images/:id/toggle-visibility** | User não é dono | Status 403 Forbidden | ⬜ |
| **POST /api/images/:id/toggle-visibility** | Não autenticado | Status 401 Unauthorized | ⬜ |
| **Toggle public → private** | Remover de galeria | Não retorna em GET /api/gallery | ⬜ |
| **Toggle private → public** | Adicionar à galeria | Retorna em GET /api/gallery | ⬜ |
| **Moderação após toggle** | Tornar pública | Passa por filtro automático | ⬜ |
| **Cache invalidation** | Após toggle | Cache limpo para /api/gallery | ⬜ |

### E2E Tests - Navegação na Galeria

| Fluxo | Caso de Teste | Asserção | Status |
|------|---------------|----------|--------|
| **Carregar página** | /gallery abre | Primeira página com 20 imagens | ⬜ |
| **Scroll infinito** | Scroll próximo fim | Próximas 20 imagens carregadas | ⬜ |
| **Load More button** | Clique "Load More" | Próximas imagens aparecem | ⬜ |
| **Loading state** | Durante carregamento | Spinner/skeleton visível | ⬜ |
| **Fim da lista** | hasMore = false | "Load More" desaparece | ⬜ |
| **Resposta mobile** | 320px viewport | 2 colunas, layout ajustado | ⬜ |
| **Resposta tablet** | 768px viewport | 3 colunas, layout ajustado | ⬜ |
| **Resposta desktop** | 1024px+ viewport | 4 colunas, layout ajustado | ⬜ |
| **Performance** | Carregar 100 imagens | Tempo < 2 segundos | ⬜ |

### E2E Tests - Click em Imagem para Detalhes

| Interação | Caso de Teste | Asserção | Status |
|-----------|---------------|----------|--------|
| **Click na thumbnail** | Abrir modal | Modal com imagem full size | ⬜ |
| **Imagem no modal** | Qualidade full size | Imagem clara sem pixelação | ⬜ |
| **Prompt completo** | Modal exibe prompt | Prompt integral visível | ⬜ |
| **Botão Download** | Clique (autenticado) | Inicia download PDF | ⬜ |
| **Botão Download** | Clique (não autenticado) | Redireciona para login | ⬜ |
| **Botão Save** | Clique (autenticado) | Confirmação de salvo | ⬜ |
| **Botão Save** | Clique (não autenticado) | Redireciona para login | ⬜ |
| **Fechar modal** | Clique X | Modal fecha suavemente | ⬜ |
| **Fechar modal** | Clique backdrop | Modal fecha suavemente | ⬜ |
| **ESC key** | Pressionar Esc | Modal fecha | ⬜ |
| **Navegação entre modais** | Setas prev/next | Abre imagem anterior/próxima | ⬜ |
| **Social sharing** | Copiar link | Link para imagem funciona | ⬜ |

### E2E Tests - Infinite Scroll

| Cenário | Caso de Teste | Asserção | Status |
|---------|---------------|----------|--------|
| **Scroll automático** | User scroll próximo fim | Próximas imagens carregadas automaticamente | ⬜ |
| **Múltiplas páginas** | 3+ páginas carregadas | Todas visíveis no scroll | ⬜ |
| **Prefetch** | Ao scroll página 1 | Página 2 já prefetchada | ⬜ |
| **Network throttle** | 3G lento | Infinite scroll ainda funciona | ⬜ |
| **Duplication** | Após multiplos scrolls | Nenhuma imagem duplicada | ⬜ |
| **Memory usage** | Scroll 1000 imagens | Sem memory leaks | ⬜ |
| **Intersection Observer** | Componente no viewport | Callback dispara corretamente | ⬜ |
| **Scroll position** | Voltar da página | Position scroll restaurada | ⬜ |
| **Offline fallback** | Internet cai | Último estado preservado | ⬜ |
| **Pagination alternative** | Mobile sem scroll | "Load More" button funciona | ⬜ |
