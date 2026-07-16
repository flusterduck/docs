# @flusterduck/next

The Next.js wrapper. Add `FlusterduckScript` to your root layout and signal detection starts app-wide. Use `useFlusterduck` in client components for tracking, consent, and opt-out.

## Install

```bash
npm install @flusterduck/next flusterduck
```

## Usage

```tsx
// app/layout.tsx
import { FlusterduckScript } from '@flusterduck/next'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <FlusterduckScript apiKey="fd_pub_xxxxxxxxxxxx" />
        {children}
      </body>
    </html>
  )
}
```

```tsx
// A client component.
'use client'
import { useFlusterduck } from '@flusterduck/next'

export function ConsentToggle() {
  const { setConsent, optOut } = useFlusterduck()
  return (
    <>
      <button onClick={() => setConsent(true)}>Allow</button>
      <button onClick={() => optOut()}>Opt out</button>
    </>
  )
}
```

## Links

Published on npm as `@flusterduck/next`. Install pulls the latest published version.
