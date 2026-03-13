# NEXTJS-003: Authentication Design

> Migration guide for the authentication system from Angular's `AuthenticationService` + `angular-oauth2-oidc` to NextAuth.js v5 with App Router.

---

## Table of Contents

1. [Current Angular Auth](#1-current-angular-auth)
2. [NextAuth.js Configuration](#2-nextauthjs-configuration)
3. [Session Management](#3-session-management)
4. [Middleware](#4-middleware)
5. [Auth Context](#5-auth-context)
6. [Remember Me](#6-remember-me)
7. [Multi-Tenant Support](#7-multi-tenant-support)
8. [2FA Support](#8-2fa-support)
9. [Migration Checklist](#9-migration-checklist)

---

## 1. Current Angular Auth

The existing Angular authentication system supports three authentication modes, determined at runtime by environment configuration.

### AuthMode Enum (`src/app/core/authentication/oauth.config.ts`)

```
AuthMode.Basic  -- Direct username/password authentication against Fineract's /authentication endpoint
AuthMode.OAuth2 -- OAuth2 Authorization Code flow with PKCE against a Fineract OAuth server
AuthMode.OIDC   -- OpenID Connect Authorization Code flow with PKCE against Zitadel
```

The active mode is determined by `getActiveAuthMode()`:

- If `environment.OIDC.oidcServerEnabled` is true, use OIDC
- Else if `environment.oauth.enabled` is true, use OAuth2
- Otherwise, use Basic Auth

### AuthenticationService (`src/app/core/authentication/authentication.service.ts`)

Key behaviors:

1. **Login (Basic Auth)**: POST to `/authentication` with `{ username, password }`. Returns a `Credentials` object containing `base64EncodedAuthenticationKey`, `userId`, `username`, `permissions[]`, `roles[]`, `officeId`, `officeName`, and flags like `isTwoFactorAuthenticationRequired` and `shouldRenewPassword`.

2. **Login (OAuth2/OIDC)**: Redirects to the authorization server via `oauthService.initCodeFlow()`. After callback, exchanges the code for tokens using `tryLoginCodeFlow()`, then fetches user details from a Fineract endpoint with the access token.

3. **Session Storage**: Credentials are serialized to JSON and stored in `sessionStorage` (default) or `localStorage` (when Remember Me is enabled). The storage key is `mifosXCredentials`.

4. **Token Management**: For OAuth2/OIDC, the `angular-oauth2-oidc` library handles token storage, silent refresh, and token events. When a token is received or refreshed, the `AuthenticationInterceptor` is updated with the new access token.

5. **Logout**: Clears credentials from storage, removes authorization headers, invalidates 2FA token if present, and for OIDC calls `oauthService.logOut()` to redirect to the provider's logout endpoint.

### AuthenticationGuard (`src/app/core/authentication/authentication.guard.ts`)

A route guard that calls `authenticationService.isAuthenticated()` before allowing route activation. If not authenticated, triggers logout and redirects to `/login`.

### AuthenticationInterceptor (`src/app/core/authentication/authentication.interceptor.ts`)

An HTTP interceptor that adds headers to every request:

- `Authorization` header (Basic or Bearer depending on auth mode)
- `Fineract-Platform-TenantId` header (from SettingsService)
- `Fineract-Platform-TFA-Token` header (when 2FA is active)
- Skips all Fineract headers for external URLs (starting with `http://` or `https://`)

### Credentials Model (`src/app/core/authentication/credentials.model.ts`)

```typescript
interface Credentials {
  accessToken?: string; // OAuth2/OIDC access token
  authenticated: boolean;
  base64EncodedAuthenticationKey?: string; // Basic Auth key
  isTwoFactorAuthenticationRequired?: boolean;
  officeId: number;
  officeName: string;
  staffId?: number;
  staffDisplayName?: string;
  organizationalRole?: any;
  permissions: string[];
  roles: any;
  userId: number;
  username: string;
  shouldRenewPassword: boolean;
  rememberMe?: boolean;
}
```

---

## 2. NextAuth.js Configuration

NextAuth.js v5 (Auth.js) replaces the entire Angular auth stack: `AuthenticationService`, `AuthenticationGuard`, `AuthenticationInterceptor`, and `angular-oauth2-oidc`. It handles all three auth modes through its provider system.

### Route Handler

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/lib/auth/auth-options';

export const { GET, POST } = handlers;
```

### Auth Configuration

```typescript
// lib/auth/auth-options.ts
import NextAuth, { type NextAuthConfig, type User } from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import type { JWT } from 'next-auth/jwt';

/**
 * NextAuth.js v5 configuration.
 *
 * Supports three authentication modes matching the Angular app:
 * 1. Basic Auth (CredentialsProvider) -- default
 * 2. OAuth2 (custom OAuth provider for Fineract OAuth)
 * 3. OIDC (generic OIDC provider for Zitadel)
 *
 * The active mode is determined by NEXT_PUBLIC_AUTH_MODE env var.
 */

const authMode = process.env.NEXT_PUBLIC_AUTH_MODE ?? 'basic';

// Build providers array based on auth mode
function getProviders(): NextAuthConfig['providers'] {
  const providers: NextAuthConfig['providers'] = [];

  // Always include the Credentials provider for Basic Auth
  // (also used as fallback when OAuth/OIDC is configured)
  providers.push(
    CredentialsProvider({
      id: 'fineract-basic',
      name: 'Fineract Basic Auth',
      credentials: {
        username: { label: 'Username', type: 'text' },
        password: { label: 'Password', type: 'password' },
        tenantId: { label: 'Tenant ID', type: 'text' }
      },
      async authorize(credentials): Promise<User | null> {
        if (!credentials?.username || !credentials?.password) {
          return null;
        }

        const tenantId = (credentials.tenantId as string) || process.env.FINERACT_TENANT_ID || 'default';
        const apiUrl = process.env.FINERACT_API_URL!;

        try {
          // Call Fineract /authentication endpoint (same as Angular's login())
          const response = await fetch(`${apiUrl}/authentication`, {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json',
              'Fineract-Platform-TenantId': tenantId
            },
            body: JSON.stringify({
              username: credentials.username,
              password: credentials.password
            })
          });

          if (!response.ok) {
            return null;
          }

          const fineractUser = await response.json();

          // Map Fineract Credentials to NextAuth User
          return {
            id: String(fineractUser.userId),
            name: fineractUser.username,
            username: fineractUser.username,
            userId: fineractUser.userId,
            officeId: fineractUser.officeId,
            officeName: fineractUser.officeName,
            staffId: fineractUser.staffId,
            staffDisplayName: fineractUser.staffDisplayName,
            roles: fineractUser.roles,
            permissions: fineractUser.permissions,
            base64EncodedAuthenticationKey: fineractUser.base64EncodedAuthenticationKey,
            authenticated: fineractUser.authenticated,
            isTwoFactorAuthenticationRequired: fineractUser.isTwoFactorAuthenticationRequired,
            shouldRenewPassword: fineractUser.shouldRenewPassword,
            tenantId,
            authMode: 'basic' as const
          };
        } catch (error) {
          console.error('Fineract authentication failed:', error);
          return null;
        }
      }
    })
  );

  // OIDC provider for Zitadel (when auth mode is oidc)
  if (authMode === 'oidc' && process.env.OIDC_BASE_URL && process.env.OIDC_CLIENT_ID) {
    providers.push({
      id: 'zitadel',
      name: 'Zitadel',
      type: 'oidc',
      issuer: process.env.OIDC_BASE_URL,
      clientId: process.env.OIDC_CLIENT_ID,
      authorization: {
        params: {
          scope: 'openid profile email offline_access',
          response_type: 'code'
        }
      },
      // Map Zitadel profile claims to NextAuth user
      profile(profile) {
        return {
          id: profile.sub,
          name: profile.name ?? profile.preferred_username,
          email: profile.email
        };
      }
    });
  }

  // OAuth2 provider for Fineract OAuth (when auth mode is oauth2)
  if (authMode === 'oauth2' && process.env.OAUTH_APP_ID) {
    providers.push({
      id: 'fineract-oauth2',
      name: 'Fineract OAuth2',
      type: 'oauth',
      authorization: {
        url: process.env.OAUTH_AUTHORIZE_URL!,
        params: {
          scope: process.env.OAUTH_SCOPE ?? '',
          response_type: 'code'
        }
      },
      token: process.env.OAUTH_TOKEN_URL!,
      clientId: process.env.OAUTH_APP_ID,
      // Fineract OAuth2 doesn't have a standard userinfo endpoint,
      // so we fetch user details manually in the jwt callback
      userinfo: undefined,
      profile(profile) {
        return { id: profile.sub ?? profile.id, name: profile.name };
      }
    });
  }

  return providers;
}

const config: NextAuthConfig = {
  providers: getProviders(),
  session: {
    strategy: 'jwt', // JWT strategy (no database required)
    maxAge: 24 * 60 * 60 // 24 hours (matches Fineract token expiry)
  },
  pages: {
    signIn: '/login',
    error: '/login'
  },
  callbacks: {
    /**
     * JWT callback -- called when the JWT is created or updated.
     *
     * For Basic Auth: Store the Fineract credentials in the JWT.
     * For OAuth2/OIDC: Fetch Fineract user details after first login,
     *   store access token and user details in the JWT.
     */
    async jwt({ token, user, account }): Promise<JWT> {
      // Initial sign-in: merge user data into JWT
      if (user) {
        token.userId = user.userId;
        token.username = user.username ?? user.name;
        token.officeId = user.officeId;
        token.officeName = user.officeName;
        token.staffId = user.staffId;
        token.staffDisplayName = user.staffDisplayName;
        token.roles = user.roles;
        token.permissions = user.permissions;
        token.tenantId = user.tenantId;
        token.authMode = user.authMode ?? authMode;
        token.isTwoFactorAuthenticationRequired = user.isTwoFactorAuthenticationRequired;
        token.shouldRenewPassword = user.shouldRenewPassword;

        if (user.authMode === 'basic') {
          // Basic Auth: store the base64 key as the access token
          token.accessToken = user.base64EncodedAuthenticationKey;
        }
      }

      // OAuth2/OIDC: store the access token from the account
      if (account) {
        token.accessToken = account.access_token;
        token.refreshToken = account.refresh_token;
        token.accessTokenExpires = account.expires_at ? account.expires_at * 1000 : undefined;
        token.authMode = account.provider === 'zitadel' ? 'oidc' : 'oauth2';

        // Fetch Fineract user details for OAuth2/OIDC providers
        if (account.provider === 'zitadel' || account.provider === 'fineract-oauth2') {
          const userDetails = await fetchFineractUserDetails(account.access_token!, account.provider);
          if (userDetails) {
            token.userId = userDetails.userId;
            token.username = userDetails.username;
            token.officeId = userDetails.officeId;
            token.officeName = userDetails.officeName;
            token.staffId = userDetails.staffId;
            token.staffDisplayName = userDetails.staffDisplayName;
            token.roles = userDetails.roles;
            token.permissions = userDetails.permissions;
          }
        }
      }

      // Token rotation for OAuth2/OIDC: refresh if expired
      if (token.accessTokenExpires && Date.now() > (token.accessTokenExpires as number) && token.refreshToken) {
        return await refreshAccessToken(token);
      }

      return token;
    },

    /**
     * Session callback -- controls what is exposed to the client via useSession().
     *
     * Only safe, non-secret data is exposed. The actual access token
     * is available but should be used carefully.
     */
    async session({ session, token }) {
      return {
        ...session,
        user: {
          ...session.user,
          id: token.userId as number,
          username: token.username as string,
          officeId: token.officeId as number,
          officeName: token.officeName as string,
          staffId: token.staffId as number | undefined,
          staffDisplayName: token.staffDisplayName as string | undefined,
          roles: token.roles as any[],
          permissions: token.permissions as string[]
        },
        accessToken: token.accessToken as string,
        authMode: token.authMode as string,
        tenantId: token.tenantId as string,
        twoFactorToken: token.twoFactorToken as string | undefined,
        isTwoFactorAuthenticationRequired: token.isTwoFactorAuthenticationRequired as boolean,
        shouldRenewPassword: token.shouldRenewPassword as boolean
      };
    },

    /**
     * Authorized callback -- used by middleware to check if a route is accessible.
     */
    authorized({ auth, request: { nextUrl } }) {
      const isAuthenticated = !!auth?.user;
      const isAuthPage = nextUrl.pathname.startsWith('/login') || nextUrl.pathname.startsWith('/callback');

      if (isAuthPage) {
        if (isAuthenticated) {
          // Redirect authenticated users away from login page
          return Response.redirect(new URL('/', nextUrl));
        }
        return true; // Allow access to login page
      }

      return isAuthenticated; // Protect all other routes
    }
  }
};

/**
 * Fetches Fineract user details after OAuth2/OIDC login.
 *
 * Mirrors the Angular AuthenticationService.getUserDetails() method.
 * For OIDC (Zitadel): POST to the OIDC API URL with the access token.
 * For OAuth2 (Fineract): GET the userdetails endpoint with Bearer token.
 */
async function fetchFineractUserDetails(accessToken: string, provider: string): Promise<FineractUserDetails | null> {
  const tenantId = process.env.FINERACT_TENANT_ID ?? 'default';

  try {
    if (provider === 'zitadel') {
      const oidcApiUrl = process.env.OIDC_API_URL!;
      const response = await fetch(`${oidcApiUrl}authentication/userdetails`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Fineract-Platform-TenantId': tenantId
        },
        body: JSON.stringify({ token: accessToken })
      });
      if (!response.ok) return null;
      const data = await response.json();
      return data.object; // Zitadel wraps the response in { object: Credentials }
    }

    if (provider === 'fineract-oauth2') {
      const oauthServerUrl = process.env.OAUTH_SERVER_URL!;
      const response = await fetch(`${oauthServerUrl}/userdetails`, {
        headers: {
          Authorization: `Bearer ${accessToken}`,
          'Fineract-Platform-TenantId': tenantId
        }
      });
      if (!response.ok) return null;
      return await response.json();
    }

    return null;
  } catch (error) {
    console.error(`Failed to fetch Fineract user details for ${provider}:`, error);
    return null;
  }
}

/**
 * Refreshes the OAuth2/OIDC access token using the refresh token.
 */
async function refreshAccessToken(token: JWT): Promise<JWT> {
  try {
    // Token refresh logic depends on the provider
    // For OIDC (Zitadel), use the standard token endpoint
    // For OAuth2 (Fineract), use the configured token URL
    const tokenUrl = token.authMode === 'oidc' ? `${process.env.OIDC_BASE_URL}/oauth/v2/token` : process.env.OAUTH_TOKEN_URL!;

    const clientId = token.authMode === 'oidc' ? process.env.OIDC_CLIENT_ID! : process.env.OAUTH_APP_ID!;

    const response = await fetch(tokenUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        grant_type: 'refresh_token',
        refresh_token: token.refreshToken as string,
        client_id: clientId
      })
    });

    const refreshed = await response.json();

    if (!response.ok) {
      throw new Error('Token refresh failed');
    }

    return {
      ...token,
      accessToken: refreshed.access_token,
      refreshToken: refreshed.refresh_token ?? token.refreshToken,
      accessTokenExpires: Date.now() + refreshed.expires_in * 1000
    };
  } catch (error) {
    console.error('Token refresh failed:', error);
    return { ...token, error: 'RefreshAccessTokenError' };
  }
}

interface FineractUserDetails {
  userId: number;
  username: string;
  officeId: number;
  officeName: string;
  staffId?: number;
  staffDisplayName?: string;
  roles: any[];
  permissions: string[];
}

export const { handlers, auth, signIn, signOut } = NextAuth(config);
export const authOptions = config;
```

---

## 3. Session Management

### JWT Strategy

The application uses the JWT session strategy (no database required). All session data is encrypted and stored in an HTTP-only cookie.

```typescript
// Session shape (available via useSession() on client, getServerSession() on server)
interface Session {
  user: {
    id: number; // Fineract userId
    username: string;
    officeId: number;
    officeName: string;
    staffId?: number;
    staffDisplayName?: string;
    roles: Array<{ id: number; name: string; description: string }>;
    permissions: string[];
  };
  accessToken: string; // Base64 key (Basic) or OAuth access token
  authMode: 'basic' | 'oauth2' | 'oidc';
  tenantId: string;
  twoFactorToken?: string;
  isTwoFactorAuthenticationRequired: boolean;
  shouldRenewPassword: boolean;
  expires: string; // ISO date string
}
```

### Type Augmentation

```typescript
// types/next-auth.d.ts
import type { DefaultSession, DefaultUser } from 'next-auth';
import type { DefaultJWT } from 'next-auth/jwt';

declare module 'next-auth' {
  interface Session extends DefaultSession {
    user: {
      id: number;
      username: string;
      officeId: number;
      officeName: string;
      staffId?: number;
      staffDisplayName?: string;
      roles: any[];
      permissions: string[];
    } & DefaultSession['user'];
    accessToken: string;
    authMode: 'basic' | 'oauth2' | 'oidc';
    tenantId: string;
    twoFactorToken?: string;
    isTwoFactorAuthenticationRequired: boolean;
    shouldRenewPassword: boolean;
  }

  interface User extends DefaultUser {
    userId?: number;
    username?: string;
    officeId?: number;
    officeName?: string;
    staffId?: number;
    staffDisplayName?: string;
    roles?: any[];
    permissions?: string[];
    base64EncodedAuthenticationKey?: string;
    authenticated?: boolean;
    isTwoFactorAuthenticationRequired?: boolean;
    shouldRenewPassword?: boolean;
    tenantId?: string;
    authMode?: 'basic' | 'oauth2' | 'oidc';
  }
}

declare module 'next-auth/jwt' {
  interface JWT extends DefaultJWT {
    userId?: number;
    username?: string;
    officeId?: number;
    officeName?: string;
    staffId?: number;
    staffDisplayName?: string;
    roles?: any[];
    permissions?: string[];
    accessToken?: string;
    refreshToken?: string;
    accessTokenExpires?: number;
    authMode?: string;
    tenantId?: string;
    twoFactorToken?: string;
    isTwoFactorAuthenticationRequired?: boolean;
    shouldRenewPassword?: boolean;
    error?: string;
  }
}
```

---

## 4. Middleware

The Next.js middleware replaces Angular's `AuthenticationGuard`. It runs on the Edge Runtime before every request.

```typescript
// middleware.ts
export { auth as middleware } from '@/lib/auth/auth-options';

/**
 * Matcher configuration.
 *
 * This defines which routes the middleware runs on.
 * The authorized() callback in auth-options.ts handles the actual logic.
 */
export const config = {
  matcher: [
    /*
     * Match all request paths EXCEPT:
     * - api/auth/* (NextAuth.js API routes -- must be publicly accessible)
     * - _next/static (static files)
     * - _next/image (image optimization)
     * - favicon.ico, images, locales (public assets)
     */
    '/((?!api/auth|_next/static|_next/image|favicon\\.ico|images|locales).*)'
  ]
};
```

### Route Protection Summary

| Route Pattern                         | Access    | Behavior                                  |
| ------------------------------------- | --------- | ----------------------------------------- |
| `/login`                              | Public    | Redirect to `/` if already authenticated  |
| `/callback`                           | Public    | OAuth2/OIDC callback handler              |
| `/api/auth/*`                         | Public    | NextAuth.js API routes                    |
| `/_next/*`, `/images/*`, `/locales/*` | Public    | Static assets (excluded from middleware)  |
| `/*` (everything else)                | Protected | Redirect to `/login` if not authenticated |

### Password Reset and 2FA Interception

After login, the middleware also checks for special conditions that the Angular app handled via alerts:

```typescript
// Extended middleware logic (inside auth-options.ts authorized callback)
authorized({ auth, request: { nextUrl } }) {
  const isAuthenticated = !!auth?.user;
  const isAuthPage = nextUrl.pathname.startsWith('/login') ||
                     nextUrl.pathname.startsWith('/callback');
  const isPasswordResetPage = nextUrl.pathname === '/reset-password';
  const isTwoFactorPage = nextUrl.pathname === '/two-factor';

  // Allow access to auth pages
  if (isAuthPage) {
    return isAuthenticated ? Response.redirect(new URL('/', nextUrl)) : true;
  }

  // Must be authenticated for everything below
  if (!isAuthenticated) return false;

  // Force password reset if shouldRenewPassword is true
  if (auth.shouldRenewPassword && !isPasswordResetPage) {
    return Response.redirect(new URL('/reset-password', nextUrl));
  }

  // Force 2FA if required and not yet verified
  if (
    auth.isTwoFactorAuthenticationRequired &&
    !auth.twoFactorToken &&
    !isTwoFactorPage
  ) {
    return Response.redirect(new URL('/two-factor', nextUrl));
  }

  return true;
},
```

---

## 5. Auth Context

A React context provides permission-checking utilities throughout the component tree. This replaces the Angular pattern of injecting `AuthenticationService` and manually checking `credentials.permissions`.

```typescript
// lib/auth/auth-context.tsx
'use client';

import { createContext, useContext, useMemo, type ReactNode } from 'react';
import { useSession, signOut } from 'next-auth/react';
import type { Session } from 'next-auth';

interface AuthContextValue {
  /** The full NextAuth session object. */
  session: Session | null;

  /** Whether the user is authenticated. */
  isAuthenticated: boolean;

  /** Whether the session is still loading. */
  isLoading: boolean;

  /**
   * Check if the user has a specific Fineract permission.
   *
   * Fineract permissions are strings like:
   * - 'ALL_FUNCTIONS' (superuser)
   * - 'READ_CLIENT'
   * - 'CREATE_CLIENT'
   * - 'APPROVE_LOAN'
   * - 'DISBURSE_LOAN'
   *
   * @param permissionCode The Fineract permission code to check.
   * @returns True if the user has the permission.
   */
  hasPermission: (permissionCode: string) => boolean;

  /**
   * Check if the user has ANY of the specified permissions.
   * Useful for showing/hiding UI elements that require one of several permissions.
   */
  hasAnyPermission: (permissionCodes: string[]) => boolean;

  /**
   * Check if the user has ALL of the specified permissions.
   */
  hasAllPermissions: (permissionCodes: string[]) => boolean;

  /**
   * Check if the user has a specific role.
   * @param roleName The role name to check (e.g., 'Super user').
   */
  hasRole: (roleName: string) => boolean;

  /**
   * The current user's office ID.
   */
  officeId: number | undefined;

  /**
   * The current tenant ID.
   */
  tenantId: string | undefined;

  /**
   * Log out the user.
   */
  logout: () => Promise<void>;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const { data: session, status } = useSession();
  const isLoading = status === 'loading';
  const isAuthenticated = status === 'authenticated';

  const permissions = useMemo(
    () => new Set(session?.user?.permissions ?? []),
    [session?.user?.permissions],
  );

  const value = useMemo<AuthContextValue>(
    () => ({
      session: session ?? null,
      isAuthenticated,
      isLoading,

      hasPermission(code: string): boolean {
        if (permissions.has('ALL_FUNCTIONS')) return true;
        return permissions.has(code);
      },

      hasAnyPermission(codes: string[]): boolean {
        if (permissions.has('ALL_FUNCTIONS')) return true;
        return codes.some((code) => permissions.has(code));
      },

      hasAllPermissions(codes: string[]): boolean {
        if (permissions.has('ALL_FUNCTIONS')) return true;
        return codes.every((code) => permissions.has(code));
      },

      hasRole(roleName: string): boolean {
        return session?.user?.roles?.some(
          (role: any) => role.name === roleName,
        ) ?? false;
      },

      officeId: session?.user?.officeId,
      tenantId: session?.tenantId,

      async logout() {
        await signOut({ callbackUrl: '/login' });
      },
    }),
    [session, isAuthenticated, isLoading, permissions],
  );

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

/**
 * Hook to access authentication context.
 *
 * Usage:
 *   const { hasPermission, logout } = useAuth();
 *   if (hasPermission('APPROVE_LOAN')) { ... }
 */
export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

### Usage Examples

```tsx
// Conditional rendering based on permissions
function LoanActions({ loanId }: { loanId: number }) {
  const { hasPermission } = useAuth();

  return (
    <div className="flex gap-2">
      {hasPermission('APPROVE_LOAN') && <Button onClick={() => approveLoan(loanId)}>Approve</Button>}
      {hasPermission('DISBURSE_LOAN') && <Button onClick={() => disburseLoan(loanId)}>Disburse</Button>}
      {hasPermission('REJECT_LOAN') && (
        <Button variant="destructive" onClick={() => rejectLoan(loanId)}>
          Reject
        </Button>
      )}
    </div>
  );
}

// Protecting a page component
function AdminSettingsPage() {
  const { hasRole, isLoading } = useAuth();

  if (isLoading) return <Skeleton />;
  if (!hasRole('Super user')) return <Forbidden />;

  return <div>{/* admin settings */}</div>;
}
```

### Provider Setup in Root Layout

```tsx
// app/layout.tsx
import { SessionProvider } from 'next-auth/react';
import { AuthProvider } from '@/lib/auth/auth-context';
import { QueryProvider } from '@/lib/providers/query-provider';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <SessionProvider>
          <AuthProvider>
            <QueryProvider>{children}</QueryProvider>
          </AuthProvider>
        </SessionProvider>
      </body>
    </html>
  );
}
```

---

## 6. Remember Me

The Angular app supports Remember Me through `environment.enableRememberMe`. When enabled and the user checks "Remember Me", credentials are stored in `localStorage` (persistent) instead of `sessionStorage` (cleared on tab close).

### Next.js Approach

NextAuth.js uses HTTP-only cookies for session management. The "Remember Me" feature is implemented by varying the cookie `maxAge`:

```typescript
// In auth-options.ts, modify session config based on remember me
session: {
  strategy: 'jwt',
  // maxAge is set dynamically in the jwt callback based on the login request
  maxAge: 24 * 60 * 60, // Default: 24 hours (no remember me)
},

// Cookie configuration for remember me
cookies: {
  sessionToken: {
    name: process.env.NODE_ENV === 'production'
      ? '__Secure-next-auth.session-token'
      : 'next-auth.session-token',
    options: {
      httpOnly: true,
      sameSite: 'lax',
      path: '/',
      secure: process.env.NODE_ENV === 'production',
      // maxAge is set per-response based on remember me preference
    },
  },
},
```

### Login Page Implementation

```tsx
// app/(auth)/login/page.tsx
'use client';

import { useState } from 'react';
import { signIn } from 'next-auth/react';
import { useRouter } from 'next/navigation';

export default function LoginPage() {
  const router = useRouter();
  const [
    rememberMe,
    setRememberMe
  ] = useState(false);
  const [
    error,
    setError
  ] = useState<string | null>(null);
  const [
    isLoading,
    setIsLoading
  ] = useState(false);
  const authMode = process.env.NEXT_PUBLIC_AUTH_MODE ?? 'basic';

  async function handleBasicLogin(formData: FormData) {
    setIsLoading(true);
    setError(null);

    const result = await signIn('fineract-basic', {
      username: formData.get('username') as string,
      password: formData.get('password') as string,
      tenantId: formData.get('tenantId') as string,
      redirect: false
    });

    setIsLoading(false);

    if (result?.error) {
      setError('Invalid username or password.');
    } else {
      router.push('/');
    }
  }

  async function handleOAuthLogin() {
    const provider = authMode === 'oidc' ? 'zitadel' : 'fineract-oauth2';
    await signIn(provider, { callbackUrl: '/' });
  }

  // Render login form based on auth mode
  if (authMode !== 'basic') {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <Button onClick={handleOAuthLogin} size="lg">
          Sign in with {authMode === 'oidc' ? 'Zitadel' : 'OAuth2'}
        </Button>
      </div>
    );
  }

  return (
    <form action={handleBasicLogin} className="space-y-4">
      <Input name="username" placeholder="Username" required />
      <Input name="password" type="password" placeholder="Password" required />
      {process.env.NEXT_PUBLIC_DISPLAY_TENANT_SELECTOR === 'true' && <Input name="tenantId" placeholder="Tenant ID" defaultValue="default" />}
      <div className="flex items-center gap-2">
        <Checkbox id="remember" checked={rememberMe} onCheckedChange={(checked) => setRememberMe(checked === true)} />
        <label htmlFor="remember">Remember me</label>
      </div>
      {error && <p className="text-red-500 text-sm">{error}</p>}
      <Button type="submit" className="w-full" disabled={isLoading}>
        {isLoading ? 'Signing in...' : 'Sign In'}
      </Button>
    </form>
  );
}
```

---

## 7. Multi-Tenant Support

The Angular app supports multiple tenants through `SettingsService.tenantIdentifier`, configurable via the settings page and stored in `localStorage`.

### Next.js Approach

The tenant ID is stored in the NextAuth JWT and passed on every API request.

```typescript
// Tenant is set during login and stored in the JWT
// See Section 2: credentials.tenantId is captured in the authorize() function

// To switch tenants, the user must log out and log back in with a different tenant ID
// This matches the Angular behavior where changing the tenant requires re-authentication
```

### Tenant Selector Component

```tsx
// components/shared/tenant-selector.tsx
'use client';

import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';

const tenantIds = (process.env.NEXT_PUBLIC_TENANT_IDS ?? 'default').split(',');

interface TenantSelectorProps {
  value: string;
  onChange: (tenantId: string) => void;
}

export function TenantSelector({ value, onChange }: TenantSelectorProps) {
  if (process.env.NEXT_PUBLIC_DISPLAY_TENANT_SELECTOR !== 'true') {
    return null;
  }

  return (
    <Select value={value} onValueChange={onChange}>
      <SelectTrigger className="w-48">
        <SelectValue placeholder="Select tenant" />
      </SelectTrigger>
      <SelectContent>
        {tenantIds.map((id) => (
          <SelectItem key={id} value={id.trim()}>
            {id.trim()}
          </SelectItem>
        ))}
      </SelectContent>
    </Select>
  );
}
```

---

## 8. 2FA Support

The Angular app supports two-factor authentication via Fineract's `/twofactor` endpoints. After initial login, if `isTwoFactorAuthenticationRequired` is true, the user must complete 2FA before accessing the application.

### Flow

1. User logs in with username/password
2. Fineract returns `Credentials` with `isTwoFactorAuthenticationRequired: true`
3. User is redirected to `/two-factor` page
4. User selects a delivery method (e.g., SMS, email)
5. OTP is sent via `POST /twofactor?deliveryMethod=...`
6. User enters OTP
7. OTP is validated via `POST /twofactor/validate?token=...`
8. On success, the 2FA token is stored in the session

### Implementation

```typescript
// lib/services/two-factor.ts
'use client';

import fineractApi from '@/lib/api/client';

export async function getDeliveryMethods() {
  const { data } = await fineractApi.get('/twofactor');
  return data;
}

export async function requestOTP(deliveryMethod: string, extendedToken: boolean) {
  const { data } = await fineractApi.post(
    '/twofactor',
    {},
    {
      params: { deliveryMethod, extendedToken: String(extendedToken) }
    }
  );
  return data;
}

export async function validateOTP(token: string) {
  const { data } = await fineractApi.post(
    '/twofactor/validate',
    {},
    {
      params: { token }
    }
  );
  return data;
}
```

```tsx
// app/(auth)/two-factor/page.tsx
'use client';

import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { getDeliveryMethods, requestOTP, validateOTP } from '@/lib/services/two-factor';

export default function TwoFactorPage() {
  const router = useRouter();
  const [
    step,
    setStep
  ] = useState<'select' | 'verify'>('select');
  const [
    deliveryMethods,
    setDeliveryMethods
  ] = useState<any[]>([]);
  const [
    selectedMethod,
    setSelectedMethod
  ] = useState<string>('');
  const [
    otp,
    setOtp
  ] = useState('');

  // Load delivery methods on mount
  // ... (useEffect to call getDeliveryMethods)

  async function handleRequestOTP() {
    await requestOTP(selectedMethod, false);
    setStep('verify');
  }

  async function handleValidateOTP() {
    const result = await validateOTP(otp);
    // Update the NextAuth session with the 2FA token
    // This requires a custom API route to update the JWT
    await fetch('/api/auth/update-2fa', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ twoFactorToken: result.token })
    });
    router.push('/');
  }

  // Render delivery method selection or OTP input based on step
  return (
    <div className="flex min-h-screen items-center justify-center">
      {step === 'select' ? (
        <div>
          <h1>Two-Factor Authentication</h1>
          {/* Delivery method selector */}
          <Button onClick={handleRequestOTP}>Send Code</Button>
        </div>
      ) : (
        <div>
          <h1>Enter Verification Code</h1>
          <Input value={otp} onChange={(e) => setOtp(e.target.value)} placeholder="Enter OTP" />
          <Button onClick={handleValidateOTP}>Verify</Button>
        </div>
      )}
    </div>
  );
}
```

### Updating the JWT with 2FA Token

```typescript
// app/api/auth/update-2fa/route.ts
import { auth } from '@/lib/auth/auth-options';
import { NextResponse } from 'next/server';
import { encode } from 'next-auth/jwt';

export async function POST(request: Request) {
  const session = await auth();
  if (!session) {
    return NextResponse.json({ error: 'Not authenticated' }, { status: 401 });
  }

  const { twoFactorToken } = await request.json();

  // The 2FA token will be picked up by the jwt callback on the next session check
  // Store it in a cookie that the jwt callback reads
  const response = NextResponse.json({ success: true });

  // Encode the updated JWT with the 2FA token
  // This is a simplified approach; in production you may need to
  // update the session token cookie directly
  response.cookies.set('mifos-2fa-token', twoFactorToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 // 24 hours
  });

  return response;
}
```

---

## 9. Migration Checklist

Step-by-step guide for migrating from Angular auth to NextAuth.js.

### Phase 1: Setup

- [ ] Install NextAuth.js v5: `npm install next-auth@beta`
- [ ] Create `app/api/auth/[...nextauth]/route.ts`
- [ ] Create `lib/auth/auth-options.ts` with CredentialsProvider
- [ ] Create `types/next-auth.d.ts` with type augmentations
- [ ] Set `NEXTAUTH_SECRET` and `NEXTAUTH_URL` in `.env.local`
- [ ] Set `FINERACT_API_URL` and `FINERACT_TENANT_ID` in `.env.local`

### Phase 2: Basic Auth

- [ ] Implement CredentialsProvider `authorize()` function
- [ ] Verify login/logout flow works with Fineract `/authentication` endpoint
- [ ] Implement `jwt` callback to store Fineract credentials in JWT
- [ ] Implement `session` callback to expose user data
- [ ] Test: Login with valid credentials returns session with userId, permissions, etc.
- [ ] Test: Login with invalid credentials shows error
- [ ] Test: Logout clears session and redirects to `/login`

### Phase 3: Route Protection

- [ ] Create `middleware.ts` with route matcher
- [ ] Implement `authorized` callback in auth config
- [ ] Verify protected routes redirect to `/login`
- [ ] Verify `/login` redirects to `/` when already authenticated
- [ ] Test: Direct URL access to protected route without session redirects to login

### Phase 4: Auth Context

- [ ] Create `lib/auth/auth-context.tsx` with `AuthProvider` and `useAuth()`
- [ ] Add `SessionProvider` and `AuthProvider` to root layout
- [ ] Implement `hasPermission()`, `hasRole()`, `hasAnyPermission()`
- [ ] Test: Components correctly show/hide based on permissions

### Phase 5: OAuth2/OIDC

- [ ] Add OIDC provider for Zitadel
- [ ] Add OAuth2 provider for Fineract OAuth
- [ ] Implement `fetchFineractUserDetails()` for post-login user data fetch
- [ ] Implement token refresh in `jwt` callback
- [ ] Create `/callback` page for OAuth redirect handling
- [ ] Test: Full OIDC flow with Zitadel (login, callback, session, logout)
- [ ] Test: Full OAuth2 flow with Fineract OAuth
- [ ] Test: Token refresh works when access token expires

### Phase 6: 2FA

- [ ] Create `/two-factor` page
- [ ] Implement delivery method selection
- [ ] Implement OTP request and validation
- [ ] Create API route to update JWT with 2FA token
- [ ] Add middleware check for `isTwoFactorAuthenticationRequired`
- [ ] Test: 2FA flow end-to-end

### Phase 7: Additional Features

- [ ] Implement Remember Me (cookie maxAge variation)
- [ ] Implement password reset flow (`shouldRenewPassword`)
- [ ] Implement tenant selector on login page
- [ ] Port session idle timeout (use `next-auth` events + client-side timer)
- [ ] Implement `signOut` cleanup (invalidate 2FA token on server)

### Phase 8: Cleanup

- [ ] Remove all Angular auth files after migration is complete
- [ ] Remove `angular-oauth2-oidc` dependency
- [ ] Update all components using `AuthenticationService` to use `useAuth()`
- [ ] Update all HTTP calls to use the new API client (which reads auth from NextAuth session)
- [ ] Run full E2E test suite against all three auth modes
