# APL MVP — Diseño: Circuito de firmado simulado (web)

| Campo | Valor |
|---|---|
| Producto | AutoPen for Lawyers (APL) — MVP |
| Versión diseño | 0.1 |
| Fecha | 2026-06-16 |
| Status | Aprobado para spec → plan |
| Base | PRD técnico APL v0.1 |
| Owner producto | Gabriel Coronel Lavecchia |

---

## 1. Objetivo del MVP

Construir un MVP **web full-stack** que demuestre, de punta a punta, el circuito
**abogado → cliente firmante**, **simulando** el proceso de firma digital.

La integración real con Lakaut (AC: KYC, RENAPER, HSM, emisión de certificados,
firma PAdES) queda **fuera del MVP** y se reemplaza por un proveedor mock. El
firmado lo ejecuta localmente el motor criptográfico reciclado del repo `autopen`
(`P12Signer` + `test.p12`), server-side.

Objetivo de negocio: validar UX y flujo del producto sin dependencia de Lakaut.

---

## 2. Decisiones de arquitectura (aprobadas)

1. **Repo nuevo standalone** en GitHub del owner, historia git limpia. Sin monorepo.
2. **Next.js (App Router) full-stack único**: un solo proyecto, un solo deploy.
   - `/dashboard` → web del abogado (protegida).
   - `/sign/[token]` → PWA pública del firmante (link que se envía).
   - `/api/*` (Route Handlers) → backend.
3. **Persistencia**: SQLite vía Prisma (file-based, cero config; swap a Postgres
   trivial cuando escale). Necesaria porque el firmante abre el link en otro
   dispositivo/navegador que el abogado — localStorage no alcanza.
4. **Límite de simulación**: interfaz `SigningProvider` con dos implementaciones
   intercambiables. MVP usa `MockLakautProvider`; el producto real usará
   `LakautProvider` sin tocar el resto del sistema. Esta interfaz es la mitigación
   directa al riesgo "concentración de Lakaut como single point" (PRD §12).
5. **Notificaciones**: sin email real en v0. El dashboard muestra el link con
   botón "copiar". Resend/SES se agregan después sin cambios de arquitectura.
6. **Auth**: mínima pero real — email+password, un abogado seedeado, sesión en
   cookie httpOnly. Sin MFA en MVP.

---

## 3. Qué se recicla / construye / descarta

