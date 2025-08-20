# Reusable Patterns & Libraries Guide

This comprehensive guide extracts production-ready patterns, libraries, and architectural decisions from the Papermark codebase, **adapted for your primary tech stack: Supabase (Database), Umami (Analytics), and Supabase Storage (File Storage)**. All patterns are battle-tested in a real-world SaaS application and optimized for the Supabase ecosystem.

---

## Table of Contents
1. [Core Technology Stack](#core-technology-stack)
2. [Project Architecture](#project-architecture)
3. [UI/UX Component Library](#uiux-component-library)
4. [Database & Data Management](#database--data-management)
5. [Authentication & Security](#authentication--security)
6. [File Storage & Processing](#file-storage--processing)
7. [Background Jobs & Processing](#background-jobs--processing)
8. [API Design Patterns](#api-design-patterns)
9. [Monitoring & Analytics](#monitoring--analytics)
10. [Configuration Management](#configuration-management)
11. [Utility Functions](#utility-functions)
12. [Performance Optimization](#performance-optimization)
13. [Security Best Practices](#security-best-practices)
14. [Deployment & Infrastructure](#deployment--infrastructure)

---

## Core Technology Stack

### Frontend Foundation
```json
{
  "framework": "Next.js 14.2.31",
  "ui_framework": "React 18.3.1",
  "styling": "Tailwind CSS 3.4.17",
  "ui_components": "@radix-ui/*",
  "state_management": "SWR 2.3.6",
  "type_safety": "TypeScript 5",
  "form_handling": "React Hook Form",
  "animation": "Framer Motion (motion 12.23.12)"
}
```

### Backend Infrastructure (Supabase-Optimized)
```json
{
  "runtime": "Node.js >=18.18.0",
  "api_routes": "Next.js API Routes",
  "database": "Supabase PostgreSQL",
  "orm": "Prisma 6.5.0 + Supabase Client",
  "authentication": "Supabase Auth + NextAuth.js",
  "realtime": "Supabase Realtime",
  "storage": "Supabase Storage",
  "analytics": "Umami Analytics",
  "background_jobs": "Trigger.dev 3.3.17",
  "email": "Resend 4.8.0",
  "edge_functions": "Supabase Edge Functions"
}
```

### Key Libraries Worth Adopting (Supabase Stack)
```typescript
// Essential utility libraries
import { clsx } from "clsx";
import { twMerge } from "tailwind-merge";
import { nanoid } from "nanoid";
import ms from "ms";
import slugify from "@sindresorhus/slugify";

// Supabase ecosystem
import { createClient } from "@supabase/supabase-js";
import { createServerClient } from "@supabase/ssr";
import { Database } from "@/types/supabase";

// Data processing and validation
import { z } from "zod";
import bcryptjs from "bcryptjs";
import jsonwebtoken from "jsonwebtoken";
import sanitizeHtml from "sanitize-html";

// File handling with Supabase Storage
import exceljs from "exceljs";
import { PDFDocument } from "pdf-lib";

// Analytics (Umami)
import { umami } from "@umami/node";
```

---

## Project Architecture

### Directory Structure Pattern
```
project/
├── app/                      # Next.js 14 app router
│   ├── (auth)/              # Route groups
│   ├── api/                 # API endpoints
│   └── layout.tsx           # Root layout
├── pages/                   # Pages router (hybrid approach)
│   ├── api/                 # API routes
│   └── _app.tsx             # App wrapper
├── components/              # Reusable UI components
│   ├── ui/                  # Base UI primitives
│   ├── shared/              # Shared components
│   └── layouts/             # Layout components
├── lib/                     # Business logic & utilities
│   ├── api/                 # API helpers
│   ├── auth/                # Authentication logic
│   ├── utils/               # Utility functions
│   ├── hooks/               # Custom React hooks
│   └── types/               # TypeScript types
├── prisma/                  # Database schema & migrations
├── public/                  # Static assets
└── docs/                    # Documentation
```

### Next.js Configuration Pattern
```javascript
// next.config.mjs
const nextConfig = {
  reactStrictMode: true,
  pageExtensions: ["js", "jsx", "ts", "tsx", "mdx"],
  images: {
    minimumCacheTTL: 2592000, // 30 days
    remotePatterns: prepareRemotePatterns(),
  },
  skipTrailingSlashRedirect: true,
  experimental: {
    outputFileTracingIncludes: {
      "/api/mupdf/*": ["./node_modules/mupdf/dist/*.wasm"],
    },
    missingSuspenseWithCSRBailout: false,
  },
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [
          { key: "Referrer-Policy", value: "no-referrer-when-downgrade" },
          { key: "X-DNS-Prefetch-Control", value: "on" },
          { key: "X-Frame-Options", value: "SAMEORIGIN" },
          {
            key: "Content-Security-Policy-Report-Only",
            value: generateCSPHeader(),
          },
        ],
      },
    ];
  },
};
```

### Middleware Pattern
```typescript
// middleware.ts
export default async function middleware(req: NextRequest) {
  const path = req.nextUrl.pathname;
  const host = req.headers.get("host");

  // Handle analytics routes
  if (isAnalyticsPath(path)) {
    return PostHogMiddleware(req);
  }

  // Handle incoming webhooks
  if (isWebhookPath(host)) {
    return IncomingWebhookMiddleware(req);
  }

  // Handle custom domains
  if (isCustomDomain(host || "")) {
    return DomainMiddleware(req);
  }

  // Handle authenticated routes
  if (!path.startsWith("/view/") && !path.startsWith("/verify")) {
    return AppMiddleware(req);
  }

  return NextResponse.next();
}

export const config = {
  matcher: [
    "/((?!api/|_next/|_static|vendor|_icons|_vercel|favicon.ico|sitemap.xml).*)",
  ],
};
```

---

## UI/UX Component Library

### Tailwind Configuration Pattern
```javascript
// tailwind.config.js
module.exports = {
  darkMode: ["class"],
  content: [
    "./pages/**/*.{ts,tsx}",
    "./components/**/*.{ts,tsx}",
    "./app/**/*.{ts,tsx}",
    "./node_modules/@tremor/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        // ... semantic color system
      },
      keyframes: {
        "scale-in": {
          "0%": { transform: "scale(0.95)" },
          "100%": { transform: "scale(1)" },
        },
        "fade-in": {
          "0%": { opacity: "0" },
          "100%": { opacity: "1" },
        },
      },
    },
  },
  plugins: [
    require("@tailwindcss/forms"),
    require("tailwindcss-animate"),
    require("@tailwindcss/typography"),
    require("tailwind-scrollbar-hide"),
  ],
};
```

### Utility Class Name Helper
```typescript
// lib/utils.ts
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// Usage:
// <div className={cn("base-styles", condition && "conditional-styles", className)} />
```

### Component Composition Pattern
```typescript
// components/ui/button.tsx
import * as React from "react";
import { Slot } from "@radix-ui/react-slot";
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils";

const buttonVariants = cva(
  "inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  },
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button";
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    );
  }
);
Button.displayName = "Button";

export { Button, buttonVariants };
```

### Animation Constants
```typescript
// lib/constants.ts
export const FADE_IN_ANIMATION_SETTINGS = {
  initial: { opacity: 0 },
  animate: { opacity: 1 },
  exit: { opacity: 0 },
  transition: { duration: 0.2 },
};

export const STAGGER_CHILD_VARIANTS = {
  hidden: { opacity: 0, y: 20 },
  show: {
    opacity: 1,
    y: 0,
    transition: { duration: 0.4, type: "spring" as const },
  },
};
```

---

## Database & Data Management (Supabase)

### Supabase Client Setup
```typescript
// lib/supabase/client.ts
import { createClient } from "@supabase/supabase-js";
import { Database } from "@/types/supabase";

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!;
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!;

export const supabase = createClient<Database>(supabaseUrl, supabaseAnonKey);

// Server-side client with service role key
const supabaseServiceRoleKey = process.env.SUPABASE_SERVICE_ROLE_KEY!;

export const supabaseAdmin = createClient<Database>(
  supabaseUrl,
  supabaseServiceRoleKey,
  {
    auth: {
      autoRefreshToken: false,
      persistSession: false,
    },
  }
);
```

### Server Components Client
```typescript
// lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";
import { Database } from "@/types/supabase";

export function createClient() {
  const cookieStore = cookies();

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) {
          return cookieStore.get(name)?.value;
        },
        set(name: string, value: string, options: any) {
          try {
            cookieStore.set({ name, value, ...options });
          } catch (error) {
            // Server component cookie setting limitation
          }
        },
        remove(name: string, options: any) {
          try {
            cookieStore.set({ name, value: "", ...options });
          } catch (error) {
            // Server component cookie setting limitation
          }
        },
      },
    }
  );
}
```

### Database Schema with Row Level Security
```sql
-- Example schema with RLS policies
-- supabase/migrations/001_initial_schema.sql

-- Enable RLS
alter table if exists public.documents enable row level security;
alter table if exists public.teams enable row level security;
alter table if exists public.links enable row level security;

-- Documents policies
create policy "Users can view documents from their teams"
  on public.documents for select
  using (
    team_id in (
      select team_id from public.team_members 
      where user_id = auth.uid()
    )
  );

create policy "Users can insert documents to their teams"
  on public.documents for insert
  with check (
    team_id in (
      select team_id from public.team_members 
      where user_id = auth.uid() 
      and role in ('owner', 'admin', 'member')
    )
  );

-- Teams policies
create policy "Users can view their team memberships"
  on public.teams for select
  using (
    id in (
      select team_id from public.team_members 
      where user_id = auth.uid()
    )
  );
```

### Hybrid Prisma + Supabase Pattern
```typescript
// lib/database.ts
import { PrismaClient } from "@prisma/client";
import { createClient } from "@supabase/supabase-js";

// Use Prisma for complex queries and type safety
const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const prisma = globalForPrisma.prisma ?? 
  new PrismaClient({
    datasources: {
      db: {
        url: process.env.SUPABASE_DATABASE_URL,
      },
    },
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;

// Use Supabase client for auth, realtime, and RLS
export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

// Prisma schema configuration for Supabase
// prisma/schema.prisma
/*
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("SUPABASE_DATABASE_URL")
  directUrl = env("SUPABASE_DIRECT_URL")
}
*/
```

### SWR Data Fetching with Supabase
```typescript
// lib/swr/use-documents.ts
import useSWR from "swr";
import { supabase } from "@/lib/supabase/client";
import { Database } from "@/types/supabase";

type Document = Database['public']['Tables']['documents']['Row'];

export function useDocuments(teamId?: string) {
  const { data, error, mutate } = useSWR<Document[]>(
    teamId ? `documents-${teamId}` : null,
    async () => {
      const { data, error } = await supabase
        .from('documents')
        .select('*')
        .eq('team_id', teamId!)
        .order('created_at', { ascending: false });

      if (error) throw error;
      return data;
    },
    {
      revalidateOnFocus: false,
      dedupingInterval: 30000,
    }
  );

  return {
    documents: data,
    loading: !error && !data,
    error,
    mutate,
  };
}

// Realtime subscription hook
export function useDocumentsRealtime(teamId?: string) {
  const { data, mutate } = useDocuments(teamId);

  useEffect(() => {
    if (!teamId) return;

    const channel = supabase
      .channel('documents-changes')
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'documents',
          filter: `team_id=eq.${teamId}`,
        },
        (payload) => {
          // Optimistically update local state
          mutate();
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [teamId, mutate]);

  return { documents: data, mutate };
}
```

### Optimistic Updates Pattern
```typescript
// hooks/use-optimistic-update.ts
import { useMemo } from "react";
import { mutate } from "swr";

export function useOptimisticUpdate<T>(
  key: string,
  data: T[] | undefined,
  updateFn: (items: T[], newItem: Partial<T>) => T[]
) {
  const optimisticUpdate = useMemo(
    () => ({
      add: async (newItem: Partial<T>) => {
        if (!data) return;
        
        // Optimistically update UI
        await mutate(key, updateFn(data, newItem), false);
        
        try {
          // Make API call
          const response = await fetch("/api/endpoint", {
            method: "POST",
            body: JSON.stringify(newItem),
          });
          
          if (!response.ok) throw new Error("Failed to update");
          
          // Revalidate data
          mutate(key);
        } catch (error) {
          // Revert optimistic update on error
          mutate(key);
          throw error;
        }
      },
    }),
    [key, data, updateFn]
  );

  return optimisticUpdate;
}
```

---

## Authentication & Security

### NextAuth.js Configuration
```typescript
// pages/api/auth/[...nextauth].ts
import NextAuth from "next-auth";
import GoogleProvider from "next-auth/providers/google";
import EmailProvider from "next-auth/providers/email";
import { PrismaAdapter } from "@next-auth/prisma-adapter";
import prisma from "@/lib/prisma";

export default NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    EmailProvider({
      server: {
        host: process.env.EMAIL_SERVER_HOST,
        port: process.env.EMAIL_SERVER_PORT,
        auth: {
          user: process.env.EMAIL_SERVER_USER,
          pass: process.env.EMAIL_SERVER_PASSWORD,
        },
      },
      from: process.env.EMAIL_FROM,
    }),
  ],
  session: {
    strategy: "jwt",
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },
  callbacks: {
    jwt: async ({ token, user, account }) => {
      if (user) {
        token.user = user;
      }
      return token;
    },
    session: async ({ session, token }) => {
      if (token?.user) {
        session.user = token.user;
      }
      return session;
    },
  },
  pages: {
    signIn: "/login",
    verifyRequest: "/auth/verify-request",
    error: "/auth/error",
  },
});
```

### Password Security Utilities
```typescript
// lib/utils.ts
import bcrypt from "bcryptjs";
import crypto from "crypto";

export async function hashPassword(password: string): Promise<string> {
  const saltRounds = 10;
  return await bcrypt.hash(password, saltRounds);
}

export async function checkPassword(
  password: string,
  hashedPassword: string,
): Promise<boolean> {
  return await bcrypt.compare(password, hashedPassword);
}

// Document password encryption
export async function generateEncryptedPassword(
  password: string,
): Promise<string> {
  if (!password) return "";
  
  const textParts: string[] = password.split(":");
  if (textParts.length === 2) return password; // Already encrypted
  
  const encryptedKey: string = crypto
    .createHash("sha256")
    .update(String(process.env.NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY))
    .digest("base64")
    .substring(0, 32);
    
  const IV: Buffer = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv("aes-256-ctr", encryptedKey, IV);
  let encryptedText: string = cipher.update(password, "utf8", "hex");
  encryptedText += cipher.final("hex");
  
  return IV.toString("hex") + ":" + encryptedText;
}
```

### Rate Limiting with Upstash
```typescript
// lib/rate-limit.ts
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, "10 s"),
  analytics: true,
});

// Usage in API routes
export default async function handler(req: NextRequest) {
  const ip = req.ip ?? "127.0.0.1";
  const { success } = await ratelimit.limit(ip);
  
  if (!success) {
    return new Response("Rate limit exceeded", { status: 429 });
  }
  
  // Handle request...
}
```

### JWT Token Management
```typescript
// lib/utils/generate-jwt.ts
import jwt from "jsonwebtoken";

export function generateJWT(payload: any, expiresIn = "1h") {
  return jwt.sign(payload, process.env.NEXTAUTH_SECRET!, { expiresIn });
}

export function verifyJWT(token: string) {
  try {
    return jwt.verify(token, process.env.NEXTAUTH_SECRET!);
  } catch (error) {
    return null;
  }
}
```

---

## File Storage & Processing (Supabase Storage)

### Supabase Storage Client Setup
```typescript
// lib/storage/supabase-storage.ts
import { supabase, supabaseAdmin } from "@/lib/supabase/client";

export class SupabaseStorageService {
  private bucket: string;

  constructor(bucket = "documents") {
    this.bucket = bucket;
  }

  async uploadFile(
    file: File, 
    path: string, 
    options?: {
      upsert?: boolean;
      cacheControl?: string;
      contentType?: string;
    }
  ) {
    const { data, error } = await supabase.storage
      .from(this.bucket)
      .upload(path, file, {
        cacheControl: options?.cacheControl || "3600",
        upsert: options?.upsert || false,
        contentType: options?.contentType || file.type,
      });

    if (error) throw error;
    return data;
  }

  async getPublicUrl(path: string) {
    const { data } = supabase.storage
      .from(this.bucket)
      .getPublicUrl(path);

    return data.publicUrl;
  }

  async getSignedUrl(path: string, expiresIn = 3600) {
    const { data, error } = await supabase.storage
      .from(this.bucket)
      .createSignedUrl(path, expiresIn);

    if (error) throw error;
    return data.signedUrl;
  }

  async deleteFile(path: string) {
    const { error } = await supabase.storage
      .from(this.bucket)
      .remove([path]);

    if (error) throw error;
  }

  // Admin operations (requires service role key)
  async adminUploadFile(file: File, path: string) {
    const { data, error } = await supabaseAdmin.storage
      .from(this.bucket)
      .upload(path, file, { upsert: true });

    if (error) throw error;
    return data;
  }
}

export const documentStorage = new SupabaseStorageService("documents");
export const profileStorage = new SupabaseStorageService("profiles");
```

### File Upload Hook with Supabase Storage
```typescript
// hooks/use-supabase-upload.ts
import { useState, useCallback } from "react";
import { documentStorage } from "@/lib/storage/supabase-storage";
import { nanoid } from "@/lib/utils";

export function useSupabaseUpload() {
  const [progress, setProgress] = useState(0);
  const [uploading, setUploading] = useState(false);

  const uploadFile = useCallback(async (
    file: File, 
    folder = "uploads",
    options?: {
      onProgress?: (progress: number) => void;
      filename?: string;
    }
  ) => {
    setUploading(true);
    setProgress(0);

    try {
      // Generate unique filename
      const fileExtension = file.name.split('.').pop();
      const filename = options?.filename || 
        `${nanoid()}.${fileExtension}`;
      const filePath = `${folder}/${filename}`;

      // Simulate progress for user feedback
      const progressInterval = setInterval(() => {
        setProgress(prev => {
          const newProgress = Math.min(prev + 10, 90);
          options?.onProgress?.(newProgress);
          return newProgress;
        });
      }, 100);

      // Upload to Supabase Storage
      const uploadResult = await documentStorage.uploadFile(
        file, 
        filePath,
        {
          contentType: file.type,
          cacheControl: "3600",
        }
      );

      clearInterval(progressInterval);
      setProgress(100);
      options?.onProgress?.(100);

      // Get public URL
      const publicUrl = await documentStorage.getPublicUrl(filePath);

      return {
        path: uploadResult.path,
        url: publicUrl,
        fullPath: uploadResult.fullPath,
      };
    } catch (error) {
      console.error("Upload failed:", error);
      throw error;
    } finally {
      setUploading(false);
      setTimeout(() => setProgress(0), 1000);
    }
  }, []);

  return { uploadFile, progress, uploading };
}
```

### Drag & Drop Upload Zone
```typescript
// components/upload-zone.tsx
import { useDropzone } from "react-dropzone";
import { cn } from "@/lib/utils";

interface UploadZoneProps {
  onDrop: (files: File[]) => void;
  accept?: Record<string, string[]>;
  maxSize?: number;
  className?: string;
}

export function UploadZone({ 
  onDrop, 
  accept, 
  maxSize, 
  className 
}: UploadZoneProps) {
  const { getRootProps, getInputProps, isDragActive, isDragReject } = useDropzone({
    onDrop,
    accept,
    maxSize,
    multiple: false,
  });

  return (
    <div
      {...getRootProps()}
      className={cn(
        "border-2 border-dashed rounded-lg p-8 text-center cursor-pointer transition-colors",
        isDragActive && "border-primary bg-primary/10",
        isDragReject && "border-destructive bg-destructive/10",
        className
      )}
    >
      <input {...getInputProps()} />
      <div className="space-y-2">
        <div className="text-lg font-medium">
          {isDragActive ? "Drop files here" : "Choose files or drag and drop"}
        </div>
        <div className="text-sm text-muted-foreground">
          Support for PDF, DOCX, PPTX, and more
        </div>
      </div>
    </div>
  );
}
```

### TUS Resumable Upload Pattern
```typescript
// lib/files/tus-upload.ts
import * as tus from "tus-js-client";

export function createTusUpload(
  file: File,
  {
    endpoint,
    onProgress,
    onSuccess,
    onError,
    metadata = {},
  }: {
    endpoint: string;
    onProgress?: (bytesUploaded: number, bytesTotal: number) => void;
    onSuccess?: (upload: tus.Upload) => void;
    onError?: (error: Error) => void;
    metadata?: Record<string, string>;
  }
) {
  const upload = new tus.Upload(file, {
    endpoint,
    retryDelays: [0, 3000, 5000, 10000, 20000],
    metadata: {
      filename: file.name,
      filetype: file.type,
      ...metadata,
    },
    onError: (error) => {
      console.error("Upload failed:", error);
      onError?.(error);
    },
    onProgress: (bytesUploaded, bytesTotal) => {
      const percentage = ((bytesUploaded / bytesTotal) * 100).toFixed(2);
      console.log(bytesUploaded, bytesTotal, percentage + "%");
      onProgress?.(bytesUploaded, bytesTotal);
    },
    onSuccess: () => {
      console.log("Upload completed successfully");
      onSuccess?.(upload);
    },
  });

  return upload;
}
```

---

## Background Jobs & Processing

### Trigger.dev Task Configuration
```typescript
// trigger.config.ts
import type { TriggerConfig } from "@trigger.dev/sdk/v3";

export const config: TriggerConfig = {
  project: "proj_xxx",
  logLevel: "info",
  retries: {
    enabledInDev: true,
    default: {
      maxAttempts: 3,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 10000,
      factor: 2,
      randomize: true,
    },
  },
  dirs: ["./lib/trigger"],
  build: {
    extensions: [
      {
        name: "Prisma",
        path: "@trigger.dev/build/extensions/prisma",
        config: {
          directUrlEnvVarName: "POSTGRES_PRISMA_URL_NON_POOLING",
          schema: "prisma/schema",
        },
      },
    ],
  },
};
```

### Background Task Pattern
```typescript
// lib/trigger/process-document.ts
import { logger, task } from "@trigger.dev/sdk/v3";
import { updateStatus } from "@/lib/utils/generate-trigger-status";

export const processDocumentTask = task({
  id: "process-document",
  retry: { maxAttempts: 3 },
  queue: { concurrencyLimit: 5 },
  run: async (payload: {
    documentId: string;
    teamId: string;
    userId: string;
  }) => {
    updateStatus({ progress: 0, text: "Starting document processing..." });

    try {
      const { documentId, teamId, userId } = payload;
      
      // Step 1: Validate document
      updateStatus({ progress: 20, text: "Validating document..." });
      const document = await prisma.document.findUnique({
        where: { id: documentId },
      });
      
      if (!document) {
        throw new Error("Document not found");
      }

      // Step 2: Process file
      updateStatus({ progress: 50, text: "Processing file..." });
      const processedFile = await processFile(document.file);

      // Step 3: Update database
      updateStatus({ progress: 80, text: "Updating database..." });
      await prisma.document.update({
        where: { id: documentId },
        data: { processedFile },
      });

      updateStatus({ progress: 100, text: "Processing complete!" });
      
      logger.info("Document processed successfully", {
        documentId,
        teamId,
        userId,
      });

      return { success: true, documentId };
    } catch (error) {
      logger.error("Document processing failed", {
        error: error instanceof Error ? error.message : String(error),
        payload,
      });
      throw error;
    }
  },
});
```

### Scheduled Cron Jobs
```typescript
// lib/trigger/cleanup-tasks.ts
import { logger, schedules } from "@trigger.dev/sdk/v3";

export const cleanupExpiredData = schedules.task({
  id: "cleanup-expired-data",
  cron: "0 2 * * *", // Daily at 2 AM UTC
  run: async (payload) => {
    logger.info("Starting cleanup job", {
      timestamp: payload.timestamp,
    });

    try {
      // Cleanup expired sessions
      const expiredSessions = await prisma.session.deleteMany({
        where: {
          expires: {
            lt: new Date(),
          },
        },
      });

      // Cleanup old verification tokens
      const expiredTokens = await prisma.verificationToken.deleteMany({
        where: {
          expires: {
            lt: new Date(),
          },
        },
      });

      logger.info("Cleanup completed", {
        deletedSessions: expiredSessions.count,
        deletedTokens: expiredTokens.count,
      });

      return {
        success: true,
        deletedSessions: expiredSessions.count,
        deletedTokens: expiredTokens.count,
      };
    } catch (error) {
      logger.error("Cleanup failed", {
        error: error instanceof Error ? error.message : String(error),
      });
      throw error;
    }
  },
});
```

---

## API Design Patterns

### Consistent API Response Pattern
```typescript
// lib/api/response-helpers.ts
export type ApiResponse<T = any> = {
  success: boolean;
  data?: T;
  error?: string;
  message?: string;
};

export function successResponse<T>(data: T, message?: string): ApiResponse<T> {
  return {
    success: true,
    data,
    message,
  };
}

export function errorResponse(error: string, data?: any): ApiResponse {
  return {
    success: false,
    error,
    data,
  };
}

// Usage in API routes
export default async function handler(req: NextRequest) {
  try {
    const result = await someOperation();
    return Response.json(successResponse(result));
  } catch (error) {
    return Response.json(
      errorResponse(error instanceof Error ? error.message : "Unknown error"),
      { status: 500 }
    );
  }
}
```

### Request Validation with Zod
```typescript
// lib/zod/schemas/documents.ts
import { z } from "zod";

export const createDocumentSchema = z.object({
  name: z.string().min(1, "Name is required").max(100),
  description: z.string().optional(),
  folderId: z.string().optional(),
  type: z.enum(["pdf", "docx", "pptx", "xlsx"]),
});

export const updateDocumentSchema = createDocumentSchema.partial();

// Usage in API route
export default async function handler(req: NextRequest) {
  try {
    const body = await req.json();
    const validatedData = createDocumentSchema.parse(body);
    
    // Process validated data...
    return Response.json(successResponse(result));
  } catch (error) {
    if (error instanceof z.ZodError) {
      return Response.json(
        errorResponse("Validation failed", error.errors),
        { status: 400 }
      );
    }
    throw error;
  }
}
```

### Authentication Middleware Pattern
```typescript
// lib/api/auth/auth-middleware.ts
import { getToken } from "next-auth/jwt";
import { NextRequest } from "next/server";

export async function withAuth(
  handler: (req: NextRequest, context: { user: any; team: any }) => Promise<Response>
) {
  return async (req: NextRequest) => {
    const token = await getToken({
      req,
      secret: process.env.NEXTAUTH_SECRET,
    });

    if (!token?.email) {
      return Response.json(
        errorResponse("Unauthorized"),
        { status: 401 }
      );
    }

    // Get user and team information
    const user = await prisma.user.findUnique({
      where: { email: token.email },
      include: { teams: true },
    });

    if (!user) {
      return Response.json(
        errorResponse("User not found"),
        { status: 404 }
      );
    }

    const team = user.teams[0]?.team; // Assuming first team for simplicity

    return handler(req, { user, team });
  };
}

// Usage
export const POST = withAuth(async (req, { user, team }) => {
  // Handler logic with authenticated user and team
});
```

### API Rate Limiting
```typescript
// lib/api/rate-limit.ts
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export const apiRateLimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(100, "1 h"), // 100 requests per hour
  analytics: true,
});

export const uploadRateLimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(10, "1 h"), // 10 uploads per hour
  analytics: true,
});

// Usage in API routes
export default async function handler(req: NextRequest) {
  const ip = req.ip ?? "127.0.0.1";
  const { success, limit, reset, remaining } = await apiRateLimit.limit(ip);

  if (!success) {
    return Response.json(
      errorResponse("Rate limit exceeded"),
      {
        status: 429,
        headers: {
          "X-RateLimit-Limit": limit.toString(),
          "X-RateLimit-Remaining": remaining.toString(),
          "X-RateLimit-Reset": new Date(reset).toISOString(),
        },
      }
    );
  }

  // Handle request...
}
```

---

## Monitoring & Analytics

### PostHog Integration Pattern
```typescript
// lib/posthog.ts
import { PostHog } from 'posthog-node';

export const posthog = new PostHog(
  process.env.NEXT_PUBLIC_POSTHOG_KEY!,
  {
    api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST,
    flushAt: 1,
    flushInterval: 0,
  }
);

// Client-side tracking
// components/providers/posthog-provider.tsx
'use client';
import posthog from 'posthog-js';
import { PostHogProvider } from 'posthog-js/react';

if (typeof window !== 'undefined') {
  posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
    api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST,
    capture_pageview: false,
  });
}

export function PHProvider({ children }: { children: React.ReactNode }) {
  return <PostHogProvider client={posthog}>{children}</PostHogProvider>;
}
```

### Event Tracking with Tinybird
```typescript
// lib/tinybird/publish.ts
import { z } from "zod";

const eventSchema = z.object({
  timestamp: z.string(),
  event_type: z.string(),
  user_id: z.string().optional(),
  session_id: z.string().optional(),
  properties: z.record(z.any()).optional(),
});

export async function publishEvent(event: z.infer<typeof eventSchema>) {
  if (!process.env.TINYBIRD_API_KEY) {
    console.warn("Tinybird API key not configured");
    return;
  }

  try {
    const validatedEvent = eventSchema.parse(event);
    
    const response = await fetch(
      `${process.env.TINYBIRD_BASE_URL}/v0/events`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${process.env.TINYBIRD_API_KEY}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify(validatedEvent),
      }
    );

    if (!response.ok) {
      throw new Error(`Tinybird API error: ${response.statusText}`);
    }
  } catch (error) {
    console.error("Failed to publish event to Tinybird:", error);
  }
}

// Usage
publishEvent({
  timestamp: new Date().toISOString(),
  event_type: "document_viewed",
  user_id: user.id,
  session_id: req.sessionId,
  properties: {
    document_id: document.id,
    team_id: team.id,
    view_duration: viewDuration,
  },
});
```

### Error Logging Pattern
```typescript
// lib/error-handler.ts
import { log } from "@/lib/utils";

export class AppError extends Error {
  public statusCode: number;
  public isOperational: boolean;

  constructor(message: string, statusCode = 500, isOperational = true) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;

    Error.captureStackTrace(this, this.constructor);
  }
}

export function handleError(error: Error, context?: Record<string, any>) {
  // Log to external service (Slack, Sentry, etc.)
  log({
    message: `Error: ${error.message}`,
    type: "error",
    mention: error instanceof AppError && error.statusCode >= 500,
  });

  // Log additional context
  if (context) {
    console.error("Error context:", context);
  }

  console.error(error.stack);
}

// Usage in API routes
export default async function handler(req: NextRequest) {
  try {
    // Handler logic...
  } catch (error) {
    handleError(error, {
      url: req.url,
      method: req.method,
      headers: Object.fromEntries(req.headers.entries()),
    });

    if (error instanceof AppError && error.isOperational) {
      return Response.json(
        errorResponse(error.message),
        { status: error.statusCode }
      );
    }

    return Response.json(
      errorResponse("Internal server error"),
      { status: 500 }
    );
  }
}
```

---

## Configuration Management

### Environment Variables Pattern
```bash
# .env.example
# Database
POSTGRES_PRISMA_URL=""
POSTGRES_PRISMA_URL_NON_POOLING=""
POSTGRES_PRISMA_SHADOW_URL=""

# Authentication
NEXTAUTH_URL=""
NEXTAUTH_SECRET=""
GOOGLE_CLIENT_ID=""
GOOGLE_CLIENT_SECRET=""

# File Storage
BLOB_READ_WRITE_TOKEN=""
NEXT_PRIVATE_UPLOAD_DISTRIBUTION_HOST=""

# Background Jobs
TRIGGER_PROJECT_ID=""
TRIGGER_API_KEY=""

# Analytics
NEXT_PUBLIC_POSTHOG_KEY=""
NEXT_PUBLIC_POSTHOG_HOST=""
TINYBIRD_API_KEY=""

# Email
RESEND_API_KEY=""
EMAIL_FROM=""

# Redis
UPSTASH_REDIS_REST_URL=""
UPSTASH_REDIS_REST_TOKEN=""

# Monitoring
PPMK_SLACK_WEBHOOK_URL=""
```

### Feature Flags Pattern
```typescript
// lib/feature-flags.ts
import { unstable_flag as flag } from "@vercel/flags/next";

export const showAdvancedFeatures = flag({
  key: "advanced-features",
  decide: () => process.env.NODE_ENV === "development",
});

export const enableBetaFeatures = flag({
  key: "beta-features", 
  decide: () => false, // Default off
});

// Usage in components
import { showAdvancedFeatures } from "@/lib/feature-flags";

export async function AdvancedSection() {
  const showAdvanced = await showAdvancedFeatures();
  
  if (!showAdvanced) return null;
  
  return <div>Advanced features content</div>;
}
```

### Constants Organization
```typescript
// lib/constants.ts
// File type configurations
export const SUPPORTED_DOCUMENT_MIME_TYPES = [
  "application/pdf",
  "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  "application/vnd.openxmlformats-officedocument.presentationml.presentation",
  "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
  "image/png",
  "image/jpeg",
  "video/mp4",
];

export const FREE_PLAN_ACCEPTED_FILE_TYPES = {
  "application/pdf": [],
  "image/png": [],
  "image/jpeg": [],
};

export const FULL_PLAN_ACCEPTED_FILE_TYPES = {
  ...FREE_PLAN_ACCEPTED_FILE_TYPES,
  "application/vnd.ms-powerpoint": [],
  "video/mp4": [],
  // ... more types
};

// Time constants
export const ONE_SECOND = 1000;
export const ONE_MINUTE = ONE_SECOND * 60;
export const ONE_HOUR = ONE_MINUTE * 60;
export const ONE_DAY = ONE_HOUR * 24;

// Security
export const BLOCKED_PATHNAMES = [
  "/phpmyadmin",
  "/server-status", 
  "/wordpress",
  "/_all_dbs",
  "/wp-json",
];
```

---

## Utility Functions

### Date & Time Utilities
```typescript
// lib/utils.ts
import ms from "ms";

export const timeAgo = (timestamp?: Date): string => {
  if (!timestamp) return "Just now";
  const diff = Date.now() - new Date(timestamp).getTime();
  
  if (diff < 60000) return "Just now";
  if (diff > 82800000) {
    return new Date(timestamp).toLocaleDateString("en-US", {
      month: "short",
      day: "numeric",
      year: new Date(timestamp).getFullYear() !== new Date().getFullYear() 
        ? "numeric" 
        : undefined,
    });
  }
  return `${ms(diff)} ago`;
};

export const durationFormat = (durationInMilliseconds?: number): string => {
  if (!durationInMilliseconds) return "0 secs";

  if (durationInMilliseconds < 60000) {
    return `${Math.round(durationInMilliseconds / 1000)} secs`;
  } else {
    const minutes = Math.floor(durationInMilliseconds / 60000);
    const seconds = Math.round((durationInMilliseconds % 60000) / 1000);
    return `${minutes}:${seconds.toString().padStart(2, "0")} mins`;
  }
};

export const getDateTimeLocal = (timestamp?: Date): string => {
  const d = timestamp ? new Date(timestamp) : new Date();
  if (d.toString() === "Invalid Date") return "";
  return new Date(d.getTime() - d.getTimezoneOffset() * 60000)
    .toISOString()
    .split(":")
    .slice(0, 2)
    .join(":");
};
```

### String & Text Processing
```typescript
// lib/utils.ts
import slugify from "@sindresorhus/slugify";
import { customAlphabet } from "nanoid";

export function capitalize(str: string) {
  if (!str || typeof str !== "string") return str;
  return str.charAt(0).toUpperCase() + str.slice(1);
}

export const nanoid = customAlphabet(
  "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz",
  7, // 7-character random string
);

export function getDomainWithoutWWW(url: string) {
  if (isValidUrl(url)) {
    return new URL(url).hostname.replace(/^www\./, "");
  }
  try {
    if (url.includes(".") && !url.includes(" ")) {
      return new URL(`https://${url}`).hostname.replace(/^www\./, "");
    }
  } catch (e) {
    return "(direct)";
  }
}

const isValidUrl = (url: string) => {
  try {
    new URL(url);
    return true;
  } catch (e) {
    return false;
  }
};
```

### File & Data Processing
```typescript
// lib/utils.ts
export function bytesToSize(bytes: number) {
  const sizes = ["Bytes", "KB", "MB", "GB", "TB"];
  if (bytes === 0) return "n/a";
  const i = Math.floor(Math.log(bytes) / Math.log(1000));
  if (i === 0) return `${bytes} ${sizes[i]}`;
  const sizeInCurrentUnit = bytes / Math.pow(1000, i);
  if (sizeInCurrentUnit >= 1000 && i < sizes.length - 1) {
    return `1 ${sizes[i + 1]}`;
  }
  return `${Math.round(sizeInCurrentUnit)} ${sizes[i]}`;
}

export function nFormatter(num?: number, digits?: number) {
  if (!num) return "0";
  const lookup = [
    { value: 1, symbol: "" },
    { value: 1e3, symbol: "K" },
    { value: 1e6, symbol: "M" },
    { value: 1e9, symbol: "G" },
    { value: 1e12, symbol: "T" },
  ];
  const rx = /\.0+$|(\.[0-9]*[1-9])0+$/;
  var item = lookup
    .slice()
    .reverse()
    .find(function (item) {
      return num >= item.value;
    });
  return item
    ? (num / item.value).toFixed(digits || 1).replace(rx, "$1") + item.symbol
    : "0";
}

export function copyToClipboard(text: string, message: string): void {
  navigator.clipboard
    .writeText(text)
    .then(() => {
      toast.success(message);
    })
    .catch((error) => {
      toast.warning("Please copy your link manually.");
    });
}
```

### Data Validation & Sanitization
```typescript
// lib/utils.ts
export const sanitizeList = (
  list: string,
  mode: "email" | "domain" | "both" = "both",
): string[] => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  const domainRegex = /^@[^\s@]+\.[^\s@]+$/;

  const sanitized = list
    .split("\n")
    .map((item) => item.trim().replace(/,$/, "").toLowerCase())
    .filter((item) => item !== "")
    .filter((item) => {
      if (mode === "email") return emailRegex.test(item);
      if (mode === "domain") return domainRegex.test(item);
      return emailRegex.test(item) || domainRegex.test(item);
    });

  return [...new Set(sanitized)];
};

// Safe template replacement (prevents injection)
export function safeTemplateReplace(
  template: string,
  data: Record<string, any>,
): string {
  const allowedVariables = ["email", "date", "time", "link", "ipAddress"];
  let result = template;

  for (const key of allowedVariables) {
    if (data[key] !== undefined && data[key] !== null) {
      const regex = new RegExp(`{{\\s*${key}\\s*}}`, "gi");
      result = result.replace(regex, String(data[key]));
    }
  }

  return result;
}
```

---

## Performance Optimization

### Image Optimization Pattern
```typescript
// next.config.mjs
const nextConfig = {
  images: {
    minimumCacheTTL: 2592000, // 30 days
    remotePatterns: [
      { protocol: "https", hostname: "assets.papermark.io" },
      { protocol: "https", hostname: "lh3.googleusercontent.com" },
      // ... other allowed domains
    ],
  },
};

// Usage
import Image from "next/image";

<Image
  src={imageUrl}
  alt="Description"
  width={300}
  height={200}
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."
  className="rounded-lg"
/>
```

### Bundle Optimization
```javascript
// next.config.mjs
const nextConfig = {
  experimental: {
    outputFileTracingIncludes: {
      "/api/mupdf/*": ["./node_modules/mupdf/dist/*.wasm"],
    },
  },
  webpack: (config, { isServer }) => {
    if (!isServer) {
      config.resolve.fallback = {
        fs: false,
        net: false,
        tls: false,
      };
    }
    return config;
  },
};
```

### Caching Strategies
```typescript
// lib/redis.ts
import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

export async function getCachedData<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl = 3600, // 1 hour default
): Promise<T> {
  try {
    const cached = await redis.get(key);
    if (cached) {
      return cached as T;
    }

    const fresh = await fetcher();
    await redis.setex(key, ttl, JSON.stringify(fresh));
    return fresh;
  } catch (error) {
    console.error("Cache error:", error);
    return await fetcher(); // Fallback to direct fetch
  }
}

// Usage
const userData = await getCachedData(
  `user:${userId}`,
  () => prisma.user.findUnique({ where: { id: userId } }),
  1800 // 30 minutes
);
```

### React Performance Patterns
```typescript
// hooks/use-debounce.ts
import { useEffect, useState } from "react";

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
}

// Usage in search component
function SearchInput() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 300);

  useEffect(() => {
    if (debouncedQuery) {
      performSearch(debouncedQuery);
    }
  }, [debouncedQuery]);

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

---

## Security Best Practices

### Content Security Policy
```typescript
// next.config.mjs
function generateCSPHeader() {
  const isDev = process.env.NODE_ENV === "development";
  
  return [
    `default-src 'self' https: ${isDev ? "http:" : ""}`,
    `script-src 'self' 'unsafe-inline' 'unsafe-eval' https: ${isDev ? "http:" : ""}`,
    `style-src 'self' 'unsafe-inline' https: ${isDev ? "http:" : ""}`,
    `img-src 'self' data: blob: https: ${isDev ? "http:" : ""}`,
    `font-src 'self' data: https: ${isDev ? "http:" : ""}`,
    `frame-ancestors 'none'`,
    `connect-src 'self' https: ${isDev ? "http: ws: wss:" : ""}`,
    `${isDev ? "" : "upgrade-insecure-requests;"}`,
    "report-to csp-endpoint;",
  ].join("; ");
}
```

### Input Sanitization
```typescript
// lib/utils/sanitize-html.ts
import sanitizeHtml from "sanitize-html";

export function sanitizeUserInput(input: string): string {
  return sanitizeHtml(input, {
    allowedTags: ["b", "i", "em", "strong", "a", "p", "br"],
    allowedAttributes: {
      a: ["href", "target"],
    },
    allowedSchemes: ["https", "mailto"],
  });
}

// Usage in API routes
export default async function handler(req: NextRequest) {
  const { content } = await req.json();
  const sanitizedContent = sanitizeUserInput(content);
  
  // Save sanitized content...
}
```

### Environment Variable Validation
```typescript
// lib/env.ts
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  NEXTAUTH_SECRET: z.string().min(1),
  POSTGRES_PRISMA_URL: z.string().url(),
  UPSTASH_REDIS_REST_URL: z.string().url(),
  RESEND_API_KEY: z.string().min(1),
});

export const env = envSchema.parse(process.env);

// Use `env` instead of `process.env` for type safety
```

---

## Deployment & Infrastructure

### Vercel Configuration
```json
{
  "functions": {
    "pages/api/**/*.ts": {
      "maxDuration": 30
    },
    "app/api/**/*.ts": {
      "maxDuration": 30
    }
  },
  "crons": [
    {
      "path": "/api/cron/cleanup",
      "schedule": "0 2 * * *"
    }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options", 
          "value": "DENY"
        },
        {
          "key": "Referrer-Policy",
          "value": "origin-when-cross-origin"
        }
      ]
    }
  ]
}
```

### Build Process
```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "postinstall": "prisma generate",
    "vercel-build": "prisma migrate deploy && next build",
    "trigger:deploy": "npx trigger.dev@latest deploy"
  },
  "engines": {
    "node": ">=18.18.0"
  }
}
```

### Health Check Endpoint
```typescript
// pages/api/health.ts
import prisma from "@/lib/prisma";

export default async function handler(req: NextRequest) {
  if (req.method !== "GET") {
    return Response.json({ error: "Method not allowed" }, { status: 405 });
  }

  try {
    // Check database connection
    await prisma.$queryRaw`SELECT 1`;
    
    // Check Redis connection
    await redis.ping();

    return Response.json({ 
      status: "ok", 
      timestamp: new Date().toISOString(),
      environment: process.env.NODE_ENV,
    });
  } catch (error) {
    console.error("Health check failed:", error);
    return Response.json(
      { 
        status: "error", 
        error: "Service unavailable",
        timestamp: new Date().toISOString(),
      },
      { status: 503 }
    );
  }
}
```

---

## Conclusion

This guide captures production-ready patterns from Papermark that can be applied across different projects:

**🏗️ Architecture**: Hybrid Next.js approach with both App and Pages Router, middleware-based routing, and modular organization

**🎨 UI/UX**: Tailwind CSS with Radix UI components, class variance authority for component variants, and consistent design tokens

**🗄️ Data**: Prisma with PostgreSQL, SWR for client-side data fetching, optimistic updates, and efficient caching strategies

**🔐 Security**: NextAuth.js with multiple providers, rate limiting, input sanitization, and CSP headers

**📁 Files**: Multi-provider storage (S3/Vercel Blob), TUS resumable uploads, and comprehensive file processing

**⚡ Background Jobs**: Trigger.dev v3 for reliable background processing with queues, retries, and monitoring

**🔧 APIs**: Consistent response patterns, Zod validation, authentication middleware, and comprehensive error handling

**📊 Monitoring**: PostHog analytics, Tinybird for events, structured logging, and performance tracking

**🚀 Performance**: Image optimization, bundle splitting, caching strategies, and React performance patterns

All patterns are production-tested and can be adapted to your specific needs while maintaining reliability and scalability.