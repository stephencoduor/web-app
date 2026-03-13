# NEXTJS-007: Internationalization

## Status: Draft

## Last Updated: 2026-03-13

---

## 1. Current Angular i18n Implementation

The M-SACCO Angular app uses **ngx-translate** for internationalization with 13 JSON translation files stored in `src/assets/translations/`.

### 1.1 Supported Locales

| Locale Code | Language         | File         |
| ----------- | ---------------- | ------------ |
| `en-US`     | English (US)     | `en-US.json` |
| `cs-CS`     | Czech            | `cs-CS.json` |
| `de-DE`     | German           | `de-DE.json` |
| `es-CL`     | Spanish (Chile)  | `es-CL.json` |
| `es-MX`     | Spanish (Mexico) | `es-MX.json` |
| `fr-FR`     | French           | `fr-FR.json` |
| `it-IT`     | Italian          | `it-IT.json` |
| `ko-KO`     | Korean           | `ko-KO.json` |
| `lt-LT`     | Lithuanian       | `lt-LT.json` |
| `lv-LV`     | Latvian          | `lv-LV.json` |
| `ne-NE`     | Nepali           | `ne-NE.json` |
| `pt-PT`     | Portuguese       | `pt-PT.json` |
| `sw-SW`     | Swahili          | `sw-SW.json` |

### 1.2 Current Translation File Format

The Angular translation files use a **mixed** JSON format with both flat dot-notation keys and nested objects:

```json
{
  "APP_NAME": "Mifos X WebApp",
  "Logged in as": "Logged in as",
  "Remember me": "Remember me",
  "errors": {
    "accountingRule": {
      "duplicateName": "Sorry, but an Accounting Rule with this Name exists already."
    },
    "linkedSavingsAccountOwnership": "Linked savings account does not belong to the selected client.",
    "clientNotInGSIM": "Client with ID {{id}} is not present in GSIM."
  },
  "validation.msg.charge.amount.cannot.be.blank": "Amount cannot be blank."
}
```

Key observations:

- Mix of flat keys (`"APP_NAME"`) and nested objects (`"errors": { ... }`)
- Some keys use dot notation as literal key names (`"validation.msg.charge..."`)
- Interpolation uses double-brace syntax: `{{id}}`, `{{params[0].value}}`
- Many keys are human-readable English strings used directly as keys (e.g., `"Logged in as"`)

### 1.3 Current Usage in Angular

```html
<!-- Template pipe usage -->
<span>{{ 'labels.clients' | translate }}</span>

<!-- With parameters -->
<span>{{ 'errors.clientNotInGSIM' | translate: { id: clientId } }}</span>

<!-- In TypeScript -->
this.translateService.instant('labels.clients');
```

---

## 2. next-intl Setup

The Next.js app uses **next-intl** which provides first-class support for the App Router, Server Components, and Client Components.

### 2.1 Package Installation

```bash
npm install next-intl
```

### 2.2 Core Configuration Files

**`src/lib/i18n/config.ts`** -- Locale definitions:

```typescript
export const locales = [
  'en-US',
  'cs-CS',
  'de-DE',
  'es-CL',
  'es-MX',
  'fr-FR',
  'it-IT',
  'ko-KO',
  'lt-LT',
  'lv-LV',
  'ne-NE',
  'pt-PT',
  'sw-SW'
] as const;

export type Locale = (typeof locales)[number];
export const defaultLocale: Locale = 'en-US';
```

**`src/lib/i18n/request.ts`** -- Server-side locale resolution:

```typescript
import { getRequestConfig } from 'next-intl/server';
import { cookies, headers } from 'next/headers';
import { defaultLocale, locales, type Locale } from './config';

export default getRequestConfig(async () => {
  // Priority: 1) Cookie, 2) Accept-Language header, 3) Default
  const cookieStore = await cookies();
  const localeCookie = cookieStore.get('NEXT_LOCALE')?.value;

  let locale: Locale = defaultLocale;

  if (localeCookie && locales.includes(localeCookie as Locale)) {
    locale = localeCookie as Locale;
  } else {
    const headerStore = await headers();
    const acceptLanguage = headerStore.get('accept-language');
    if (acceptLanguage) {
      const preferred = acceptLanguage
        .split(',')
        .map((lang) => lang.split(';')[0].trim())
        .find((lang) => locales.includes(lang as Locale));
      if (preferred) {
        locale = preferred as Locale;
      }
    }
  }

  return {
    locale,
    messages: (await import(`./messages/${locale}.json`)).default
  };
});
```

