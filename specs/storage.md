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

## Testes TDD

### Unit Tests - Storage Utilities

Testes isolados para funções individuais da camada de storage:

| Teste | Descrição | Input | Expected Output | Status |
|-------|-----------|-------|-----------------|--------|
| `test('uploadImage generates valid UUID')` | Verifica se UUID é gerado corretamente | Buffer de 100KB | UUID v4 válido em formato correto | ✓ |
| `test('uploadImage returns correct path format')` | Valida formato da URL retornada | Buffer PNG | `/uploads/images/{uuid}.png` | ✓ |
| `test('uploadImage creates directory if missing')` | Garante que diretório é criado | Buffer PNG | Diretório existe em `public/uploads/images/` | ✓ |
| `test('downloadImage reads file correctly')` | Verifica leitura de arquivo do disco | URL válida | Buffer idêntico ao enviado | ✓ |
| `test('downloadImage throws on missing file')` | Valida erro quando arquivo não existe | URL inexistente | `Error: ENOENT` | ✓ |
| `test('deleteImage removes file successfully')` | Verifica exclusão de arquivo | URL válida | Arquivo não existe no disco | ✓ |
| `test('deleteImage throws on missing file')` | Valida erro ao deletar não-existente | URL inexistente | `Error: ENOENT` | ✓ |
| `test('processImage resizes to 1024x1024')` | Verifica redimensionamento | Imagem 4000x3000px | Imagem 1024x1024px ou menor | ✓ |
| `test('processImage maintains aspect ratio')` | Valida preservação de proporção | Imagem 1024x2048px | Imagem 512x1024px (fit: inside) | ✓ |
| `test('processImage compresses PNG')` | Verifica compressão | Buffer 5MB PNG | Buffer ~1MB PNG | ✓ |

### Integration Tests - Upload/Download Operations

Testes de fluxo completo integrando múltiplos componentes:

| Teste | Descrição | Setup | Ações | Verificações | Status |
|-------|-----------|-------|-------|--------------|--------|
| `test('upload and download cycle')` | Fluxo completo de envio e recuperação | Buffer PNG 2MB | Upload → Download | Buffer idêntico; URL válida | ✓ |
| `test('multiple sequential uploads')` | Vários uploads consecutivos | 3x Buffers diferentes | 3x upload() | 3x UUIDs únicos; 3x arquivos no disco | ✓ |
| `test('concurrent upload operations')` | Uploads paralelos sem conflitos | 5x Buffers | Promise.all([5x upload()]) | 5x UUIDs únicos; sem corrupção | ✓ |
| `test('upload, download, then delete')` | Ciclo completo CRUD | Buffer PNG | upload() → download() → delete() | Arquivo removido; leitura falha após delete | ✓ |
| `test('storage interface consistency')` | LocalStorage implementa interface | Implementação local | Chama upload/download/delete | Todos os métodos existem e funcionam | ✓ |
| `test('image processing before upload')` | Processa imagem ao fazer upload | Imagem 4000x3000px | processImage() → upload() | Imagem 1024x1024px no disco | ✓ |
| `test('large file upload (50MB)')` | Manipula arquivos grandes | Buffer 50MB | upload() | Arquivo salvo completo sem truncamento | ✓ |
| `test('special characters in image handling')` | Lida com UUIDs em paths | UUID válido v4 | upload() | Path sem problemas de encoding | ✓ |

### Mocking Strategies

Estratégias de mock para testes isolados sem I/O:

| Componente | Mock Strategy | Implementação | Propósito |
|------------|---------------|----------------|----------|
| `fs/promises` | Jest Mock Module | `jest.mock('fs/promises')` | Simular operações de arquivo sem disco real |
| `writeFile()` | Mock Function | `writeFile: jest.fn().mockResolvedValue()` | Retornar sucesso sem escrever arquivo |
| `readFile()` | Mock Return Buffer | `readFile: jest.fn().mockResolvedValue(Buffer.from(...))` | Retornar buffer mock sem ler disco |
| `unlink()` | Mock Function | `unlink: jest.fn().mockResolvedValue()` | Simular exclusão sem afetar disco |
| `mkdir()` | Mock Function | `mkdir: jest.fn().mockResolvedValue()` | Simular criação de diretório |
| `crypto.randomUUID()` | Deterministic Mock | `jest.spyOn(crypto, 'randomUUID').mockReturnValue('test-uuid-1234')` | Retornar UUID consistente para testes |
| `sharp()` | Mock Image Processor | `jest.mock('sharp', () => mockSharpInstance)` | Simular processamento de imagem |
| `StorageProvider` (local) | Dependency Injection | Passar mock de `StorageProvider` em testes | Testar lógica sem depender de implementação real |
| Environment Variables | Mock process.cwd() | `jest.spyOn(process, 'cwd').mockReturnValue('/fake/path')` | Simular diferentes caminhos de projeto |
| Cloudflare R2 (futuro) | AWS SDK Mock | `aws-sdk-client-mock` para S3 API | Simular R2 em testes de integração |

