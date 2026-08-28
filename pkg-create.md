# create-flusterduck

The `npm create` entry point for the setup CLI. Run it inside an existing project and it detects your framework, then wires in the Flusterduck SDK for you. It's not a scaffolder: it doesn't generate a new directory, so don't pass a project name.

## Install

```bash
npm create flusterduck@latest
```

You can also invoke it directly with your package manager of choice:

```bash
pnpm create flusterduck
yarn create flusterduck
```

## Usage

Run it from your project's root, with no arguments:

```bash
npm create flusterduck@latest
```

It detects your framework and package manager, asks how to connect (sign in with your browser, or paste a publishable key), and installs the matching wrapper package already initialized. Under the hood this runs `flusterduck-cli init`, so `npx flusterduck-cli init` does the same thing directly. See the [quickstart](./quickstart) for what happens next.

## Links

Published on npm as `create-flusterduck`. Install pulls the latest published version.