**`next.config.ts`** -- Plugin integration:

```typescript
import createNextIntlPlugin from 'next-intl/plugin';

const withNextIntl = createNextIntlPlugin('./src/lib/i18n/request.ts');

const nextConfig = {
  // ... other config
};

export default withNextIntl(nextConfig);
```

### 2.3 Message Files Location

```
src/lib/i18n/
├── config.ts
├── request.ts
└── messages/
    ├── en-US.json
    ├── cs-CS.json
    ├── de-DE.json
    ├── es-CL.json
    ├── es-MX.json
    ├── fr-FR.json
    ├── it-IT.json
    ├── ko-KO.json
    ├── lt-LT.json
    ├── lv-LV.json
    ├── ne-NE.json
    ├── pt-PT.json
    └── sw-SW.json
```

### 2.4 Locale Detection Strategy

The detection follows this priority order:

1. **Cookie** (`NEXT_LOCALE`): Set by the language switcher component. Persists across sessions.
2. **Accept-Language header**: Browser preference, used on first visit.
3. **Default**: Falls back to `en-US`.

This approach avoids URL-prefix-based locale routing (e.g., `/en-US/clients`) to minimize disruption to existing URL structures. The locale is stored as a cookie, not in the URL path.

---

## 3. Migration of Translation Files

### 3.1 Format Decision

**Recommended: Keep flat key format.** next-intl supports both flat and nested JSON. The Angular files use a mix, and converting all to nested would require changing every key reference. Instead, keep the existing key structure and normalize only the interpolation syntax.

### 3.2 Key Changes Required

| Angular (ngx-translate)      | next-intl                         | Notes                                      |
| ---------------------------- | --------------------------------- | ------------------------------------------ |
| `{{variable}}`               | `{variable}`                      | Single braces for interpolation            |
| `{{params[0].value}}`        | `{param0Value}`                   | Rename complex expressions to simple names |
| `translate` pipe             | `t('key')` function               | Usage pattern change                       |
| `translateService.instant()` | `t('key')` or `getTranslations()` | API change                                 |

### 3.3 Automated Migration Script

Create a Node.js script to convert all translation files:

```typescript
// scripts/migrate-translations.ts
import fs from 'fs';
import path from 'path';

const SOURCE_DIR = 'src/assets/translations';
const TARGET_DIR = 'src/lib/i18n/messages';

const locales = [
  'en-US',
  'cs-CS',
  'de-DE',
  'es-CL',
  'es-MX',
  'fr-FR',
  'it-IT',
  'ko-KO',
  'lt-LT',
  'lv-LV',
  'ne-NE',
  'pt-PT',
  'sw-SW'
];

function convertInterpolation(value: string): string {
  // Convert {{variable}} to {variable}
  let result = value.replace(/\{\{(\w+)\}\}/g, '{$1}');

  // Convert {{params[0].value}} to {param0Value}
  result = result.replace(/\{\{params\[(\d+)\]\.value\}\}/g, (_, index) => `{param${index}Value}`);

  return result;
}

function processValue(value: unknown): unknown {
  if (typeof value === 'string') {
    return convertInterpolation(value);
  }
  if (typeof value === 'object' && value !== null) {
    const result: Record<string, unknown> = {};
    for (const [
      key,
      val
    ] of Object.entries(value)) {
      result[key] = processValue(val);
    }
    return result;
  }
  return value;
}

function migrateFile(locale: string) {
  const sourcePath = path.join(SOURCE_DIR, `${locale}.json`);
  const targetPath = path.join(TARGET_DIR, `${locale}.json`);

  const content = JSON.parse(fs.readFileSync(sourcePath, 'utf-8'));
  const migrated = processValue(content);

  fs.mkdirSync(TARGET_DIR, { recursive: true });
  fs.writeFileSync(targetPath, JSON.stringify(migrated, null, 2) + '\n');

  console.log(`Migrated: ${locale}`);
}

locales.forEach(migrateFile);
console.log('Migration complete.');
```

Run with:

```bash
npx tsx scripts/migrate-translations.ts
```

### 3.4 Validation Script

After migration, validate that all keys referenced in source code exist in the translation files:

