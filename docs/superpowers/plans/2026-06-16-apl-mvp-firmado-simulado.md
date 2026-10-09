# APL MVP — Firmado Simulado — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir un MVP web full-stack que demuestre el circuito abogado→firmante simulando la firma digital, con el motor criptográfico reciclado del repo `autopen` haciendo de stand-in de Lakaut.

**Architecture:** Un único proyecto Next.js (App Router) full-stack. Rutas `/dashboard` (abogado, protegido) y `/sign/[token]` (firmante público). Backend en Route Handlers. Persistencia SQLite vía Prisma. La firma la ejecuta un `MockLakautProvider` server-side que envuelve el `P12Signer`/`PDFSigner` reciclados. Todo el sistema depende de la interfaz `SigningProvider`, nunca de la implementación: el swap a Lakaut real no toca nada fuera de `src/lib/signing/`.

**Tech Stack:** Next.js 15 (App Router) · TypeScript · Prisma + SQLite · Tailwind · node-forge / pdf-lib / @signpdf (reciclados) · zod · bcrypt · jose (JWT cookie) · Vitest.

> **Nombre del proyecto:** la carpeta/repo es `legalsign/` (donde el plan dice `apl/`, leer `legalsign/`). Repo creado como sibling de `autopen-gclavecchia` en `/Users/gclavecchia/proyects/`.

## Global Constraints

- Lenguaje de UI: español Argentina (es-AR) como default.
- El firmante abre el link en otro dispositivo/navegador que el abogado → toda persistencia compartida es server-side (SQLite); nunca localStorage para estado compartido.
- `sign_type` en MVP: solo `ONE_TIME`.
- Ningún módulo fuera de `src/lib/signing/` puede importar `node-forge`, `pdf-lib`, `@signpdf` ni conocer al proveedor concreto. Solo dependen de la interfaz `SigningProvider`.
- Prohibido reintroducir `viem` o cualquier camino Ethereum.
- TDD estricto: test que falla → implementación mínima → test que pasa → commit. Un test por comportamiento.
- Commits: conventional commits, sin atribución AI / sin Co-Authored-By.
- Test runner: `vitest run <archivo>`.
- El `invite_token` se guarda hasheado (sha256) en DB; el token plano solo vive en el link.

---

## File Structure

```
apl/                                    # repo nuevo (nombre GitHub a confirmar al pushear)
├── prisma/
│   ├── schema.prisma                   # Task 3
│   └── seed.ts                         # Task 6 (abogado seedeado)
├── src/
│   ├── lib/
│   │   ├── signing/                    # === RECICLADO ===
│   │   │   ├── p12-signer.ts           # Task 1
│   │   │   ├── pdf-signer.ts           # Task 1
│   │   │   ├── signing-provider.ts     # Task 2 (interfaz)
│   │   │   ├── mock-lakaut-provider.ts # Task 2
│   │   │   └── __fixtures__/test.p12    # Task 1
│   │   ├── db.ts                        # Task 3 (cliente Prisma singleton)
│   │   ├── sign-request-state.ts        # Task 4 (máquina de estados pura)
│   │   ├── storage/
│   │   │   ├── document-storage.ts      # Task 5 (interfaz)
│   │   │   └── local-document-storage.ts# Task 5
│   │   ├── auth/
│   │   │   ├── password.ts              # Task 6
│   │   │   ├── session.ts               # Task 6 (JWT cookie)
│   │   │   └── tokens.ts                # Task 7 (invite token: gen + hash)
│   │   └── validation/schemas.ts        # Task 7 (zod)
│   ├── app/
│   │   ├── api/
│   │   │   ├── auth/login/route.ts       # Task 6
│   │   │   ├── sign-requests/route.ts    # Task 7 (POST crear, GET listar)
│   │   │   ├── sign-requests/[id]/route.ts             # Task 8 (GET detalle)
│   │   │   ├── sign-requests/[id]/signed/route.ts      # Task 10 (descarga)
│   │   │   └── public/sign/[token]/
│   │   │       ├── route.ts              # Task 9 (GET open)
│   │   │       ├── precheck/route.ts     # Task 9
│   │   │       ├── confirm/route.ts      # Task 9 (dispara firma simulada)
│   │   │       └── reject/route.ts       # Task 9
│   │   ├── login/page.tsx               # Task 11
│   │   ├── dashboard/page.tsx           # Task 12
│   │   ├── dashboard/new/page.tsx       # Task 13
│   │   └── sign/[token]/page.tsx        # Task 14 (wizard firmante)
│   └── components/                      # Tasks 11-14
└── package.json
```

---

## Task 0: Scaffold del repo nuevo

**Files:**
- Create: `apl/` (proyecto Next.js completo), `apl/.gitignore`, `apl/vitest.config.ts`

**Interfaces:**
- Produces: proyecto Next.js ejecutable con TS, Tailwind, Prisma y Vitest instalados.

- [ ] **Step 1: Crear el proyecto Next.js**

Desde `/Users/gclavecchia/proyects/`:
```bash
npx create-next-app@latest apl --typescript --tailwind --app --src-dir --import-alias "@/*" --no-eslint --use-npm
cd apl
```

- [ ] **Step 2: Instalar dependencias del proyecto**

```bash
npm install prisma @prisma/client zod bcryptjs jose node-forge pdf-lib @signpdf/signpdf @signpdf/signer-p12 @signpdf/placeholder-pdf-lib @signpdf/utils
npm install -D vitest @types/node-forge @types/bcryptjs tsx
npx prisma init --datasource-provider sqlite
```

- [ ] **Step 3: Configurar Vitest**

Create `apl/vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config'
import path from 'node:path'

export default defineConfig({
  test: { environment: 'node', include: ['src/**/*.spec.ts'] },
  resolve: { alias: { '@': path.resolve(__dirname, 'src') } },
})
```

Add to `apl/package.json` scripts: `"test": "vitest run"`.

- [ ] **Step 4: Verificar que arranca**

