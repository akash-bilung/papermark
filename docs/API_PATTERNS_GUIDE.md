# Production-Ready API Patterns & Best Practices Guide

## Overview

This comprehensive guide outlines proven API patterns and best practices based on the production-ready implementation of Papermark. It covers everything from API architecture and authentication to error handling, validation, rate limiting, and webhook integrations.

---

## Table of Contents

1. [API Architecture Overview](#api-architecture-overview)
2. [Authentication & Authorization Patterns](#authentication--authorization-patterns)
3. [Request/Response Patterns](#requestresponse-patterns)
4. [Error Handling & Validation](#error-handling--validation)
5. [Rate Limiting & Security](#rate-limiting--security)
6. [File Upload & Processing](#file-upload--processing)
7. [Database Patterns](#database-patterns)
8. [Webhook Implementation](#webhook-implementation)
9. [API Versioning & Documentation](#api-versioning--documentation)
10. [Testing Strategies](#testing-strategies)
11. [Performance Optimization](#performance-optimization)
12. [Monitoring & Logging](#monitoring--logging)
13. [Implementation Examples](#implementation-examples)

---

## API Architecture Overview

### RESTful API Structure

Papermark follows a consistent RESTful API pattern with nested resources:

```
/api/teams/{teamId}/documents/{documentId}
/api/teams/{teamId}/datarooms/{dataroomId}/documents
/api/teams/{teamId}/invitations/accept
/api/file/s3/get-presigned-post-url
/api/stripe/webhook
```

### File Organization Pattern

```
pages/api/
├── auth/
│   └── [...nextauth].ts          # Authentication endpoints
├── teams/
│   └── [teamId]/
│       ├── documents/
│       │   ├── index.ts          # GET /teams/:id/documents, POST /teams/:id/documents
│       │   └── [id]/
│       │       └── index.ts      # GET/PUT/DELETE /teams/:id/documents/:id
│       ├── datarooms/
│       └── invitations/
├── file/
│   ├── s3/
│   └── tus/
├── webhooks/
└── health.ts                     # Health check endpoint
```

### HTTP Method Conventions

```typescript
// GET - Retrieve resources
GET    /api/teams/:teamId/documents           // List documents
GET    /api/teams/:teamId/documents/:id       // Get specific document

// POST - Create resources
POST   /api/teams/:teamId/documents           // Create document
POST   /api/teams/:teamId/invitations         // Create invitation

// PUT - Update/replace resources
PUT    /api/teams/:teamId/documents/:id       // Update document

// DELETE - Remove resources
DELETE /api/teams/:teamId/documents/:id       // Delete document
DELETE /api/teams/:teamId/invitations/:token  // Cancel invitation

// PATCH - Partial updates
PATCH  /api/teams/:teamId/documents/:id       // Partial document update
```

---

## Authentication & Authorization Patterns

### Multi-Layer Authentication

```typescript
// pages/api/teams/[teamId]/documents/index.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // 1. Check for API token first
  const authHeader = req.headers.authorization;
  let userId: string;
  let token: string | null = null;

  if (authHeader?.startsWith("Bearer ")) {
    token = authHeader.replace("Bearer ", "");
    const hashedToken = hashToken(token);

    // Look up token in database
    const restrictedToken = await prisma.restrictedToken.findUnique({
      where: { hashedKey: hashedToken },
      select: { userId: true, teamId: true },
    });

    if (!restrictedToken) {
      return res.status(401).json({ error: "Unauthorized" });
    }

    if (restrictedToken.teamId !== teamId) {
      return res.status(401).json({ error: "Unauthorized" });
    }

    userId = restrictedToken.userId;
  } else {
    // 2. Fall back to session auth
    const session = await getServerSession(req, res, authOptions);
    if (!session) {
      return res.status(401).json({ error: "Unauthorized" });
    }
    userId = (session.user as CustomUser).id;
  }

  // 3. Verify team membership
  const team = await prisma.team.findUnique({
    where: {
      id: teamId,
      users: {
        some: { userId: userId },
      },
    },
  });

  if (!team) {
    return res.status(403).json({ error: "Unauthorized to access this team" });
  }

  // Continue with authorized request...
}
```

### Token-Based Authentication

```typescript
// lib/api/auth/token.ts
import crypto from 'crypto';

export function hashToken(token: string): string {
  return crypto.createHash('sha256').update(token).digest('hex');
}

export function generateApiToken(): string {
  return 'ppmk_' + crypto.randomBytes(32).toString('hex');
}

// Database schema for tokens
interface RestrictedToken {
  id: string;
  name: string;
  hashedKey: string;        // Hashed version of the token
  partialKey: string;       // First 8 chars for display
  scopes: string;           // Comma-separated permissions
  expires?: Date;
  lastUsed?: Date;
  rateLimit: number;        // Requests per minute
  userId: string;
  teamId: string;
}
```

### Authorization Middleware Pattern

```typescript
// lib/middleware/auth.ts
export interface AuthContext {
  user: { id: string; email: string };
  team: { id: string; role: string };
  token?: { scopes: string[] };
}

export async function withAuth<T>(
  handler: (req: NextApiRequest, res: NextApiResponse, ctx: AuthContext) => Promise<T>
) {
  return async (req: NextApiRequest, res: NextApiResponse) => {
    try {
      const context = await getAuthContext(req, res);
      return await handler(req, res, context);
    } catch (error) {
      if (error instanceof AuthError) {
        return res.status(error.statusCode).json({ error: error.message });
      }
      throw error;
    }
  };
}

// Usage
export default withAuth(async (req, res, { user, team }) => {
  // Handler with guaranteed auth context
});
```

---

## Request/Response Patterns

### Standardized Response Format

```typescript
// lib/types/api.ts
interface ApiResponse<T = any> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  meta?: {
    pagination?: {
      total: number;
      pages: number;
      currentPage: number;
      pageSize: number;
    };
    filters?: Record<string, any>;
  };
}

// Success response helper
export function successResponse<T>(data: T, meta?: any): ApiResponse<T> {
  return {
    success: true,
    data,
    ...(meta && { meta }),
  };
}

// Error response helper
export function errorResponse(code: string, message: string, details?: any): ApiResponse {
  return {
    success: false,
    error: {
      code,
      message,
      ...(details && { details }),
    },
  };
}
```

### Pagination Pattern

```typescript
// GET /api/teams/:teamId/documents
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { query, sort } = req.query as { query?: string; sort?: string };
  
  const usePagination = !!(query || sort);
  const page = usePagination ? Number(req.query.page) || 1 : undefined;
  const limit = usePagination ? Number(req.query.limit) || 10 : undefined;

  // Get total count for pagination
  const totalDocuments = usePagination
    ? await prisma.document.count({
        where: {
          teamId: teamId,
          ...(query && {
            name: {
              contains: query,
              mode: "insensitive",
            },
          }),
        },
      })
    : undefined;

  // Fetch paginated results
  const documents = await prisma.document.findMany({
    where: { /* filters */ },
    orderBy: getOrderBy(sort),
    ...(usePagination && {
      skip: ((page as number) - 1) * (limit as number),
      take: limit,
    }),
    include: { /* relations */ },
  });

  return res.status(200).json({
    documents,
    ...(usePagination && {
      pagination: {
        total: totalDocuments,
        pages: Math.ceil(totalDocuments! / limit!),
        currentPage: page,
        pageSize: limit,
      },
    }),
  });
}
```

### Query Parameter Validation

```typescript
// lib/validation/query.ts
import { z } from 'zod';

export const paginationSchema = z.object({
  page: z.coerce.number().min(1).default(1),
  limit: z.coerce.number().min(1).max(100).default(10),
});

export const sortSchema = z.object({
  sort: z.enum(['createdAt', 'name', 'views', 'lastViewed']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

export const searchSchema = z.object({
  query: z.string().min(1).max(100).optional(),
});

// Usage in API route
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const pagination = paginationSchema.parse(req.query);
  const sort = sortSchema.parse(req.query);
  const search = searchSchema.parse(req.query);
  
  // Use validated parameters...
}
```

---

## Error Handling & Validation

### Custom Error Classes

```typescript
// lib/errorHandler.ts
export class TeamError extends Error {
  statusCode = 400;
  constructor(public message: string) {
    super(message);
  }
}

export class DocumentError extends Error {
  statusCode = 400;
  constructor(public message: string) {
    super(message);
  }
}

export class ValidationError extends Error {
  statusCode = 422;
  constructor(public message: string, public details?: any) {
    super(message);
  }
}

export class AuthError extends Error {
  statusCode: number;
  constructor(message: string, statusCode = 401) {
    super(message);
    this.statusCode = statusCode;
  }
}

export class RateLimitError extends Error {
  statusCode = 429;
  constructor(public message: string = "Too many requests") {
    super(message);
  }
}
```

### Centralized Error Handler

```typescript
// lib/errorHandler.ts
export function errorhandler(err: unknown, res: NextApiResponse) {
  console.error('API Error:', err);

  if (err instanceof TeamError || err instanceof DocumentError) {
    return res.status(err.statusCode).json({
      success: false,
      error: {
        code: err.constructor.name,
        message: err.message,
      },
    });
  }

  if (err instanceof ValidationError) {
    return res.status(422).json({
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: err.message,
        details: err.details,
      },
    });
  }

  if (err instanceof AuthError) {
    return res.status(err.statusCode).json({
      success: false,
      error: {
        code: 'AUTH_ERROR',
        message: err.message,
      },
    });
  }

  if (err instanceof RateLimitError) {
    return res.status(429).json({
      success: false,
      error: {
        code: 'RATE_LIMIT_EXCEEDED',
        message: err.message,
      },
    });
  }

  // Default error handling
  return res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: 'An unexpected error occurred',
      ...(process.env.NODE_ENV === 'development' && {
        details: (err as Error).message,
      }),
    },
  });
}
```

### Input Validation with Zod

```typescript
// lib/validation/schemas.ts
import { z } from 'zod';

export const createDocumentSchema = z.object({
  name: z.string().min(1).max(255),
  url: z.string().url(),
  storageType: z.enum(['VERCEL_BLOB', 'S3_PATH']),
  numPages: z.number().min(1).optional(),
  type: z.string(),
  contentType: z.string(),
  createLink: z.boolean().default(false),
  fileSize: z.number().positive().optional(),
  folderPathName: z.string().optional(),
});

export const updateDocumentSchema = z.object({
  name: z.string().min(1).max(255).optional(),
  folderId: z.string().uuid().nullable().optional(),
});

// Validation middleware
export function validateBody<T>(schema: z.ZodSchema<T>) {
  return (handler: (req: NextApiRequest & { body: T }, res: NextApiResponse) => Promise<void>) => {
    return async (req: NextApiRequest, res: NextApiResponse) => {
      try {
        req.body = schema.parse(req.body);
        return await handler(req as NextApiRequest & { body: T }, res);
      } catch (error) {
        if (error instanceof z.ZodError) {
          return res.status(422).json({
            success: false,
            error: {
              code: 'VALIDATION_ERROR',
              message: 'Invalid request data',
              details: error.errors,
            },
          });
        }
        throw error;
      }
    };
  };
}

// Usage
export default validateBody(createDocumentSchema)(async (req, res) => {
  // req.body is now typed and validated
  const { name, url, storageType } = req.body;
  // ...
});
```

---

## Rate Limiting & Security

### Redis-Based Rate Limiting

```typescript
// lib/redis.ts
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL as string,
  token: process.env.UPSTASH_REDIS_REST_TOKEN as string,
});

// Flexible rate limiter factory
export const ratelimit = (
  requests: number = 10,
  window: string = "10 s", // "1 m", "1 h", "1 d"
) => {
  return new Ratelimit({
    redis: redis,
    limiter: Ratelimit.slidingWindow(requests, window),
    analytics: true,
    prefix: "papermark",
  });
};

// Common rate limit configurations
export const rateLimits = {
  // General API endpoints
  api: ratelimit(100, "1 m"),
  
  // Authentication endpoints
  auth: ratelimit(5, "1 m"),
  
  // File upload endpoints
  upload: ratelimit(10, "1 m"),
  
  // Webhook endpoints
  webhook: ratelimit(1000, "1 m"),
  
  // Per-user, per-resource limits
  perUserDocument: (docId: string, teamId: string, userId: string) =>
    ratelimit(120, "1 m").limit(`doc:${docId}:team:${teamId}:user:${userId}`),
};
```

### Rate Limiting Middleware

```typescript
// lib/middleware/rateLimit.ts
export function withRateLimit(
  limiter: Ratelimit,
  getKey?: (req: NextApiRequest) => string
) {
  return (handler: (req: NextApiRequest, res: NextApiResponse) => Promise<void>) => {
    return async (req: NextApiRequest, res: NextApiResponse) => {
      const key = getKey ? getKey(req) : req.ip || 'anonymous';
      
      const { success, limit, remaining, reset } = await limiter.limit(key);

      // Set rate limit headers
      res.setHeader("X-RateLimit-Limit", limit.toString());
      res.setHeader("X-RateLimit-Remaining", remaining.toString());
      res.setHeader("X-RateLimit-Reset", reset.toString());

      if (!success) {
        return res.status(429).json({
          success: false,
          error: {
            code: 'RATE_LIMIT_EXCEEDED',
            message: 'Too many requests',
          },
        });
      }

      return await handler(req, res);
    };
  };
}

// Usage examples
export default withRateLimit(
  rateLimits.api,
  (req) => `api:${req.ip}` // Custom key generator
)(async (req, res) => {
  // Handler code
});

// Per-resource rate limiting
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { teamId, id: docId } = req.query;
  const userId = getUserId(req);
  
  const { success, limit, remaining, reset } = await rateLimits.perUserDocument(
    docId as string,
    teamId as string,
    userId
  );
  
  if (!success) {
    return res.status(429).json({ error: "Too many requests" });
  }
  
  // Continue with request...
}
```

### Security Headers & CORS

```typescript
// lib/middleware/security.ts
export function withSecurity() {
  return (handler: (req: NextApiRequest, res: NextApiResponse) => Promise<void>) => {
    return async (req: NextApiRequest, res: NextApiResponse) => {
      // Security headers
      res.setHeader('X-Content-Type-Options', 'nosniff');
      res.setHeader('X-Frame-Options', 'DENY');
      res.setHeader('X-XSS-Protection', '1; mode=block');
      res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
      
      // CORS handling
      const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];
      const origin = req.headers.origin;
      
      if (origin && allowedOrigins.includes(origin)) {
        res.setHeader('Access-Control-Allow-Origin', origin);
      }
      
      res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
      res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
      
      if (req.method === 'OPTIONS') {
        return res.status(200).end();
      }
      
      return await handler(req, res);
    };
  };
}
```

---

## File Upload & Processing

### Presigned URL Pattern

```typescript
// pages/api/file/s3/get-presigned-post-url.ts
import { PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import slugify from "@sindresorhus/slugify";
import path from "node:path";

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== "POST") {
    return res.status(405).end("Method Not Allowed");
  }

  const { fileName, contentType, teamId, docId } = req.body;

  // Authenticate user
  const session = await getServerSession(req, res, authOptions);
  if (!session) {
    return res.status(401).end("Unauthorized");
  }

  // Verify team access
  const team = await prisma.team.findUnique({
    where: {
      id: teamId,
      users: {
        some: {
          userId: (session.user as CustomUser).id,
        },
      },
    },
  });

  if (!team) {
    return res.status(403).end("Unauthorized to access this team");
  }

  try {
    // Sanitize filename
    const { name, ext } = path.parse(fileName);
    const slugifiedName = slugify(name) + ext;
    const key = `${team.id}/${docId}/${slugifiedName}`;

    // Get S3 client and config for team
    const { client, config } = await getTeamS3ClientAndConfig(team.id);

    // Create presigned URL
    const putObjectCommand = new PutObjectCommand({
      Bucket: config.bucket,
      Key: key,
      ContentType: contentType,
      ContentDisposition: `attachment; filename="${slugifiedName}"`,
    });

    const url = await getSignedUrl(client, putObjectCommand, {
      expiresIn: 3600, // 1 hour
    });

    return res.status(200).json({ 
      url, 
      key, 
      docId, 
      fileName: slugifiedName 
    });
  } catch (error) {
    console.error('Presigned URL generation error:', error);
    return res.status(500).json({ error: "Internal server error" });
  }
}
```

### TUS Resumable Upload

```typescript
// pages/api/file/tus/[[...file]].ts
import { Server } from "@tus/server";
import { FileStore } from "@tus/file-store";

const tusServer = new Server({
  path: "/api/file/tus",
  datastore: new FileStore({ directory: "./uploads" }),
  onUploadCreate: async (req, res, upload) => {
    // Validate upload permissions
    const auth = await validateUploadAuth(req);
    if (!auth.valid) {
      throw new Error("Unauthorized");
    }
    
    // Store metadata
    upload.metadata = {
      ...upload.metadata,
      userId: auth.userId,
      teamId: auth.teamId,
    };
  },
  onUploadFinish: async (req, res, upload) => {
    // Process completed upload
    await processUploadedFile(upload);
  },
});

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  return new Promise<void>((resolve) => {
    tusServer.handle(req, res, resolve);
  });
}

export const config = {
  api: {
    bodyParser: false,
  },
};
```

### File Processing Pipeline

```typescript
// lib/files/process-document.ts
export async function processDocument({
  documentData,
  teamId,
  userId,
  teamPlan,
  createLink,
  folderPathName,
}: ProcessDocumentParams) {
  try {
    // 1. Create document record
    const document = await prisma.document.create({
      data: {
        name: documentData.name,
        file: documentData.key,
        type: documentData.supportedFileType,
        contentType: documentData.contentType,
        storageType: documentData.storageType,
        numPages: documentData.numPages,
        fileSize: documentData.fileSize,
        ownerId: userId,
        teamId: teamId,
        assistantEnabled: documentData.supportedFileType === "pdf",
        advancedExcelEnabled: documentData.enableExcelAdvancedMode,
        // Set folder if specified
        ...(folderPathName && {
          folder: {
            connect: {
              teamId_path: {
                teamId: teamId,
                path: folderPathName,
              },
            },
          },
        }),
      },
    });

    // 2. Create document version
    const version = await prisma.documentVersion.create({
      data: {
        documentId: document.id,
        versionNumber: 1,
        file: documentData.key,
        type: documentData.supportedFileType,
        contentType: documentData.contentType,
        storageType: documentData.storageType,
        numPages: documentData.numPages,
        fileSize: documentData.fileSize,
        isPrimary: true,
      },
    });

    // 3. Process file asynchronously
    await processFileAsync({
      documentId: document.id,
      versionId: version.id,
      fileType: documentData.supportedFileType,
      fileUrl: documentData.key,
    });

    // 4. Create link if requested
    if (createLink) {
      const link = await createDocumentLink({
        documentId: document.id,
        userId,
        teamId,
      });
      
      return { ...document, link };
    }

    return document;
  } catch (error) {
    console.error('Document processing error:', error);
    throw new Error('Failed to process document');
  }
}
```

---

## Database Patterns

### Repository Pattern

```typescript
// lib/repositories/documentRepository.ts
export class DocumentRepository {
  async findByTeam(teamId: string, options: FindDocumentsOptions = {}) {
    const {
      query,
      sort = 'createdAt',
      order = 'desc',
      page = 1,
      limit = 10,
      includeFolder = false,
    } = options;

    const where: Prisma.DocumentWhereInput = {
      teamId,
      ...(query && {
        name: {
          contains: query,
          mode: 'insensitive',
        },
      }),
    };

    const orderBy = this.getOrderBy(sort, order);

    const [documents, total] = await Promise.all([
      prisma.document.findMany({
        where,
        orderBy,
        skip: (page - 1) * limit,
        take: limit,
        include: {
          ...(includeFolder && {
            folder: {
              select: { name: true, path: true },
            },
          }),
          _count: {
            select: {
              links: true,
              views: true,
              versions: true,
            },
          },
        },
      }),
      prisma.document.count({ where }),
    ]);

    return {
      documents,
      pagination: {
        total,
        pages: Math.ceil(total / limit),
        currentPage: page,
        pageSize: limit,
      },
    };
  }

  async findByIdWithPermissions(
    documentId: string,
    userId: string,
    teamId: string
  ) {
    return prisma.document.findFirst({
      where: {
        id: documentId,
        teamId,
        team: {
          users: {
            some: { userId },
          },
        },
      },
      include: {
        versions: {
          where: { isPrimary: true },
          orderBy: { createdAt: 'desc' },
          take: 1,
        },
        folder: {
          select: { name: true, path: true },
        },
      },
    });
  }

  private getOrderBy(sort: string, order: 'asc' | 'desc'): Prisma.DocumentOrderByWithRelationInput {
    switch (sort) {
      case 'name':
        return { name: order };
      case 'views':
        return { views: { _count: order } };
      case 'links':
        return { links: { _count: order } };
      default:
        return { createdAt: order };
    }
  }
}
```

### Transaction Patterns

```typescript
// lib/services/documentService.ts
export class DocumentService {
  async deleteDocument(documentId: string, userId: string, teamId: string) {
    return await prisma.$transaction(async (tx) => {
      // 1. Verify permissions
      const document = await tx.document.findFirst({
        where: {
          id: documentId,
          teamId,
          team: {
            users: {
              some: { userId },
            },
          },
        },
        include: {
          versions: {
            select: {
              id: true,
              file: true,
              type: true,
              storageType: true,
            },
          },
        },
      });

      if (!document) {
        throw new DocumentError("Document not found");
      }

      // 2. Delete files from storage
      if (document.type !== "notion") {
        for (const version of document.versions) {
          await deleteFile({
            type: version.storageType,
            data: version.file,
            teamId,
          });
        }
      }

      // 3. Delete database records (cascades will handle related data)
      await tx.document.delete({
        where: { id: documentId },
      });

      // 4. Log the action
      await tx.auditLog.create({
        data: {
          action: 'DELETE_DOCUMENT',
          resourceId: documentId,
          resourceType: 'DOCUMENT',
          userId,
          teamId,
          metadata: {
            documentName: document.name,
          },
        },
      });

      return { success: true };
    });
  }
}
```

### Data Serialization

```typescript
// lib/utils/serialization.ts
export function serializeFileSize(obj: any): any {
  if (obj === null || obj === undefined) {
    return obj;
  }

  if (Array.isArray(obj)) {
    return obj.map(serializeFileSize);
  }

  if (typeof obj === "object") {
    const serialized: any = {};
    for (const key in obj) {
      if (obj.hasOwnProperty(key)) {
        if (key === "fileSize" && typeof obj[key] === "bigint") {
          // Convert BigInt fileSize to number for JSON serialization
          serialized[key] = Number(obj[key]);
        } else {
          serialized[key] = serializeFileSize(obj[key]);
        }
      }
    }
    return serialized;
  }

  return obj;
}

// Date serialization
export function serializeDates(obj: any): any {
  if (obj instanceof Date) {
    return obj.toISOString();
  }
  
  if (Array.isArray(obj)) {
    return obj.map(serializeDates);
  }
  
  if (obj && typeof obj === 'object') {
    const serialized: any = {};
    for (const key in obj) {
      serialized[key] = serializeDates(obj[key]);
    }
    return serialized;
  }
  
  return obj;
}

// Combined serialization
export function serializeResponse<T>(data: T): T {
  return serializeDates(serializeFileSize(data));
}
```

---

## Webhook Implementation

### Webhook Schema & Validation

```typescript
// lib/zod/schemas/webhooks.ts
import { z } from "zod";

export const createWebhookSchema = z.object({
  name: z.string().min(1).max(40),
  url: z.string().url().max(190),
  secret: z.string().startsWith("whsec_"),
  triggers: z.array(z.enum(["link.created", "document.created", "link.viewed"])),
});

// Event schemas
const linkEventSchema = z.object({
  id: z.string(),
  url: z.string(),
  name: z.string().nullable(),
  domain: z.string(),
  key: z.string(),
  expiresAt: z.string().datetime().nullable(),
  hasPassword: z.boolean().default(false),
  allowDownload: z.boolean().default(false),
  documentId: z.string().nullable(),
  dataroomId: z.string().nullable(),
  teamId: z.string(),
  createdAt: z.string().datetime(),
});

const viewEventSchema = z.object({
  viewedAt: z.string().datetime(),
  viewId: z.string(),
  email: z.string().email().nullable(),
  emailVerified: z.boolean().default(false),
  country: z.string().nullable(),
  city: z.string().nullable(),
  device: z.string().nullable(),
  browser: z.string().nullable(),
  os: z.string().nullable(),
});

// Webhook payload schemas
export const linkCreatedWebhookSchema = z.object({
  id: z.string().startsWith("evt_"),
  event: z.literal("link.created"),
  createdAt: z.string().datetime(),
  data: z.object({
    link: linkEventSchema,
    document: documentEventSchema.optional(),
  }),
});

export const linkViewedWebhookSchema = z.object({
  id: z.string().startsWith("evt_"),
  event: z.literal("link.viewed"),
  createdAt: z.string().datetime(),
  data: z.object({
    view: viewEventSchema,
    link: linkEventSchema,
    document: documentEventSchema.optional(),
  }),
});
```

### Webhook Delivery System

```typescript
// lib/webhook/send-webhooks.ts
import { QStash } from "@upstash/qstash";

const qstash = new QStash({
  token: process.env.QSTASH_TOKEN!,
});

export async function sendWebhook({
  webhook,
  payload,
  retryConfig = { maxRetries: 3, backoff: "exponential" },
}: {
  webhook: { url: string; secret: string };
  payload: WebhookPayload;
  retryConfig?: { maxRetries: number; backoff: string };
}) {
  try {
    // Create signature
    const signature = generateWebhookSignature(payload, webhook.secret);
    
    // Send via QStash for reliable delivery
    const response = await qstash.publishJSON({
      url: webhook.url,
      body: payload,
      headers: {
        "X-Webhook-Signature": signature,
        "X-Webhook-Event": payload.event,
        "Content-Type": "application/json",
      },
      retries: retryConfig.maxRetries,
    });

    return {
      success: true,
      messageId: response.messageId,
    };
  } catch (error) {
    console.error("Webhook delivery failed:", error);
    return {
      success: false,
      error: error.message,
    };
  }
}

function generateWebhookSignature(payload: any, secret: string): string {
  const crypto = require('crypto');
  const body = JSON.stringify(payload);
  return crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');
}
```

### Webhook Receiver Pattern

```typescript
// pages/api/stripe/webhook.ts
import { Readable } from "node:stream";
import Stripe from "stripe";

// Disable body parser for webhook verification
export const config = {
  api: {
    bodyParser: false,
  },
};

async function buffer(readable: Readable) {
  const chunks: Buffer[] = [];
  for await (const chunk of readable) {
    chunks.push(typeof chunk === "string" ? Buffer.from(chunk) : chunk);
  }
  return Buffer.concat(chunks);
}

const relevantEvents = new Set([
  "checkout.session.completed",
  "customer.subscription.updated",
  "customer.subscription.deleted",
]);

export default async function webhookHandler(
  req: NextApiRequest,
  res: NextApiResponse,
) {
  if (req.method !== "POST") {
    res.setHeader("Allow", ["POST"]);
    return res.status(405).end(`Method ${req.method} Not Allowed`);
  }

  const buf = await buffer(req);
  const sig = req.headers["stripe-signature"];
  const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;
  
  let event: Stripe.Event;
  
  try {
    if (!sig || !webhookSecret) {
      throw new Error("Missing signature or webhook secret");
    }
    
    const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
    event = stripe.webhooks.constructEvent(buf, sig, webhookSecret);
  } catch (err: any) {
    console.error("Webhook signature verification failed:", err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  // Ignore unsupported events
  if (!relevantEvents.has(event.type)) {
    return res.status(400).send(`Unhandled event type: ${event.type}`);
  }

  try {
    switch (event.type) {
      case "checkout.session.completed":
        await handleCheckoutSessionCompleted(event);
        break;
      case "customer.subscription.updated":
        await handleSubscriptionUpdated(event);
        break;
      case "customer.subscription.deleted":
        await handleSubscriptionDeleted(event);
        break;
    }
  } catch (error) {
    console.error(`Webhook handler failed for ${event.type}:`, error);
    return res.status(400).send("Webhook handler failed");
  }

  return res.status(200).json({ received: true });
}
```

---

## API Versioning & Documentation

### URL-Based Versioning

```typescript
// pages/api/v1/teams/[teamId]/documents/index.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // V1 implementation
}

// pages/api/v2/teams/[teamId]/documents/index.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  // V2 implementation with breaking changes
}
```

### Header-Based Versioning

```typescript
// lib/middleware/versioning.ts
export function withVersioning() {
  return (handler: (req: NextApiRequest, res: NextApiResponse, version: string) => Promise<void>) => {
    return async (req: NextApiRequest, res: NextApiResponse) => {
      const version = req.headers['api-version'] as string || 'v1';
      
      // Validate version
      if (!['v1', 'v2'].includes(version)) {
        return res.status(400).json({
          error: 'Unsupported API version',
          supportedVersions: ['v1', 'v2'],
        });
      }
      
      return await handler(req, res, version);
    };
  };
}
```

### OpenAPI Documentation

```typescript
// lib/docs/openapi.ts
export const openApiSpec = {
  openapi: "3.0.0",
  info: {
    title: "Papermark API",
    version: "1.0.0",
    description: "Document sharing and analytics API",
  },
  servers: [
    {
      url: "https://api.papermark.io",
      description: "Production server",
    },
    {
      url: "http://localhost:3000/api",
      description: "Development server",
    },
  ],
  paths: {
    "/teams/{teamId}/documents": {
      get: {
        summary: "List team documents",
        parameters: [
          {
            name: "teamId",
            in: "path",
            required: true,
            schema: { type: "string" },
          },
          {
            name: "page",
            in: "query",
            schema: { type: "integer", minimum: 1, default: 1 },
          },
          {
            name: "limit",
            in: "query",
            schema: { type: "integer", minimum: 1, maximum: 100, default: 10 },
          },
          {
            name: "query",
            in: "query",
            schema: { type: "string" },
            description: "Search query for document names",
          },
          {
            name: "sort",
            in: "query",
            schema: {
              type: "string",
              enum: ["createdAt", "name", "views", "lastViewed"],
              default: "createdAt",
            },
          },
        ],
        responses: {
          200: {
            description: "List of documents",
            content: {
              "application/json": {
                schema: {
                  type: "object",
                  properties: {
                    documents: {
                      type: "array",
                      items: { $ref: "#/components/schemas/Document" },
                    },
                    pagination: { $ref: "#/components/schemas/Pagination" },
                  },
                },
              },
            },
          },
          401: { $ref: "#/components/responses/Unauthorized" },
          403: { $ref: "#/components/responses/Forbidden" },
          429: { $ref: "#/components/responses/RateLimited" },
        },
        security: [{ BearerAuth: [] }],
      },
    },
  },
  components: {
    schemas: {
      Document: {
        type: "object",
        properties: {
          id: { type: "string" },
          name: { type: "string" },
          type: { type: "string" },
          createdAt: { type: "string", format: "date-time" },
          updatedAt: { type: "string", format: "date-time" },
        },
      },
      Pagination: {
        type: "object",
        properties: {
          total: { type: "integer" },
          pages: { type: "integer" },
          currentPage: { type: "integer" },
          pageSize: { type: "integer" },
        },
      },
    },
    responses: {
      Unauthorized: {
        description: "Authentication required",
        content: {
          "application/json": {
            schema: {
              type: "object",
              properties: {
                error: { type: "string" },
              },
            },
          },
        },
      },
    },
    securitySchemes: {
      BearerAuth: {
        type: "http",
        scheme: "bearer",
      },
    },
  },
};
```

---

## Testing Strategies

### Unit Tests for API Routes

```typescript
// __tests__/api/teams/documents.test.ts
import { createMocks } from 'node-mocks-http';
import handler from '@/pages/api/teams/[teamId]/documents/index';
import { getServerSession } from 'next-auth/next';

// Mock dependencies
jest.mock('next-auth/next');
jest.mock('@/lib/prisma');

const mockGetServerSession = getServerSession as jest.MockedFunction<typeof getServerSession>;

describe('/api/teams/[teamId]/documents', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('GET', () => {
    it('should return documents for authenticated user', async () => {
      // Setup mocks
      mockGetServerSession.mockResolvedValue({
        user: { id: 'user-1', email: 'test@example.com' },
      });

      const { req, res } = createMocks({
        method: 'GET',
        query: { teamId: 'team-1' },
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(200);
      const data = JSON.parse(res._getData());
      expect(data).toHaveProperty('documents');
    });

    it('should return 401 for unauthenticated user', async () => {
      mockGetServerSession.mockResolvedValue(null);

      const { req, res } = createMocks({
        method: 'GET',
        query: { teamId: 'team-1' },
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(401);
    });

    it('should handle pagination correctly', async () => {
      mockGetServerSession.mockResolvedValue({
        user: { id: 'user-1', email: 'test@example.com' },
      });

      const { req, res } = createMocks({
        method: 'GET',
        query: { 
          teamId: 'team-1',
          page: '2',
          limit: '5',
        },
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(200);
      const data = JSON.parse(res._getData());
      expect(data.pagination).toEqual({
        currentPage: 2,
        pageSize: 5,
        total: expect.any(Number),
        pages: expect.any(Number),
      });
    });
  });

  describe('POST', () => {
    it('should create document with valid data', async () => {
      mockGetServerSession.mockResolvedValue({
        user: { id: 'user-1', email: 'test@example.com' },
      });

      const { req, res } = createMocks({
        method: 'POST',
        query: { teamId: 'team-1' },
        body: {
          name: 'Test Document',
          url: 'https://example.com/doc.pdf',
          storageType: 'VERCEL_BLOB',
          type: 'pdf',
          contentType: 'application/pdf',
        },
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(201);
    });

    it('should validate required fields', async () => {
      mockGetServerSession.mockResolvedValue({
        user: { id: 'user-1', email: 'test@example.com' },
      });

      const { req, res } = createMocks({
        method: 'POST',
        query: { teamId: 'team-1' },
        body: {
          // Missing required fields
        },
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(422);
    });
  });
});
```

### Integration Tests

```typescript
// __tests__/integration/document-lifecycle.test.ts
import { testApiHandler } from 'next-test-api-route-handler';
import documentHandler from '@/pages/api/teams/[teamId]/documents/index';
import documentDetailHandler from '@/pages/api/teams/[teamId]/documents/[id]/index';

describe('Document Lifecycle Integration', () => {
  it('should create, read, update, and delete document', async () => {
    let documentId: string;

    // Create document
    await testApiHandler({
      handler: documentHandler,
      url: '/api/teams/team-1/documents',
      test: async ({ fetch }) => {
        const res = await fetch({
          method: 'POST',
          headers: {
            'Authorization': 'Bearer test-token',
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            name: 'Integration Test Doc',
            url: 'https://example.com/test.pdf',
            storageType: 'VERCEL_BLOB',
            type: 'pdf',
            contentType: 'application/pdf',
          }),
        });

        expect(res.status).toBe(201);
        const data = await res.json();
        documentId = data.id;
        expect(data.name).toBe('Integration Test Doc');
      },
    });

    // Read document
    await testApiHandler({
      handler: documentDetailHandler,
      paramsPatcher: (params) => {
        params.teamId = 'team-1';
        params.id = documentId;
      },
      test: async ({ fetch }) => {
        const res = await fetch({
          method: 'GET',
          headers: {
            'Authorization': 'Bearer test-token',
          },
        });

        expect(res.status).toBe(200);
        const data = await res.json();
        expect(data.id).toBe(documentId);
      },
    });

    // Update document
    await testApiHandler({
      handler: documentDetailHandler,
      paramsPatcher: (params) => {
        params.teamId = 'team-1';
        params.id = documentId;
      },
      test: async ({ fetch }) => {
        const res = await fetch({
          method: 'PUT',
          headers: {
            'Authorization': 'Bearer test-token',
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            folderId: null,
            currentPathName: '/',
          }),
        });

        expect(res.status).toBe(200);
      },
    });

    // Delete document
    await testApiHandler({
      handler: documentDetailHandler,
      paramsPatcher: (params) => {
        params.teamId = 'team-1';
        params.id = documentId;
      },
      test: async ({ fetch }) => {
        const res = await fetch({
          method: 'DELETE',
          headers: {
            'Authorization': 'Bearer test-token',
          },
        });

        expect(res.status).toBe(204);
      },
    });
  });
});
```

### Load Testing

```typescript
// scripts/load-test.ts
import { performance } from 'perf_hooks';

interface LoadTestConfig {
  url: string;
  concurrent: number;
  duration: number; // seconds
  headers?: Record<string, string>;
}

export async function runLoadTest(config: LoadTestConfig) {
  const results: { responseTime: number; status: number }[] = [];
  const startTime = performance.now();
  const endTime = startTime + (config.duration * 1000);

  const promises: Promise<void>[] = [];

  for (let i = 0; i < config.concurrent; i++) {
    promises.push(
      (async () => {
        while (performance.now() < endTime) {
          const requestStart = performance.now();
          
          try {
            const response = await fetch(config.url, {
              headers: config.headers,
            });
            
            const requestEnd = performance.now();
            results.push({
              responseTime: requestEnd - requestStart,
              status: response.status,
            });
          } catch (error) {
            results.push({
              responseTime: performance.now() - requestStart,
              status: 0, // Connection error
            });
          }

          // Small delay to prevent overwhelming
          await new Promise(resolve => setTimeout(resolve, 10));
        }
      })()
    );
  }

  await Promise.all(promises);

  // Calculate statistics
  const successfulRequests = results.filter(r => r.status >= 200 && r.status < 300);
  const avgResponseTime = results.reduce((sum, r) => sum + r.responseTime, 0) / results.length;
  const successRate = (successfulRequests.length / results.length) * 100;

  return {
    totalRequests: results.length,
    successfulRequests: successfulRequests.length,
    failedRequests: results.length - successfulRequests.length,
    avgResponseTime: Math.round(avgResponseTime),
    successRate: Math.round(successRate * 100) / 100,
    requestsPerSecond: Math.round(results.length / config.duration),
  };
}

// Usage
if (require.main === module) {
  runLoadTest({
    url: 'http://localhost:3000/api/health',
    concurrent: 10,
    duration: 30,
    headers: {
      'Authorization': 'Bearer test-token',
    },
  }).then(results => {
    console.log('Load Test Results:', results);
  });
}
```

---

## Performance Optimization

### Database Query Optimization

```typescript
// lib/optimization/queryOptimization.ts

// 1. Use select to limit fields
export async function getDocuments(teamId: string) {
  return prisma.document.findMany({
    where: { teamId },
    select: {
      id: true,
      name: true,
      createdAt: true,
      // Don't select large fields like file content
    },
  });
}

// 2. Use indexes for common queries
export async function searchDocuments(teamId: string, query: string) {
  return prisma.document.findMany({
    where: {
      teamId, // Indexed field first
      name: {
        contains: query,
        mode: 'insensitive',
      },
    },
    // Add index: @@index([teamId, name])
  });
}

// 3. Aggregate queries for counts
export async function getDocumentStats(teamId: string) {
  const [totalDocs, totalViews] = await Promise.all([
    prisma.document.count({
      where: { teamId },
    }),
    prisma.view.count({
      where: {
        document: {
          teamId,
        },
      },
    }),
  ]);

  return { totalDocs, totalViews };
}

// 4. Batch queries to avoid N+1 problems
export async function getDocumentsWithViewCounts(teamId: string) {
  return prisma.document.findMany({
    where: { teamId },
    include: {
      _count: {
        select: {
          views: true,
          links: true,
        },
      },
    },
  });
}
```

### Caching Strategies

```typescript
// lib/cache/redis-cache.ts
import { redis } from '@/lib/redis';

export class RedisCache {
  private defaultTTL = 3600; // 1 hour

  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await redis.get(key);
      return value ? JSON.parse(value) : null;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  async set(key: string, value: any, ttl = this.defaultTTL): Promise<void> {
    try {
      await redis.setex(key, ttl, JSON.stringify(value));
    } catch (error) {
      console.error('Cache set error:', error);
    }
  }

  async del(key: string): Promise<void> {
    try {
      await redis.del(key);
    } catch (error) {
      console.error('Cache delete error:', error);
    }
  }

  async invalidatePattern(pattern: string): Promise<void> {
    try {
      const keys = await redis.keys(pattern);
      if (keys.length > 0) {
        await redis.del(...keys);
      }
    } catch (error) {
      console.error('Cache invalidate error:', error);
    }
  }
}

export const cache = new RedisCache();

// Cache wrapper for expensive operations
export function withCache<T extends any[], R>(
  fn: (...args: T) => Promise<R>,
  getKey: (...args: T) => string,
  ttl?: number
) {
  return async (...args: T): Promise<R> => {
    const key = getKey(...args);
    
    // Try to get from cache first
    const cached = await cache.get<R>(key);
    if (cached !== null) {
      return cached;
    }

    // Execute function and cache result
    const result = await fn(...args);
    await cache.set(key, result, ttl);
    
    return result;
  };
}

// Usage example
export const getTeamDocumentsWithCache = withCache(
  async (teamId: string) => {
    return prisma.document.findMany({
      where: { teamId },
      include: { _count: { select: { views: true } } },
    });
  },
  (teamId) => `team:${teamId}:documents`,
  300 // 5 minutes
);
```

### Response Compression

```typescript
// lib/middleware/compression.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { gzip } from 'zlib';
import { promisify } from 'util';

const gzipAsync = promisify(gzip);

export function withCompression() {
  return (handler: (req: NextApiRequest, res: NextApiResponse) => Promise<void>) => {
    return async (req: NextApiRequest, res: NextApiResponse) => {
      const originalSend = res.send;
      const originalJson = res.json;

      // Override res.json to add compression
      res.json = function(body: any) {
        const acceptEncoding = req.headers['accept-encoding'] || '';
        
        if (acceptEncoding.includes('gzip')) {
          const jsonString = JSON.stringify(body);
          
          // Only compress if response is large enough
          if (jsonString.length > 1024) {
            gzipAsync(Buffer.from(jsonString))
              .then(compressed => {
                res.setHeader('Content-Encoding', 'gzip');
                res.setHeader('Content-Type', 'application/json');
                originalSend.call(res, compressed);
              })
              .catch(() => {
                // Fallback to uncompressed
                originalJson.call(res, body);
              });
            return res;
          }
        }
        
        return originalJson.call(res, body);
      };

      return await handler(req, res);
    };
  };
}
```

---

## Monitoring & Logging

### Structured Logging

```typescript
// lib/logging/logger.ts
import { createLogger, format, transports } from 'winston';

const logger = createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: format.combine(
    format.timestamp(),
    format.errors({ stack: true }),
    format.json()
  ),
  defaultMeta: { service: 'papermark-api' },
  transports: [
    new transports.File({ filename: 'logs/error.log', level: 'error' }),
    new transports.File({ filename: 'logs/combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new transports.Console({
    format: format.combine(
      format.colorize(),
      format.simple()
    )
  }));
}

export { logger };

// API request logging middleware
export function withLogging() {
  return (handler: (req: NextApiRequest, res: NextApiResponse) => Promise<void>) => {
    return async (req: NextApiRequest, res: NextApiResponse) => {
      const startTime = Date.now();
      const requestId = req.headers['x-request-id'] || generateRequestId();
      
      // Add request ID to response headers
      res.setHeader('x-request-id', requestId);
      
      logger.info('API Request', {
        requestId,
        method: req.method,
        url: req.url,
        userAgent: req.headers['user-agent'],
        ip: req.headers['x-forwarded-for'] || req.socket.remoteAddress,
      });

      try {
        await handler(req, res);
        
        const duration = Date.now() - startTime;
        logger.info('API Response', {
          requestId,
          statusCode: res.statusCode,
          duration,
        });
      } catch (error) {
        const duration = Date.now() - startTime;
        logger.error('API Error', {
          requestId,
          error: error.message,
          stack: error.stack,
          duration,
        });
        throw error;
      }
    };
  };
}

function generateRequestId(): string {
  return Math.random().toString(36).substring(2) + Date.now().toString(36);
}
```

### Health Check Endpoint

```typescript
// pages/api/health.ts
import { NextApiRequest, NextApiResponse } from 'next';
import { redis } from '@/lib/redis';
import prisma from '@/lib/prisma';

interface HealthStatus {
  status: 'healthy' | 'unhealthy';
  timestamp: string;
  checks: {
    database: 'up' | 'down';
    redis: 'up' | 'down';
    storage: 'up' | 'down';
  };
  version: string;
  uptime: number;
}

export default async function health(
  req: NextApiRequest,
  res: NextApiResponse<HealthStatus>
) {
  const startTime = Date.now();
  
  const checks = {
    database: 'down' as const,
    redis: 'down' as const,
    storage: 'up' as const, // Assume storage is up for now
  };

  // Check database
  try {
    await prisma.$queryRaw`SELECT 1`;
    checks.database = 'up';
  } catch (error) {
    console.error('Database health check failed:', error);
  }

  // Check Redis
  try {
    await redis.ping();
    checks.redis = 'up';
  } catch (error) {
    console.error('Redis health check failed:', error);
  }

  const allHealthy = Object.values(checks).every(status => status === 'up');
  const status = allHealthy ? 'healthy' : 'unhealthy';
  const responseTime = Date.now() - startTime;

  const healthStatus: HealthStatus = {
    status,
    timestamp: new Date().toISOString(),
    checks,
    version: process.env.npm_package_version || '1.0.0',
    uptime: process.uptime(),
  };

  // Return 503 if unhealthy
  const statusCode = status === 'healthy' ? 200 : 503;
  
  res.setHeader('Cache-Control', 'no-cache');
  res.status(statusCode).json(healthStatus);
}
```

### Metrics Collection

```typescript
// lib/metrics/collector.ts
import { redis } from '@/lib/redis';

export class MetricsCollector {
  async incrementCounter(metric: string, labels: Record<string, string> = {}) {
    const key = this.getMetricKey(metric, labels);
    await redis.incr(key);
    await redis.expire(key, 86400); // 24 hours
  }

  async recordHistogram(metric: string, value: number, labels: Record<string, string> = {}) {
    const key = this.getMetricKey(metric, labels);
    await redis.lpush(`${key}:values`, value);
    await redis.ltrim(`${key}:values`, 0, 999); // Keep last 1000 values
    await redis.expire(`${key}:values`, 86400);
  }

  async recordGauge(metric: string, value: number, labels: Record<string, string> = {}) {
    const key = this.getMetricKey(metric, labels);
    await redis.set(key, value);
    await redis.expire(key, 86400);
  }

  private getMetricKey(metric: string, labels: Record<string, string>): string {
    const labelStr = Object.entries(labels)
      .map(([k, v]) => `${k}=${v}`)
      .join(',');
    return `metrics:${metric}${labelStr ? `:${labelStr}` : ''}`;
  }

  async getMetrics(): Promise<Record<string, any>> {
    const keys = await redis.keys('metrics:*');
    const metrics: Record<string, any> = {};

    for (const key of keys) {
      const type = await redis.type(key);
      if (type === 'string') {
        metrics[key] = await redis.get(key);
      } else if (type === 'list') {
        const values = await redis.lrange(key, 0, -1);
        metrics[key] = values.map(Number);
      }
    }

    return metrics;
  }
}

export const metrics = new MetricsCollector();

// Metrics middleware
export function withMetrics() {
  return (handler: (req: NextApiRequest, res: NextApiResponse) => Promise<void>) => {
    return async (req: NextApiRequest, res: NextApiResponse) => {
      const startTime = Date.now();
      
      try {
        await handler(req, res);
        
        const duration = Date.now() - startTime;
        const labels = {
          method: req.method!,
          route: req.url!.split('?')[0],
          status: res.statusCode.toString(),
        };

        await metrics.incrementCounter('api_requests_total', labels);
        await metrics.recordHistogram('api_request_duration_ms', duration, labels);
        
      } catch (error) {
        const duration = Date.now() - startTime;
        const labels = {
          method: req.method!,
          route: req.url!.split('?')[0],
          status: '500',
        };

        await metrics.incrementCounter('api_requests_total', labels);
        await metrics.recordHistogram('api_request_duration_ms', duration, labels);
        
        throw error;
      }
    };
  };
}
```

---

## Implementation Examples

### Complete API Route Example

```typescript
// pages/api/teams/[teamId]/documents/[id]/index.ts
import { NextApiRequest, NextApiResponse } from "next";
import { getServerSession } from "next-auth/next";
import { z } from "zod";

import { authOptions } from "@/pages/api/auth/[...nextauth]";
import { errorhandler, DocumentError } from "@/lib/errorHandler";
import { ratelimit } from "@/lib/redis";
import { getTeamWithUsersAndDocument } from "@/lib/team/helper";
import { deleteFile } from "@/lib/files/delete-file-server";
import { serializeFileSize } from "@/lib/utils";
import { withLogging } from "@/lib/logging/logger";
import { withMetrics } from "@/lib/metrics/collector";
import { CustomUser } from "@/lib/types";
import prisma from "@/lib/prisma";

// Input validation schemas
const updateDocumentSchema = z.object({
  folderId: z.string().uuid().nullable().optional(),
  currentPathName: z.string().optional(),
});

// Compose middleware
const withMiddleware = (handler: Function) => 
  withLogging()(withMetrics()(handler));

export default withMiddleware(async function handler(
  req: NextApiRequest,
  res: NextApiResponse,
) {
  const { teamId, id: docId } = req.query as { teamId: string; id: string };

  // Method handling
  switch (req.method) {
    case "GET":
      return await handleGet(req, res, teamId, docId);
    case "PUT":
      return await handlePut(req, res, teamId, docId);
    case "DELETE":
      return await handleDelete(req, res, teamId, docId);
    default:
      res.setHeader("Allow", ["GET", "PUT", "DELETE"]);
      return res.status(405).json({
        success: false,
        error: {
          code: "METHOD_NOT_ALLOWED",
          message: `Method ${req.method} Not Allowed`,
        },
      });
  }
});

async function handleGet(
  req: NextApiRequest,
  res: NextApiResponse,
  teamId: string,
  docId: string,
) {
  // Authentication
  const session = await getServerSession(req, res, authOptions);
  if (!session) {
    return res.status(401).json({
      success: false,
      error: { code: "UNAUTHORIZED", message: "Authentication required" },
    });
  }

  const userId = (session.user as CustomUser).id;

  try {
    // Rate limiting
    const { success, limit, remaining, reset } = await ratelimit(120, "1 m")
      .limit(`doc:${docId}:team:${teamId}:user:${userId}`);

    res.setHeader("X-RateLimit-Limit", limit.toString());
    res.setHeader("X-RateLimit-Remaining", remaining.toString());
    res.setHeader("X-RateLimit-Reset", reset.toString());

    if (!success) {
      return res.status(429).json({
        success: false,
        error: { code: "RATE_LIMIT_EXCEEDED", message: "Too many requests" },
      });
    }

    // Authorization & data fetching
    const { document } = await getTeamWithUsersAndDocument({
      teamId,
      userId,
      docId,
      options: {
        include: {
          versions: {
            where: { isPrimary: true },
            orderBy: { createdAt: "desc" },
            take: 1,
          },
          folder: {
            select: { name: true, path: true },
          },
          datarooms: {
            select: {
              dataroom: { select: { id: true, name: true } },
              folder: { select: { id: true, name: true, path: true } },
            },
          },
        },
      },
    });

    if (!document || !document.versions || document.versions.length === 0) {
      return res.status(404).json({
        success: false,
        error: { code: "DOCUMENT_NOT_FOUND", message: "Document not found" },
      });
    }

    // Additional data processing
    const pages = await prisma.documentPage.findMany({
      where: { versionId: document.versions[0].id },
      select: { pageLinks: true },
    });

    const hasPageLinks = pages.some(
      (page) =>
        page.pageLinks &&
        Array.isArray(page.pageLinks) &&
        (page.pageLinks as any[]).length > 0,
    );

    return res.status(200).json({
      success: true,
      data: serializeFileSize({ ...document, hasPageLinks }),
    });
  } catch (error) {
    console.error("GET document error:", error);
    errorhandler(error, res);
  }
}

async function handlePut(
  req: NextApiRequest,
  res: NextApiResponse,
  teamId: string,
  docId: string,
) {
  // Authentication
  const session = await getServerSession(req, res, authOptions);
  if (!session) {
    return res.status(401).json({
      success: false,
      error: { code: "UNAUTHORIZED", message: "Authentication required" },
    });
  }

  const userId = (session.user as CustomUser).id;

  try {
    // Input validation
    const { folderId, currentPathName } = updateDocumentSchema.parse(req.body);

    // Update document
    const document = await prisma.document.update({
      where: {
        id: docId,
        teamId: teamId,
        team: {
          users: {
            some: { userId: userId },
          },
        },
      },
      data: { folderId },
      select: {
        folder: {
          select: { path: true },
        },
      },
    });

    if (!document) {
      return res.status(404).json({
        success: false,
        error: { code: "DOCUMENT_NOT_FOUND", message: "Document not found" },
      });
    }

    return res.status(200).json({
      success: true,
      data: {
        message: "Document updated successfully",
        newPath: document.folder?.path,
        oldPath: currentPathName,
      },
    });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(422).json({
        success: false,
        error: {
          code: "VALIDATION_ERROR",
          message: "Invalid input data",
          details: error.errors,
        },
      });
    }
    console.error("PUT document error:", error);
    errorhandler(error, res);
  }
}

async function handleDelete(
  req: NextApiRequest,
  res: NextApiResponse,
  teamId: string,
  docId: string,
) {
  // Authentication
  const session = await getServerSession(req, res, authOptions);
  if (!session) {
    return res.status(401).json({
      success: false,
      error: { code: "UNAUTHORIZED", message: "Authentication required" },
    });
  }

  const userId = (session.user as CustomUser).id;

  try {
    // Transaction for atomic delete
    await prisma.$transaction(async (tx) => {
      // Get document with versions
      const documentVersions = await tx.document.findUnique({
        where: {
          id: docId,
          teamId: teamId,
          team: {
            users: {
              some: { userId: userId },
            },
          },
        },
        include: {
          versions: {
            select: {
              id: true,
              file: true,
              type: true,
              storageType: true,
            },
          },
        },
      });

      if (!documentVersions) {
        throw new DocumentError("Document not found");
      }

      // Delete files from storage (if not Notion)
      if (documentVersions.type !== "notion") {
        for (const version of documentVersions.versions) {
          await deleteFile({
            type: version.storageType,
            data: version.file,
            teamId,
          });
        }
      }

      // Delete from database
      await tx.document.delete({
        where: { id: docId },
      });
    });

    return res.status(204).end();
  } catch (error) {
    console.error("DELETE document error:", error);
    errorhandler(error, res);
  }
}
```

### Utility Functions

```typescript
// lib/utils/api-helpers.ts

export function createApiResponse<T>(
  data: T,
  meta?: any
): { success: true; data: T; meta?: any } {
  return {
    success: true,
    data: serializeResponse(data),
    ...(meta && { meta }),
  };
}

export function createErrorResponse(
  code: string,
  message: string,
  details?: any
): { success: false; error: { code: string; message: string; details?: any } } {
  return {
    success: false,
    error: {
      code,
      message,
      ...(details && { details }),
    },
  };
}

export function parseQueryParams<T>(
  query: any,
  schema: z.ZodSchema<T>
): T {
  try {
    return schema.parse(query);
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new ValidationError("Invalid query parameters", error.errors);
    }
    throw error;
  }
}

export function getAuthUserId(req: NextApiRequest): string | null {
  const authHeader = req.headers.authorization;
  
  if (authHeader?.startsWith("Bearer ")) {
    // Handle API token auth
    const token = authHeader.replace("Bearer ", "");
    // Validate and return user ID from token
    return getUserIdFromToken(token);
  }
  
  // Handle session auth
  const session = getSession(req);
  return session?.user?.id || null;
}
```

---

This comprehensive guide provides production-ready API patterns based on Papermark's implementation. The patterns cover authentication, validation, error handling, rate limiting, file processing, database optimization, webhooks, testing, monitoring, and more. Each pattern is designed for scalability, security, and maintainability in production environments.

Use these patterns as building blocks for your own API implementations, adapting them to your specific requirements while maintaining the core principles of reliability, security, and performance.