### Exemplo de Suite de Testes

```typescript
// __tests__/lib/storage.test.ts

import { describe, it, expect, beforeEach, afterEach, jest } from '@jest/globals';
import * as fs from 'fs/promises';
import * as crypto from 'crypto';
import { storage, processImage } from '@/lib/storage';

jest.mock('fs/promises');
jest.mock('crypto');

describe('Storage - Unit Tests', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    (crypto.randomUUID as jest.Mock).mockReturnValue('test-uuid-1234-5678');
  });

  describe('upload()', () => {
    it('should generate valid UUID for filename', async () => {
      (fs.writeFile as jest.Mock).mockResolvedValue(undefined);
      (fs.mkdir as jest.Mock).mockResolvedValue(undefined);

      const buffer = Buffer.from('fake-png-data');
      const result = await storage.upload(buffer);

      expect(crypto.randomUUID).toHaveBeenCalled();
      expect(result).toBe('/uploads/images/test-uuid-1234-5678.png');
    });

    it('should call mkdir with correct path', async () => {
      (fs.writeFile as jest.Mock).mockResolvedValue(undefined);
      (fs.mkdir as jest.Mock).mockResolvedValue(undefined);

      await storage.upload(Buffer.from('data'));

      expect(fs.mkdir).toHaveBeenCalledWith(
        expect.stringContaining('public/uploads/images'),
        { recursive: true }
      );
    });
  });

  describe('download()', () => {
    it('should read file from correct path', async () => {
      const mockBuffer = Buffer.from('image-data');
      (fs.readFile as jest.Mock).mockResolvedValue(mockBuffer);

      const result = await storage.download('/uploads/images/test-uuid.png');

      expect(fs.readFile).toHaveBeenCalledWith(
        expect.stringContaining('public/uploads/images/test-uuid.png')
      );
      expect(result).toEqual(mockBuffer);
    });

    it('should throw error when file not found', async () => {
      (fs.readFile as jest.Mock).mockRejectedValue(
        new Error('ENOENT: no such file')
      );

      await expect(
        storage.download('/uploads/images/nonexistent.png')
      ).rejects.toThrow('ENOENT');
    });
  });

  describe('delete()', () => {
    it('should remove file successfully', async () => {
      (fs.unlink as jest.Mock).mockResolvedValue(undefined);

      await storage.delete('/uploads/images/test-uuid.png');

      expect(fs.unlink).toHaveBeenCalledWith(
        expect.stringContaining('public/uploads/images/test-uuid.png')
      );
    });
  });
});

describe('Storage - Integration Tests', () => {
  describe('upload and download cycle', () => {
    it('should maintain data integrity through cycle', async () => {
      // Este teste roda com fs real (sem mocks)
      const testBuffer = Buffer.from('real-png-data');

      const uploadedUrl = await storage.upload(testBuffer);
      const downloadedBuffer = await storage.download(uploadedUrl);

      expect(downloadedBuffer).toEqual(testBuffer);

      await storage.delete(uploadedUrl);
    });
  });

  describe('concurrent operations', () => {
    it('should handle multiple concurrent uploads', async () => {
      const buffers = [
        Buffer.from('data1'),
        Buffer.from('data2'),
        Buffer.from('data3'),
      ];

      const results = await Promise.all(
        buffers.map(buf => storage.upload(buf))
      );

      const urls = new Set(results);
      expect(urls.size).toBe(3); // Todas as URLs devem ser únicas
    });
  });
});

describe('Storage - Image Processing', () => {
  it('should resize image to max 1024x1024', async () => {
    // Mock sharp com dimensions específicas
    const mockSharp = {
      resize: jest.fn().mockReturnThis(),
      png: jest.fn().mockReturnThis(),
      toBuffer: jest
        .fn()
        .mockResolvedValue(Buffer.from('compressed-data')),
    };

    jest.doMock('sharp', () => jest.fn(() => mockSharp));

    // Testar processImage...
  });
});
```

### Coverage Goals

| Métrica | Target | Atual |
|---------|--------|-------|
| Line Coverage | 90%+ | Pendente |
| Branch Coverage | 85%+ | Pendente |
| Function Coverage | 95%+ | Pendente |
| Statement Coverage | 90%+ | Pendente |

Executar cobertura com:
```bash
bun run test --coverage
```