Run: `npm run dev`
Expected: Next.js levanta en `http://localhost:3000` sin errores.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore: scaffold next.js app with prisma, vitest, tailwind"
```

---

## Task 1: Migrar el motor de firma reciclado

**Files:**
- Create: `src/lib/signing/p12-signer.ts`, `src/lib/signing/pdf-signer.ts`, `src/lib/signing/__fixtures__/test.p12`
- Test: `src/lib/signing/pdf-signer.spec.ts`

**Interfaces:**
- Produces:
  - `class P12Signer` con `constructor(p12Buffer: Buffer, passphrase: string)` y `async sign(content: Buffer, signingTime?: Date): Promise<Buffer>`.
  - `class PDFSigner` con `async sign(pdfBuffer: Buffer, signer: P12Signer, options: SigningOptions): Promise<Buffer>` y `async verify(pdfBuffer: Buffer): Promise<SignatureInfo>`.
  - `type SigningOptions = { reason: string; name: string; location?: string; contactInfo?: string; signingTime?: Date }`.

- [ ] **Step 1: Copiar los fuentes reciclados y el fixture**

```bash
cp ../autopen-gclavecchia/packages/shared/crypto/signers/p12.signer.ts src/lib/signing/p12-signer.ts
cp ../autopen-gclavecchia/packages/shared/signpdf/pdf-signer.ts src/lib/signing/pdf-signer.ts
mkdir -p src/lib/signing/__fixtures__
cp ../autopen-gclavecchia/packages/shared/__fixtures__/test.p12 src/lib/signing/__fixtures__/test.p12
```

- [ ] **Step 2: Ajustar imports y quitar dependencias viem**

Abrí ambos archivos. Eliminá cualquier import o referencia a `viem`, `ViemCryptoProvider`, `KeyManager`, o stores de zustand. `P12Signer` y `PDFSigner` solo deben depender de `node-forge`, `pdf-lib`, `@signpdf/*` y de tipos locales. Mové el tipo `SigningOptions` al tope de `pdf-signer.ts` y exportalo (antes vivía en `crypto/interfaces/signer.interface.ts`).

- [ ] **Step 3: Escribir el test que falla**

Create `src/lib/signing/pdf-signer.spec.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { readFileSync } from 'node:fs'
import path from 'node:path'
import { PDFDocument } from 'pdf-lib'
import { P12Signer } from './p12-signer'
import { PDFSigner } from './pdf-signer'

const P12 = readFileSync(path.join(__dirname, '__fixtures__/test.p12'))

async function blankPdf(): Promise<Buffer> {
  const doc = await PDFDocument.create()
  doc.addPage([595, 842])
  return Buffer.from(await doc.save())
}

describe('PDFSigner', () => {
  it('firma un PDF y verify() reconoce la firma', async () => {
    const pdf = await blankPdf()
    const signer = new P12Signer(P12, 'test') // passphrase del fixture
    const pdfSigner = new PDFSigner()
    const signed = await pdfSigner.sign(pdf, signer, { reason: 'MVP test', name: 'Juan Perez' })
    expect(signed.length).toBeGreaterThan(pdf.length)
    const info = await pdfSigner.verify(signed)
    expect(info.signatureExists).toBe(true)
    expect(info.signerName).toContain('Juan Perez')
  })
})
```

> Nota para el implementador: si el passphrase del `test.p12` no es `'test'`, recuperalo del uso en `../autopen-gclavecchia/apps/mobile` (pantalla Sign) o de los specs originales en `packages/shared/crypto/signers/p12.signer.spec.ts`. Ajustá la constante.

- [ ] **Step 4: Correr el test — debe FALLAR**

Run: `npm run test -- src/lib/signing/pdf-signer.spec.ts`
Expected: FAIL (imports rotos o firma incompleta hasta terminar el Step 2).

- [ ] **Step 5: Ajustar hasta que pase**

Corregí imports/tipos en `p12-signer.ts` y `pdf-signer.ts` hasta que el test pase. No cambies la lógica de firma/verify reciclada; solo desacoplá dependencias.

- [ ] **Step 6: Correr el test — debe PASAR**

Run: `npm run test -- src/lib/signing/pdf-signer.spec.ts`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/lib/signing && git commit -m "feat: recycle P12Signer and PDFSigner engine (drop viem)"
```

---

## Task 2: Interfaz SigningProvider + MockLakautProvider

**Files:**
- Create: `src/lib/signing/signing-provider.ts`, `src/lib/signing/mock-lakaut-provider.ts`
- Test: `src/lib/signing/mock-lakaut-provider.spec.ts`

**Interfaces:**
- Consumes: `P12Signer`, `PDFSigner`, `SigningOptions` (Task 1).
- Produces:
  - `type SignType = 'ONE_TIME' | 'EXTENDED'`
  - `interface SignerData { nombreCompleto: string; dni: string; email: string }`
  - `interface SigningProvider { prepareSession(s: SignerData): Promise<{ sessionId: string }>; startKyc(sessionId: string): Promise<{ status: 'COMPLETED' | 'FAILED' }>; issueCertificate(sessionId: string, type: SignType): Promise<{ certId: string }>; sign(certId: string, pdf: Buffer, options: SigningOptions): Promise<Buffer> }`
  - `class MockLakautProvider implements SigningProvider`
  - `function getSigningProvider(): SigningProvider` (factory; hoy devuelve el mock).

- [ ] **Step 1: Escribir la interfaz**

Create `src/lib/signing/signing-provider.ts`:
```ts
import type { SigningOptions } from './pdf-signer'

export type SignType = 'ONE_TIME' | 'EXTENDED'

export interface SignerData {
  nombreCompleto: string
  dni: string
  email: string
}

export interface SigningProvider {
  prepareSession(signer: SignerData): Promise<{ sessionId: string }>
  startKyc(sessionId: string): Promise<{ status: 'COMPLETED' | 'FAILED' }>
  issueCertificate(sessionId: string, type: SignType): Promise<{ certId: string }>
  sign(certId: string, pdf: Buffer, options: SigningOptions): Promise<Buffer>
}
```

- [ ] **Step 2: Escribir el test que falla**

Create `src/lib/signing/mock-lakaut-provider.spec.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { readFileSync } from 'node:fs'
import path from 'node:path'
import { PDFDocument } from 'pdf-lib'
import { MockLakautProvider } from './mock-lakaut-provider'
import { PDFSigner } from './pdf-signer'

async function blankPdf(): Promise<Buffer> {
  const doc = await PDFDocument.create(); doc.addPage([595, 842])
  return Buffer.from(await doc.save())
}

describe('MockLakautProvider', () => {
  it('simula KYC+cert OK y firma el PDF de verdad', async () => {
    const p = new MockLakautProvider()
    const { sessionId } = await p.prepareSession({ nombreCompleto: 'Ana Diaz', dni: '30111222', email: 'a@b.com' })
    expect((await p.startKyc(sessionId)).status).toBe('COMPLETED')
    const { certId } = await p.issueCertificate(sessionId, 'ONE_TIME')
    const signed = await p.sign(certId, await blankPdf(), { reason: 'firma', name: 'Ana Diaz' })
    const info = await new PDFSigner().verify(signed)
    expect(info.signatureExists).toBe(true)
  })
})
```

- [ ] **Step 3: Run — debe FALLAR**

Run: `npm run test -- src/lib/signing/mock-lakaut-provider.spec.ts`
Expected: FAIL ("Cannot find module './mock-lakaut-provider'").

- [ ] **Step 4: Implementar el mock**

Create `src/lib/signing/mock-lakaut-provider.ts`:
```ts
import { readFileSync } from 'node:fs'
import path from 'node:path'
import { randomUUID } from 'node:crypto'
import { P12Signer } from './p12-signer'
import { PDFSigner } from './pdf-signer'
import type { SigningOptions } from './pdf-signer'
import type { SigningProvider, SignerData, SignType } from './signing-provider'

const P12 = readFileSync(path.join(__dirname, '__fixtures__/test.p12'))
const P12_PASSPHRASE = 'test' // ajustar si el fixture usa otro

export class MockLakautProvider implements SigningProvider {
  async prepareSession(_signer: SignerData) { return { sessionId: randomUUID() } }
  async startKyc(_sessionId: string) { return { status: 'COMPLETED' as const } }
  async issueCertificate(_sessionId: string, _type: SignType) { return { certId: randomUUID() } }
  async sign(_certId: string, pdf: Buffer, options: SigningOptions): Promise<Buffer> {
    const signer = new P12Signer(P12, P12_PASSPHRASE)
    return new PDFSigner().sign(pdf, signer, options)
  }
}

export function getSigningProvider(): SigningProvider {
  return new MockLakautProvider()
}
```

- [ ] **Step 5: Run — debe PASAR**

Run: `npm run test -- src/lib/signing/mock-lakaut-provider.spec.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/lib/signing && git commit -m "feat: add SigningProvider interface and MockLakautProvider"
```

---

## Task 3: Modelo de datos (Prisma + SQLite)

**Files:**
- Create: `prisma/schema.prisma`, `src/lib/db.ts`
- Test: `src/lib/db.spec.ts`

**Interfaces:**
- Produces: modelos `User`, `Cliente`, `SignRequest`, `Document`, `Evidence`; export `prisma` (PrismaClient singleton).

- [ ] **Step 1: Escribir el schema**

Replace `prisma/schema.prisma`:
```prisma
generator client { provider = "prisma-client-js" }
datasource db { provider = "sqlite"; url = "file:./dev.db" }

model User {
  id            String        @id @default(uuid())
  email         String        @unique
  passwordHash  String
  nombre        String
  signRequests  SignRequest[]
  createdAt     DateTime      @default(now())
}

model Cliente {
  id             String        @id @default(uuid())
  nombreCompleto String
  dni            String
  email          String
  celular        String
  signRequests   SignRequest[]
  createdAt      DateTime      @default(now())
}

model Document {
  id          String   @id @default(uuid())
  storageKey  String
  sha256      String
  sizeBytes   Int
  mimeType    String
  kind        String   // ORIGINAL | SIGNED
  createdAt   DateTime @default(now())
}

model SignRequest {
  id               String    @id @default(uuid())
  createdByUserId  String
  createdBy        User      @relation(fields: [createdByUserId], references: [id])
  clienteId        String
  cliente          Cliente   @relation(fields: [clienteId], references: [id])
  documentId       String
  signType         String    @default("ONE_TIME")
  status           String    @default("CREATED")
  inviteTokenHash  String    @unique
  expiresAt        DateTime
  signedDocumentId String?
  rejectReason     String?
  events           String    @default("[]") // JSON timeline
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt
  evidence         Evidence?
}

model Evidence {
  id               String      @id @default(uuid())
  signRequestId    String      @unique
  signRequest      SignRequest @relation(fields: [signRequestId], references: [id])
  events           String      // JSON
  originalPdfSha256 String
  signedPdfSha256   String
  createdAt        DateTime    @default(now())
}
```

- [ ] **Step 2: Crear el cliente singleton**

Create `src/lib/db.ts`:
```ts
import { PrismaClient } from '@prisma/client'
const g = globalThis as unknown as { prisma?: PrismaClient }
export const prisma = g.prisma ?? new PrismaClient()
if (process.env.NODE_ENV !== 'production') g.prisma = prisma
```

- [ ] **Step 3: Migrar**

Run: `npx prisma migrate dev --name init`
Expected: crea `prisma/dev.db` y la migración; genera el cliente.

- [ ] **Step 4: Escribir el test que falla**

Create `src/lib/db.spec.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { prisma } from './db'

describe('db', () => {
  it('crea y lee un User', async () => {
    const email = `t${Date.now()}@x.com`
    const u = await prisma.user.create({ data: { email, passwordHash: 'x', nombre: 'T' } })
    const found = await prisma.user.findUnique({ where: { id: u.id } })
    expect(found?.email).toBe(email)
    await prisma.user.delete({ where: { id: u.id } })
  })
})
```

- [ ] **Step 5: Run — debe PASAR** (el schema ya existe)

Run: `npm run test -- src/lib/db.spec.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add prisma src/lib/db.ts src/lib/db.spec.ts && git commit -m "feat: prisma schema and sqlite data layer"
```

---

## Task 4: Máquina de estados de SignRequest

**Files:**
- Create: `src/lib/sign-request-state.ts`
- Test: `src/lib/sign-request-state.spec.ts`

**Interfaces:**
- Produces:
  - `type SignStatus = 'CREATED'|'SENT'|'OPENED'|'SIGNING'|'SIGNED'|'REJECTED'|'EXPIRED'`
  - `function canTransition(from: SignStatus, to: SignStatus): boolean`
  - `function assertTransition(from: SignStatus, to: SignStatus): void` (lanza `Error` si inválida).

- [ ] **Step 1: Escribir el test que falla**

Create `src/lib/sign-request-state.spec.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { canTransition, assertTransition } from './sign-request-state'

describe('sign-request-state', () => {
  it('permite el happy path', () => {
    expect(canTransition('CREATED', 'SENT')).toBe(true)
    expect(canTransition('SENT', 'OPENED')).toBe(true)
    expect(canTransition('OPENED', 'SIGNING')).toBe(true)
    expect(canTransition('SIGNING', 'SIGNED')).toBe(true)
  })
  it('permite rechazo y expiración', () => {
    expect(canTransition('OPENED', 'REJECTED')).toBe(true)
    expect(canTransition('SENT', 'EXPIRED')).toBe(true)
  })
  it('rechaza saltos inválidos', () => {
    expect(canTransition('CREATED', 'SIGNED')).toBe(false)
    expect(canTransition('SIGNED', 'OPENED')).toBe(false)
    expect(() => assertTransition('SIGNED', 'OPENED')).toThrow()
  })
})
```

- [ ] **Step 2: Run — debe FALLAR**

Run: `npm run test -- src/lib/sign-request-state.spec.ts`
Expected: FAIL ("Cannot find module").

- [ ] **Step 3: Implementar**

Create `src/lib/sign-request-state.ts`:
```ts
export type SignStatus =
  | 'CREATED' | 'SENT' | 'OPENED' | 'SIGNING' | 'SIGNED' | 'REJECTED' | 'EXPIRED'

const ALLOWED: Record<SignStatus, SignStatus[]> = {
  CREATED: ['SENT', 'EXPIRED'],
  SENT: ['OPENED', 'EXPIRED'],
  OPENED: ['SIGNING', 'REJECTED', 'EXPIRED'],
  SIGNING: ['SIGNED', 'REJECTED'],
  SIGNED: [],
  REJECTED: [],
  EXPIRED: [],
}

export function canTransition(from: SignStatus, to: SignStatus): boolean {
  return ALLOWED[from]?.includes(to) ?? false
}

export function assertTransition(from: SignStatus, to: SignStatus): void {
  if (!canTransition(from, to)) {
    throw new Error(`Transición inválida: ${from} → ${to}`)
  }
}
```

- [ ] **Step 4: Run — debe PASAR**

Run: `npm run test -- src/lib/sign-request-state.spec.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/sign-request-state.* && git commit -m "feat: sign request state machine"
```

---

## Task 5: Storage de documentos (filesystem local)

**Files:**
- Create: `src/lib/storage/document-storage.ts`, `src/lib/storage/local-document-storage.ts`
- Test: `src/lib/storage/local-document-storage.spec.ts`

**Interfaces:**
- Produces:
  - `interface DocumentStorage { put(key: string, data: Buffer): Promise<void>; get(key: string): Promise<Buffer> }`
  - `class LocalDocumentStorage implements DocumentStorage` (raíz: `process.env.STORAGE_DIR ?? '.storage'`)
  - `function getStorage(): DocumentStorage`

- [ ] **Step 1: Escribir el test que falla**

Create `src/lib/storage/local-document-storage.spec.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { LocalDocumentStorage } from './local-document-storage'

describe('LocalDocumentStorage', () => {
  it('persiste y recupera bytes', async () => {
    const s = new LocalDocumentStorage('.storage-test')
    const key = `k/${Date.now()}.pdf`
    await s.put(key, Buffer.from('hola'))
    expect((await s.get(key)).toString()).toBe('hola')
  })
})
```

- [ ] **Step 2: Run — debe FALLAR**

Run: `npm run test -- src/lib/storage/local-document-storage.spec.ts`
Expected: FAIL ("Cannot find module").

- [ ] **Step 3: Implementar interfaz + impl local**

Create `src/lib/storage/document-storage.ts`:
```ts
export interface DocumentStorage {
  put(key: string, data: Buffer): Promise<void>
  get(key: string): Promise<Buffer>
}
```

Create `src/lib/storage/local-document-storage.ts`:
```ts
import { mkdir, writeFile, readFile } from 'node:fs/promises'
import path from 'node:path'
import type { DocumentStorage } from './document-storage'

export class LocalDocumentStorage implements DocumentStorage {
  constructor(private root = process.env.STORAGE_DIR ?? '.storage') {}
  private full(key: string) { return path.join(this.root, key) }
  async put(key: string, data: Buffer) {
    const f = this.full(key)
    await mkdir(path.dirname(f), { recursive: true })
    await writeFile(f, data)
  }
  async get(key: string) { return readFile(this.full(key)) }
}

export function getStorage(): DocumentStorage { return new LocalDocumentStorage() }
```

Add `.storage*` to `apl/.gitignore`.

- [ ] **Step 4: Run — debe PASAR**

Run: `npm run test -- src/lib/storage/local-document-storage.spec.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/storage .gitignore && git commit -m "feat: local document storage with swappable interface"
```

---

## Task 6: Auth del abogado (password + sesión + seed)

**Files:**
- Create: `src/lib/auth/password.ts`, `src/lib/auth/session.ts`, `prisma/seed.ts`, `src/app/api/auth/login/route.ts`
- Test: `src/lib/auth/password.spec.ts`, `src/lib/auth/session.spec.ts`

**Interfaces:**
- Produces:
  - `password.ts`: `hashPassword(p: string): Promise<string>`, `verifyPassword(p, hash): Promise<boolean>`
  - `session.ts`: `createSessionCookie(userId: string): Promise<string>` (JWT firmado con `AUTH_SECRET`), `readSession(token: string): Promise<{ userId: string } | null>`, const `SESSION_COOKIE = 'apl_session'`
  - Route `POST /api/auth/login` → set-cookie httpOnly + `{ ok: true }` o 401.

- [ ] **Step 1: Test de password que falla**

Create `src/lib/auth/password.spec.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { hashPassword, verifyPassword } from './password'

describe('password', () => {
  it('hashea y verifica', async () => {
    const h = await hashPassword('secreto123')
    expect(await verifyPassword('secreto123', h)).toBe(true)
    expect(await verifyPassword('mal', h)).toBe(false)
  })
})
```

- [ ] **Step 2: Run — FALLA**, luego implementar

Create `src/lib/auth/password.ts`:
```ts
import bcrypt from 'bcryptjs'
export function hashPassword(p: string) { return bcrypt.hash(p, 10) }
export function verifyPassword(p: string, hash: string) { return bcrypt.compare(p, hash) }
```
Run: `npm run test -- src/lib/auth/password.spec.ts` → PASS.

- [ ] **Step 3: Test de sesión que falla**

Create `src/lib/auth/session.spec.ts`:
```ts
import { describe, it, expect, beforeAll } from 'vitest'
import { createSessionCookie, readSession } from './session'

beforeAll(() => { process.env.AUTH_SECRET = 'test-secret-test-secret-test-secret' })

describe('session', () => {
  it('round-trip del JWT', async () => {
    const token = await createSessionCookie('user-1')
    expect((await readSession(token))?.userId).toBe('user-1')
    expect(await readSession('basura')).toBeNull()
  })
})
```

- [ ] **Step 4: Run — FALLA**, luego implementar

Create `src/lib/auth/session.ts`:
```ts
import { SignJWT, jwtVerify } from 'jose'

export const SESSION_COOKIE = 'apl_session'
const secret = () => new TextEncoder().encode(process.env.AUTH_SECRET!)

export async function createSessionCookie(userId: string): Promise<string> {
  return new SignJWT({ userId })
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('7d')
    .sign(secret())
}

export async function readSession(token: string): Promise<{ userId: string } | null> {
  try {
    const { payload } = await jwtVerify(token, secret())
    return { userId: payload.userId as string }
  } catch { return null }
}
```
Run: `npm run test -- src/lib/auth/session.spec.ts` → PASS.

- [ ] **Step 5: Seed del abogado**

Create `prisma/seed.ts`:
```ts
import { PrismaClient } from '@prisma/client'
import { hashPassword } from '../src/lib/auth/password'
const prisma = new PrismaClient()
async function main() {
  const email = 'abogado@apl.test'
  await prisma.user.upsert({
    where: { email },
    update: {},
    create: { email, nombre: 'Dr. Demo', passwordHash: await hashPassword('demo1234') },
  })
  console.log('Seeded:', email, '/ demo1234')
}
main().finally(() => prisma.$disconnect())
```
Add to `package.json`: `"prisma": { "seed": "tsx prisma/seed.ts" }`.
Run: `npx prisma db seed`
Expected: imprime el usuario seedeado.

- [ ] **Step 6: Route de login**

Create `src/app/api/auth/login/route.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { verifyPassword } from '@/lib/auth/password'
import { createSessionCookie, SESSION_COOKIE } from '@/lib/auth/session'

export async function POST(req: NextRequest) {
  const { email, password } = await req.json()
  const user = await prisma.user.findUnique({ where: { email } })
  if (!user || !(await verifyPassword(password, user.passwordHash))) {
    return NextResponse.json({ error: 'Credenciales inválidas' }, { status: 401 })
  }
  const token = await createSessionCookie(user.id)
  const res = NextResponse.json({ ok: true })
  res.cookies.set(SESSION_COOKIE, token, { httpOnly: true, sameSite: 'lax', path: '/' })
  return res
}
```

- [ ] **Step 7: Commit**

```bash
git add src/lib/auth prisma/seed.ts src/app/api/auth package.json && git commit -m "feat: lawyer auth (bcrypt + jwt cookie) and seed"
```

---

## Task 7: Crear y listar solicitudes (POST/GET /api/sign-requests)

**Files:**
- Create: `src/lib/auth/tokens.ts`, `src/lib/validation/schemas.ts`, `src/lib/auth/require-user.ts`, `src/app/api/sign-requests/route.ts`
- Test: `src/lib/auth/tokens.spec.ts`, `src/app/api/sign-requests/route.spec.ts`

**Interfaces:**
- Consumes: `prisma` (T3), `assertTransition`/`SignStatus` (T4), `getStorage` (T5), `readSession`/`SESSION_COOKIE` (T6).
- Produces:
  - `tokens.ts`: `generateInviteToken(): { token: string; hash: string }`, `hashInviteToken(token: string): string`
  - `require-user.ts`: `requireUser(req: NextRequest): Promise<{ userId: string } | null>`
  - `POST /api/sign-requests` (multipart: `clienteId?`, datos de cliente nuevo, `file` PDF, `expiresInDays?`) → `{ id, token, link }`
  - `GET /api/sign-requests` → lista del abogado autenticado.

- [ ] **Step 1: Test de tokens que falla**

Create `src/lib/auth/tokens.spec.ts`:
```ts
import { describe, it, expect } from 'vitest'
import { generateInviteToken, hashInviteToken } from './tokens'

describe('invite tokens', () => {
  it('hash determinístico y token random', () => {
    const a = generateInviteToken(); const b = generateInviteToken()
    expect(a.token).not.toBe(b.token)
    expect(hashInviteToken(a.token)).toBe(a.hash)
    expect(a.hash).not.toBe(a.token)
  })
})
```

- [ ] **Step 2: Run — FALLA**, luego implementar

Create `src/lib/auth/tokens.ts`:
```ts
import { randomBytes, createHash } from 'node:crypto'
export function hashInviteToken(token: string): string {
  return createHash('sha256').update(token).digest('hex')
}
export function generateInviteToken() {
  const token = randomBytes(24).toString('base64url')
  return { token, hash: hashInviteToken(token) }
}
```
Run: `npm run test -- src/lib/auth/tokens.spec.ts` → PASS.

- [ ] **Step 3: Schemas zod + helper de auth**

Create `src/lib/validation/schemas.ts`:
```ts
import { z } from 'zod'
export const nuevoClienteSchema = z.object({
  nombreCompleto: z.string().min(1),
  dni: z.string().min(6),
  email: z.string().email(),
  celular: z.string().min(6),
})
export const MAX_PDF_BYTES = 25 * 1024 * 1024
```

Create `src/lib/auth/require-user.ts`:
```ts
import { NextRequest } from 'next/server'
import { readSession, SESSION_COOKIE } from './session'
export async function requireUser(req: NextRequest) {
  const token = req.cookies.get(SESSION_COOKIE)?.value
  if (!token) return null
  return readSession(token)
}
```

- [ ] **Step 4: Route de creación + listado**

Create `src/app/api/sign-requests/route.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { createHash } from 'node:crypto'
import { prisma } from '@/lib/db'
import { requireUser } from '@/lib/auth/require-user'
import { generateInviteToken } from '@/lib/auth/tokens'
import { getStorage } from '@/lib/storage/local-document-storage'
import { nuevoClienteSchema, MAX_PDF_BYTES } from '@/lib/validation/schemas'

export async function POST(req: NextRequest) {
  const session = await requireUser(req)
  if (!session) return NextResponse.json({ error: 'No autorizado' }, { status: 401 })

  const form = await req.formData()
  const file = form.get('file') as File | null
  if (!file || file.type !== 'application/pdf') {
    return NextResponse.json({ error: 'PDF requerido' }, { status: 400 })
  }
  const bytes = Buffer.from(await file.arrayBuffer())
  if (bytes.length > MAX_PDF_BYTES) {
    return NextResponse.json({ error: 'PDF supera 25MB' }, { status: 400 })
  }

  let clienteId = form.get('clienteId') as string | null
  if (!clienteId) {
    const parsed = nuevoClienteSchema.parse({
      nombreCompleto: form.get('nombreCompleto'), dni: form.get('dni'),
      email: form.get('email'), celular: form.get('celular'),
    })
    const cliente = await prisma.cliente.create({ data: parsed })
    clienteId = cliente.id
  }

  const sha256 = createHash('sha256').update(bytes).digest('hex')
  const now = new Date()
  const storageKey = `${session.userId}/${now.getFullYear()}/${now.getMonth() + 1}/${sha256}.pdf`
  await getStorage().put(storageKey, bytes)
  const doc = await prisma.document.create({
    data: { storageKey, sha256, sizeBytes: bytes.length, mimeType: 'application/pdf', kind: 'ORIGINAL' },
  })

  const days = Number(form.get('expiresInDays') ?? 7)
  const expiresAt = new Date(now.getTime() + days * 86400000)
  const { token, hash } = generateInviteToken()
  const sr = await prisma.signRequest.create({
    data: {
      createdByUserId: session.userId, clienteId, documentId: doc.id,
      status: 'SENT', inviteTokenHash: hash, expiresAt,
      events: JSON.stringify([{ at: now.toISOString(), event: 'CREATED' }, { at: now.toISOString(), event: 'SENT' }]),
    },
  })
  const link = `${req.nextUrl.origin}/sign/${token}`
  return NextResponse.json({ id: sr.id, token, link })
}

export async function GET(req: NextRequest) {
  const session = await requireUser(req)
  if (!session) return NextResponse.json({ error: 'No autorizado' }, { status: 401 })
  const list = await prisma.signRequest.findMany({
    where: { createdByUserId: session.userId },
    include: { cliente: true },
    orderBy: { createdAt: 'desc' },
  })
  return NextResponse.json(list)
}
```

- [ ] **Step 5: Test de integración de la route**

Create `src/app/api/sign-requests/route.spec.ts`:
```ts
import { describe, it, expect, beforeAll } from 'vitest'
import { NextRequest } from 'next/server'
import { PDFDocument } from 'pdf-lib'
import { prisma } from '@/lib/db'
import { hashPassword } from '@/lib/auth/password'
import { createSessionCookie, SESSION_COOKIE } from '@/lib/auth/session'
import { POST } from './route'

beforeAll(() => { process.env.AUTH_SECRET = 'test-secret-test-secret-test-secret' })

async function pdf(): Promise<Buffer> {
  const d = await PDFDocument.create(); d.addPage(); return Buffer.from(await d.save())
}

describe('POST /api/sign-requests', () => {
  it('crea solicitud con cliente nuevo y devuelve link', async () => {
    const user = await prisma.user.create({
      data: { email: `l${Date.now()}@x.com`, nombre: 'L', passwordHash: await hashPassword('x') },
    })
    const token = await createSessionCookie(user.id)
    const form = new FormData()
    form.set('file', new File([await pdf()], 'd.pdf', { type: 'application/pdf' }))
    form.set('nombreCompleto', 'Ana Diaz'); form.set('dni', '30111222')
    form.set('email', 'ana@x.com'); form.set('celular', '1133334444')
    const req = new NextRequest('http://localhost/api/sign-requests', { method: 'POST', body: form })
    req.cookies.set(SESSION_COOKIE, token)
    const res = await POST(req)
    expect(res.status).toBe(200)
    const json = await res.json()
    expect(json.link).toContain('/sign/')
  })
})
```

- [ ] **Step 6: Run — debe PASAR**

Run: `npm run test -- src/app/api/sign-requests/route.spec.ts`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/lib/auth/tokens.ts src/lib/auth/require-user.ts src/lib/validation src/app/api/sign-requests && git commit -m "feat: create and list sign requests"
```

---

## Task 8: Detalle de solicitud (GET /api/sign-requests/[id])

**Files:**
- Create: `src/app/api/sign-requests/[id]/route.ts`
- Test: `src/app/api/sign-requests/[id]/route.spec.ts`

**Interfaces:**
- Consumes: `prisma`, `requireUser`.
- Produces: `GET /api/sign-requests/:id` → solicitud con `cliente` y `events` parseados; 404 si no pertenece al abogado.

- [ ] **Step 1: Test que falla** — crea user+cliente+document+signRequest vía prisma, llama `GET` con cookie válida, espera 200 y `events` como array. Con cookie ajena espera 404.

```ts
// src/app/api/sign-requests/[id]/route.spec.ts — patrón idéntico al Task 7 spec para setup.
// Aserción clave:
//   const res = await GET(req, { params: Promise.resolve({ id: sr.id }) })
//   expect(res.status).toBe(200)
//   expect(Array.isArray((await res.json()).events)).toBe(true)
```

- [ ] **Step 2: Run — FALLA**

Run: `npm run test -- src/app/api/sign-requests/[id]/route.spec.ts`
Expected: FAIL ("Cannot find module './route'").

- [ ] **Step 3: Implementar**

Create `src/app/api/sign-requests/[id]/route.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { requireUser } from '@/lib/auth/require-user'

export async function GET(req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const session = await requireUser(req)
  if (!session) return NextResponse.json({ error: 'No autorizado' }, { status: 401 })
  const { id } = await params
  const sr = await prisma.signRequest.findFirst({
    where: { id, createdByUserId: session.userId },
    include: { cliente: true },
  })
  if (!sr) return NextResponse.json({ error: 'No encontrado' }, { status: 404 })
  return NextResponse.json({ ...sr, events: JSON.parse(sr.events) })
}
```

- [ ] **Step 4: Run — PASA.** Run: `npm run test -- src/app/api/sign-requests/[id]/route.spec.ts`
- [ ] **Step 5: Commit** `git commit -m "feat: sign request detail endpoint"`

---

## Task 9: Endpoints públicos del firmante (open / precheck / confirm / reject)

**Files:**
- Create: `src/lib/sign-requests/service.ts`, `src/app/api/public/sign/[token]/route.ts`, `.../precheck/route.ts`, `.../confirm/route.ts`, `.../reject/route.ts`
- Test: `src/lib/sign-requests/service.spec.ts`

**Interfaces:**
- Consumes: `prisma`, `hashInviteToken` (T7), `assertTransition` (T4), `getStorage` (T5), `getSigningProvider` (T2), `PDFSigner` (T1).
- Produces (en `service.ts`, lógica testeable sin HTTP):
  - `findByToken(token: string)` → signRequest+cliente+document o null (chequea expiración → marca EXPIRED).
  - `precheck(token: string, last4: string)` → bool; si OK transiciona SENT→OPENED.
  - `confirmSign(token: string)` → corre `prepareSession→startKyc→issueCertificate→sign`, persiste PDF firmado (`Document` kind SIGNED), crea `Evidence`, transiciona OPENED→SIGNING→SIGNED. Devuelve `{ signedDocumentId }`.
  - `reject(token: string, motivo: string)` → OPENED→REJECTED.
- Las routes son wrappers finos sobre `service.ts`.

- [ ] **Step 1: Test del service (happy path completo) que falla**

Create `src/lib/sign-requests/service.spec.ts`:
```ts
import { describe, it, expect, beforeAll } from 'vitest'
import { PDFDocument } from 'pdf-lib'
import { prisma } from '@/lib/db'
import { getStorage } from '@/lib/storage/local-document-storage'
import { generateInviteToken } from '@/lib/auth/tokens'
import { hashPassword } from '@/lib/auth/password'
import { findByToken, precheck, confirmSign } from './service'

beforeAll(() => { process.env.AUTH_SECRET = 'test-secret-test-secret-test-secret' })

async function seedRequest() {
  const user = await prisma.user.create({ data: { email: `l${Date.now()}@x.com`, nombre: 'L', passwordHash: await hashPassword('x') } })
  const cliente = await prisma.cliente.create({ data: { nombreCompleto: 'Ana Diaz', dni: '30111222', email: 'a@x.com', celular: '11' } })
  const d = await PDFDocument.create(); d.addPage()
  const bytes = Buffer.from(await d.save())
  const storageKey = `t/${Date.now()}.pdf`
  await getStorage().put(storageKey, bytes)
  const doc = await prisma.document.create({ data: { storageKey, sha256: 'x', sizeBytes: bytes.length, mimeType: 'application/pdf', kind: 'ORIGINAL' } })
  const { token, hash } = generateInviteToken()
  await prisma.signRequest.create({ data: { createdByUserId: user.id, clienteId: cliente.id, documentId: doc.id, status: 'SENT', inviteTokenHash: hash, expiresAt: new Date(Date.now() + 86400000), events: '[]' } })
  return { token, dni: '30111222' }
}

describe('sign-requests service', () => {
  it('open → precheck → confirm firma el documento', async () => {
    const { token, dni } = await seedRequest()
    expect(await findByToken(token)).not.toBeNull()
    expect(await precheck(token, dni.slice(-4))).toBe(true)
    const { signedDocumentId } = await confirmSign(token)
    expect(signedDocumentId).toBeTruthy()
    const sr = await prisma.signRequest.findFirst({ where: { signedDocumentId } })
    expect(sr?.status).toBe('SIGNED')
  })
})
```

- [ ] **Step 2: Run — FALLA**

Run: `npm run test -- src/lib/sign-requests/service.spec.ts`
Expected: FAIL ("Cannot find module './service'").

- [ ] **Step 3: Implementar el service**

Create `src/lib/sign-requests/service.ts`:
```ts
import { createHash } from 'node:crypto'
import { prisma } from '@/lib/db'
import { hashInviteToken } from '@/lib/auth/tokens'
import { assertTransition, type SignStatus } from '@/lib/sign-request-state'
import { getStorage } from '@/lib/storage/local-document-storage'
import { getSigningProvider } from '@/lib/signing/mock-lakaut-provider'

function appendEvent(events: string, event: string) {
  const arr = JSON.parse(events) as Array<{ at: string; event: string }>
  arr.push({ at: new Date().toISOString(), event })
  return JSON.stringify(arr)
}

async function loadByToken(token: string) {
  const hash = hashInviteToken(token)
  return prisma.signRequest.findUnique({
    where: { inviteTokenHash: hash },
    include: { cliente: true },
  })
}

export async function findByToken(token: string) {
  const sr = await loadByToken(token)
  if (!sr) return null
  if (!['SIGNED', 'REJECTED', 'EXPIRED'].includes(sr.status) && sr.expiresAt < new Date()) {
    await prisma.signRequest.update({ where: { id: sr.id }, data: { status: 'EXPIRED' } })
    return { ...sr, status: 'EXPIRED' }
  }
  return sr
}

export async function precheck(token: string, last4: string): Promise<boolean> {
  const sr = await loadByToken(token)
  if (!sr) return false
  if (sr.cliente.dni.slice(-4) !== last4) return false
  if (sr.status === 'SENT') {
    assertTransition(sr.status as SignStatus, 'OPENED')
    await prisma.signRequest.update({
      where: { id: sr.id },
      data: { status: 'OPENED', events: appendEvent(sr.events, 'OPENED') },
    })
  }
  return true
}

export async function confirmSign(token: string): Promise<{ signedDocumentId: string }> {
  const sr = await loadByToken(token)
  if (!sr) throw new Error('Solicitud no encontrada')
  assertTransition(sr.status as SignStatus, 'SIGNING')
  await prisma.signRequest.update({ where: { id: sr.id }, data: { status: 'SIGNING', events: appendEvent(sr.events, 'SIGNING') } })

  const original = await prisma.document.findUniqueOrThrow({ where: { id: sr.documentId } })
  const pdf = await getStorage().get(original.storageKey)

  const provider = getSigningProvider()
  const { sessionId } = await provider.prepareSession({ nombreCompleto: sr.cliente.nombreCompleto, dni: sr.cliente.dni, email: sr.cliente.email })
  await provider.startKyc(sessionId)
  const { certId } = await provider.issueCertificate(sessionId, 'ONE_TIME')
  const signed = await provider.sign(certId, pdf, { reason: 'Firma digital APL (simulada)', name: sr.cliente.nombreCompleto })

  const signedSha = createHash('sha256').update(signed).digest('hex')
  const signedKey = original.storageKey.replace(/\.pdf$/, '.signed.pdf')
  await getStorage().put(signedKey, signed)
  const signedDoc = await prisma.document.create({
    data: { storageKey: signedKey, sha256: signedSha, sizeBytes: signed.length, mimeType: 'application/pdf', kind: 'SIGNED' },
  })

  const events = appendEvent(sr.events, 'SIGNED')
  await prisma.signRequest.update({ where: { id: sr.id }, data: { status: 'SIGNED', signedDocumentId: signedDoc.id, events } })
  await prisma.evidence.create({
    data: { signRequestId: sr.id, events, originalPdfSha256: original.sha256, signedPdfSha256: signedSha },
  })
  return { signedDocumentId: signedDoc.id }
}

export async function reject(token: string, motivo: string): Promise<void> {
  const sr = await loadByToken(token)
  if (!sr) throw new Error('Solicitud no encontrada')
  assertTransition(sr.status as SignStatus, 'REJECTED')
  await prisma.signRequest.update({
    where: { id: sr.id },
    data: { status: 'REJECTED', rejectReason: motivo, events: appendEvent(sr.events, 'REJECTED') },
  })
}
```

- [ ] **Step 4: Run — debe PASAR**

Run: `npm run test -- src/lib/sign-requests/service.spec.ts`
Expected: PASS.

- [ ] **Step 5: Escribir las 4 routes (wrappers finos)**

Create `src/app/api/public/sign/[token]/route.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { findByToken } from '@/lib/sign-requests/service'

export async function GET(_req: NextRequest, { params }: { params: Promise<{ token: string }> }) {
  const { token } = await params
  const sr = await findByToken(token)
  if (!sr) return NextResponse.json({ error: 'Link inválido' }, { status: 404 })
  return NextResponse.json({
    status: sr.status,
    abogado: undefined, // se completa en T12 si se quiere mostrar nombre
    clienteNombre: sr.cliente.nombreCompleto,
    expiresAt: sr.expiresAt,
  })
}
```

Create `src/app/api/public/sign/[token]/precheck/route.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { precheck } from '@/lib/sign-requests/service'

export async function POST(req: NextRequest, { params }: { params: Promise<{ token: string }> }) {
  const { token } = await params
  const { last4 } = await req.json()
  const ok = await precheck(token, String(last4 ?? ''))
  return NextResponse.json({ ok }, { status: ok ? 200 : 403 })
}
```

Create `src/app/api/public/sign/[token]/confirm/route.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { confirmSign } from '@/lib/sign-requests/service'

export async function POST(_req: NextRequest, { params }: { params: Promise<{ token: string }> }) {
  const { token } = await params
  try {
    const { signedDocumentId } = await confirmSign(token)
    return NextResponse.json({ ok: true, signedDocumentId })
  } catch (e) {
    return NextResponse.json({ error: (e as Error).message }, { status: 409 })
  }
}
```

Create `src/app/api/public/sign/[token]/reject/route.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { reject } from '@/lib/sign-requests/service'

export async function POST(req: NextRequest, { params }: { params: Promise<{ token: string }> }) {
  const { token } = await params
  const { motivo } = await req.json()
  await reject(token, String(motivo ?? 'Sin motivo'))
  return NextResponse.json({ ok: true })
}
```

- [ ] **Step 6: Commit**

```bash
git add src/lib/sign-requests src/app/api/public && git commit -m "feat: public signer endpoints with simulated signing"
```

---

## Task 10: Descarga del PDF firmado

**Files:**
- Create: `src/app/api/sign-requests/[id]/signed/route.ts`
- Test: cubierto vía `service.spec.ts` (firma) + verificación manual; agregar test de la route opcional.

**Interfaces:**
- Consumes: `prisma`, `requireUser`, `getStorage`.
- Produces: `GET /api/sign-requests/:id/signed` → stream del PDF firmado con `Content-Type: application/pdf`; 404 si no es del abogado o no está firmado.

- [ ] **Step 1: Implementar la route**

Create `src/app/api/sign-requests/[id]/signed/route.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/db'
import { requireUser } from '@/lib/auth/require-user'
import { getStorage } from '@/lib/storage/local-document-storage'

export async function GET(req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const session = await requireUser(req)
  if (!session) return NextResponse.json({ error: 'No autorizado' }, { status: 401 })
  const { id } = await params
  const sr = await prisma.signRequest.findFirst({ where: { id, createdByUserId: session.userId } })
  if (!sr?.signedDocumentId) return NextResponse.json({ error: 'No firmado' }, { status: 404 })
  const doc = await prisma.document.findUniqueOrThrow({ where: { id: sr.signedDocumentId } })
  const bytes = await getStorage().get(doc.storageKey)
  return new NextResponse(bytes, {
    headers: { 'Content-Type': 'application/pdf', 'Content-Disposition': `attachment; filename="firmado-${id}.pdf"` },
  })
}
```

- [ ] **Step 2: Verificación manual**

Run: `npm run dev`, crear y firmar una solicitud (vía UI de tasks siguientes o curl), luego `GET /api/sign-requests/<id>/signed`.
Expected: descarga un PDF válido.

- [ ] **Step 3: Commit** `git commit -m "feat: signed pdf download endpoint"`

---

## Task 11: UI Login

**Files:**
- Create: `src/app/login/page.tsx`

**Interfaces:**
- Consumes: `POST /api/auth/login`.
- Produces: form de login que ante éxito redirige a `/dashboard`.

- [ ] **Step 1: Implementar la página**

Create `src/app/login/page.tsx`:
```tsx
'use client'
import { useState } from 'react'
import { useRouter } from 'next/navigation'

export default function LoginPage() {
  const router = useRouter()
  const [email, setEmail] = useState('abogado@apl.test')
  const [password, setPassword] = useState('demo1234')
  const [error, setError] = useState('')
  async function submit(e: React.FormEvent) {
    e.preventDefault(); setError('')
    const res = await fetch('/api/auth/login', {
      method: 'POST', headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    })
    if (res.ok) router.push('/dashboard')
    else setError('Credenciales inválidas')
  }
  return (
    <main className="min-h-screen flex items-center justify-center bg-slate-50">
      <form onSubmit={submit} className="bg-white p-8 rounded-xl shadow w-full max-w-sm space-y-4">
        <h1 className="text-xl font-semibold">AutoPen for Lawyers</h1>
        <input className="w-full border rounded px-3 py-2" value={email} onChange={e => setEmail(e.target.value)} placeholder="Email" />
        <input className="w-full border rounded px-3 py-2" type="password" value={password} onChange={e => setPassword(e.target.value)} placeholder="Contraseña" />
        {error && <p className="text-red-600 text-sm">{error}</p>}
        <button className="w-full bg-slate-900 text-white rounded py-2">Ingresar</button>
      </form>
    </main>
  )
}
```

- [ ] **Step 2: Verificación manual** — `npm run dev`, ir a `/login`, ingresar con el seed → redirige a `/dashboard`.
- [ ] **Step 3: Commit** `git commit -m "feat: login page"`

---

## Task 12: UI Dashboard (lista + timeline)

**Files:**
- Create: `src/app/dashboard/page.tsx`, `src/components/StatusBadge.tsx`

**Interfaces:**
- Consumes: `GET /api/sign-requests`.
- Produces: tabla de solicitudes con cliente, estado (badge), fecha y acción "Descargar firmado" cuando `SIGNED`; botón "Nueva solicitud" → `/dashboard/new`.

- [ ] **Step 1: Componente de badge**

Create `src/components/StatusBadge.tsx`:
```tsx
const COLORS: Record<string, string> = {
  CREATED: 'bg-gray-200 text-gray-800', SENT: 'bg-blue-100 text-blue-800',
  OPENED: 'bg-indigo-100 text-indigo-800', SIGNING: 'bg-amber-100 text-amber-800',
  SIGNED: 'bg-green-100 text-green-800', REJECTED: 'bg-red-100 text-red-800',
  EXPIRED: 'bg-gray-300 text-gray-700',
}
export function StatusBadge({ status }: { status: string }) {
  return <span className={`px-2 py-0.5 rounded text-xs font-medium ${COLORS[status] ?? ''}`}>{status}</span>
}
```

- [ ] **Step 2: Página dashboard (server component que fetchea)**

Create `src/app/dashboard/page.tsx`:
```tsx
import Link from 'next/link'
import { cookies } from 'next/headers'
import { prisma } from '@/lib/db'
import { readSession, SESSION_COOKIE } from '@/lib/auth/session'
import { redirect } from 'next/navigation'
import { StatusBadge } from '@/components/StatusBadge'

export default async function Dashboard() {
  const token = (await cookies()).get(SESSION_COOKIE)?.value
  const session = token ? await readSession(token) : null
  if (!session) redirect('/login')
  const list = await prisma.signRequest.findMany({
    where: { createdByUserId: session.userId }, include: { cliente: true }, orderBy: { createdAt: 'desc' },
  })
  return (
    <main className="max-w-4xl mx-auto p-8">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-2xl font-semibold">Solicitudes de firma</h1>
        <Link href="/dashboard/new" className="bg-slate-900 text-white px-4 py-2 rounded">Nueva solicitud</Link>
      </div>
      <table className="w-full text-sm">
        <thead><tr className="text-left text-gray-500 border-b">
          <th className="py-2">Cliente</th><th>Estado</th><th>Vence</th><th></th>
        </tr></thead>
        <tbody>
          {list.map(sr => (
            <tr key={sr.id} className="border-b">
              <td className="py-3">{sr.cliente.nombreCompleto}</td>
              <td><StatusBadge status={sr.status} /></td>
              <td>{new Date(sr.expiresAt).toLocaleDateString('es-AR')}</td>
              <td className="text-right">
                {sr.status === 'SIGNED' && (
                  <a className="text-blue-600" href={`/api/sign-requests/${sr.id}/signed`}>Descargar firmado</a>
                )}
              </td>
            </tr>
          ))}
          {list.length === 0 && <tr><td colSpan={4} className="py-6 text-gray-400">Sin solicitudes todavía.</td></tr>}
        </tbody>
      </table>
    </main>
  )
}
```

- [ ] **Step 3: Verificación manual** — login → `/dashboard` renderiza (vacío al principio).
- [ ] **Step 4: Commit** `git commit -m "feat: dashboard list with status and download"`

---

## Task 13: UI Nueva solicitud (form + copiar link)

**Files:**
- Create: `src/app/dashboard/new/page.tsx`

**Interfaces:**
- Consumes: `POST /api/sign-requests` (multipart).
- Produces: form con datos de cliente nuevo + upload PDF + vencimiento; al crear muestra el link con botón "Copiar".

- [ ] **Step 1: Implementar la página**

Create `src/app/dashboard/new/page.tsx`:
```tsx
'use client'
import { useState } from 'react'

export default function NuevaSolicitud() {
  const [link, setLink] = useState('')
  const [error, setError] = useState('')
  async function submit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault(); setError('')
    const res = await fetch('/api/sign-requests', { method: 'POST', body: new FormData(e.currentTarget) })
    const json = await res.json()
    if (res.ok) setLink(json.link); else setError(json.error ?? 'Error')
  }
  if (link) return (
    <main className="max-w-lg mx-auto p-8 space-y-4">
      <h1 className="text-xl font-semibold">Solicitud creada</h1>
      <p className="text-sm text-gray-600">Compartí este link con tu cliente:</p>
      <div className="flex gap-2">
        <input readOnly className="flex-1 border rounded px-3 py-2 text-sm" value={link} />
        <button onClick={() => navigator.clipboard.writeText(link)} className="bg-slate-900 text-white px-3 rounded">Copiar</button>
      </div>
      <a href="/dashboard" className="text-blue-600 text-sm">← Volver al dashboard</a>
    </main>
  )
  return (
    <main className="max-w-lg mx-auto p-8">
      <h1 className="text-xl font-semibold mb-4">Nueva solicitud de firma</h1>
      <form onSubmit={submit} className="space-y-3">
        <input name="nombreCompleto" placeholder="Nombre completo del cliente" className="w-full border rounded px-3 py-2" required />
        <input name="dni" placeholder="DNI" className="w-full border rounded px-3 py-2" required />
        <input name="email" type="email" placeholder="Email" className="w-full border rounded px-3 py-2" required />
        <input name="celular" placeholder="Celular" className="w-full border rounded px-3 py-2" required />
        <input name="file" type="file" accept="application/pdf" className="w-full" required />
        <input name="expiresInDays" type="number" defaultValue={7} className="w-full border rounded px-3 py-2" />
        {error && <p className="text-red-600 text-sm">{error}</p>}
        <button className="w-full bg-slate-900 text-white rounded py-2">Crear y generar link</button>
      </form>
    </main>
  )
}
```

- [ ] **Step 2: Verificación manual** — crear una solicitud con un PDF real → muestra link copiable; aparece en el dashboard.
- [ ] **Step 3: Commit** `git commit -m "feat: new sign request form with copyable link"`

---

## Task 14: UI Wizard del firmante (`/sign/[token]`)

**Files:**
- Create: `src/app/sign/[token]/page.tsx`, `src/components/SignerWizard.tsx`

**Interfaces:**
- Consumes: `GET /api/public/sign/[token]`, `.../precheck`, `.../confirm`, `.../reject`.
- Produces: wizard lineal mobile-first: bienvenida → precheck (últimos 4 DNI) → KYC simulado (spinner) → ver documento con scroll-para-habilitar → consentimiento + confirmar → resultado. Rama de rechazo.

- [ ] **Step 1: Page que carga el estado inicial (server)**

Create `src/app/sign/[token]/page.tsx`:
```tsx
import { findByToken } from '@/lib/sign-requests/service'
import { SignerWizard } from '@/components/SignerWizard'

export default async function SignPage({ params }: { params: Promise<{ token: string }> }) {
  const { token } = await params
  const sr = await findByToken(token)
  if (!sr) return <main className="p-8 text-center">Link inválido o vencido.</main>
  if (sr.status === 'SIGNED') return <main className="p-8 text-center">Este documento ya fue firmado. ✅</main>
  if (sr.status === 'REJECTED') return <main className="p-8 text-center">Esta solicitud fue rechazada.</main>
  if (sr.status === 'EXPIRED') return <main className="p-8 text-center">El link venció.</main>
  return <SignerWizard token={token} clienteNombre={sr.cliente.nombreCompleto} />
}
```

- [ ] **Step 2: Wizard client component**

Create `src/components/SignerWizard.tsx`:
```tsx
'use client'
import { useState, useRef } from 'react'

type Step = 'welcome' | 'precheck' | 'kyc' | 'review' | 'done' | 'rejected'

export function SignerWizard({ token, clienteNombre }: { token: string; clienteNombre: string }) {
  const [step, setStep] = useState<Step>('welcome')
  const [last4, setLast4] = useState('')
  const [error, setError] = useState('')
  const [scrolledEnd, setScrolledEnd] = useState(false)
  const docRef = useRef<HTMLDivElement>(null)

  async function doPrecheck() {
    setError('')
    const res = await fetch(`/api/public/sign/${token}/precheck`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ last4 }),
    })
    if (res.ok) { setStep('kyc'); setTimeout(() => setStep('review'), 2500) } // KYC simulado
    else setError('Los últimos 4 dígitos no coinciden.')
  }
  async function doConfirm() {
    const res = await fetch(`/api/public/sign/${token}/confirm`, { method: 'POST' })
    if (res.ok) setStep('done'); else setError('No se pudo firmar. Intentá de nuevo.')
  }
  async function doReject() {
    await fetch(`/api/public/sign/${token}/reject`, {
      method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ motivo: 'Rechazado por el firmante' }),
    })
    setStep('rejected')
  }
  function onScroll() {
    const el = docRef.current; if (!el) return
    if (el.scrollHeight - el.scrollTop - el.clientHeight < 24) setScrolledEnd(true)
  }

  return (
    <main className="max-w-md mx-auto p-6 min-h-screen flex flex-col">
      {step === 'welcome' && (
        <div className="space-y-4 my-auto">
          <h1 className="text-2xl font-semibold">Hola 👋</h1>
          <p>Tu abogado te envió un documento para firmar digitalmente. El proceso lleva pocos minutos.</p>
          <button onClick={() => setStep('precheck')} className="w-full bg-slate-900 text-white rounded py-3">Comenzar</button>
        </div>
      )}
      {step === 'precheck' && (
        <div className="space-y-4 my-auto">
          <h2 className="text-lg font-semibold">Verificá tu identidad</h2>
          <p className="text-sm text-gray-600">Ingresá los últimos 4 dígitos de tu DNI.</p>
          <input value={last4} onChange={e => setLast4(e.target.value)} maxLength={4} inputMode="numeric"
            className="w-full border rounded px-3 py-3 text-center text-2xl tracking-widest" />
          {error && <p className="text-red-600 text-sm">{error}</p>}
          <button onClick={doPrecheck} className="w-full bg-slate-900 text-white rounded py-3">Continuar</button>
        </div>
      )}
      {step === 'kyc' && (
        <div className="space-y-4 my-auto text-center">
          <div className="animate-spin h-10 w-10 border-4 border-slate-300 border-t-slate-900 rounded-full mx-auto" />
          <p>Validando tu identidad y emitiendo tu certificado…</p>
        </div>
      )}
      {step === 'review' && (
        <div className="space-y-4 flex flex-col flex-1">
          <h2 className="text-lg font-semibold">Leé el documento</h2>
          <div ref={docRef} onScroll={onScroll} className="flex-1 overflow-y-auto border rounded p-4 bg-gray-50 text-sm">
            <p className="text-gray-500 mb-4">[Vista previa del documento de {clienteNombre}]</p>
            {Array.from({ length: 40 }).map((_, i) => <p key={i} className="mb-2">Línea {i + 1} del escrito a firmar…</p>)}
            <p className="font-semibold mt-4">— Fin del documento —</p>
          </div>
          <label className="text-xs text-gray-600 block">
            Declaro firmar digitalmente este documento con plenos efectos legales conforme Ley 25.506.
          </label>
          {error && <p className="text-red-600 text-sm">{error}</p>}
          <div className="flex gap-2">
            <button onClick={doReject} className="flex-1 border rounded py-3">Rechazar</button>
            <button onClick={doConfirm} disabled={!scrolledEnd}
              className="flex-1 bg-slate-900 text-white rounded py-3 disabled:opacity-40">
              {scrolledEnd ? 'Firmar' : 'Scrolleá hasta el final'}
            </button>
          </div>
        </div>
      )}
      {step === 'done' && (
        <div className="space-y-4 my-auto text-center">
          <p className="text-4xl">✅</p>
          <h2 className="text-xl font-semibold">¡Documento firmado!</h2>
          <p className="text-sm text-gray-600">Tu abogado ya puede descargar el documento firmado.</p>
        </div>
      )}
      {step === 'rejected' && (
        <div className="my-auto text-center"><h2 className="text-lg font-semibold">Rechazaste la firma.</h2></div>
      )}
    </main>
  )
}
```

- [ ] **Step 3: Verificación manual end-to-end**

Run: `npm run dev`. En un navegador: login → crear solicitud → copiar link. En otro navegador (o incógnito): abrir el link → completar wizard (DNI últimos 4 correctos) → firmar. Volver al dashboard → estado `SIGNED` → descargar PDF firmado y abrirlo.
Expected: PDF firmado válido; estados correctos en ambas puntas.

- [ ] **Step 4: Commit** `git commit -m "feat: signer wizard with scroll-to-sign and simulated kyc"`

---

## Task 15: Cierre — middleware de protección + manifest PWA

**Files:**
- Create: `src/middleware.ts`, `public/manifest.webmanifest`
- Modify: `src/app/layout.tsx` (link al manifest, lang es-AR)

**Interfaces:**
- Produces: redirección a `/login` para rutas `/dashboard*` sin sesión; PWA instalable para el firmante.

- [ ] **Step 1: Middleware de protección**

Create `src/middleware.ts`:
```ts
import { NextRequest, NextResponse } from 'next/server'
import { SESSION_COOKIE } from '@/lib/auth/session'

export function middleware(req: NextRequest) {
  const hasCookie = req.cookies.has(SESSION_COOKIE)
  if (!hasCookie) return NextResponse.redirect(new URL('/login', req.url))
  return NextResponse.next()
}
export const config = { matcher: ['/dashboard/:path*'] }
```

- [ ] **Step 2: Manifest PWA**

Create `public/manifest.webmanifest`:
```json
{
  "name": "AutoPen for Lawyers",
  "short_name": "APL",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#f8fafc",
  "theme_color": "#0f172a",
  "icons": []
}
```

Modify `src/app/layout.tsx`: `<html lang="es-AR">` y agregar `<link rel="manifest" href="/manifest.webmanifest" />`.

- [ ] **Step 3: Verificación** — abrir `/dashboard` sin sesión redirige a `/login`; el firmante puede "Agregar a inicio".
- [ ] **Step 4: Commit** `git commit -m "feat: dashboard auth middleware and pwa manifest"`

---

## Task 16: Test E2E del circuito completo (Playwright)

**Files:**
- Create: `playwright.config.ts`, `e2e/circuito.spec.ts`, `e2e/fixtures/test.pdf`
- Modify: `package.json` (script `test:e2e`)

**Interfaces:**
- Consumes: la app completa corriendo (`npm run dev`), seed del abogado (Task 6).
- Produces: un test E2E que recorre el happy-path abogado→firmante→descarga.

- [ ] **Step 1: Instalar Playwright**

```bash
cd legalsign
npm install -D @playwright/test
npx playwright install chromium
```

- [ ] **Step 2: Config de Playwright**

Create `legalsign/playwright.config.ts`:
```ts
import { defineConfig } from '@playwright/test'
export default defineConfig({
  testDir: './e2e',
  use: { baseURL: 'http://localhost:3000' },
  webServer: { command: 'npm run dev', url: 'http://localhost:3000', reuseExistingServer: true, timeout: 120_000 },
})
```
Add to `package.json` scripts: `"test:e2e": "playwright test"`.

- [ ] **Step 3: Preparar fixture PDF y seed**

Generar `e2e/fixtures/test.pdf` (cualquier PDF válido pequeño; se puede crear con un script node usando pdf-lib o copiar uno existente). Asegurar `npx prisma migrate deploy` + `npx prisma db seed` corridos antes del test.

- [ ] **Step 4: Escribir el test E2E (happy path)**

Create `legalsign/e2e/circuito.spec.ts`:
```ts
import { test, expect } from '@playwright/test'
import path from 'node:path'

test('circuito completo abogado → firmante → descarga', async ({ page, context }) => {
  // 1. Abogado: login
  await page.goto('/login')
  await page.fill('input[placeholder="Email"]', 'abogado@apl.test')
  await page.fill('input[placeholder="Contraseña"]', 'demo1234')
  await page.click('button:has-text("Ingresar")')
  await expect(page).toHaveURL(/dashboard/)

  // 2. Abogado: crea solicitud
  await page.goto('/dashboard/new')
  await page.fill('input[name="nombreCompleto"]', 'Ana Diaz')
  await page.fill('input[name="dni"]', '30111222')
  await page.fill('input[name="email"]', 'ana@x.com')
  await page.fill('input[name="celular"]', '1133334444')
  await page.setInputFiles('input[name="file"]', path.join(__dirname, 'fixtures/test.pdf'))
  await page.click('button:has-text("Crear y generar link")')
  const link = await page.locator('input[readonly]').inputValue()
  expect(link).toContain('/sign/')

  // 3. Firmante: abre el link en una pestaña nueva (sin sesión de abogado)
  const signer = await context.browser()!.newContext()
  const sp = await signer.newPage()
  await sp.goto(link)
  await sp.click('button:has-text("Comenzar")')
  await sp.fill('input[inputmode="numeric"]', '1222')
  await sp.click('button:has-text("Continuar")')
  // KYC simulado → review (esperar a que aparezca el documento)
  await expect(sp.locator('text=Fin del documento')).toBeVisible({ timeout: 10_000 })
  await sp.locator('div[class*="overflow-y-auto"]').evaluate(el => el.scrollTo(0, el.scrollHeight))
  await sp.click('button:has-text("Firmar")')
  await expect(sp.locator('text=Documento firmado')).toBeVisible({ timeout: 15_000 })

  // 4. Abogado: el estado pasa a SIGNED y puede descargar
  await page.reload()
  await expect(page.locator('text=SIGNED')).toBeVisible()
})
```

- [ ] **Step 5: Correr el E2E**

Run: `npm run test:e2e`
Expected: PASS (1 test verde).

> Si algún selector no matchea por diferencias en el markup final de Tasks 11–14, ajustar los selectores del test a lo realmente renderizado (no cambiar la UI para que matchee un selector arbitrario).

- [ ] **Step 6: Commit**

```bash
git add playwright.config.ts e2e package.json && git commit -m "test: e2e happy path for full signing circuit"
```

---

## Self-Review (cobertura del spec)

| Requisito del spec | Task |
|---|---|
| Repo nuevo standalone, sin monorepo | Task 0 |
| Next.js full-stack, /dashboard + /sign/[token] | Tasks 0, 11–14 |
| Reciclar P12Signer/PDFSigner, descartar viem | Task 1 |
| Interfaz SigningProvider + MockLakautProvider | Task 2 |
| SQLite + Prisma + modelo (User/Cliente/SignRequest/Document/Evidence) | Task 3 |
| Máquina de estados colapsada | Task 4 |
| DocumentStorage swappable (local) | Task 5 |
| Auth mínima real (email+password, cookie httpOnly, seed) | Task 6 |
| Crear solicitud (PDF, token hasheado, vencimiento) | Task 7 |
| Listar/detalle + timeline | Tasks 7, 8 |
| Flujo firmante (precheck, KYC simulado, scroll-to-sign, consentimiento) | Tasks 9, 14 |
| Firma simulada con P12 server-side + Evidence | Task 9 |
| Descarga del PDF firmado | Task 10 |
| Rechazo con motivo | Tasks 9, 14 |
| Expiración | Task 9 (`findByToken`) |
| Sin email real (link copiable) | Task 13 |
| es-AR, PWA | Tasks 14, 15 |

**Fuera de scope (confirmado, no en este plan):** Lakaut real, KYC/RENAPER real, PAdES-LTV real, multi-estudio, billing, SMS/WhatsApp, verificación pública, MFA, portal PJN.

**Notas de consistencia de tipos:** `SignStatus` (T4) se reutiliza en T9 vía cast desde `string` (Prisma guarda string). `SigningProvider`/`SignerData`/`SignType` (T2) consumidos en T9. `getStorage` (T5) usado en T7/T9/T10. `hashInviteToken` (T7) usado en T9. `SESSION_COOKIE`/`readSession` (T6) usados en T7/T8/T10/T12/T15.
