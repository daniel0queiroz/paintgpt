# Especificação: Storage (Local)

## Visão Geral

Armazenamento local de imagens para desenvolvimento. Migração para R2 será feita antes do deploy em produção.

## Estrutura de Diretórios

```
paintgpt/
└── public/
    └── uploads/
        └── images/
            └── {uuid}.png
```

## Por que Local?

- **Simplicidade:** Sem configuração de cloud
- **Velocidade:** Desenvolvimento rápido
- **Custo:** Zero durante desenvolvimento
- **Debug:** Fácil inspecionar arquivos

## Nomenclatura de Arquivos

```
/uploads/images/{uuid}.png
```

- UUID v4 para unicidade
- Extensão sempre `.png`
- Servido estaticamente via Next.js

## Upload de Imagens

### Implementação

```typescript
import { writeFile, mkdir } from 'fs/promises';
import { join } from 'path';
import { randomUUID } from 'crypto';

const UPLOAD_DIR = join(process.cwd(), 'public', 'uploads', 'images');

async function uploadImage(imageBuffer: Buffer): Promise<string> {
  // Garantir que diretório existe
  await mkdir(UPLOAD_DIR, { recursive: true });

  const imageId = randomUUID();
  const filename = `${imageId}.png`;
  const filepath = join(UPLOAD_DIR, filename);

  await writeFile(filepath, imageBuffer);

  // URL pública (servida pelo Next.js)
  return `/uploads/images/${filename}`;
}
```

## Download de Imagens

### Para PDF Generation

```typescript
import { readFile } from 'fs/promises';
import { join } from 'path';

async function downloadImage(imageUrl: string): Promise<Buffer> {
  // imageUrl = "/uploads/images/{uuid}.png"
  const filepath = join(process.cwd(), 'public', imageUrl);
  return readFile(filepath);
}
```

## Deleção de Imagens

```typescript
import { unlink } from 'fs/promises';
import { join } from 'path';

async function deleteImage(imageUrl: string): Promise<void> {
  const filepath = join(process.cwd(), 'public', imageUrl);
  await unlink(filepath);
}
```

## Abstração para Migração Futura

Criar interface para facilitar troca para R2:

```typescript
// lib/storage.ts

export interface StorageProvider {
  upload(buffer: Buffer): Promise<string>;
  download(url: string): Promise<Buffer>;
  delete(url: string): Promise<void>;
}

// Implementação local
export const localStorage: StorageProvider = {
  async upload(buffer: Buffer): Promise<string> {
    const imageId = randomUUID();
    const filename = `${imageId}.png`;
    const filepath = join(UPLOAD_DIR, filename);

    await mkdir(UPLOAD_DIR, { recursive: true });
    await writeFile(filepath, buffer);

    return `/uploads/images/${filename}`;
  },

  async download(url: string): Promise<Buffer> {
    const filepath = join(process.cwd(), 'public', url);
    return readFile(filepath);
  },

  async delete(url: string): Promise<void> {
    const filepath = join(process.cwd(), 'public', url);
    await unlink(filepath);
  },
};

// Exportar provider ativo
export const storage = localStorage;
```

### Uso

```typescript
import { storage } from '@/lib/storage';

// Upload
const url = await storage.upload(imageBuffer);

// Download
const buffer = await storage.download(url);

// Delete
await storage.delete(url);
```

## Processamento de Imagem

```typescript
import sharp from 'sharp';

async function processImage(rawBuffer: Buffer): Promise<Buffer> {
  return sharp(rawBuffer)
    .resize(1024, 1024, {
      fit: 'inside',
      withoutEnlargement: true,
    })
    .png({
      compressionLevel: 9,
    })
    .toBuffer();
}
```

## .gitignore

Adicionar uploads ao gitignore:

```gitignore
# Uploaded images (local dev)
public/uploads/
```

## Variáveis de Ambiente

```env
# Nenhuma necessária para storage local
# Quando migrar para R2:
# R2_ENDPOINT=
# R2_ACCESS_KEY_ID=
# R2_SECRET_ACCESS_KEY=
# R2_BUCKET_NAME=
# R2_PUBLIC_URL=
```

## Limitações (Dev Only)

- Arquivos perdidos se limpar `public/uploads`
- Não escala para produção
- Sem CDN/cache
- Sem backup

## Checklist para Produção

Antes de deploy, migrar para R2:

- [ ] Criar bucket no Cloudflare R2
- [ ] Configurar variáveis de ambiente
- [ ] Implementar `r2Storage: StorageProvider`
- [ ] Trocar export: `export const storage = r2Storage`
- [ ] Migrar imagens existentes (se houver)
- [ ] Testar upload/download/delete