### Recicla (del repo `autopen`)
- `packages/shared` cripto → entra como `src/lib/signing/`:
  - `P12Signer` (PKCS#7 sobre node-forge) → motor de firma del mock.
  - `PDFSigner.sign()` (pdf-lib + @signpdf) → firma el PDF server-side.
  - `PDFSigner.verify()` (parser PKCS#7) → reservado para verificación pública (post-MVP).
  - Interfaz `ICryptoProvider` → patrón para el `SigningProvider`.
- Fixture `__fixtures__/test.p12`.
- Conceptos del `apps/mobile`: wizard del firmante, i18n es-AR, patrón Zustand+persist.

### Construye nuevo
- App Next.js (App Router) con las dos caras.
- Route Handlers (backend) + Prisma schema + SQLite.
- `MockLakautProvider` (KYC simulado + firma con P12Signer).
- UI web (los componentes RN-`StyleSheet` del mobile NO se portan; se reescriben en web).

### Descarta
- `viem` y todo el camino Ethereum (`ViemCryptoProvider`, `KeyManager` viem).
- `apps/api` (scaffold), `apps/quark-firmador` (hello world).
- `apps/mobile` queda en el repo viejo como semilla de la v2.0 nativa.

---

## 4. Modelo de datos (subset MVP)

Subconjunto del modelo del PRD §5, recortado a lo que el circuito necesita.

### Incluido
- **User** (abogado): id, email, password_hash, nombre, created_at. Single estudio implícito.
- **Cliente** (firmante): id, nombre_completo, dni, email, celular, created_at.
- **SignRequest**: id, created_by_user_id, cliente_id, document_id, sign_type
  (`ONE_TIME` en MVP), status (ver §5), invite_token (hash, unique), expires_at,
  signed_document_id (nullable), events (JSON timeline), created_at, updated_at.
- **Document**: id, storage_key, sha256, size_bytes, mime_type, kind
  (`ORIGINAL`/`SIGNED`), created_at.
- **Evidence** (básica): sign_request_id, events (JSON), original_pdf_sha256,
  signed_pdf_sha256, created_at.

### Diferido (post-MVP)
- `Estudio` multi-user, `BillingEvent`, `AuditLog` formal. El timeline de eventos
  vive como JSON dentro de `SignRequest`/`Evidence` — suficiente para trazabilidad demo.

### Storage de PDFs
- MVP: filesystem local del server (carpeta fuera de git) referenciado por `storage_key`.
  Interfaz `DocumentStorage` para swap a S3 después.

---

## 5. Máquina de estados de SignRequest (colapsada)

Como el KYC es simulado, se colapsan los estados de Lakaut del PRD §5.2:

```
CREATED → SENT → OPENED → SIGNING → SIGNED
   │        │       │
   ▼        ▼       ▼
EXPIRED  EXPIRED  REJECTED
```

- `CREATED`: abogado creó la solicitud (doc subido, token generado).
- `SENT`: link disponible/"enviado" (en MVP, copiado del dashboard).
- `OPENED`: firmante abrió el link y pasó el precheck de DNI.
- `SIGNING`: KYC simulado + emisión de cert mock en curso.
- `SIGNED`: PDF firmado y disponible para descarga.
- `REJECTED`: firmante rechazó (con motivo).
- `EXPIRED`: venció `expires_at` sin firmar.

Transiciones validadas server-side en el módulo SignRequest.

---

## 6. Flujos

### 6.1 Abogado (`/dashboard`, protegido)
1. Login (email+password).
2. Lista de solicitudes con estado y timeline.
3. "Nueva solicitud": elige/crea cliente (nombre, DNI, email, celular) → sube PDF
   (validación MIME + tamaño + sha256) → define vencimiento (default 7 días).
4. Se genera `invite_token` + link `/sign/[token]`. Botón "copiar link".
5. Tracking del estado en timeline.
6. Descarga del PDF firmado cuando estado = `SIGNED`.

### 6.2 Firmante (`/sign/[token]`, público, mobile-first)
1. Abre link.
2. Bienvenida: identifica al abogado que envía + explicación breve.
3. Precheck anti-link-leak: últimos 4 dígitos del DNI.
4. KYC **simulado**: pantalla con spinner + texto, se saltea (mock).
5. Emisión de cert **simulada** (mock).
6. Visualiza el documento con **scroll obligatorio hasta el final** para habilitar "Firmar".
7. Confirmación con texto explícito de consentimiento (Ley 25.506).
8. Firma simulada: `MockLakautProvider.sign()` → `P12Signer` firma el PDF server-side.
9. Resultado: descarga del PDF firmado + comprobante básico.
10. Rama de rechazo: `/sign/[token]/reject` con motivo → estado `REJECTED`.

---

## 7. Límite de simulación — `SigningProvider`

Interfaz conceptual (la real se define en el plan de implementación):

```
interface SigningProvider {
  prepareSession(signerData): Promise<{ sessionId }>
  startKyc(sessionId): Promise<{ status }>            // mock: resuelve OK
  issueCertificate(sessionId, type): Promise<{ certId }> // mock: cert dummy
  sign(certId, pdfBytes, options): Promise<Buffer>    // mock: P12Signer real
}
```

- `MockLakautProvider`: KYC/cert resuelven OK al instante; `sign` usa `P12Signer`
  con `test.p12`.
- `LakautProvider` (futuro): misma interfaz, llamadas reales + webhooks.

El resto del sistema (SignRequest, dashboard, firmante) depende solo de la
interfaz, nunca de la implementación.

---

## 8. Fuera de scope del MVP (puntos de enganche dejados listos)

Lakaut real · KYC/RENAPER real · PAdES-LTV real · multi-estudio · billing ·
SMS/WhatsApp · página pública de verificación de firma · MFA · integración portal PJN.

---

## 9. Stack propuesto

- Next.js (App Router) + TypeScript.
- Prisma + SQLite.
- Tailwind (ya presente en quark-firmador) para la UI web.
- node-forge / pdf-lib / @signpdf (reciclados) server-side para la firma simulada.
- Zustand para estado de cliente en el wizard del firmante (patrón reciclado).
- zod para validación de inputs (ya instalado en shared, hoy sin uso).

---

## 10. Criterios de éxito del MVP

- Un abogado puede loguearse, crear una solicitud con un PDF real y obtener un link.
- Un firmante, en otro dispositivo, abre el link y completa el flujo simulado.
- El PDF resultante queda firmado (firma técnica real vía P12, simulando Lakaut) y
  descargable por ambas partes.
- El estado de la solicitud refleja correctamente la máquina de estados con su timeline.
- El swap a `LakautProvider` no requiere cambios fuera de `src/lib/signing/`.
