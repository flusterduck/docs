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
// A client component. FlusterduckScript loads the SDK onto window, so
// consent controls read that same instance instead of importing the SDK
// again (a separate useFlusterduck() init would create a second SDK
// instance, since the script build and the npm build don't share state).
'use client'

declare global {
  interface Window {
    flusterduck?: { setConsent: (consented: boolean) => void; optOut: () => void }
  }
}

export function ConsentToggle() {
  return (
    <>
      <button onClick={() => window.flusterduck?.setConsent(true)}>Allow</button>
      <button onClick={() => window.flusterduck?.optOut()}>Opt out</button>
    </>
  )
}
```

If you're initializing with `useFlusterduck` instead of `FlusterduckScript`, call the hook again with the same key in the deeper component: it returns `{ signal, track, identify, setConsent, optOut }` and re-init is a no-op. See the [Next.js guide](./next) for both paths.

## Links

Published on npm as `@flusterduck/next`. Install pulls the latest published version.