```typescript
// scripts/validate-translations.ts
import fs from 'fs';
import path from 'path';
import { glob } from 'glob';

const MESSAGES_DIR = 'src/lib/i18n/messages';

async function extractKeysFromCode(): Promise<Set<string>> {
  const keys = new Set<string>();
  const files = await glob('src/**/*.{ts,tsx}', { ignore: 'node_modules/**' });

  for (const file of files) {
    const content = fs.readFileSync(file, 'utf-8');
    // Match t('key'), t("key"), getTranslations('namespace')
    const matches = content.matchAll(/t\(['"]([^'"]+)['"]\)/g);
    for (const match of matches) {
      keys.add(match[1]);
    }
  }

  return keys;
}

async function validate() {
  const enMessages = JSON.parse(fs.readFileSync(path.join(MESSAGES_DIR, 'en-US.json'), 'utf-8'));

  const codeKeys = await extractKeysFromCode();
  const messageKeys = new Set(flattenKeys(enMessages));

  const missing = [...codeKeys].filter((k) => !messageKeys.has(k));
  const unused = [...messageKeys].filter((k) => !codeKeys.has(k));

  if (missing.length) {
    console.warn(`Missing ${missing.length} keys in en-US.json:`);
    missing.forEach((k) => console.warn(`  - ${k}`));
  }

  if (unused.length) {
    console.info(`Potentially unused ${unused.length} keys in en-US.json`);
  }
}

function flattenKeys(obj: Record<string, unknown>, prefix = ''): string[] {
  const keys: string[] = [];
  for (const [
    key,
    value
  ] of Object.entries(obj)) {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === 'object' && value !== null) {
      keys.push(...flattenKeys(value as Record<string, unknown>, fullKey));
    } else {
      keys.push(fullKey);
    }
  }
  return keys;
}

validate();
```

---

## 4. Usage Patterns

### 4.1 Server Components

```typescript
// app/(dashboard)/clients/page.tsx
import { getTranslations } from 'next-intl/server';

export default async function ClientsPage() {
  const t = await getTranslations();

  return (
    <div>
      <h1>{t('labels.clients')}</h1>
      {/* ... */}
    </div>
  );
}
```

With namespaced translations:

```typescript
export default async function ClientsPage() {
  const t = await getTranslations('clients');

  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('description')}</p>
    </div>
  );
}
```

### 4.2 Client Components

```typescript
'use client';
import { useTranslations } from 'next-intl';

export function ClientForm() {
  const t = useTranslations();

  return (
    <form>
      <label>{t('labels.firstName')}</label>
      <input placeholder={t('placeholders.enterFirstName')} />
      <button type="submit">{t('labels.buttons.submit')}</button>
    </form>
  );
}
```

### 4.3 Angular to Next.js Translation Reference

**Angular template pipe:**

```html
<!-- Before -->
<mat-label>{{ 'labels.firstName' | translate }}</mat-label>
<span>{{ 'errors.clientNotInGSIM' | translate: { id: clientId } }}</span>
```

**Next.js equivalent:**

```tsx
// After (Client Component)
const t = useTranslations();
<Label>{t('labels.firstName')}</Label>
<span>{t('errors.clientNotInGSIM', { id: clientId })}</span>
```

**Angular TypeScript:**

```typescript
// Before
const msg = this.translateService.instant('labels.success');
```

**Next.js equivalent:**

```typescript
// After (Server Component)
const t = await getTranslations();
const msg = t('labels.success');

// After (Client Component)
const t = useTranslations();
const msg = t('labels.success');
```

### 4.4 Metadata Translation

```typescript
// app/(dashboard)/clients/page.tsx
import { getTranslations } from 'next-intl/server';

export async function generateMetadata() {
  const t = await getTranslations();
  return { title: t('labels.clients') };
}
```

---

## 5. Language Switcher Component

```typescript
// components/language-switcher.tsx
'use client';

import { useLocale } from 'next-intl';
import { useRouter } from 'next/navigation';
import { useTransition } from 'react';
import { locales, type Locale } from '@/lib/i18n/config';

const LOCALE_NAMES: Record<Locale, string> = {
  'en-US': 'English',
  'cs-CS': 'Cestina',
  'de-DE': 'Deutsch',
  'es-CL': 'Espanol (Chile)',
  'es-MX': 'Espanol (Mexico)',
  'fr-FR': 'Francais',
  'it-IT': 'Italiano',
  'ko-KO': 'Korean',
  'lt-LT': 'Lietuviu',
  'lv-LV': 'Latviesu',
  'ne-NE': 'Nepali',
  'pt-PT': 'Portugues',
  'sw-SW': 'Kiswahili',
};

export function LanguageSwitcher() {
  const locale = useLocale();
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  function handleChange(newLocale: string) {
    // Set cookie and refresh
    document.cookie = `NEXT_LOCALE=${newLocale};path=/;max-age=31536000;samesite=lax`;
    startTransition(() => {
      router.refresh();
    });
  }

  return (
    <select
      value={locale}
      onChange={(e) => handleChange(e.target.value)}
      disabled={isPending}
      className="rounded border px-2 py-1 text-sm"
    >
      {locales.map((loc) => (
        <option key={loc} value={loc}>
          {LOCALE_NAMES[loc]}
        </option>
      ))}
    </select>
  );
}
```

