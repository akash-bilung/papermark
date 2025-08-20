# Complete Authentication Implementation Guide

## Overview

This comprehensive guide details the complete authentication system implemented in Papermark, which provides a production-ready authentication solution using NextAuth.js with multiple authentication providers including Google OAuth, LinkedIn, Email (Magic Links), and WebAuthn/Passkeys.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Dependencies & Setup](#dependencies--setup)
3. [NextAuth.js Configuration](#nextauthjs-configuration)
4. [Authentication Providers](#authentication-providers)
5. [Database Schema](#database-schema)
6. [Session Management](#session-management)
7. [Middleware & Route Protection](#middleware--route-protection)
8. [Authentication Pages & UI](#authentication-pages--ui)
9. [Email Verification System](#email-verification-system)
10. [WebAuthn/Passkey Implementation](#webauthpasskey-implementation)
11. [Team Invitations](#team-invitations)
12. [Security Features](#security-features)
13. [Implementation Steps](#implementation-steps)
14. [Environment Variables](#environment-variables)
15. [Testing](#testing)
16. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

### Authentication Flow Architecture

```mermaid
graph TD
    A[User Visits App] --> B{Is Authenticated?}
    B -->|No| C[Redirect to /login]
    B -->|Yes| D{New User?}
    D -->|Yes| E[Redirect to /welcome]
    D -->|No| F[Access Dashboard]
    
    C --> G[Select Auth Method]
    G --> H[Google OAuth]
    G --> I[LinkedIn OAuth]
    G --> J[Email Magic Link]
    G --> K[WebAuthn/Passkey]
    
    H --> L[OAuth Callback]
    I --> L
    J --> M[Email Verification]
    K --> N[WebAuthn Challenge]
    
    L --> O[Create/Update User]
    M --> O
    N --> O
    
    O --> P[Generate Session]
    P --> F
```

### Key Components

- **NextAuth.js**: Core authentication framework
- **Prisma**: ORM for user/session management
- **Middleware**: Route protection and redirects
- **Email System**: Resend for transactional emails
- **WebAuthn**: Hanko for passkey authentication
- **Session Strategy**: JWT-based sessions

---

## Dependencies & Setup

### Required Dependencies

```json
{
  "dependencies": {
    "next-auth": "^4.24.11",
    "@next-auth/prisma-adapter": "^1.0.7",
    "@teamhanko/passkeys-next-auth-provider": "^0.3.1",
    "@prisma/client": "^6.5.0",
    "resend": "^4.8.0",
    "@react-email/components": "^0.0.42",
    "zod": "^3.25.76",
    "sonner": "^2.0.7"
  },
  "devDependencies": {
    "prisma": "^6.5.0"
  }
}
```

### Installation

```bash
npm install next-auth @next-auth/prisma-adapter @teamhanko/passkeys-next-auth-provider
npm install @prisma/client resend @react-email/components
npm install zod sonner
npm install -D prisma
```

---

## NextAuth.js Configuration

### Core Configuration (`pages/api/auth/[...nextauth].ts`)

```typescript
import { PrismaAdapter } from "@next-auth/prisma-adapter";
import PasskeyProvider from "@teamhanko/passkeys-next-auth-provider";
import NextAuth, { type NextAuthOptions } from "next-auth";
import EmailProvider from "next-auth/providers/email";
import GoogleProvider from "next-auth/providers/google";
import LinkedInProvider from "next-auth/providers/linkedin";

import { identifyUser, trackAnalytics } from "@/lib/analytics";
import { isBlacklistedEmail } from "@/lib/edge-config/blacklist";
import { sendVerificationRequestEmail } from "@/lib/emails/send-verification-request";
import { sendWelcomeEmail } from "@/lib/emails/send-welcome";
import hanko from "@/lib/hanko";
import prisma from "@/lib/prisma";
import { CreateUserEmailProps, CustomUser } from "@/lib/types";
import { subscribe } from "@/lib/unsend";
import { generateChecksum } from "@/lib/utils/generate-checksum";

const VERCEL_DEPLOYMENT = !!process.env.VERCEL_URL;

function getMainDomainUrl(): string {
  if (process.env.NODE_ENV === "development") {
    return process.env.NEXTAUTH_URL || "http://localhost:3000";
  }
  return process.env.NEXTAUTH_URL || "https://app.papermark.com";
}

export const authOptions: NextAuthOptions = {
  pages: {
    error: "/login", // Redirect errors to login page
  },
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID as string,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET as string,
      allowDangerousEmailAccountLinking: true,
    }),
    LinkedInProvider({
      clientId: process.env.LINKEDIN_CLIENT_ID as string,
      clientSecret: process.env.LINKEDIN_CLIENT_SECRET as string,
      authorization: {
        params: { scope: "openid profile email" },
      },
      issuer: "https://www.linkedin.com/oauth",
      jwks_endpoint: "https://www.linkedin.com/oauth/openid/jwks",
      profile(profile, tokens) {
        const defaultImage = "https://cdn-icons-png.flaticon.com/512/174/174857.png";
        return {
          id: profile.sub,
          name: profile.name,
          email: profile.email,
          image: profile.picture ?? defaultImage,
        };
      },
      allowDangerousEmailAccountLinking: true,
    }),
    EmailProvider({
      async sendVerificationRequest({ identifier, url }) {
        const hasValidNextAuthUrl = !!process.env.NEXTAUTH_URL;
        let finalUrl = url;

        // Handle URL construction for different environments
        if (!hasValidNextAuthUrl) {
          const mainDomainUrl = getMainDomainUrl();
          const urlObj = new URL(url);
          const mainDomainObj = new URL(mainDomainUrl);
          urlObj.hostname = mainDomainObj.hostname;
          urlObj.protocol = mainDomainObj.protocol;
          urlObj.port = mainDomainObj.port || "";
          finalUrl = urlObj.toString();
        }

        if (process.env.NODE_ENV === "development") {
          // In development, log the verification URL with checksum
          const checksum = generateChecksum(finalUrl);
          const verificationUrlParams = new URLSearchParams({
            verification_url: finalUrl,
            checksum,
          });

          const baseUrl = hasValidNextAuthUrl
            ? process.env.NEXTAUTH_URL
            : getMainDomainUrl();

          const verificationUrl = `${baseUrl}/verify?${verificationUrlParams}`;
          console.log("[Login URL]", verificationUrl);
          return;
        } else {
          // In production, send email
          await sendVerificationRequestEmail({
            url: finalUrl,
            email: identifier,
          });
        }
      },
    }),
    PasskeyProvider({
      tenant: hanko,
      async authorize({ userId }) {
        const user = await prisma.user.findUnique({ where: { id: userId } });
        if (!user) return null;
        return user;
      },
    }),
  ],
  adapter: PrismaAdapter(prisma),
  session: { strategy: "jwt" },
  
  // Cookie configuration for production
  cookies: {
    sessionToken: {
      name: `${VERCEL_DEPLOYMENT ? "__Secure-" : ""}next-auth.session-token`,
      options: {
        httpOnly: true,
        sameSite: "lax",
        path: "/",
        domain: VERCEL_DEPLOYMENT ? ".papermark.com" : undefined,
        secure: VERCEL_DEPLOYMENT,
      },
    },
  },
  
  callbacks: {
    signIn: async ({ user }) => {
      // Check for blacklisted emails
      if (!user.email || (await isBlacklistedEmail(user.email))) {
        await identifyUser(user.email ?? user.id);
        await trackAnalytics({
          event: "User Sign In Attempted",
          email: user.email ?? undefined,
          userId: user.id,
        });
        return false;
      }
      return true;
    },

    jwt: async (params) => {
      const { token, user, trigger } = params;
      if (!token.email) {
        return {};
      }
      
      if (user) {
        token.user = user;
      }
      
      // Refresh user data on token update
      if (trigger === "update") {
        const user = token?.user as CustomUser;
        const refreshedUser = await prisma.user.findUnique({
          where: { id: user.id },
        });
        
        if (refreshedUser) {
          token.user = refreshedUser;
        } else {
          return {};
        }

        // Handle email change
        if (refreshedUser?.email !== user.email) {
          if (user.id && refreshedUser.email) {
            await prisma.account.deleteMany({
              where: { userId: user.id },
            });
          }
        }
      }
      return token;
    },
    
    session: async ({ session, token }) => {
      (session.user as CustomUser) = {
        id: token.sub,
        ...(token || session).user,
      };
      return session;
    },
  },
  
  events: {
    async createUser(message) {
      // Send welcome email and track analytics
      const params: CreateUserEmailProps = {
        user: {
          name: message.user.name,
          email: message.user.email,
        },
      };

      await identifyUser(message.user.email ?? message.user.id);
      await trackAnalytics({
        event: "User Signed Up",
        email: message.user.email,
        userId: message.user.id,
      });

      await sendWelcomeEmail(params);

      if (message.user.email) {
        await subscribe(message.user.email);
      }
    },
    
    async signIn(message) {
      // Set additional auth cookies and track sign in
      if (typeof window !== "undefined") {
        try {
          await fetch("/api/auth-plus/set-cookie");
        } catch (error) {
          console.error("Failed to set additional cookie", error);
        }
      }
      
      await identifyUser(message.user.email ?? message.user.id);
      await trackAnalytics({
        event: "User Signed In",
        email: message.user.email,
      });
    },
  },
};

export default NextAuth(authOptions);
```

---

## Authentication Providers

### 1. Google OAuth Provider

**Setup Steps:**

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing
3. Enable Google+ API
4. Create OAuth 2.0 credentials
5. Set authorized redirect URIs:
   - Development: `http://localhost:3000/api/auth/callback/google`
   - Production: `https://yourdomain.com/api/auth/callback/google`

**Environment Variables:**
```bash
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
```

**Configuration Features:**
- `allowDangerousEmailAccountLinking: true` - Links accounts with same email
- Automatic profile extraction

### 2. LinkedIn Provider

**Setup Steps:**

1. Go to [LinkedIn Developer Portal](https://developer.linkedin.com/)
2. Create a LinkedIn App
3. Request "Sign In with LinkedIn" product
4. Configure OAuth 2.0 settings
5. Set redirect URLs

**Environment Variables:**
```bash
LINKEDIN_CLIENT_ID=your_linkedin_client_id
LINKEDIN_CLIENT_SECRET=your_linkedin_client_secret
```

**Custom Profile Handling:**
```typescript
profile(profile, tokens) {
  const defaultImage = "https://cdn-icons-png.flaticon.com/512/174/174857.png";
  return {
    id: profile.sub,
    name: profile.name,
    email: profile.email,
    image: profile.picture ?? defaultImage,
  };
}
```

### 3. Email Provider (Magic Links)

**Features:**
- Passwordless authentication
- Checksum verification for security
- Development logging
- Production email sending

**Custom Email Sending:**
```typescript
async sendVerificationRequest({ identifier, url }) {
  // Custom URL construction and checksum generation
  const checksum = generateChecksum(finalUrl);
  
  if (process.env.NODE_ENV === "development") {
    console.log("[Login URL]", verificationUrl);
  } else {
    await sendVerificationRequestEmail({ url: finalUrl, email: identifier });
  }
}
```

### 4. WebAuthn/Passkey Provider

**Setup with Hanko:**
```typescript
PasskeyProvider({
  tenant: hanko,
  async authorize({ userId }) {
    const user = await prisma.user.findUnique({ where: { id: userId } });
    if (!user) return null;
    return user;
  },
})
```

**Environment Variables:**
```bash
HANKO_API_KEY=your_hanko_api_key
NEXT_PUBLIC_HANKO_TENANT_ID=your_tenant_id
```

---

## Database Schema

### Core Authentication Tables

```prisma
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model User {
  id             String     @id @default(cuid())
  name           String?
  email          String?    @unique
  emailVerified  DateTime?
  image          String?
  createdAt      DateTime   @default(now())
  accounts       Account[]
  sessions       Session[]
  documents      Document[]
  teams          UserTeam[]
  domains        Domain[]
  chats          Chat[]
  contactId      String?
  plan           String     @default("free")
  stripeId       String?    @unique
  subscriptionId String?    @unique
  startsAt       DateTime?
  endsAt         DateTime?

  restrictedTokens RestrictedToken[]
  participatedConversations ConversationParticipant[]
  messages                  Message[]
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}
```

### Team & Invitation System

```prisma
model Team {
  id          String      @id @default(cuid())
  name        String
  slug        String?     @unique
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt
  users       UserTeam[]
  invitations Invitation[]
  documents   Document[]
  // ... other fields
}

model UserTeam {
  id         String   @id @default(cuid())
  userId     String
  teamId     String
  role       String   @default("member")
  createdAt  DateTime @default(now())
  user       User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  team       Team     @relation(fields: [teamId], references: [id], onDelete: Cascade)

  @@unique([userId, teamId])
}

model Invitation {
  email     String
  expires   DateTime
  teamId    String
  team      Team     @relation(fields: [teamId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
  token     String   @unique

  @@unique([email, teamId])
}
```

---

## Session Management

### JWT Strategy Configuration

```typescript
export const authOptions: NextAuthOptions = {
  session: { strategy: "jwt" },
  
  callbacks: {
    jwt: async (params) => {
      const { token, user, trigger } = params;
      
      // Initial sign in
      if (user) {
        token.user = user;
      }
      
      // Token refresh/update
      if (trigger === "update") {
        const refreshedUser = await prisma.user.findUnique({
          where: { id: user.id },
        });
        
        if (refreshedUser) {
          token.user = refreshedUser;
        }
      }
      
      return token;
    },
    
    session: async ({ session, token }) => {
      (session.user as CustomUser) = {
        id: token.sub,
        ...(token || session).user,
      };
      return session;
    },
  },
}
```

### Session Provider Setup

```typescript
// pages/_app.tsx
import { SessionProvider } from "next-auth/react";

export default function App({
  Component,
  pageProps: { session, ...pageProps },
}: AppProps<{ session: Session }>) {
  return (
    <SessionProvider session={session}>
      <Component {...pageProps} />
    </SessionProvider>
  );
}
```

### Using Sessions in Components

```typescript
import { useSession, signIn, signOut } from "next-auth/react";

function Component() {
  const { data: session, status } = useSession();
  
  if (status === "loading") return <p>Loading...</p>;
  
  if (status === "unauthenticated") {
    return <button onClick={() => signIn()}>Sign in</button>;
  }
  
  return (
    <div>
      <p>Signed in as {session?.user?.email}</p>
      <button onClick={() => signOut()}>Sign out</button>
    </div>
  );
}
```

---

## Middleware & Route Protection

### Main Middleware (`middleware.ts`)

```typescript
import { NextFetchEvent, NextRequest, NextResponse } from "next/server";
import AppMiddleware from "@/lib/middleware/app";
import DomainMiddleware from "@/lib/middleware/domain";

export const config = {
  matcher: [
    "/((?!api/|_next/|_static|vendor|_icons|_vercel|favicon.ico|sitemap.xml).*)",
  ],
};

export default async function middleware(req: NextRequest, ev: NextFetchEvent) {
  const path = req.nextUrl.pathname;
  const host = req.headers.get("host");

  // Handle custom domains
  if (isCustomDomain(host || "")) {
    return DomainMiddleware(req);
  }

  // Handle standard app routes
  if (!path.startsWith("/view/") && !path.startsWith("/verify") && !path.startsWith("/unsubscribe")) {
    return AppMiddleware(req);
  }

  return NextResponse.next();
}
```

### App Middleware (`lib/middleware/app.ts`)

```typescript
import { NextRequest, NextResponse } from "next/server";
import { getToken } from "next-auth/jwt";

export default async function AppMiddleware(req: NextRequest) {
  const url = req.nextUrl;
  const path = url.pathname;
  const isInvited = url.searchParams.has("invitation");
  
  const token = await getToken({
    req,
    secret: process.env.NEXTAUTH_SECRET,
  });

  // UNAUTHENTICATED - Redirect to login
  if (!token?.email && path !== "/login") {
    const loginUrl = new URL(`/login`, req.url);
    
    if (path !== "/") {
      const nextPath = path === "/auth/confirm-email-change" 
        ? `${path}${url.search}` 
        : path;
      loginUrl.searchParams.set("next", encodeURIComponent(nextPath));
    }
    
    return NextResponse.redirect(loginUrl);
  }

  // NEW USER - Redirect to welcome
  if (
    token?.email &&
    token?.user?.createdAt &&
    new Date(token?.user?.createdAt).getTime() > Date.now() - 10000 &&
    path !== "/welcome" &&
    !isInvited
  ) {
    return NextResponse.redirect(new URL("/welcome", req.url));
  }

  // AUTHENTICATED on login page - Redirect to dashboard
  if (token?.email && path === "/login") {
    const nextPath = url.searchParams.get("next") || "/dashboard";
    return NextResponse.redirect(new URL(decodeURIComponent(nextPath), req.url));
  }
}
```

### Server-side Route Protection

```typescript
// pages/api/protected-route.ts
import { getServerSession } from "next-auth/next";
import { authOptions } from "./auth/[...nextauth]";

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const session = await getServerSession(req, res, authOptions);
  
  if (!session) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  
  // Protected route logic
  res.json({ message: "Protected content" });
}
```

---

## Authentication Pages & UI

### Login Page (`app/(auth)/login/page-client.tsx`)

**Key Features:**
- Multiple authentication methods
- Form validation with Zod
- Loading states
- Last used method memory
- Toast notifications

```typescript
"use client";

import { useState } from "react";
import { signIn } from "next-auth/react";
import { signInWithPasskey } from "@teamhanko/passkeys-next-auth-provider/client";
import { toast } from "sonner";
import { z } from "zod";

export default function Login() {
  const [clickedMethod, setClickedMethod] = useState<AuthMethod | undefined>();
  const [email, setEmail] = useState<string>("");
  const [emailButtonText, setEmailButtonText] = useState<string>("Continue with Email");

  const emailSchema = z
    .string()
    .trim()
    .toLowerCase()
    .min(3, { message: "Please enter a valid email." })
    .email({ message: "Please enter a valid email." });

  const emailValidation = emailSchema.safeParse(email);

  // Email sign in handler
  const handleEmailSignIn = (e: React.FormEvent) => {
    e.preventDefault();
    
    if (!emailValidation.success) {
      toast.error(emailValidation.error.errors[0].message);
      return;
    }

    setClickedMethod("email");
    signIn("email", {
      email: emailValidation.data,
      redirect: false,
      ...(next && next.length > 0 ? { callbackUrl: next } : {}),
    }).then((res) => {
      if (res?.ok && !res?.error) {
        setEmail("");
        setEmailButtonText("Email sent - check your inbox!");
        toast.success("Email sent - check your inbox!");
      } else {
        setEmailButtonText("Error sending email - try again?");
        toast.error("Error sending email - try again?");
      }
      setClickedMethod(undefined);
    });
  };

  // Google sign in
  const handleGoogleSignIn = () => {
    setClickedMethod("google");
    signIn("google", {
      ...(next && next.length > 0 ? { callbackUrl: next } : {}),
    }).then(() => setClickedMethod(undefined));
  };

  // LinkedIn sign in
  const handleLinkedInSignIn = () => {
    setClickedMethod("linkedin");
    signIn("linkedin", {
      ...(next && next.length > 0 ? { callbackUrl: next } : {}),
    }).then(() => setClickedMethod(undefined));
  };

  // Passkey sign in
  const handlePasskeySignIn = () => {
    setClickedMethod("passkey");
    signInWithPasskey({
      tenantId: process.env.NEXT_PUBLIC_HANKO_TENANT_ID as string,
    }).then(() => setClickedMethod(undefined));
  };

  return (
    <div className="flex h-screen w-full flex-wrap">
      {/* Login form implementation */}
    </div>
  );
}
```

### UI Components

**Email Input with Validation:**
```typescript
<Input
  id="email"
  placeholder="name@example.com"
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  className={cn(
    "standard-input-styles",
    email.length > 0 && !emailValidation.success
      ? "ring-red-500"
      : "ring-gray-200",
  )}
/>
```

**Provider Buttons:**
```typescript
<Button
  onClick={handleGoogleSignIn}
  loading={clickedMethod === "google"}
  disabled={clickedMethod && clickedMethod !== "google"}
  className="provider-button-styles"
>
  <GoogleIcon className="h-5 w-5" />
  <span>Continue with Google</span>
</Button>
```

---

## Email Verification System

### Email Sending Service (`lib/emails/send-verification-request.ts`)

```typescript
import LoginLink from "@/components/emails/verification-link";
import { sendEmail } from "@/lib/resend";
import { generateChecksum } from "../utils/generate-checksum";

export const sendVerificationRequestEmail = async (params: {
  email: string;
  url: string;
}) => {
  const { url, email } = params;
  
  // Generate security checksum
  const checksum = generateChecksum(url);
  const verificationUrlParams = new URLSearchParams({
    verification_url: url,
    checksum,
  });

  const verificationUrl = `${process.env.NEXTAUTH_URL}/verify?${verificationUrlParams}`;
  const emailTemplate = LoginLink({ url: verificationUrl });
  
  try {
    await sendEmail({
      to: email as string,
      subject: "Welcome to Papermark!",
      react: emailTemplate,
      test: process.env.NODE_ENV === "development",
    });
  } catch (e) {
    console.error(e);
  }
};
```

### Checksum Generation (`lib/utils/generate-checksum.ts`)

```typescript
import crypto from 'crypto';

export function generateChecksum(data: string): string {
  return crypto
    .createHash('sha256')
    .update(data + process.env.NEXTAUTH_SECRET)
    .digest('hex')
    .substring(0, 16);
}
```

### Email Template (`components/emails/verification-link.tsx`)

```typescript
import React from "react";
import {
  Body,
  Button,
  Container,
  Head,
  Hr,
  Html,
  Preview,
  Section,
  Tailwind,
  Text,
} from "@react-email/components";

const VerificationLinkEmail = ({ url }: { url: string }) => {
  return (
    <Html>
      <Head />
      <Preview>Login to your Papermark account with a link</Preview>
      <Tailwind>
        <Body className="mx-auto my-auto bg-white font-sans">
          <Container className="mx-auto my-10 max-w-[500px] rounded border border-solid border-gray-200 px-10 py-5">
            <Text className="mx-0 mb-8 mt-4 p-0 text-center text-2xl font-normal">
              <span className="font-bold tracking-tighter">Papermark</span>
            </Text>
            <Text className="mx-0 my-7 p-0 text-center text-xl font-semibold text-black">
              Your Papermark Login Link
            </Text>
            
            <Text className="text-sm leading-6 text-black">
              Please click the magic link below to sign in to your account.
            </Text>
            
            <Section className="my-8 text-center">
              <Button
                className="rounded bg-black text-center text-xs font-semibold text-white no-underline"
                href={url}
                style={{ padding: "12px 20px" }}
              >
                Sign in
              </Button>
            </Section>
            
            <Text className="text-sm leading-6 text-black">
              or copy and paste this URL into your browser:
            </Text>
            <Text className="max-w-sm flex-wrap break-words font-medium text-purple-600 no-underline">
              {url.replace(/^https?:\/\//, "")}
            </Text>
            
            <Hr />
            <Section className="mt-8 text-gray-400">
              <Text className="text-xs">
                © {new Date().getFullYear()}{" "}
                <a href="https://www.papermark.com" className="text-gray-400 no-underline">
                  papermark.com
                </a>
              </Text>
            </Section>
          </Container>
        </Body>
      </Tailwind>
    </Html>
  );
};

export default VerificationLinkEmail;
```

### Verification Page (`app/(auth)/verify/page.tsx`)

```typescript
export default function VerifyPage({
  searchParams,
}: {
  searchParams: { verification_url?: string; checksum?: string };
}) {
  const { verification_url, checksum } = searchParams;

  if (!verification_url || !checksum) {
    return <NotFound />;
  }

  // Server-side validation
  const isValidVerificationUrl = (url: string, checksum: string): boolean => {
    try {
      const urlObj = new URL(url);
      if (urlObj.origin !== process.env.NEXTAUTH_URL) return false;
      
      const expectedChecksum = generateChecksum(url);
      return checksum === expectedChecksum;
    } catch {
      return false;
    }
  };

  if (!isValidVerificationUrl(verification_url, checksum)) {
    return <NotFound />;
  }

  return (
    <div className="verify-page">
      <Link href={verification_url}>
        <Button>Verify email</Button>
      </Link>
    </div>
  );
}
```

---

## WebAuthn/Passkey Implementation

### Hanko Setup (`lib/hanko.ts`)

```typescript
import { tenant } from "@teamhanko/passkeys-next-auth-provider";

if (!process.env.HANKO_API_KEY || !process.env.NEXT_PUBLIC_HANKO_TENANT_ID) {
  throw new Error(
    "Please set HANKO_API_KEY and NEXT_PUBLIC_HANKO_TENANT_ID in your .env.local file.",
  );
}

const hanko = tenant({
  apiKey: process.env.HANKO_API_KEY!,
  tenantId: process.env.NEXT_PUBLIC_HANKO_TENANT_ID!,
});

export default hanko;
```

### Passkey Registration API (`pages/api/passkeys/register.ts`)

```typescript
import { NextApiRequest, NextApiResponse } from "next";
import { getServerSession } from "next-auth/next";
import {
  finishServerPasskeyRegistration,
  startServerPasskeyRegistration,
} from "@/lib/api/auth/passkey";
import { authOptions } from "../auth/[...nextauth]";

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === "POST") {
    const session = await getServerSession(req, res, authOptions);
    if (!session) {
      return res.status(401).end("Unauthorized");
    }

    const { start, finish, credential } = req.body;

    try {
      if (start) {
        const createOptions = await startServerPasskeyRegistration({ session });
        res.status(200).json({ createOptions });
        return;
      }
      
      if (finish) {
        await finishServerPasskeyRegistration({ credential, session });
        res.status(200).json({ message: "Registered Passkey" });
        return;
      }
    } catch (error) {
      console.error(error);
      return res.status(500).json({ error: "Internal server error" });
    }
  }

  return res.status(405).json({ error: "Method not allowed" });
}
```

### Client-side Passkey Registration

```typescript
import { startRegistration, finishRegistration } from "@teamhanko/passkeys-next-auth-provider/client";

const handlePasskeyRegistration = async () => {
  try {
    // Start registration
    const startResponse = await fetch("/api/passkeys/register", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ start: true }),
    });
    
    const { createOptions } = await startResponse.json();
    
    // Get credential from browser
    const credential = await startRegistration(createOptions);
    
    // Finish registration
    await fetch("/api/passkeys/register", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ finish: true, credential }),
    });
    
    toast.success("Passkey registered successfully!");
  } catch (error) {
    toast.error("Failed to register passkey");
  }
};
```

---

## Team Invitations

### Invitation Flow

1. **Create Invitation**: Admin generates invitation with unique token
2. **Send Email**: Invitation email sent to user
3. **Click Link**: User clicks invitation link
4. **Authentication Check**: If not logged in, redirect to login with invitation context
5. **Accept Invitation**: Add user to team and delete invitation

### Accept Invitation API (`pages/api/teams/[teamId]/invitations/accept.ts`)

```typescript
export default async function handle(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === "GET") {
    const session = await getServerSession(req, res, authOptions);
    const { teamId, token, email } = req.query;

    // Check if user is authenticated
    if (!session) {
      const redirectUrl = `/login?next=/api/teams/${teamId}/invitations/accept`;
      const params = new URLSearchParams();
      
      if (token) params.append("token", token);
      if (email) params.append("email", email);

      const finalRedirectUrl = params.toString()
        ? `${redirectUrl}&${params.toString()}`
        : redirectUrl;

      res.redirect(finalRedirectUrl);
      return;
    }

    const userId = (session.user as CustomUser).id;
    const userEmail = (session.user as CustomUser).email;

    try {
      // Check if user is already in team
      const userTeam = await prisma.userTeam.findFirst({
        where: { teamId, userId },
      });

      if (userTeam) {
        return res.redirect(`/documents?invitation=teamMember`);
      }

      // Find and validate invitation
      let invitation;
      
      if (token && email) {
        invitation = await prisma.invitation.findFirst({
          where: { token, email, teamId },
        });
      }

      if (!invitation && userEmail) {
        invitation = await prisma.invitation.findUnique({
          where: {
            email_teamId: { email: userEmail, teamId },
          },
        });
      }

      if (!invitation) {
        return res.status(404).json("Invalid invitation");
      }

      if (invitation.email !== userEmail) {
        return res.status(403).json("You are not invited to this team");
      }

      if (new Date() > invitation.expires) {
        return res.status(410).json("Invitation link has expired");
      }

      // Add user to team
      await prisma.team.update({
        where: { id: teamId },
        data: {
          users: {
            create: { userId },
          },
        },
      });

      // Track analytics
      await identifyUser(invitation.email);
      await trackAnalytics({
        event: "Team Member Invitation Accepted",
        teamId: teamId,
      });

      // Clean up invitation
      await prisma.invitation.delete({
        where: { token: invitation.token },
      });

      return res.redirect(`/documents?invitation=accepted`);
    } catch (error) {
      console.error(error);
      return res.status(500).json({ error: "Internal server error" });
    }
  }
}
```

### Invitation Email Template

```typescript
const TeamInvitationEmail = ({ 
  invitedBy, 
  teamName, 
  inviteLink 
}: {
  invitedBy: string;
  teamName: string;
  inviteLink: string;
}) => {
  return (
    <Html>
      <Head />
      <Preview>You've been invited to join {teamName}</Preview>
      <Tailwind>
        <Body>
          <Container>
            <Text>
              {invitedBy} has invited you to join {teamName} on Papermark.
            </Text>
            <Section className="text-center">
              <Button href={inviteLink}>
                Accept Invitation
              </Button>
            </Section>
          </Container>
        </Body>
      </Tailwind>
    </Html>
  );
};
```

---

## Security Features

### 1. Email Blacklisting

```typescript
import { isBlacklistedEmail } from "@/lib/edge-config/blacklist";

// In NextAuth callbacks
signIn: async ({ user }) => {
  if (!user.email || (await isBlacklistedEmail(user.email))) {
    return false;
  }
  return true;
}
```

### 2. Checksum Verification

```typescript
export function generateChecksum(data: string): string {
  return crypto
    .createHash('sha256')
    .update(data + process.env.NEXTAUTH_SECRET)
    .digest('hex')
    .substring(0, 16);
}
```

### 3. CSRF Protection

NextAuth.js includes built-in CSRF protection through:
- CSRF tokens in state parameter
- Secure cookie settings
- Origin validation

### 4. Secure Cookie Configuration

```typescript
cookies: {
  sessionToken: {
    name: `${VERCEL_DEPLOYMENT ? "__Secure-" : ""}next-auth.session-token`,
    options: {
      httpOnly: true,
      sameSite: "lax",
      path: "/",
      domain: VERCEL_DEPLOYMENT ? ".papermark.com" : undefined,
      secure: VERCEL_DEPLOYMENT,
    },
  },
}
```

### 5. Rate Limiting

Implement rate limiting on authentication endpoints:

```typescript
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, "1 m"), // 5 requests per minute
});

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const identifier = req.ip || "anonymous";
  const { success } = await ratelimit.limit(identifier);
  
  if (!success) {
    return res.status(429).json({ error: "Too many requests" });
  }
  
  // Handle request
}
```

---

## Implementation Steps

### Step 1: Project Setup

```bash
# Install dependencies
npm install next-auth @next-auth/prisma-adapter @teamhanko/passkeys-next-auth-provider
npm install @prisma/client resend @react-email/components zod sonner
npm install -D prisma

# Initialize Prisma
npx prisma init
```

### Step 2: Environment Configuration

Create `.env.local`:

```bash
# NextAuth
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-secret-key

# Database
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"

# Google OAuth
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

# LinkedIn OAuth
LINKEDIN_CLIENT_ID=your-linkedin-client-id
LINKEDIN_CLIENT_SECRET=your-linkedin-client-secret

# Hanko (WebAuthn)
HANKO_API_KEY=your-hanko-api-key
NEXT_PUBLIC_HANKO_TENANT_ID=your-tenant-id

# Email
RESEND_API_KEY=your-resend-api-key
```

### Step 3: Database Setup

```bash
# Generate Prisma client
npx prisma generate

# Run migrations
npx prisma db push

# (Optional) Seed database
npx prisma db seed
```

### Step 4: NextAuth Configuration

Create `pages/api/auth/[...nextauth].ts` with the configuration shown above.

### Step 5: Middleware Setup

Create `middleware.ts` and `lib/middleware/app.ts` for route protection.

### Step 6: Authentication Pages

Create login/register pages in the `app/(auth)` directory.

### Step 7: Email Templates

Set up React Email components for verification and invitation emails.

### Step 8: WebAuthn Integration

Configure Hanko integration for passkey authentication.

### Step 9: Testing

Test all authentication flows:
- Google OAuth
- LinkedIn OAuth
- Email magic links
- WebAuthn/Passkeys
- Team invitations

---

## Environment Variables

### Complete Environment Variables List

```bash
# Required - NextAuth
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your-nextauth-secret-minimum-32-characters

# Required - Database
DATABASE_URL="postgresql://username:password@localhost:5432/database"

# Optional - Google OAuth
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

# Optional - LinkedIn OAuth
LINKEDIN_CLIENT_ID=your-linkedin-client-id
LINKEDIN_CLIENT_SECRET=your-linkedin-client-secret

# Optional - WebAuthn/Passkeys
HANKO_API_KEY=your-hanko-api-key
NEXT_PUBLIC_HANKO_TENANT_ID=your-tenant-id

# Required for Email - Resend
RESEND_API_KEY=your-resend-api-key

# Optional - Analytics
NEXT_PUBLIC_POSTHOG_KEY=your-posthog-key
NEXT_PUBLIC_POSTHOG_HOST=https://app.posthog.com

# Optional - Marketing Site
NEXT_PUBLIC_MARKETING_URL=https://www.papermark.com

# Production - Deployment
VERCEL_URL=your-vercel-url
```

### Environment Variable Generation

```bash
# Generate NextAuth Secret
openssl rand -base64 32

# Or using Node.js
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

---

## Testing

### Authentication Flow Testing

```typescript
// __tests__/auth.test.ts
import { authOptions } from '@/pages/api/auth/[...nextauth]';
import { getServerSession } from 'next-auth/next';

describe('Authentication', () => {
  test('should redirect unauthenticated users to login', async () => {
    // Mock session
    jest.mocked(getServerSession).mockResolvedValue(null);
    
    // Test middleware behavior
    // Implementation depends on your testing setup
  });

  test('should allow authenticated users to access protected routes', async () => {
    const mockSession = {
      user: { id: '1', email: 'test@example.com' },
      expires: new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString(),
    };
    
    jest.mocked(getServerSession).mockResolvedValue(mockSession);
    
    // Test protected route access
  });
});
```

### Manual Testing Checklist

- [ ] Google OAuth sign-in/sign-up
- [ ] LinkedIn OAuth sign-in/sign-up
- [ ] Email magic link authentication
- [ ] WebAuthn/Passkey registration and sign-in
- [ ] New user welcome flow
- [ ] Route protection middleware
- [ ] Team invitation flow
- [ ] Email verification with checksum
- [ ] Session persistence across page reloads
- [ ] Sign out functionality
- [ ] Error handling for expired/invalid tokens

---

## Troubleshooting

### Common Issues

**1. NextAuth Error: "Invalid provider"**
```bash
# Solution: Check provider configuration
# Ensure all required environment variables are set
# Verify callback URLs match provider settings
```

**2. Database Connection Errors**
```bash
# Solution: Verify DATABASE_URL
# Check database is running
# Run: npx prisma db push
```

**3. Email Not Sending (Development)**
```bash
# In development, check console for login URL
# Emails are logged instead of sent
# Set NODE_ENV=production to actually send emails
```

**4. WebAuthn/Passkey Issues**
```bash
# Ensure HTTPS in production
# Check HANKO_API_KEY and NEXT_PUBLIC_HANKO_TENANT_ID
# Verify browser supports WebAuthn
```

**5. Cookie/Session Issues**
```bash
# Clear browser cookies
# Check NEXTAUTH_URL matches actual URL
# Verify NEXTAUTH_SECRET is set
```

**6. OAuth Callback Errors**
```bash
# Verify redirect URIs in OAuth app settings
# Check client ID and secret
# Ensure allowDangerousEmailAccountLinking if needed
```

### Debugging Tips

**Enable Debug Logging:**
```bash
NEXTAUTH_DEBUG=true
```

**Check Session in Browser:**
```javascript
// In browser console
console.log(await fetch('/api/auth/session').then(r => r.json()))
```

**Verify JWT Token:**
```javascript
import jwt from 'jsonwebtoken';
console.log(jwt.decode(token));
```

---

## Production Considerations

### 1. Security Hardening

- Use secure cookies in production
- Implement proper CORS policies
- Add rate limiting to auth endpoints
- Regular security audits
- Monitor for suspicious login attempts

### 2. Performance Optimization

- Database connection pooling
- Session caching with Redis
- Optimize database queries
- CDN for static assets

### 3. Monitoring & Analytics

- Track authentication metrics
- Monitor error rates
- User authentication journey analytics
- Performance monitoring

### 4. Backup & Recovery

- Regular database backups
- Session data backup strategy
- Disaster recovery procedures

---

This comprehensive guide provides everything needed to implement a production-ready authentication system based on the Papermark implementation. The system supports multiple authentication providers, includes robust security features, and provides excellent user experience across all authentication flows.