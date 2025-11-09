# Direct HTTP Fetch & HTML Scraping Implementation Guide

**Based on Gemini-CLI Codebase Analysis**

> Complete technical documentation for implementing direct HTTP fetching and HTML content retrieval for web scraping projects.

---

## Table of Contents

1. [Overview](#overview)
2. [Core Architecture](#core-architecture)
3. [Implementation Details](#implementation-details)
4. [HTML to Text Conversion](#html-to-text-conversion)
5. [Error Handling & Retry Logic](#error-handling--retry-logic)
6. [Proxy Configuration](#proxy-configuration)
7. [Timeout Management](#timeout-management)
8. [Private IP Detection](#private-ip-detection)
9. [Complete Code Examples](#complete-code-examples)
10. [Dependencies](#dependencies)
11. [Best Practices](#best-practices)
12. [Testing Strategies](#testing-strategies)

---

## Overview

The Gemini-CLI implements a **fallback-based HTTP fetch mechanism** for retrieving web content when the primary Gemini API method fails or when dealing with private/local network URLs. This guide extracts the complete implementation details for replication in scraping projects.

### When Direct HTTP Fetch is Used

1. **Private IP addresses** (localhost, 192.168.x.x, 10.x.x.x, etc.)
2. **Primary API fetch failures** (when Gemini's urlContext tool fails)
3. **GitHub blob URLs** (automatically converted to raw URLs)

### Key Features

- ✅ 10-second timeout protection
- ✅ AbortController-based cancellation
- ✅ HTML-to-text conversion with customizable selectors
- ✅ Content size limiting (100KB cap)
- ✅ Private IP range detection
- ✅ Proxy support via undici
- ✅ GitHub URL auto-conversion
- ✅ Content-type based processing
- ✅ Comprehensive error handling

---

## Core Architecture

### File Structure

```
packages/core/src/
├── utils/
│   └── fetch.ts           # Core fetch utilities
├── tools/
│   └── web-fetch.ts       # WebFetch tool implementation
└── telemetry/
    └── loggers.ts         # Fallback logging
```

### Data Flow

```
┌─────────────────────────────────────────┐
│  URL Request                             │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│  Check if Private IP                    │
│  (10.x, 127.x, 192.168.x, etc.)        │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│  Convert GitHub Blob → Raw URL          │
│  github.com/user/repo/blob/main/file   │
│  → raw.githubusercontent.com/...        │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│  fetchWithTimeout(url, 10000ms)         │
│  - AbortController for timeout          │
│  - Native fetch API                     │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│  Check Response Status                  │
│  - throw if !response.ok                │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│  Get Content-Type Header                │
└──────────┬──────────────────────────────┘
           │
           ├─── text/html or empty ────────┐
           │                               │
           │                               ▼
           │                    ┌──────────────────────┐
           │                    │  HTML-to-Text        │
           │                    │  Conversion          │
           │                    │  - wordwrap: false   │
           │                    │  - skip images       │
           │                    │  - ignore hrefs      │
           │                    └──────────┬───────────┘
           │                               │
           ├─── other types ───────────────┤
           │                               │
           ▼                               ▼
┌─────────────────────────────────────────┐
│  Cap at 100,000 characters              │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│  Return processed content               │
└─────────────────────────────────────────┘
```

---

## Implementation Details

### 1. Core Fetch Function

**Location:** `packages/core/src/utils/fetch.ts:40-58`

```typescript
export async function fetchWithTimeout(
  url: string,
  timeout: number,
): Promise<Response> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return response;
  } catch (error) {
    if (isNodeError(error) && error.code === 'ABORT_ERR') {
      throw new FetchError(`Request timed out after ${timeout}ms`, 'ETIMEDOUT');
    }
    throw new FetchError(getErrorMessage(error));
  } finally {
    clearTimeout(timeoutId);
  }
}
```

**Key Features:**
- Uses native Node.js `fetch()` API (requires Node >= 18)
- `AbortController` for timeout enforcement
- Custom `FetchError` class with error codes
- Automatic cleanup with `finally` block

### 2. Private IP Detection

**Location:** `packages/core/src/utils/fetch.ts:31-38`

```typescript
const PRIVATE_IP_RANGES = [
  /^10\./,                                 // 10.0.0.0/8
  /^127\./,                                // 127.0.0.0/8 (localhost)
  /^172\.(1[6-9]|2[0-9]|3[0-1])\./,       // 172.16.0.0/12
  /^192\.168\./,                           // 192.168.0.0/16
  /^::1$/,                                 // IPv6 localhost
  /^fc00:/,                                // IPv6 unique local
  /^fe80:/,                                // IPv6 link-local
];

export function isPrivateIp(url: string): boolean {
  try {
    const hostname = new URL(url).hostname;
    return PRIVATE_IP_RANGES.some((range) => range.test(hostname));
  } catch (_e) {
    return false;
  }
}
```

**Use Cases:**
- Prevent SSRF (Server-Side Request Forgery) attacks
- Enable local development testing
- Support localhost API scraping

### 3. GitHub URL Conversion

**Location:** `packages/core/src/tools/web-fetch.ts:127-131`

```typescript
// Convert GitHub blob URL to raw URL
if (url.includes('github.com') && url.includes('/blob/')) {
  url = url
    .replace('github.com', 'raw.githubusercontent.com')
    .replace('/blob/', '/');
}
```

**Example:**
```
Input:  https://github.com/user/repo/blob/main/README.md
Output: https://raw.githubusercontent.com/user/repo/main/README.md
```

### 4. Fallback Execution Flow

**Location:** `packages/core/src/tools/web-fetch.ts:121-196`

```typescript
private async executeFallback(signal: AbortSignal): Promise<ToolResult> {
  const { validUrls: urls } = parsePrompt(this.params.prompt);
  let url = urls[0];

  // 1. Convert GitHub URLs
  if (url.includes('github.com') && url.includes('/blob/')) {
    url = url
      .replace('github.com', 'raw.githubusercontent.com')
      .replace('/blob/', '/');
  }

  try {
    // 2. Fetch with timeout
    const response = await fetchWithTimeout(url, URL_FETCH_TIMEOUT_MS);

    if (!response.ok) {
      throw new Error(
        `Request failed with status code ${response.status} ${response.statusText}`
      );
    }

    // 3. Get content and content-type
    const rawContent = await response.text();
    const contentType = response.headers.get('content-type') || '';
    let textContent: string;

    // 4. Convert HTML to text if needed
    if (
      contentType.toLowerCase().includes('text/html') ||
      contentType === ''
    ) {
      textContent = convert(rawContent, {
        wordwrap: false,
        selectors: [
          { selector: 'a', options: { ignoreHref: true } },
          { selector: 'img', format: 'skip' },
        ],
      });
    } else {
      textContent = rawContent;
    }

    // 5. Cap content length
    textContent = textContent.substring(0, MAX_CONTENT_LENGTH);

    // 6. Return processed content
    return {
      llmContent: textContent,
      returnDisplay: `Content for ${url} processed using fallback fetch.`,
    };
  } catch (e) {
    const error = e as Error;
    const errorMessage = `Error during fallback fetch for ${url}: ${error.message}`;
    return {
      llmContent: `Error: ${errorMessage}`,
      returnDisplay: `Error: ${errorMessage}`,
      error: {
        message: errorMessage,
        type: ToolErrorType.WEB_FETCH_FALLBACK_FAILED,
      },
    };
  }
}
```

---

## HTML to Text Conversion

### Library: html-to-text v9.0.5

**Installation:**
```bash
npm install html-to-text@9.0.5 @types/html-to-text@9.0.4
```

### Configuration Used

**Location:** `packages/core/src/tools/web-fetch.ts:150-156`

```typescript
import { convert } from 'html-to-text';

const textContent = convert(rawContent, {
  wordwrap: false,
  selectors: [
    { selector: 'a', options: { ignoreHref: true } },
    { selector: 'img', format: 'skip' },
  ],
});
```

### Available Options

```typescript
interface HtmlToTextOptions {
  // Core Options
  wordwrap?: number | false | null;  // Line wrap limit
  preserveNewlines?: boolean;        // Keep newlines from HTML
  decodeEntities?: boolean;          // Decode HTML entities

  // Selectors
  selectors?: Array<{
    selector: string;
    format?: 'skip' | 'inline' | 'block' | string;
    options?: {
      ignoreHref?: boolean;
      itemPrefix?: string;
      uppercase?: boolean;
      // ... more options
    };
  }>;

  // Formatting
  formatters?: Record<string, Function>;

  // Base Elements
  baseElements?: {
    selectors?: string[];
    orderBy?: 'selectors' | 'occurrence';
    returnDomByDefault?: boolean;
  };

  // Limits
  limits?: {
    ellipsis?: string;
    maxBaseElements?: number;
    maxChildNodes?: number;
  };

  // Whitespace
  whitespaceCharacters?: string;

  // Long Words
  longWordSplit?: {
    wrapCharacters?: string[];
    forceWrapOnLimit?: boolean;
  };
}
```

### Custom Selector Examples

```typescript
// Example 1: Extract only main content
convert(html, {
  wordwrap: 80,
  selectors: [
    { selector: 'article', format: 'block' },
    { selector: 'aside', format: 'skip' },
    { selector: 'nav', format: 'skip' },
    { selector: 'footer', format: 'skip' },
  ]
});

// Example 2: Keep links but clean format
convert(html, {
  wordwrap: false,
  selectors: [
    { selector: 'a', format: 'inline' },
    { selector: 'script', format: 'skip' },
    { selector: 'style', format: 'skip' },
  ]
});

// Example 3: Extract lists with custom prefixes
convert(html, {
  selectors: [
    {
      selector: 'ul',
      options: {
        itemPrefix: '• ',
      }
    },
    {
      selector: 'ol',
      options: {
        itemPrefix: (index) => `${index + 1}. `,
      }
    }
  ]
});
```

### Content Type Detection

```typescript
const contentType = response.headers.get('content-type') || '';

if (
  contentType.toLowerCase().includes('text/html') ||
  contentType === ''  // Default to HTML if no content-type header
) {
  // Apply HTML-to-text conversion
  textContent = convert(rawContent, options);
} else {
  // Use raw text for JSON, plain text, etc.
  textContent = rawContent;
}
```

**Supported Content Types:**
- `text/html` → HTML-to-text conversion
- `text/plain` → Raw text
- `application/json` → Raw text
- `application/xml` → Raw text
- *(empty)* → Treated as HTML (default)

---

## Error Handling & Retry Logic

### Custom Error Classes

**Location:** `packages/core/src/utils/fetch.ts:21-28`

```typescript
export class FetchError extends Error {
  constructor(
    message: string,
    public code?: string,
  ) {
    super(message);
    this.name = 'FetchError';
  }
}
```

### Error Detection

```typescript
export function isNodeError(error: unknown): error is NodeJS.ErrnoException {
  return error instanceof Error && 'code' in error;
}

export function getErrorMessage(error: unknown): string {
  if (error instanceof Error) {
    return error.message;
  }
  try {
    return String(error);
  } catch {
    return 'Failed to get error details';
  }
}
```

### Retry Logic with Exponential Backoff

**Location:** `packages/core/src/utils/retry.ts:89-215`

```typescript
interface RetryOptions {
  maxAttempts: number;           // Default: 3
  initialDelayMs: number;        // Default: 5000
  maxDelayMs: number;            // Default: 30000
  shouldRetryOnError: (error: Error) => boolean;
  retryFetchErrors?: boolean;    // Retry on "fetch failed" errors
  signal?: AbortSignal;
}

export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options?: Partial<RetryOptions>,
): Promise<T> {
  const {
    maxAttempts,
    initialDelayMs,
    maxDelayMs,
    shouldRetryOnError,
    retryFetchErrors,
    signal,
  } = { ...DEFAULT_RETRY_OPTIONS, ...options };

  let attempt = 0;
  let currentDelay = initialDelayMs;

  while (attempt < maxAttempts) {
    if (signal?.aborted) {
      throw createAbortError();
    }

    attempt++;

    try {
      const result = await fn();
      return result;
    } catch (error) {
      // Check if should retry
      if (
        attempt >= maxAttempts ||
        !shouldRetryOnError(error as Error, retryFetchErrors)
      ) {
        throw error;
      }

      // Exponential backoff with jitter
      const jitter = currentDelay * 0.3 * (Math.random() * 2 - 1);
      const delayWithJitter = Math.max(0, currentDelay + jitter);

      await delay(delayWithJitter, signal);

      currentDelay = Math.min(maxDelayMs, currentDelay * 2);
    }
  }

  throw new Error('Retry attempts exhausted');
}
```

### Default Retry Conditions

```typescript
function defaultShouldRetry(
  error: Error | unknown,
  retryFetchErrors?: boolean,
): boolean {
  // Retry on "fetch failed" errors
  if (
    retryFetchErrors &&
    error instanceof Error &&
    error.message.includes('exception TypeError: fetch failed sending request')
  ) {
    return true;
  }

  const status = getErrorStatus(error);
  if (status !== undefined) {
    // Don't retry 400 Bad Request
    if (status === 400) return false;

    // Retry 429 (Too Many Requests) and 5xx (Server Errors)
    return status === 429 || (status >= 500 && status < 600);
  }

  return false;
}
```

### Usage Example

```typescript
import { retryWithBackoff } from './utils/retry';
import { fetchWithTimeout } from './utils/fetch';

async function fetchWithRetry(url: string): Promise<string> {
  return retryWithBackoff(
    async () => {
      const response = await fetchWithTimeout(url, 10000);

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return await response.text();
    },
    {
      maxAttempts: 3,
      initialDelayMs: 2000,
      maxDelayMs: 10000,
      retryFetchErrors: true,
    }
  );
}
```

---

## Proxy Configuration

### Setup via undici

**Location:** `packages/core/src/utils/fetch.ts:60-62`

```typescript
import { ProxyAgent, setGlobalDispatcher } from 'undici';

export function setGlobalProxy(proxy: string) {
  setGlobalDispatcher(new ProxyAgent(proxy));
}
```

### Usage in Application

**Location:** `packages/core/src/config/config.ts:560-564`

```typescript
const proxy = this.getProxy();
if (proxy) {
  try {
    setGlobalProxy(proxy);
  } catch (error) {
    console.error('Failed to set proxy:', error);
  }
}
```

### Proxy URL Format

```typescript
// HTTP Proxy
setGlobalProxy('http://proxy.example.com:8080');

// HTTPS Proxy
setGlobalProxy('https://proxy.example.com:8080');

// Authenticated Proxy
setGlobalProxy('http://username:password@proxy.example.com:8080');

// SOCKS5 Proxy
setGlobalProxy('socks5://proxy.example.com:1080');
```

### Environment Variable Support

```bash
# Set proxy via environment
export HTTP_PROXY=http://proxy.example.com:8080
export HTTPS_PROXY=http://proxy.example.com:8080
export NO_PROXY=localhost,127.0.0.1
```

### Advanced Proxy Configuration

```typescript
import { ProxyAgent } from 'undici';

const proxyAgent = new ProxyAgent({
  uri: 'http://proxy.example.com:8080',

  // Authentication
  token: 'Basic ' + Buffer.from('user:pass').toString('base64'),

  // Keep-alive
  keepAliveTimeout: 10000,
  keepAliveMaxTimeout: 600000,

  // Connection limits
  connections: 256,
  pipelining: 1,
});

// Use with fetch
const response = await fetch(url, { dispatcher: proxyAgent });
```

---

## Timeout Management

### Constants

**Location:** `packages/core/src/tools/web-fetch.ts:35-36`

```typescript
const URL_FETCH_TIMEOUT_MS = 10000;    // 10 seconds
const MAX_CONTENT_LENGTH = 100000;     // 100,000 characters
```

### Timeout Implementation

```typescript
export async function fetchWithTimeout(
  url: string,
  timeout: number = 10000,
): Promise<Response> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return response;
  } catch (error) {
    if (isNodeError(error) && error.code === 'ABORT_ERR') {
      throw new FetchError(
        `Request timed out after ${timeout}ms`,
        'ETIMEDOUT'
      );
    }
    throw new FetchError(getErrorMessage(error));
  } finally {
    clearTimeout(timeoutId);
  }
}
```

### Configurable Timeouts

```typescript
class FetchConfig {
  private timeout: number = 10000;

  setTimeout(ms: number) {
    this.timeout = ms;
  }

  async fetch(url: string): Promise<Response> {
    return fetchWithTimeout(url, this.timeout);
  }
}

// Usage
const config = new FetchConfig();
config.setTimeout(30000);  // 30 seconds for slow sites
const response = await config.fetch('https://example.com');
```

### Per-Request Timeout

```typescript
async function fetchWithCustomTimeout(
  url: string,
  customTimeout?: number
): Promise<Response> {
  const timeout = customTimeout ?? URL_FETCH_TIMEOUT_MS;
  return fetchWithTimeout(url, timeout);
}

// Fast timeout for health checks
await fetchWithCustomTimeout('https://api.example.com/health', 3000);

// Long timeout for large downloads
await fetchWithCustomTimeout('https://cdn.example.com/file.zip', 60000);
```

---

## Private IP Detection

### Complete Implementation

```typescript
const PRIVATE_IP_RANGES = [
  /^10\./,                                 // 10.0.0.0/8
  /^127\./,                                // 127.0.0.0/8 (localhost)
  /^172\.(1[6-9]|2[0-9]|3[0-1])\./,       // 172.16.0.0/12
  /^192\.168\./,                           // 192.168.0.0/16
  /^::1$/,                                 // IPv6 localhost
  /^fc00:/,                                // IPv6 unique local address
  /^fe80:/,                                // IPv6 link-local address
];

export function isPrivateIp(url: string): boolean {
  try {
    const hostname = new URL(url).hostname;
    return PRIVATE_IP_RANGES.some((range) => range.test(hostname));
  } catch (_e) {
    return false;
  }
}
```

### IP Range Explanations

| Pattern | Range | Description |
|---------|-------|-------------|
| `^10\.` | 10.0.0.0/8 | Private Class A network |
| `^127\.` | 127.0.0.0/8 | Loopback addresses (localhost) |
| `^172\.(1[6-9]\|2[0-9]\|3[0-1])\.` | 172.16.0.0/12 | Private Class B networks |
| `^192\.168\.` | 192.168.0.0/16 | Private Class C networks |
| `^::1$` | ::1 | IPv6 loopback |
| `^fc00:` | fc00::/7 | IPv6 unique local addresses |
| `^fe80:` | fe80::/10 | IPv6 link-local addresses |

### Usage Examples

```typescript
// Test various URLs
console.log(isPrivateIp('http://localhost:3000'));           // true
console.log(isPrivateIp('http://127.0.0.1:8080'));           // true
console.log(isPrivateIp('http://192.168.1.1'));              // true
console.log(isPrivateIp('http://10.0.0.1'));                 // true
console.log(isPrivateIp('http://172.16.0.1'));               // true
console.log(isPrivateIp('http://172.31.255.255'));           // true
console.log(isPrivateIp('http://[::1]:3000'));               // true
console.log(isPrivateIp('http://[fe80::1]'));                // true

console.log(isPrivateIp('https://example.com'));             // false
console.log(isPrivateIp('http://8.8.8.8'));                  // false
console.log(isPrivateIp('http://172.32.0.1'));               // false (outside range)
```

### Security Use Case

```typescript
async function safeFetch(url: string): Promise<Response> {
  // Prevent SSRF attacks
  if (isPrivateIp(url)) {
    throw new Error(
      'Access to private IP addresses is not allowed for security reasons'
    );
  }

  return fetchWithTimeout(url, 10000);
}

// Allow private IPs in development
async function devSafeFetch(url: string): Promise<Response> {
  const isDev = process.env.NODE_ENV === 'development';

  if (!isDev && isPrivateIp(url)) {
    throw new Error('Private IP access not allowed in production');
  }

  return fetchWithTimeout(url, 10000);
}
```

---

## Complete Code Examples

### Example 1: Basic Scraper

```typescript
import { fetchWithTimeout, isPrivateIp } from './utils/fetch';
import { convert } from 'html-to-text';

interface ScraperResult {
  url: string;
  content: string;
  contentType: string;
  length: number;
  error?: string;
}

async function scrapeUrl(url: string): Promise<ScraperResult> {
  try {
    // Validate URL
    if (isPrivateIp(url)) {
      throw new Error('Private IP addresses are not allowed');
    }

    // Fetch with timeout
    const response = await fetchWithTimeout(url, 10000);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    // Get content
    const rawContent = await response.text();
    const contentType = response.headers.get('content-type') || '';

    // Convert HTML to text if needed
    let content: string;
    if (contentType.toLowerCase().includes('text/html')) {
      content = convert(rawContent, {
        wordwrap: false,
        selectors: [
          { selector: 'script', format: 'skip' },
          { selector: 'style', format: 'skip' },
          { selector: 'nav', format: 'skip' },
          { selector: 'footer', format: 'skip' },
          { selector: 'a', options: { ignoreHref: true } },
          { selector: 'img', format: 'skip' },
        ],
      });
    } else {
      content = rawContent;
    }

    // Cap content length
    const maxLength = 100000;
    if (content.length > maxLength) {
      content = content.substring(0, maxLength);
    }

    return {
      url,
      content,
      contentType,
      length: content.length,
    };
  } catch (error) {
    return {
      url,
      content: '',
      contentType: '',
      length: 0,
      error: error instanceof Error ? error.message : String(error),
    };
  }
}

// Usage
const result = await scrapeUrl('https://example.com');
console.log(result);
```

### Example 2: Batch Scraper with Retry

```typescript
import { retryWithBackoff } from './utils/retry';
import { fetchWithTimeout } from './utils/fetch';
import { convert } from 'html-to-text';

interface BatchScraperOptions {
  maxRetries?: number;
  timeout?: number;
  maxContentLength?: number;
  concurrency?: number;
}

class BatchScraper {
  constructor(private options: BatchScraperOptions = {}) {
    this.options = {
      maxRetries: 3,
      timeout: 10000,
      maxContentLength: 100000,
      concurrency: 5,
      ...options,
    };
  }

  async scrapeUrl(url: string): Promise<string> {
    return retryWithBackoff(
      async () => {
        const response = await fetchWithTimeout(url, this.options.timeout!);

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`);
        }

        const rawContent = await response.text();
        const contentType = response.headers.get('content-type') || '';

        let content: string;
        if (contentType.includes('text/html')) {
          content = convert(rawContent, {
            wordwrap: false,
            selectors: [
              { selector: 'script', format: 'skip' },
              { selector: 'style', format: 'skip' },
            ],
          });
        } else {
          content = rawContent;
        }

        return content.substring(0, this.options.maxContentLength!);
      },
      {
        maxAttempts: this.options.maxRetries!,
        initialDelayMs: 2000,
        retryFetchErrors: true,
      }
    );
  }

  async scrapeMultiple(urls: string[]): Promise<Map<string, string>> {
    const results = new Map<string, string>();
    const chunks = this.chunk(urls, this.options.concurrency!);

    for (const chunk of chunks) {
      const promises = chunk.map(async (url) => {
        try {
          const content = await this.scrapeUrl(url);
          results.set(url, content);
        } catch (error) {
          console.error(`Failed to scrape ${url}:`, error);
          results.set(url, '');
        }
      });

      await Promise.all(promises);
    }

    return results;
  }

  private chunk<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
}

// Usage
const scraper = new BatchScraper({
  maxRetries: 3,
  timeout: 15000,
  concurrency: 10,
});

const urls = [
  'https://example.com/page1',
  'https://example.com/page2',
  'https://example.com/page3',
];

const results = await scraper.scrapeMultiple(urls);
for (const [url, content] of results) {
  console.log(`${url}: ${content.substring(0, 100)}...`);
}
```

### Example 3: Advanced Scraper with Proxy & Headers

```typescript
import { ProxyAgent, setGlobalDispatcher } from 'undici';
import { fetchWithTimeout } from './utils/fetch';
import { convert } from 'html-to-text';

interface AdvancedScraperConfig {
  proxy?: string;
  userAgent?: string;
  headers?: Record<string, string>;
  timeout?: number;
  followRedirects?: boolean;
}

class AdvancedScraper {
  private config: Required<AdvancedScraperConfig>;

  constructor(config: AdvancedScraperConfig = {}) {
    this.config = {
      proxy: config.proxy || '',
      userAgent: config.userAgent || 'Mozilla/5.0 (compatible; Bot/1.0)',
      headers: config.headers || {},
      timeout: config.timeout || 10000,
      followRedirects: config.followRedirects ?? true,
    };

    // Setup proxy if configured
    if (this.config.proxy) {
      setGlobalDispatcher(new ProxyAgent(this.config.proxy));
    }
  }

  async scrape(url: string): Promise<{
    content: string;
    metadata: {
      statusCode: number;
      contentType: string;
      contentLength: number;
      finalUrl: string;
    };
  }> {
    const controller = new AbortController();
    const timeoutId = setTimeout(
      () => controller.abort(),
      this.config.timeout
    );

    try {
      const response = await fetch(url, {
        signal: controller.signal,
        headers: {
          'User-Agent': this.config.userAgent,
          ...this.config.headers,
        },
        redirect: this.config.followRedirects ? 'follow' : 'manual',
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const rawContent = await response.text();
      const contentType = response.headers.get('content-type') || '';

      let content: string;
      if (contentType.includes('text/html')) {
        content = convert(rawContent, {
          wordwrap: 80,
          preserveNewlines: true,
          selectors: [
            { selector: 'article', format: 'block' },
            { selector: 'script', format: 'skip' },
            { selector: 'style', format: 'skip' },
            { selector: 'nav', format: 'skip' },
            { selector: 'aside', format: 'skip' },
            { selector: 'footer', format: 'skip' },
            {
              selector: 'a',
              options: {
                ignoreHref: false  // Keep links for scraping
              }
            },
          ],
          baseElements: {
            selectors: ['article', 'main', 'body'],
            returnDomByDefault: true,
          },
        });
      } else {
        content = rawContent;
      }

      return {
        content,
        metadata: {
          statusCode: response.status,
          contentType,
          contentLength: content.length,
          finalUrl: response.url,
        },
      };
    } finally {
      clearTimeout(timeoutId);
    }
  }

  async scrapeWithMetadata(url: string) {
    const result = await this.scrape(url);

    // Extract additional metadata
    const links = this.extractLinks(result.content);
    const images = this.extractImages(result.content);

    return {
      ...result,
      links,
      images,
    };
  }

  private extractLinks(content: string): string[] {
    const linkRegex = /\[([^\]]+)\]\(([^\)]+)\)/g;
    const links: string[] = [];
    let match;

    while ((match = linkRegex.exec(content)) !== null) {
      links.push(match[2]);
    }

    return links;
  }

  private extractImages(content: string): string[] {
    const imgRegex = /!\[([^\]]*)\]\(([^\)]+)\)/g;
    const images: string[] = [];
    let match;

    while ((match = imgRegex.exec(content)) !== null) {
      images.push(match[2]);
    }

    return images;
  }
}

// Usage
const scraper = new AdvancedScraper({
  proxy: 'http://proxy.example.com:8080',
  userAgent: 'MyBot/1.0 (+https://example.com/bot)',
  headers: {
    'Accept': 'text/html,application/xhtml+xml',
    'Accept-Language': 'en-US,en;q=0.9',
  },
  timeout: 30000,
});

const result = await scraper.scrapeWithMetadata('https://example.com');
console.log('Content:', result.content);
console.log('Metadata:', result.metadata);
console.log('Links found:', result.links.length);
```

### Example 4: GitHub File Fetcher

```typescript
import { fetchWithTimeout } from './utils/fetch';

class GitHubFileFetcher {
  async fetchFile(githubUrl: string): Promise<string> {
    // Convert GitHub blob URL to raw URL
    let url = githubUrl;

    if (url.includes('github.com') && url.includes('/blob/')) {
      url = url
        .replace('github.com', 'raw.githubusercontent.com')
        .replace('/blob/', '/');

      console.log(`Converted URL: ${url}`);
    }

    const response = await fetchWithTimeout(url, 10000);

    if (!response.ok) {
      throw new Error(`Failed to fetch: HTTP ${response.status}`);
    }

    return await response.text();
  }

  async fetchMultipleFiles(urls: string[]): Promise<Map<string, string>> {
    const results = new Map<string, string>();

    await Promise.all(
      urls.map(async (url) => {
        try {
          const content = await this.fetchFile(url);
          results.set(url, content);
        } catch (error) {
          console.error(`Failed to fetch ${url}:`, error);
          results.set(url, '');
        }
      })
    );

    return results;
  }
}

// Usage
const fetcher = new GitHubFileFetcher();

const readmeUrl = 'https://github.com/user/repo/blob/main/README.md';
const readme = await fetcher.fetchFile(readmeUrl);
console.log(readme);
```

---

## Dependencies

### Required Dependencies

```json
{
  "dependencies": {
    "undici": "^7.10.0",
    "html-to-text": "^9.0.5"
  },
  "devDependencies": {
    "@types/html-to-text": "^9.0.4"
  }
}
```

### Installation

```bash
npm install undici@7.10.0 html-to-text@9.0.5
npm install -D @types/html-to-text@9.0.4
```

### Optional Dependencies

```json
{
  "dependencies": {
    "https-proxy-agent": "^7.0.6"  // For HTTPS proxy support
  }
}
```

### Node.js Version Requirement

```json
{
  "engines": {
    "node": ">=20"
  }
}
```

**Note:** Native `fetch()` API requires Node.js 18+, but undici ProxyAgent works best with Node.js 20+.

---

## Best Practices

### 1. Always Use Timeouts

```typescript
// ❌ Bad: No timeout
const response = await fetch(url);

// ✅ Good: With timeout
const response = await fetchWithTimeout(url, 10000);
```

### 2. Handle Errors Gracefully

```typescript
// ❌ Bad: Unhandled errors
const content = await scrapeUrl(url);

// ✅ Good: Proper error handling
try {
  const content = await scrapeUrl(url);
  processContent(content);
} catch (error) {
  if (error instanceof FetchError && error.code === 'ETIMEDOUT') {
    console.error('Request timed out');
  } else {
    console.error('Fetch failed:', error);
  }
  // Fallback or retry logic
}
```

### 3. Respect Rate Limits

```typescript
class RateLimitedScraper {
  private lastRequest = 0;
  private minDelay = 1000; // 1 second between requests

  async scrape(url: string): Promise<string> {
    const now = Date.now();
    const timeSinceLastRequest = now - this.lastRequest;

    if (timeSinceLastRequest < this.minDelay) {
      await new Promise(resolve =>
        setTimeout(resolve, this.minDelay - timeSinceLastRequest)
      );
    }

    this.lastRequest = Date.now();
    return this.doScrape(url);
  }

  private async doScrape(url: string): Promise<string> {
    const response = await fetchWithTimeout(url, 10000);
    return response.text();
  }
}
```

### 4. Use Content Length Limits

```typescript
const MAX_CONTENT_LENGTH = 100000; // 100KB

async function scrapeWithLimit(url: string): Promise<string> {
  const response = await fetchWithTimeout(url, 10000);
  let content = await response.text();

  if (content.length > MAX_CONTENT_LENGTH) {
    console.warn(`Content truncated from ${content.length} to ${MAX_CONTENT_LENGTH}`);
    content = content.substring(0, MAX_CONTENT_LENGTH);
  }

  return content;
}
```

### 5. Implement Retry with Exponential Backoff

```typescript
async function scrapeWithRetry(url: string, maxAttempts = 3): Promise<string> {
  return retryWithBackoff(
    () => fetchAndParse(url),
    {
      maxAttempts,
      initialDelayMs: 1000,
      maxDelayMs: 10000,
      retryFetchErrors: true,
    }
  );
}
```

### 6. Use Appropriate User Agents

```typescript
const userAgents = {
  bot: 'MyBot/1.0 (+https://example.com/bot)',
  browser: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
  mobile: 'Mozilla/5.0 (iPhone; CPU iPhone OS 14_0 like Mac OS X)',
};

async function fetchWithUserAgent(url: string, type: keyof typeof userAgents) {
  return fetch(url, {
    headers: {
      'User-Agent': userAgents[type],
    },
  });
}
```

### 7. Validate URLs Before Fetching

```typescript
function isValidUrl(urlString: string): boolean {
  try {
    const url = new URL(urlString);
    return ['http:', 'https:'].includes(url.protocol);
  } catch {
    return false;
  }
}

async function safeScrape(urlString: string): Promise<string> {
  if (!isValidUrl(urlString)) {
    throw new Error('Invalid URL');
  }

  if (isPrivateIp(urlString)) {
    throw new Error('Private IP not allowed');
  }

  return scrapeUrl(urlString);
}
```

### 8. Clean HTML Properly

```typescript
const cleanHtmlOptions = {
  wordwrap: false,
  preserveNewlines: false,
  selectors: [
    // Remove unwanted elements
    { selector: 'script', format: 'skip' },
    { selector: 'style', format: 'skip' },
    { selector: 'noscript', format: 'skip' },
    { selector: 'iframe', format: 'skip' },
    { selector: 'svg', format: 'skip' },

    // Remove navigation and auxiliary content
    { selector: 'nav', format: 'skip' },
    { selector: 'aside', format: 'skip' },
    { selector: 'footer', format: 'skip' },
    { selector: 'header', format: 'skip' },

    // Clean links and images
    { selector: 'a', options: { ignoreHref: true } },
    { selector: 'img', format: 'skip' },
  ],
  baseElements: {
    selectors: ['article', 'main', '[role="main"]', 'body'],
    returnDomByDefault: true,
  },
};
```

### 9. Log Telemetry for Debugging

```typescript
interface FetchTelemetry {
  url: string;
  startTime: number;
  endTime: number;
  duration: number;
  statusCode?: number;
  contentLength?: number;
  error?: string;
}

async function fetchWithTelemetry(url: string): Promise<Response> {
  const telemetry: FetchTelemetry = {
    url,
    startTime: Date.now(),
    endTime: 0,
    duration: 0,
  };

  try {
    const response = await fetchWithTimeout(url, 10000);

    telemetry.statusCode = response.status;
    telemetry.contentLength = parseInt(
      response.headers.get('content-length') || '0'
    );

    return response;
  } catch (error) {
    telemetry.error = error instanceof Error ? error.message : String(error);
    throw error;
  } finally {
    telemetry.endTime = Date.now();
    telemetry.duration = telemetry.endTime - telemetry.startTime;

    console.log('Fetch telemetry:', JSON.stringify(telemetry, null, 2));
  }
}
```

### 10. Handle Different Content Types

```typescript
async function fetchAndProcess(url: string): Promise<string> {
  const response = await fetchWithTimeout(url, 10000);
  const contentType = response.headers.get('content-type') || '';
  const rawContent = await response.text();

  // HTML
  if (contentType.includes('text/html')) {
    return convert(rawContent, htmlToTextOptions);
  }

  // JSON
  if (contentType.includes('application/json')) {
    const json = JSON.parse(rawContent);
    return JSON.stringify(json, null, 2);
  }

  // XML
  if (contentType.includes('application/xml') || contentType.includes('text/xml')) {
    // Parse and format XML
    return rawContent;
  }

  // Plain text
  return rawContent;
}
```

---

## Testing Strategies

### 1. Unit Tests for Utilities

```typescript
import { describe, it, expect, vi } from 'vitest';
import { fetchWithTimeout, isPrivateIp } from './utils/fetch';

describe('fetchWithTimeout', () => {
  it('should fetch successfully within timeout', async () => {
    const response = await fetchWithTimeout('https://example.com', 10000);
    expect(response.ok).toBe(true);
  });

  it('should timeout after specified duration', async () => {
    const slowUrl = 'https://httpbin.org/delay/20';

    await expect(
      fetchWithTimeout(slowUrl, 1000)
    ).rejects.toThrow('Request timed out after 1000ms');
  });

  it('should handle network errors', async () => {
    await expect(
      fetchWithTimeout('https://invalid-url-12345.com', 5000)
    ).rejects.toThrow();
  });
});

describe('isPrivateIp', () => {
  it('should detect localhost', () => {
    expect(isPrivateIp('http://localhost:3000')).toBe(true);
    expect(isPrivateIp('http://127.0.0.1')).toBe(true);
  });

  it('should detect private IP ranges', () => {
    expect(isPrivateIp('http://192.168.1.1')).toBe(true);
    expect(isPrivateIp('http://10.0.0.1')).toBe(true);
    expect(isPrivateIp('http://172.16.0.1')).toBe(true);
  });

  it('should not detect public IPs as private', () => {
    expect(isPrivateIp('https://example.com')).toBe(false);
    expect(isPrivateIp('http://8.8.8.8')).toBe(false);
  });

  it('should handle IPv6', () => {
    expect(isPrivateIp('http://[::1]:3000')).toBe(true);
    expect(isPrivateIp('http://[fe80::1]')).toBe(true);
  });
});
```

### 2. Integration Tests

```typescript
import { describe, it, expect } from 'vitest';
import { scrapeUrl } from './scraper';

describe('Scraper Integration Tests', () => {
  it('should scrape real website', async () => {
    const result = await scrapeUrl('https://example.com');

    expect(result.error).toBeUndefined();
    expect(result.content).toBeTruthy();
    expect(result.content.length).toBeGreaterThan(0);
  });

  it('should handle 404 errors', async () => {
    const result = await scrapeUrl('https://example.com/nonexistent-page');

    expect(result.error).toBeTruthy();
    expect(result.error).toContain('404');
  });

  it('should convert HTML to text', async () => {
    const result = await scrapeUrl('https://example.com');

    expect(result.content).not.toContain('<html>');
    expect(result.content).not.toContain('<body>');
    expect(result.content).not.toContain('<script>');
  });
});
```

### 3. Mock Testing

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { fetchWithTimeout } from './utils/fetch';

// Mock fetch globally
global.fetch = vi.fn();

describe('Scraper with Mocks', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should handle successful response', async () => {
    const mockResponse = {
      ok: true,
      status: 200,
      headers: new Headers({ 'content-type': 'text/html' }),
      text: async () => '<html><body>Hello</body></html>',
    };

    vi.mocked(fetch).mockResolvedValue(mockResponse as Response);

    const response = await fetchWithTimeout('https://example.com', 10000);
    const text = await response.text();

    expect(text).toBe('<html><body>Hello</body></html>');
  });

  it('should handle fetch errors', async () => {
    vi.mocked(fetch).mockRejectedValue(new Error('Network error'));

    await expect(
      fetchWithTimeout('https://example.com', 10000)
    ).rejects.toThrow('Network error');
  });
});
```

### 4. Performance Testing

```typescript
import { describe, it, expect } from 'vitest';
import { fetchWithTimeout } from './utils/fetch';

describe('Performance Tests', () => {
  it('should complete within timeout', async () => {
    const startTime = Date.now();

    await fetchWithTimeout('https://example.com', 10000);

    const duration = Date.now() - startTime;
    expect(duration).toBeLessThan(10000);
  });

  it('should handle concurrent requests', async () => {
    const urls = Array(10).fill('https://example.com');

    const startTime = Date.now();

    await Promise.all(
      urls.map(url => fetchWithTimeout(url, 10000))
    );

    const duration = Date.now() - startTime;

    // Should complete faster than sequential (10 * timeout)
    expect(duration).toBeLessThan(100000);
  });
});
```

---

## Conclusion

This guide provides a complete implementation reference for direct HTTP fetching and HTML scraping based on the Gemini-CLI codebase. The implementation includes:

- ✅ Robust timeout handling with AbortController
- ✅ Comprehensive error handling and retry logic
- ✅ HTML-to-text conversion with customizable options
- ✅ Private IP detection for security
- ✅ Proxy support via undici
- ✅ GitHub URL conversion
- ✅ Content type detection and processing
- ✅ Configurable content length limits
- ✅ Production-ready testing strategies

### Key Takeaways

1. **Always use timeouts** - Prevents hanging on slow/unresponsive servers
2. **Implement retry logic** - Handles transient network failures
3. **Clean HTML properly** - Use html-to-text with appropriate selectors
4. **Validate inputs** - Check URLs and detect private IPs
5. **Handle errors gracefully** - Provide meaningful error messages
6. **Respect rate limits** - Add delays between requests
7. **Log telemetry** - Track performance and debug issues
8. **Test thoroughly** - Unit, integration, and performance tests

### References

- **Source Code**: `/home/user/gemini-cli/packages/core/src/`
- **Main Files**:
  - `utils/fetch.ts` - Core fetch utilities
  - `tools/web-fetch.ts` - WebFetch tool implementation
  - `utils/retry.ts` - Retry logic with exponential backoff

### License

The Gemini-CLI codebase is licensed under Apache License 2.0.

---

**Document Version:** 1.0
**Last Updated:** 2025-11-09
**Based on:** Gemini-CLI v0.15.0-nightly.20251107.b8eeb553
