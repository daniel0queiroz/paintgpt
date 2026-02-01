# Especificação: Download PDF

## Visão Geral

Sistema para gerar PDFs otimizados para impressão das páginas de colorir.

## Formato do PDF

| Propriedade | Valor |
|-------------|-------|
| Tamanho | A4 (210mm x 297mm) |
| Orientação | Retrato |
| Margens | 10mm (todas) |
| Área útil | 190mm x 277mm |
| Resolução | 300 DPI |
| Cor | Preto e branco |

## Fluxo de Download

```
[Click Download] → [Gera PDF] → [Stream para browser] → [Salva arquivo]
```

1. Usuário clica "Download PDF"
2. Request para `/api/download/[id]`
3. Server busca imagem do R2
4. Gera PDF com a imagem
5. Retorna PDF como stream
6. Browser baixa arquivo

## API Endpoint

### GET /api/download/[id]

**Headers Response:**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="coloring-page-{id}.pdf"
```

**Fluxo:**
```typescript
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  // 1. Buscar imagem
  const image = await db.query.images.findFirst({
    where: eq(images.id, params.id)
  });

  if (!image) {
    return new Response('Not found', { status: 404 });
  }

  // 2. Verificar acesso
  // - Se pública: qualquer um pode baixar
  // - Se privada: apenas o dono
  const { userId } = auth();
  if (!image.isPublic && image.userId !== userId) {
    return new Response('Forbidden', { status: 403 });
  }

  // 3. Buscar imagem do R2
  const imageBuffer = await fetchFromR2(image.imageUrl);

  // 4. Gerar PDF
  const pdfBuffer = await generatePDF(imageBuffer, image.prompt);

  // 5. Retornar
  return new Response(pdfBuffer, {
    headers: {
      'Content-Type': 'application/pdf',
      'Content-Disposition': `attachment; filename="coloring-page-${params.id.slice(0, 8)}.pdf"`
    }
  });
}
```

## Geração do PDF

### Biblioteca

**pdf-lib** - Leve, sem dependências nativas.

```typescript
import { PDFDocument, rgb } from 'pdf-lib';

async function generatePDF(
  imageBuffer: Buffer,
  prompt: string
): Promise<Uint8Array> {
  // Criar documento
  const pdfDoc = await PDFDocument.create();

  // Tamanho A4 em pontos (1 inch = 72 points, A4 = 8.27 x 11.69 inches)
  const pageWidth = 595.28;  // 210mm
  const pageHeight = 841.89; // 297mm
  const margin = 28.35;      // 10mm

  // Adicionar página
  const page = pdfDoc.addPage([pageWidth, pageHeight]);

  // Embed imagem (PNG)
  const image = await pdfDoc.embedPng(imageBuffer);

  // Calcular dimensões mantendo aspect ratio
  const availableWidth = pageWidth - (margin * 2);
  const availableHeight = pageHeight - (margin * 2) - 30; // -30 para rodapé

  const imageAspect = image.width / image.height;
  const availableAspect = availableWidth / availableHeight;

  let drawWidth, drawHeight;
  if (imageAspect > availableAspect) {
    drawWidth = availableWidth;
    drawHeight = availableWidth / imageAspect;
  } else {
    drawHeight = availableHeight;
    drawWidth = availableHeight * imageAspect;
  }

  // Centralizar imagem
  const x = margin + (availableWidth - drawWidth) / 2;
  const y = margin + 30 + (availableHeight - drawHeight) / 2;

  // Desenhar imagem
  page.drawImage(image, {
    x,
    y,
    width: drawWidth,
    height: drawHeight,
  });

  // Rodapé (opcional)
  page.drawText('paintgpt.com', {
    x: pageWidth / 2 - 30,
    y: margin,
    size: 8,
    color: rgb(0.7, 0.7, 0.7),
  });

  return pdfDoc.save();
}
```

## Otimizações

### Compressão de Imagem

Antes de embedar no PDF:
```typescript
import sharp from 'sharp';

const optimizedImage = await sharp(imageBuffer)
  .grayscale() // Garantir preto e branco
  .png({ compressionLevel: 9 })
  .toBuffer();
```

### Cache de PDF

Para imagens populares da galeria:
```typescript
// Checar cache
const cachedPdf = await r2.get(`pdfs/${imageId}.pdf`);
if (cachedPdf) {
  return new Response(cachedPdf.body, {...});
}

