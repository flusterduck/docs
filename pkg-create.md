# create-flusterduck

The project scaffolder. It creates a new app preconfigured with Flusterduck so signals flow from the first run.

## Install

```bash
npm create flusterduck@latest
```

You can also invoke it directly with your package manager of choice:

```bash
npm create flusterduck@latest my-app
pnpm create flusterduck my-app
yarn create flusterduck my-app
```

## Usage

```bash
# Scaffold into a new directory and answer the prompts.
npm create flusterduck@latest my-app

cd my-app
npm install
npm run dev
```

The scaffolder asks for your framework and publishable key, generates the project, and includes the matching Flusterduck wrapper already initialized.

## Links

Published on npm as `create-flusterduck`. Install pulls the latest published version.
