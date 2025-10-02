# AuraForge Platform – Project Guide

AuraForge es una plataforma de generación creativa que integra flujos frontend en React 18 + Vite y un backend Cloudflare Worker sobre D1. El repositorio contiene tanto la experiencia web como la API multi-servicio que orquesta créditos, suscripciones y pipeline de activos.

## Comandos esenciales

| Propósito | Directorio | Comando |
| --- | --- | --- |
| Instalar dependencias frontend | raíz | `npm install` |
| Instalar dependencias backend | `backend/` | `npm install` |
| Build producción frontend | raíz | `npm run build` |
| Build/validación backend | `backend/` | `npm run build` |

*(No hay suite de tests configurada actualmente.)*

## Arquitectura de alto nivel

### Frontend (React + Vite)
- **Entrada**: `src/main.tsx` monta la SPA en `index.html`; `src/index.css` aplica Tailwind.
- **Shell principal**: `src/App.tsx` gestiona el estudio creativo de imagen Nano Banana (Gemini 2.5 Flash Image), maneja estado de generación/upscaling, registra métricas y persiste galería en `localStorage`.
- **Integración AI**: `yw_manifest.json` define escenas `nano_banana_generator` y `super_resolution_upscaler`. El front invoca `https://api.youware.com/public/v1/ai` directamente (backend no llama AI).
- **Persistencia de activos**: resultados se suben a R2 mediante presign del backend (`/storage/*`).

### Backend (Cloudflare Worker + D1)
- **Entrada**: `backend/src/index.ts` usa `itty-router` para exponer endpoints REST.
- **Autenticación**: extrae `X-Encrypted-Yw-ID` / `X-Is-Login`. `ensureUser` crea sincronamente registro y balance inicial.
- **Orquestación Agentica** (Fase 2):
  - `conversation_sessions`, `conversation_messages`, `job_events` almacenan contexto conversacional.
  - Endpoints `/sessions`, `/sessions/:id/messages`, `/jobs` gestionan planner + ejecutores.
- **Monetización (Fase 3)**:
  - Semillas automáticas de planes (`subscription_plans`) y paquetes (`credit_packages`).
  - `/subscription/*` cubre alta, cancelación, renovación, consulta y administra `user_subscriptions` + `subscription_invoices`.
  - `/credits/orders` y `/credits/orders/:id/complete` manejan compra de créditos (`credit_orders`) y ledger.
  - `/credits/daily-bonus` registra bonus en `daily_bonus_log` y ledger.
  - `/financial/*` ofrece snapshots y métricas agregadas (`financial_snapshots`) sólo para el creador (admin).
- **Ledger Créditos**: `applyCreditDelta` actualiza balance y `credit_ledger` de forma atómica (batch).
- **R2 Storage**: `/storage/presign-upload|download|upload-from-url` proxyean a `https://storage.youware.me` con `STORAGE_API_KEY` opcional.

## Esquema D1 consolidado

- `users`, `user_balances`, `credit_ledger`: identidad y saldo.
- `assets`, `jobs`, `job_events`: orquestación de generación.
- `conversation_sessions`, `conversation_messages`: estado del chatbot agente.
- `daily_bonus_log`: registro de bonus diarios para evitar duplicados.
- **Monetización** (`0003_monetization.sql`):
  - `subscription_plans`, `user_subscriptions`, `subscription_invoices`.
  - `credit_packages`, `credit_orders`.
  - `financial_snapshots` con agregados diarios de revenue, créditos y suscripciones.
- **API / Marketplace / Mobile** (`0004_phase4.sql`):
  - `api_clients`, `api_usage_logs`: llaves Creator Supreme y bitácora por request.
  - `templates`, `template_purchases`, `template_reviews`: marketplace con precios en créditos y reviews únicos.
  - `mobile_devices`, `mobile_sessions`, `notifications`: sesiones con tokens hashados y cola de avisos.

`backend/schema.sql` concatena todas las migraciones en orden para facilitar recreación desde cero.

## Convenciones y consideraciones

- **Administrador**: se valida contra el `encrypted_yw_id` del creador (`ensureAdmin`). Sólo admin opera `/admin/seed-monetization`, endpoints financieros y `/admin/notifications`.
- **Semillas**: `seedPlans` y `seedCreditPackages` se ejecutan al consultar planes/paquetes y poblan datos por defecto si la tabla está vacía.
- **Créditos**: toda mutación usa `applyCreditDelta` para mantener historial consistente y actualizar `user_balances.updated_at`.
- **API externa**: claves comienzan con `af_live_`; se listan/crean vía `/api/clients`, requieren plan Creator Supreme y se consumen con Bearer token.
- **Marketplace**: compradores sólo reseñan si poseen el template; el creador recibe notificación y créditos tras cada venta.
- **Mobile**: sesiones `af_sess_…` requieren encabezado `Authorization: Bearer <token>` y `X-Mobile-Session` para operaciones.
- **Snapshots**: `updateFinancialSnapshot` agrega o crea fila diaria asegurando sumatoria de revenue, churn y actividad.
- **Frontend**: Mantener assets estáticos con rutas absolutas `/assets/...` para builds productivos.