// Gerar e cachear
const pdfBuffer = await generatePDF(...);
await r2.put(`pdfs/${imageId}.pdf`, pdfBuffer);
```

## UI de Download

### Botão
```typescript
<Button onClick={handleDownload} disabled={isDownloading}>
  {isDownloading ? (
    <>
      <Spinner /> Generating PDF...
    </>
  ) : (
    <>
      <DownloadIcon /> Download PDF
    </>
  )}
</Button>
```

### Feedback
- Loading state durante geração
- Toast de sucesso após download
- Toast de erro se falhar

## Métricas

- Downloads por imagem
- Tempo de geração de PDF
- Taxa de cache hit

## Considerações

- Não armazenar PDFs indefinidamente (gerar on-demand)
- Cache apenas para top 100 imagens mais baixadas
- Timeout de 30 segundos para geração
- Max file size: ~5MB por PDF

## Testes TDD

### Testes Unitários - Geração de PDF

| ID | Descrição | Entrada | Saída Esperada | Critérios de Aceitação |
|----|-----------|---------|-----------------|------------------------|
| U001 | Gerar PDF com imagem válida | PNG 800x600 + "dog drawing" | Uint8Array | PDF contém imagem, documento criado com sucesso |
| U002 | Manter aspect ratio em landscape | PNG 1200x400 | Uint8Array | Imagem não distorcida, usa altura máxima disponível |
| U003 | Manter aspect ratio em portrait | PNG 300x900 | Uint8Array | Imagem não distorcida, usa largura máxima disponível |
| U004 | Centralizar imagem na página | PNG 400x400 | Uint8Array | Imagem posicionada no centro (margem igual) |
| U005 | Gerar PDF com prompt vazio | PNG 640x480 + "" | Uint8Array | PDF gerado sem erros |
| U006 | Gerar PDF com prompt longo | PNG 640x480 + "very long prompt..." | Uint8Array | PDF gerado, rodapé incluído |
| U007 | Validar dimensões A4 | Qualquer imagem | Uint8Array | pageWidth=595.28, pageHeight=841.89 |
| U008 | Validar margens corretas | Qualquer imagem | Uint8Array | Todas as margens = 28.35 pontos (10mm) |
| U009 | Rodapé contém domínio correto | Qualquer imagem | Uint8Array | Texto "paintgpt.com" presente em (y: margin) |
| U010 | Rejeitar buffer nulo | null + "prompt" | Erro/Exception | Lança erro descriptivo |
| U011 | Rejeitar imagem inválida | Buffer corrupto | Erro/Exception | Lança erro de embed PNG |
| U012 | Comprimir imagem para escala de cinza | PNG colorido | Uint8Array | Imagem em preto/branco, compressão nível 9 |

### Testes de Integração - Endpoint de Download

| ID | Descrição | Setup | Request | Resposta Esperada | Validações |
|----|-----------|-------|---------|-------------------|------------|
| I001 | Download imagem pública sem auth | Image {id, isPublic: true, imageUrl} | GET /api/download/[id] | 200 PDF stream | Content-Type: application/pdf, Content-Disposition correto |
| I002 | Download imagem privada com auth | Image {id, isPublic: false, userId: "123"} + auth user 123 | GET /api/download/[id] | 200 PDF stream | Usuario autenticado consegue baixar seu PDF |
| I003 | Rejeitar acesso imagem privada sem auth | Image {id, isPublic: false} | GET /api/download/[id] no auth | 401 Unauthorized | Sem acesso, erro claro |
| I004 | Rejeitar acesso imagem privada outro user | Image {id, isPublic: false, userId: "123"} + auth user 456 | GET /api/download/[id] | 403 Forbidden | Usuário diferente rejeitado |
| I005 | Imagem não encontrada | Image não existe | GET /api/download/nonexistent | 404 Not Found | Mensagem "Not found" |
| I006 | Validar header Content-Type | Qualquer imagem válida | GET /api/download/[id] | 200 | Header "Content-Type: application/pdf" |
| I007 | Validar header Content-Disposition | Image {id: "abc123..."} | GET /api/download/[id] | 200 | Header contém 'filename="coloring-page-abc12345.pdf"' |
| I008 | Buscar do R2 corretamente | Image com imageUrl válida | GET /api/download/[id] | 200 PDF | Chama fetchFromR2 com imageUrl correto |
| I009 | Timeout em geração lenta | Image muito grande | GET /api/download/[id] | 504 Timeout | Retorna erro após 30 segundos |
| I010 | Cache hit R2 | PDF já cacheado em r2://pdfs/[id].pdf | GET /api/download/[id] (2x) | 200 (ambos) | Segundo request mais rápido, do cache |
| I011 | Cache miss e armazenar | Image popular nova | GET /api/download/[id] | 200 PDF | Armazena em r2://pdfs/[id].pdf |
| I012 | Size limit check | PDF gerado > 5MB | GET /api/download/[id] | 413 Payload Too Large | Valida max 5MB |
| I013 | Filename com caracteres especiais | Image {id: "abc-123_xyz"} | GET /api/download/[id] | 200 | Filename sanitizado, apenas alfanuméricos/hífen/underscore |

### Testes E2E - Fluxo Completo de Download

| ID | Descrição | Precondições | Ações | Resultado Esperado | Validações |
|----|-----------|--------------|-------|-------------------|------------|
| E001 | Download completo no navegador | Usuário na página de imagem pública | 1. Clica botão "Download PDF" 2. Aguarda geração | 3. Arquivo baixado com nome correto | Arquivo .pdf existe, tamanho > 0, cabeçalho PDF válido (%PDF) |
| E002 | Loading state durante geração | Usuário clicou download | Spinner visível, botão desabilitado | Após 2-5s desaparece | Loading state gerenciado corretamente |
| E003 | Toast sucesso após download | Download concluído | Aguarda toast render | Toast com mensagem positiva | Mensagem "PDF downloaded successfully" visível por 3s |
| E004 | Toast erro em falha | Imagem não encontrada no R2 | Clica download de imagem corrompida | Toast com erro | Mensagem "Failed to generate PDF" e motivo do erro |
| E005 | Não permite múltiplos downloads simultâneos | Usuário clica download 2x | Primeiro clique ativa loading | Segundo clique ignorado | Botão permanece desabilitado até conclusão |
| E006 | Senha protegida (imagem privada) | Usuário não autenticado na imagem privada | Clica download | Redireciona para login | URL include redirect_to=/image/[id] |
| E007 | Download após login | Usuário faz login | Clica download imagem privada | Arquivo baixado | PDF salvo com sucesso |
| E008 | Filename no download | Browser salva arquivo | Verifica nome do arquivo | "coloring-page-abc12345.pdf" | Sem espaços, extensão .pdf |
| E009 | Múltiplos downloads sequenciais | Usuário em galeria | 1. Download imagem A 2. Download imagem B | Ambos salvos corretamente | Ambos PDF válidos, nomes diferentes |
| E010 | Recuperação de erro de rede | Network failure na fetch R2 | Clica retry no toast | PDF gerado do cache ou nova tentativa | Download bem-sucedido |
| E011 | PDF abrível em visualizador | PDF baixado no dispositivo | Abre em Adobe Reader, Preview (Mac), Chrome | PDF exibe corretamente | Imagem visível, rodapé "paintgpt.com" presente |
| E012 | Compatibilidade cross-browser | Chrome, Firefox, Safari | Clica download em cada browser | Todos downloadam corretamente | Downloads funcionam em todos os navegadores |
| E013 | Mobile responsivo | Dispositivo mobile (iPhone/Android) | Touch download button | Arquivo baixado | Funciona em navegadores mobile |
| E014 | Métrica de download registrada | Usuário faz download | Requisição POST analytics | Analytics atualizado | DB registra download com timestamp e userId |

### Setup de Testes

```typescript
// Mock R2 Storage
jest.mock('@/lib/r2', () => ({
  fetchFromR2: jest.fn((url) => Promise.resolve(validPngBuffer)),
  putToR2: jest.fn((path, buffer) => Promise.resolve()),
  getFromR2: jest.fn((path) => Promise.resolve(null)), // cache miss
}));

// Mock Database
jest.mock('@/lib/db', () => ({
  db: {
    query: {
      images: {
        findFirst: jest.fn((where) => Promise.resolve(mockImage)),
      },
    },
  },
}));

// Mock Auth
jest.mock('@/lib/auth', () => ({
  auth: jest.fn(() => ({ userId: 'test-user-123' })),
}));

// Fixtures
const validPngBuffer = Buffer.from([...PNG_HEADER_BYTES...]);
const mockImage = {
  id: 'img-001',
  imageUrl: 's3://bucket/image-001.png',
  prompt: 'dog drawing',
  isPublic: true,
  userId: 'user-123',
  createdAt: new Date(),
};
```

### Comandos de Teste

```bash
# Rodar todos os testes
bun test

# Apenas unitários
bun test --testPathPattern="pdf.unit"

# Apenas integração
bun test --testPathPattern="pdf.integration"

# Apenas E2E
bun test:e2e --testPathPattern="pdf.e2e"

# Com cobertura
bun test --coverage

# Watch mode
bun test --watch
```