---

## 6. Date, Number, and Currency Formatting

### 6.1 Date Formatting

next-intl provides formatting utilities that use the `Intl` API under the hood. This replaces Angular's `DatePipe` with locale-dependent formatting.

```typescript
// Server Component
import { getFormatter } from 'next-intl/server';

export default async function LoanDetails({ loan }: { loan: LoanData }) {
  const format = await getFormatter();

  return (
    <div>
      <p>Disbursed: {format.dateTime(new Date(loan.disbursementDate), {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
      })}</p>
      <p>Amount: {format.number(loan.principal, {
        style: 'currency',
        currency: loan.currency.code,
      })}</p>
    </div>
  );
}
```

```typescript
// Client Component
'use client';
import { useFormatter } from 'next-intl';

export function TransactionRow({ txn }: { txn: Transaction }) {
  const format = useFormatter();

  return (
    <tr>
      <td>{format.dateTime(new Date(txn.date), { dateStyle: 'medium' })}</td>
      <td>{format.number(txn.amount, { style: 'currency', currency: txn.currency })}</td>
    </tr>
  );
}
```

### 6.2 Fineract Date Format Handling

Fineract returns dates as arrays `[year, month, day]` (e.g., `[2024, 3, 15]`). Create a utility for consistent conversion:

```typescript
// lib/utils/date.ts
export function fineractDateToDate(dateArray: number[]): Date {
  const [
    year,
    month,
    day
  ] = dateArray;
  return new Date(year, month - 1, day); // month is 0-indexed in JS
}

export function dateToFineractFormat(date: Date, format = 'dd MMMM yyyy'): string {
  // Fineract expects specific date format strings
  return date.toISOString().split('T')[0]; // or use date-fns/format
}
```

### 6.3 Relative Time

```typescript
import { useFormatter } from 'next-intl';

function TimeAgo({ date }: { date: Date }) {
  const format = useFormatter();
  return <span>{format.relativeTime(date)}</span>;
}
```

---

## 7. RTL Support

Currently none of the 13 supported locales are RTL languages, but the architecture should support future additions (Arabic, Hebrew, Urdu are common in microfinance).

### 7.1 Direction Detection

```typescript
// lib/i18n/direction.ts
const RTL_LOCALES = new Set([
  'ar-SA',
  'he-IL',
  'ur-PK'
]);

export function getDirection(locale: string): 'ltr' | 'rtl' {
  return RTL_LOCALES.has(locale) ? 'rtl' : 'ltr';
}
```

### 7.2 Root Layout Integration

```typescript
// app/layout.tsx
import { getLocale } from 'next-intl/server';
import { getDirection } from '@/lib/i18n/direction';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const locale = await getLocale();
  const dir = getDirection(locale);

  return (
    <html lang={locale} dir={dir} suppressHydrationWarning>
      <body>{children}</body>
    </html>
  );
}
```

### 7.3 CSS Considerations

Use CSS logical properties instead of physical ones:

```css
/* Instead of */
.sidebar {
  margin-left: 16px;
  padding-right: 8px;
}

/* Use logical properties */
.sidebar {
  margin-inline-start: 16px;
  padding-inline-end: 8px;
}
```

Tailwind CSS supports logical properties via `ms-*` (margin-start), `me-*` (margin-end), `ps-*` (padding-start), `pe-*` (padding-end).

---

## 8. Translation Key Extraction Tooling

### 8.1 During Development

Use the `next-intl` development warning feature. When a key is missing, it logs a warning in development mode:

```typescript
// next-intl configuration in request.ts
export default getRequestConfig(async () => {
  return {
    locale,
    messages: (await import(`./messages/${locale}.json`)).default,
    onError(error) {
      if (error.code === 'MISSING_MESSAGE') {
        console.warn(`Missing translation: ${error.originalMessage}`);
      }
    },
    getMessageFallback({ namespace, key }) {
      return `${namespace}.${key}`; // Show the key as fallback
    }
  };
});
```

