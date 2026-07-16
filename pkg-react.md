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
// Any component.
import { useFlusterduck } from '@flusterduck/react'

export function CheckoutButton() {
  const { track } = useFlusterduck()
  return (
    <button onClick={() => track('checkout_completed', { value: 49 })}>
      Pay
    </button>
  )
}
```

## Links

Published on npm as `@flusterduck/react`. Install pulls the latest published version.
