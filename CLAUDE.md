# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Papermark is an open-source document sharing alternative to DocSend, built with Next.js and featuring document analytics, custom branding, and data rooms. It's a full-stack TypeScript application with both traditional pages router and app router patterns.

## Development Commands

### Essential Commands
- `npm run dev` - Start development server
- `npm run build` - Build for production  
- `npm run lint` - Run ESLint
- `npm run format` - Format code with Prettier
- `npm run dev:prisma` - Generate Prisma client and run migrations
- `npm run email` - Start email preview server on port 3001

### Database Commands
- `npx prisma generate` - Generate Prisma client
- `npx prisma migrate deploy` - Deploy pending migrations
- `npx prisma studio` - Open Prisma Studio

### Trigger.dev Commands
- `npm run trigger:v3:dev` - Start Trigger.dev development
- `npm run trigger:v3:deploy` - Deploy Trigger.dev tasks

### Other Utilities
- `npm run stripe:webhook` - Listen to Stripe webhooks locally
- `pkgx pipenv` - Manage Python dependencies (for Tinybird)

## Architecture Overview

### Routing Strategy
The application uses **both** Next.js routing patterns:
- **App Router** (`/app`) - Modern Next.js 13+ routing for new features
- **Pages Router** (`/pages`) - Legacy routing, still actively used for main application routes

### Key Architectural Patterns

#### Hybrid Pages/App Router Structure
- `/pages` contains main application routes (dashboard, documents, datarooms, etc.)
- `/app` contains authentication routes and some API routes
- Both routers coexist - check both directories when working on routing

#### Database Architecture 
- **Prisma ORM** with PostgreSQL
- Schema split across multiple files in `prisma/schema/`
- Main models: User, Team, Document, Link, View, Dataroom, Viewer
- Uses `prisma/schema/schema.prisma` as the main schema file

#### File Storage Strategy
- Primary: **AWS S3** with CloudFront distribution
- Fallback: **Vercel Blob** storage
- Enum `DocumentStorageType` controls storage backend
- File operations in `lib/files/` directory

### Core Business Logic Locations

#### Document Management
- `/lib/documents/` - Document CRUD operations
- `/lib/files/` - File upload/download/streaming
- `/pages/api/documents/` & `/pages/api/file/` - Document APIs

#### Analytics & Tracking  
- `/lib/analytics/` - Analytics utilities
- `/lib/tinybird/` - Tinybird integration for real-time analytics
- `/lib/tracking/` - View tracking and metrics

#### Authentication & Authorization
- **NextAuth.js** for authentication
- `/lib/auth/` - Custom auth helpers for dataroom access
- **Hanko** integration for passkeys (WebAuthn)

#### Email System
- **Resend** for transactional emails  
- `/lib/emails/` - Email sending functions
- `/components/emails/` - React Email templates

#### Background Jobs
- **Trigger.dev v3** for background processing
- Tasks located in `/lib/trigger/`
- Configuration in `trigger.config.ts`

## Important Implementation Notes

### Trigger.dev Integration
- Uses Trigger.dev v3 SDK (`@trigger.dev/sdk/v3`)
- Task files in `/lib/trigger/` 
- **Always export tasks** and use `task()` function (never `client.defineJob`)
- Prisma extension configured for database access in tasks

### Database Considerations
- Uses connection pooling via `POSTGRES_PRISMA_URL`
- Prisma schema folder structure enabled
- Foreign key constraints actively used - consider cascading deletes

### File Upload Patterns
- **TUS protocol** for resumable uploads
- Both regular uploads (`/pages/api/file/tus/`) and viewer uploads (`/pages/api/file/tus-viewer/`) 
- S3 multipart uploads for large files
- File processing via background jobs

### Security Features
- Content Security Policy configured in `next.config.mjs`
- Domain verification system for custom domains
- Watermarking and screen protection features
- Rate limiting via Upstash Redis

### Frontend Patterns
- **shadcn/ui** components in `/components/ui/`
- **Tailwind CSS** with Tremor charts integration
- **SWR** for client-side data fetching (patterns in `/lib/swr/`)
- Custom hooks in `/lib/hooks/` and `/components/hooks/`

## Testing

The codebase does not include explicit test frameworks. When adding tests, determine the appropriate framework by checking for existing test configurations or ask the user for their preferred testing approach.

## Key Dependencies to Understand

### Core Framework
- **Next.js 14** with both app and pages router
- **TypeScript** with strict configuration
- **Tailwind CSS** for styling

### Database & ORM  
- **Prisma** as ORM with PostgreSQL
- **@upstash/redis** for caching and rate limiting

### Authentication
- **NextAuth.js** for OAuth and credential auth
- **@teamhanko/passkeys-next-auth-provider** for WebAuthn

### File Storage & Processing
- **AWS SDK** for S3 operations
- **@tus/server** for resumable uploads  
- **mupdf** for PDF processing
- **fluent-ffmpeg** for video processing

### Background Processing
- **Trigger.dev v3** for reliable background jobs
- **@upstash/qstash** for cron jobs

### Analytics & Tracking
- **Tinybird** for real-time analytics
- **PostHog** for product analytics
- **@vercel/analytics** for performance tracking

When working on this codebase, always check both `/pages` and `/app` directories for existing implementations, and follow the established patterns for file storage, authentication, and background processing.