### 8.2 CI Extraction Check

Add a lint step that ensures all `t('...')` and `getTranslations('...')` calls reference existing keys:

```json
// package.json
{
  "scripts": {
    "i18n:validate": "tsx scripts/validate-translations.ts",
    "i18n:check-missing": "tsx scripts/check-missing-translations.ts"
  }
}
```

The `check-missing-translations.ts` script compares all locale files against `en-US.json` to find untranslated keys:

```typescript
// scripts/check-missing-translations.ts
import fs from 'fs';
import path from 'path';

const MESSAGES_DIR = 'src/lib/i18n/messages';
const BASE_LOCALE = 'en-US';

function flattenKeys(obj: Record<string, unknown>, prefix = ''): string[] {
  const keys: string[] = [];
  for (const [
    key,
    value
  ] of Object.entries(obj)) {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === 'object' && value !== null) {
      keys.push(...flattenKeys(value as Record<string, unknown>, fullKey));
    } else {
      keys.push(fullKey);
    }
  }
  return keys;
}

const baseMessages = JSON.parse(fs.readFileSync(path.join(MESSAGES_DIR, `${BASE_LOCALE}.json`), 'utf-8'));
const baseKeys = new Set(flattenKeys(baseMessages));

const localeFiles = fs.readdirSync(MESSAGES_DIR).filter((f) => f.endsWith('.json') && f !== `${BASE_LOCALE}.json`);

let hasErrors = false;

for (const file of localeFiles) {
  const locale = file.replace('.json', '');
  const messages = JSON.parse(fs.readFileSync(path.join(MESSAGES_DIR, file), 'utf-8'));
  const localeKeys = new Set(flattenKeys(messages));

  const missing = [...baseKeys].filter((k) => !localeKeys.has(k));
  if (missing.length > 0) {
    console.warn(`${locale}: ${missing.length} missing translations`);
    hasErrors = true;
  }
}

if (hasErrors) {
  process.exit(1);
}
```

---

## 9. Type Safety for Translation Keys

### 9.1 Generating Types from JSON

next-intl supports TypeScript-native type checking for translation keys. Add a global type declaration:

```typescript
// src/lib/i18n/types.ts (auto-generated)
// Run: npx tsx scripts/generate-i18n-types.ts

import messages from './messages/en-US.json';

type Messages = typeof messages;

declare global {
  // Use type-safe keys with next-intl
  interface IntlMessages extends Messages {}
}
```

### 9.2 Type Generation Script

```typescript
// scripts/generate-i18n-types.ts
import fs from 'fs';

const TYPE_FILE = 'src/lib/i18n/types.ts';

const content = `// Auto-generated -- do not edit manually
// Run: npx tsx scripts/generate-i18n-types.ts

import messages from './messages/en-US.json';

type Messages = typeof messages;

declare global {
  interface IntlMessages extends Messages {}
}

export {};
`;

fs.writeFileSync(TYPE_FILE, content);
console.log('Generated i18n types at', TYPE_FILE);
```

Add to package.json:

```json
{
  "scripts": {
    "i18n:types": "tsx scripts/generate-i18n-types.ts"
  }
}
```

### 9.3 Usage with Type Safety

With the types in place, `t('nonexistent.key')` will produce a TypeScript error at build time. This catches translation key typos during development rather than at runtime.

---

## 10. Migration Checklist

1. [ ] Install `next-intl` package
2. [ ] Create `src/lib/i18n/config.ts` with locale definitions
3. [ ] Create `src/lib/i18n/request.ts` with locale detection logic
4. [ ] Update `next.config.ts` with next-intl plugin
5. [ ] Run migration script to convert translation files (interpolation syntax)
6. [ ] Copy migrated files to `src/lib/i18n/messages/`
7. [ ] Add `NextIntlClientProvider` to root layout providers
8. [ ] Generate TypeScript types from `en-US.json`
9. [ ] Build `LanguageSwitcher` component
10. [ ] Replace all `| translate` pipe usages with `t()` calls during component migration
11. [ ] Replace all `translateService.instant()` calls with `t()` or `getTranslations()`
12. [ ] Add `i18n:validate` script to CI pipeline
13. [ ] Test all 13 locales for rendering correctness
14. [ ] Verify date/number formatting per locale
