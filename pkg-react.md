# @flusterduck/react

The React wrapper. Wrap your app root with `FlusterduckProvider` and signal detection starts across the whole tree. Read the SDK from any component with `useFlusterduck`.

## Install

```bash
npm install @flusterduck/react flusterduck
```

## Usage

```tsx
// main.tsx
import { FlusterduckProvider } from '@flusterduck/react'
import App from './App'

export function Root() {
  return (
    <FlusterduckProvider apiKey="fd_pub_xxxxxxxxxxxx">
      <App />
    </FlusterduckProvider>
  )
}
```

```tsx
// Any other component. Track directly from the core package: the provider
// above already initialized it, and it's the same running instance.
import { track } from 'flusterduck'

export function CheckoutButton() {
  return (
    <button onClick={() => track('checkout_completed', { value: 49 })}>
      Pay
    </button>
  )
}
```

`useFlusterduck` itself takes the same config as `FlusterduckProvider` and returns `{ signal, track, identify, setConsent, optOut }`; call it again with the same key in a deeper component if you'd rather not import from `flusterduck` directly. See the [React guide](./react) for the full option list.

## Links

Published on npm as `@flusterduck/react`. Install pulls the latest published version.
