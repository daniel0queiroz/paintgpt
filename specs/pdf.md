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
