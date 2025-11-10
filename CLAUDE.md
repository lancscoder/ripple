# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ripple is a TypeScript-first UI framework that uses a custom `.ripple` file extension for components. It combines fine-grained reactivity (similar to Solid/Svelte) with React-like patterns, featuring the unique `track()` API with `@` operator for reactive state access.

**Key Philosophy**: `.ripple` files are TypeScript with custom syntax for components, reactivity, and control flow. The codebase is transitioning from JavaScript to TypeScript, so some TypeScript errors are expected.

## Essential Commands

### Development
```bash
pnpm install              # Install dependencies (requires pnpm >=10.18.2, node >=20.0.0)
pnpm test                 # Run all tests with Vitest
pnpm format               # Format code with Prettier
pnpm format:check         # Check code formatting
```

### Versioning
```bash
pnpm bump:patch           # Bump patch version (0.0.x)
pnpm bump:minor           # Bump minor version (0.x.0)
pnpm bump:major           # Bump major version (x.0.0)
pnpm bump:editors:patch   # Bump editor plugins only
```

### Development Tools
```bash
pnpm regenerate-textmate                # Regenerate TextMate grammar
pnpm copy-tree-sitter-queries          # Copy Tree-sitter queries to editor plugins
```

### Testing
Tests are organized by project in `vitest.config.js`:
- **Client tests**: `packages/ripple/tests/client/**/*.test.ripple` (jsdom environment)
- **Server tests**: `packages/ripple/tests/server/**/*.test.ripple` (node environment)
- **Plugin tests**: Individual test suites for prettier, eslint, cli, etc.

Run specific test projects:
```bash
npx vitest run --project ripple-client
npx vitest run --project ripple-server
npx vitest run --project prettier-plugin
```

## Architecture

### Monorepo Structure

This is a pnpm workspace monorepo with packages organized by functionality:

```
packages/
├── ripple/              # Core framework (compiler + runtime)
├── vite-plugin/         # Vite integration (main build tool)
├── create-ripple/       # CLI scaffolding tool
├── prettier-plugin/     # Code formatter
├── eslint-parser/       # Parse .ripple for ESLint
├── eslint-plugin/       # Linting rules
├── typescript-plugin/   # TypeScript language service plugin
├── language-server/     # LSP server (Volar-based)
├── vscode-plugin/       # VSCode extension
├── tree-sitter/         # Tree-sitter grammar
├── nvim-plugin/         # Neovim plugin
├── zed-plugin/          # Zed editor plugin
├── sublime-text-plugin/ # Sublime Text plugin
├── compat-react/        # React compatibility layer
├── cli/                 # CLI tools
└── rollup-plugin/       # Rollup integration

templates/
└── basic/               # Template for new projects

playground/              # Interactive playground
website/                 # Documentation site (VitePress)
```

### Core Package: `packages/ripple/`

The heart of Ripple contains both the compiler and runtime:

```
packages/ripple/src/
├── compiler/
│   ├── phases/
│   │   ├── 1-parse/          # Parse .ripple → ESTree AST
│   │   ├── 2-analyze/        # Scope analysis, validation
│   │   └── 3-transform/      # Transform to JS/CSS
│   ├── index.js              # Main compiler API
│   ├── index.d.ts            # Types: compile(), parse(), compile_to_volar_mappings()
│   ├── scope.js              # Scope tracking
│   └── utils.js              # Compiler utilities
├── runtime/
│   ├── internal/
│   │   ├── client/           # Client-side reactivity engine
│   │   └── server/           # SSR rendering (coming soon)
│   ├── index-client.js       # Client runtime exports
│   ├── index-server.js       # Server runtime exports
│   ├── reactive-value.js     # Core track() implementation
│   ├── array.js              # TrackedArray (#[])
│   ├── object.js             # TrackedObject (#{})
│   └── [other reactivity primitives]
├── server/                   # SSR utilities
└── jsx-runtime.js           # JSX factory (internal use)
```

### Compiler Architecture

The Ripple compiler is a **3-phase pipeline**:

1. **Parse** (`1-parse/`): Converts `.ripple` source → ESTree AST with custom nodes
   - Based on Acorn with TypeScript support
   - Handles `component`, control flow (`if`, `for`, `switch`), and `@` operator

