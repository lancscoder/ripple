# Ripple Extension for Zed

This extension provides comprehensive Ripple language support for the [Zed editor](https://zed.dev).

## Features

- **Syntax Highlighting**: Full tree-sitter-based syntax highlighting for Ripple's custom syntax
  - Component and fragment declarations
  - Reactive operators (`@`, `#[]`, `#{}`)
  - JSX/TSX elements
  - TypeScript integration
- **IntelliSense**: Powered by the Ripple Language Server
  - Autocomplete for components, functions, and variables
  - Parameter hints
  - Hover documentation
  - Go to Definition
  - Find All References
  - Rename symbol
- **Diagnostics**: Real-time TypeScript error checking
- **Code Navigation**:
  - Outline view showing components and functions
  - Code folding for blocks and components
  - Bracket matching
- **Editing Features**:
  - Auto-closing pairs for brackets, quotes, and JSX tags
  - Comment toggling (`Cmd/Ctrl + /`)
  - Language injection for CSS in `<style>` blocks

## Installation

### From Zed Extensions

Once published to the Zed extensions registry:

1. Open Zed
2. Press `Cmd/Ctrl + Shift + X` to open extensions
3. Search for "Ripple"
4. Click "Install"

### Development Installation

1. Clone this repository
2. Install Rust with the wasm32-wasip1 target:
   ```bash
   rustup target add wasm32-wasip1
   ```
3. Open Zed
4. Press `Cmd/Ctrl + Shift + P`
5. Run "zed: install dev extension"
6. Select the `packages/zed-plugin` directory

## Language Server Setup

The extension automatically detects and uses the Ripple Language Server in the following priority order:

1. **Monorepo language server** (if working in the Ripple repository): Uses `packages/language-server/bin/language-server.js`
2. **System-wide installation**: Checks for globally installed `ripple-language-server`
3. **Project-local installation**: Checks `node_modules/.bin/ripple-language-server`
4. **Automatic download**: Downloads from npm the first time it runs

For development in the Ripple monorepo, the extension will automatically use the local language server, making it easy to test changes without publishing.

If you prefer to manage the language server yourself, install it via npm:

```bash
npm install -g @ripple-ts/language-server
```

The version downloaded automatically is pinned via the `config` entry for `@ripple-ts/language-server` in this package's `package.json`.

## Requirements

- Zed editor (latest version recommended)
- Node.js and npm (for language server)
- A Ripple project with proper configuration

## Formatting

The extension works with Ripple's Prettier plugin for code formatting. To enable formatting:

1. Install Prettier and the Ripple plugin in your project:
   ```bash
   pnpm install --save-dev prettier @ripple-ts/prettier-plugin
   ```

2. Create a `.prettierrc` file in your project root:
   ```json
   {
     "plugins": ["@ripple-ts/prettier-plugin"],
     "overrides": [
       {
         "files": "*.ripple",
         "options": {
           "parser": "ripple"
         }
       }
     ]
   }
   ```

3. Enable "Format on Save" in Zed settings if desired.

## Troubleshooting

### Syntax highlighting not working

- Ensure the extension is installed and enabled
- Try reloading extensions: `Cmd/Ctrl + Shift + P` → "zed: reload extensions"
- Check Zed logs: `Cmd/Ctrl + Shift + P` → "zed: open log"

### Language server not starting

- Verify Node.js is installed: `node --version`
- Try installing the language server manually: `npm install -g @ripple-ts/language-server`
- Check if the binary is accessible: `which ripple-language-server`
- Review Zed logs for error messages

### Formatting not working

- Ensure Prettier and `@ripple-ts/prettier-plugin` are installed in your project
- Verify `.prettierrc` configuration exists
- Test formatting from command line: `prettier --write file.ripple`

## Contributing

Contributions are welcome! Please see the main [Ripple repository](https://github.com/Ripple-TS/ripple) for contribution guidelines.

## License

MIT