2. **Analyze** (`2-analyze/`): Validates and analyzes the AST
   - Scope analysis and binding resolution
   - Reactivity dependency tracking
   - Component validation

3. **Transform** (`3-transform/`): Generates output
   - **Client mode**: Reactive JS with runtime calls + scoped CSS
   - **Server mode**: SSR-compatible JS (future)
   - Outputs: `{ ast, js: { code, map }, css }`

### Reactivity System

Ripple uses **signals-based fine-grained reactivity**:

- `track(value)` or `track(fn)` creates reactive state/derived values
- `@` operator accesses reactive values (compile-time syntax)
- `#[]` and `#{}` create `TrackedArray` and `TrackedObject`
- Changes trigger precise DOM updates without re-rendering components

Example compilation:
```ripple
let count = track(0);
<div>{@count}</div>
```
→ Compiles to reactive subscriptions that update only the text node

### Build Integration

**Primary**: Vite plugin (`packages/vite-plugin/`) - transforms `.ripple` files via compiler
**Secondary**: Rollup plugin (`packages/rollup-plugin/`) - for non-Vite builds

Both plugins:
- Call `ripple/compiler` API
- Handle HMR for `.ripple` files
- Extract and inject scoped CSS

### Editor Integration

All editor plugins depend on the **Language Server** (`packages/language-server/`):
- Built on Volar framework (used by Vue, Svelte)
- Provides TypeScript integration for `.ripple` files
- Uses `compile_to_volar_mappings()` to map `.ripple` → `.ts` virtual files

The **Tree-sitter grammar** (`packages/tree-sitter/`) powers syntax highlighting and is copied to Zed and Neovim plugins via `pnpm copy-tree-sitter-queries`.

## Code Conventions

### File Extensions
- `.ripple` - Component files (TypeScript + custom syntax)
- `.js` - Internal implementation (transitioning to `.ts`)
- `.d.ts` - Type definitions

### Formatting
Use the Prettier plugin with project settings:
- Tabs (width 2) for `.ripple` and `.js`
- Spaces for `.md` and `.json`
- Single quotes for JS, double quotes for JSX
- 100 character line width

### Component Syntax
Components use the `component` keyword (not `export default`):
```ripple
component Button(props: { text: string }) {
  <button>{props.text}</button>
}

export component App() {
  <Button text="Click me" />
}
```

Note: Direct JSX (no return statement), reactive access with `@`, control flow is inline.

### Testing Patterns
Test files use `.test.ripple` extension and include components:
```ripple
import { track, flushSync } from 'ripple';

describe('feature', () => {
  it('test case', () => {
    component App() {
      let state = track(0);
      <button onClick={() => @state++}>{'Click'}</button>
    }

    render(App);
    expect(container.textContent).toBe('Click');
    container.querySelector('button').click();
    flushSync();
    expect(container.textContent).toBe('Click');
  });
});
```

Test utilities (`render`, `container`) are provided by setup files in `packages/ripple/tests/`.

## Important Notes

### Current State
- **Early alpha**: APIs and internal structure are evolving
- **SSR**: Not yet implemented (SPA-only currently)
- **TypeScript migration**: In progress, expect some type errors in internal code
- **Test coverage**: Being expanded

### When Working on Compiler
- Changes to compiler phases affect all downstream consumers
- Test both client and server compilation modes
- Verify source map generation works correctly
- Check Volar mappings for editor integration

### When Working on Runtime
- Client runtime must be highly optimized (size and speed are critical)
- All reactive primitives use the same signals system
- Be careful with reactivity tracking to avoid memory leaks

### When Working on Editor Plugins
- VSCode plugin is the primary/reference implementation
- Language server changes affect all editor integrations
- Tree-sitter changes require running `pnpm copy-tree-sitter-queries`
- Test both TypeScript and syntax highlighting

### Package Dependencies
- Most packages depend on `ripple` (core)
- Editor plugins depend on `language-server`
- Build plugins depend on `ripple/compiler`
- Use `workspace:*` protocol for internal dependencies
- External deps use pnpm catalog (see `pnpm-workspace.yaml`)
