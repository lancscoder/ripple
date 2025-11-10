# Ripple Zed Extension Implementation Plan

## Executive Summary

This document provides a detailed, step-by-step plan to complete the Ripple extension for the Zed editor. The extension already has a foundational structure in `packages/zed-plugin/` but requires several components to achieve feature parity with the VSCode extension.

**Current State**: Basic extension skeleton with language server integration exists
**Goal**: Full-featured language extension with syntax highlighting, IntelliSense, formatting, and all language features

---

## Phase 1: Foundation & Setup

### 1.1 Verify Development Environment

**Priority**: High | **Effort**: 15 minutes | **Blocker**: Yes

**Description**: Ensure all required tools are installed and working

**Tasks**:
- [ ] Verify Rust is installed via rustup (not Homebrew): `rustc --version`
- [ ] Verify wasm32-wasip1 target is installed: `rustup target add wasm32-wasip1`
- [ ] Verify Zed is installed and can load dev extensions
- [ ] Verify pnpm is available for running build scripts

**Success Criteria**:
- `rustc --version` shows Rust compiler version
- `rustup target list --installed` includes `wasm32-wasip1`
- Zed opens and can access "Install Dev Extension" command

**Testing**:
```bash
rustc --version
rustup target list --installed | grep wasm32-wasip1
zed --version
```

---

### 1.2 Understand Current Extension Structure

**Priority**: High | **Effort**: 30 minutes | **Blocker**: No

**Description**: Thoroughly review existing Zed extension files to understand what's implemented

**Files to Review**:
- `packages/zed-plugin/extension.toml` - Extension metadata
- `packages/zed-plugin/Cargo.toml` - Rust dependencies
- `packages/zed-plugin/src/lib.rs` - Language server integration logic
- `packages/zed-plugin/languages/ripple/config.toml` - Language configuration
- `packages/zed-plugin/package.json` - NPM scripts and language server version

**Current Implementation Analysis**:
- ✅ Language server binary detection (system path, worktree, npm installation)
- ✅ Automatic installation of `@ripple-ts/language-server` from npm
- ✅ Version management based on `package.json` config
- ✅ Basic language configuration (comments, brackets, tab size)
- ❌ Missing tree-sitter query files for syntax highlighting
- ❌ Missing advanced language configuration features

**Success Criteria**:
- Complete understanding of how language server is discovered and launched
- Clear picture of what's missing vs VSCode extension

---

## Phase 2: Tree-sitter Query Files (Critical Path)

### 2.1 Copy Tree-sitter Query Files

**Priority**: Critical | **Effort**: 10 minutes | **Blocker**: Yes

**Description**: Copy existing tree-sitter query files from `packages/tree-sitter/queries/` to `packages/zed-plugin/languages/ripple/`

**Why This is Critical**: Without these files, there will be no syntax highlighting in Zed

**Available Query Files** (from `packages/tree-sitter/queries/`):
- `highlights.scm` - Syntax highlighting rules
- `brackets.scm` - Bracket matching
- `folds.scm` - Code folding regions
- `injections.scm` - Language injection (CSS in `<style>`, etc.)
- `outline.scm` - Code structure for outline view
- `locals.scm` - Local variable scoping

**Tasks**:
```bash
# Run the existing copy script
cd packages/zed-plugin
pnpm run copy-scm
```

**OR manually**:
```bash
cp packages/tree-sitter/queries/*.scm packages/zed-plugin/languages/ripple/
```

**Success Criteria**:
- All `.scm` files exist in `packages/zed-plugin/languages/ripple/`
- Files are identical to source (byte-for-byte)

**Verification**:
```bash
ls -la packages/zed-plugin/languages/ripple/*.scm
diff packages/tree-sitter/queries/highlights.scm packages/zed-plugin/languages/ripple/highlights.scm
```

---

### 2.2 Validate Tree-sitter Query Syntax

**Priority**: High | **Effort**: 20 minutes | **Blocker**: No

**Description**: Ensure all copied `.scm` files are valid and use correct capture names

**Zed-Supported Capture Names** (from highlights.scm):
- `@keyword` - Keywords
- `@function` - Function names
- `@function.builtin` - Built-in functions (track, untrack)
- `@function.call` - Function calls
- `@function.method` - Method names
- `@variable` - Variables
- `@variable.parameter` - Function parameters
- `@string` - String literals
- `@number` - Number literals
- `@comment` - Comments
- `@tag` - JSX/HTML tags
- `@tag.delimiter` - Tag brackets (< >)
- `@operator` - Operators
- `@operator.special` - Special operators (@)
- `@punctuation.bracket` - Regular brackets
- `@punctuation.bracket.special` - Reactive brackets (#[ #{)

**Tasks**:
- [ ] Review `highlights.scm` for Ripple-specific syntax:
  - `component` keyword highlighting
  - `@` operator (unbox) highlighting
  - `#[]` and `#{}` reactive construct highlighting
  - JSX tag highlighting
- [ ] Check `brackets.scm` includes Ripple-specific brackets
- [ ] Verify `outline.scm` shows components, functions, and fragments
- [ ] Test `injections.scm` handles CSS in `<style>` blocks

**Common Issues to Check**:
- Incorrect capture names (Zed may ignore or error on unknown names)
- Missing patterns for Ripple-specific syntax
- Overly broad or narrow queries

**Success Criteria**:
- All query files use standard tree-sitter syntax
- Ripple-specific constructs are properly captured
- No syntax errors in query files

**Testing Method**:
Load extension in Zed and check for syntax highlighting (done in Phase 3)

---

### 2.3 Add Optional Query Files

**Priority**: Medium | **Effort**: 1-2 hours | **Blocker**: No

**Description**: Add additional query files for enhanced Zed features

**Optional Query Files** (not in tree-sitter package):
- `indents.scm` - Smart indentation rules
- `textobjects.scm` - Vim mode text object support
- `overrides.scm` - Scope-specific editor settings
- `redactions.scm` - Privacy protection for screen sharing
- `runnables.scm` - Detectable executable code (test cases, etc.)

**Priority Order**:
1. **indents.scm** (High value) - Makes editing more pleasant
2. **textobjects.scm** (Medium value) - For Vim users
3. **runnables.scm** (Low value) - Nice-to-have for test files
4. **overrides.scm** (Low value) - Edge case customization
5. **redactions.scm** (Low value) - Privacy feature

**Example `indents.scm`**:
```scheme
; Increase indent on opening braces/brackets
[
  (statement_block)
  (object)
  (array)
  (jsx_element)
  (component_declaration)
] @indent

; Decrease indent on closing
[
  "}"
  "]"
  ")"
] @outdent
```

**Success Criteria**:
- Smart indentation works when pressing Enter in code blocks
- (If added) Vim text objects work for Vim mode users

**Defer If**: Time-constrained or need to ship quickly

---

## Phase 3: Build & Test Extension

### 3.1 Build Extension for First Time

**Priority**: Critical | **Effort**: 15 minutes | **Blocker**: Yes

**Description**: Compile the Rust extension to WebAssembly

**Tasks**:
```bash
cd packages/zed-plugin
cargo build --target wasm32-wasip1 --release
```

**Expected Output**:
- Successful compilation message
- WASM file created at: `target/wasm32-wasip1/release/ripple_zed_plugin.wasm`

**Common Build Errors**:
1. **Missing wasm32-wasip1 target**: Run `rustup target add wasm32-wasip1`
2. **Cargo.toml syntax error**: Check TOML formatting
3. **API version mismatch**: Update `zed_extension_api` version in `Cargo.toml`

**Success Criteria**:
- Build completes without errors
- `.wasm` file exists in target directory

---

### 3.2 Install Extension in Zed (Dev Mode)

**Priority**: Critical | **Effort**: 10 minutes | **Blocker**: No

**Description**: Load the extension in Zed for testing

**Steps**:
1. Open Zed editor
2. Press `Cmd+Shift+P` (Mac) or `Ctrl+Shift+P` (Windows/Linux)
3. Type "zed: install dev extension"
4. Navigate to `packages/zed-plugin/` directory
5. Select folder

**Expected Result**:
- Extension appears in Extensions list with "(Dev)" label
- No error messages in Zed logs

**If Installation Fails**:
1. Check Zed logs: `Cmd/Ctrl+Shift+P` → "zed: open log"
2. Look for extension loading errors
3. Verify `extension.toml` syntax
4. Ensure `.wasm` file was built

**Success Criteria**:
- Extension shows in Zed's extension list
- No errors in Zed logs

---

### 3.3 Test Syntax Highlighting

**Priority**: Critical | **Effort**: 20 minutes | **Blocker**: No

**Description**: Verify that tree-sitter queries produce correct syntax highlighting

**Test Files**:
Create or use existing `.ripple` test files:
- `packages/ripple/tests/client/basic.test.ripple`
- `packages/ripple/tests/client/switch.test.ripple`
- `templates/basic/src/App.ripple` (if exists)

**Visual Checks**:
1. **Keywords**: `component`, `if`, `for`, `switch`, `import`, `export`
2. **Reactive operators**: `@` in `@count`, `@value`
3. **Reactive collections**: `#[...]` and `#{...}`
4. **Functions**: `track()`, `effect()`, component names
5. **JSX tags**: `<div>`, `<Button>`, etc.
6. **Strings**: Both single and double quoted
7. **Comments**: `//` and `/* */`
8. **Types**: TypeScript type annotations

**Known Good Visual Reference**:
Compare side-by-side with VSCode extension highlighting

**If Highlighting Doesn't Work**:
1. Check if tree-sitter grammar compiled: Look in Zed logs for grammar errors
2. Verify `highlights.scm` was copied correctly
3. Check `extension.toml` grammar reference points to correct repo/rev
4. Try reloading: "zed: reload extensions"

**Success Criteria**:
- All syntax elements have appropriate colors
- Ripple-specific syntax (`@`, `#[]`, `component`) is highlighted
- No obviously broken or missing highlighting

---

### 3.4 Test Language Server Integration

**Priority**: Critical | **Effort**: 30 minutes | **Blocker**: No

**Description**: Verify language server connects and provides IntelliSense

**Prerequisites**:
- Language server installed globally or locally:
  ```bash
  npm install -g @ripple-ts/language-server
  # OR in project
  pnpm install
  ```

**Test Cases**:

**TC1: Language Server Starts**
- Open a `.ripple` file in Zed
- Check status bar: Should show "Ripple Language Server" as running
- Check Zed logs for "Language server started" message

**TC2: Autocomplete**
- Type `track(` and wait for popup
- Should see parameter hints for `track()` function
- Try completing with Ripple imports

**TC3: Go to Definition**
- Place cursor on a component name like `<Button>`
- Press `F12` or right-click → "Go to Definition"
- Should jump to component definition

**TC4: Hover Documentation**
- Hover over `track` function
- Should see TypeScript signature and documentation

**TC5: Diagnostics**
- Introduce a TypeScript error (wrong type, missing import)
- Should see red squiggle and error message

**TC6: Find References**
- Right-click on a variable/component
- Select "Find All References"
- Should show all usages

**If Language Server Doesn't Start**:
1. Check Zed logs: `Cmd/Ctrl+Shift+P` → "zed: open log"
2. Look for "Language server binary not found" errors
3. Verify language server is installed: `which ripple-language-server`
4. Check `lib.rs` binary path detection logic
5. Manually specify path if needed

**Success Criteria**:
- Language server starts without errors
- All 6 test cases pass
- IntelliSense feels responsive (< 1 second delays)

---

### 3.5 Test Additional Language Features

**Priority**: High | **Effort**: 20 minutes | **Blocker**: No

**Description**: Test other language features beyond basic IntelliSense

**Features to Test**:

**Bracket Matching**
- Place cursor on opening `{` or `(`
- Closing bracket should highlight
- Try with JSX tags: `<div>` should match `</div>`

**Code Folding**
- Look for fold icons next to functions, components, blocks
- Click to fold/unfold
- Verify entire scope collapses correctly

**Outline View**
- Open outline panel (`Cmd/Ctrl+Shift+O`)
- Should show:
  - Components (e.g., `component App`)
  - Functions
  - Exports
- Click to navigate

**Comment Toggling**
- Select code
- Press `Cmd/Ctrl+/`
- Should toggle `//` comments
- Block comment: Select + `Cmd/Ctrl+Shift+/` should add `/* */`

**Auto-Closing Pairs**
- Type `{` → should auto-insert `}`
- Type `(` → should auto-insert `)`
- Type `"` → should auto-insert `"`
- Type `<` in JSX context → should auto-insert `>`

**Success Criteria**:
- All features work as expected
- No crashes or freezes
- Behavior matches VSCode where applicable

---

## Phase 4: Formatting Integration

### 4.1 Verify Prettier Integration

**Priority**: High | **Effort**: 30 minutes | **Blocker**: No

**Description**: Ensure formatting works with Ripple's Prettier plugin

**Background**:
- Ripple uses `@ripple-ts/prettier-plugin` for formatting
- VSCode extension configures Prettier automatically
- Zed may need manual Prettier configuration

**Test Setup**:
```bash
# Ensure Prettier and plugin are installed in project
pnpm install --save-dev prettier @ripple-ts/prettier-plugin
```

**Configuration Needed** (if not exists):
Create/verify `.prettierrc` in project root:
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

**Test Cases**:

**TC1: Manual Format**
- Open a `.ripple` file with messy formatting
- Press `Cmd/Ctrl+Shift+I` (format document)
- Code should reformat according to Ripple style

**TC2: Format on Save**
- Enable "Format on Save" in Zed settings
- Edit and save a `.ripple` file
- Should auto-format

**TC3: Format Selection**
- Select portion of code
- Right-click → "Format Selection"
- Only selected code should format

**If Formatting Doesn't Work**:
1. Check if Prettier is installed: `which prettier` or check `node_modules/.bin/prettier`
2. Verify Prettier plugin is installed
3. Check Zed settings for "Format on Save"
4. Look in Zed logs for formatter errors
5. May need to configure Zed to use project's Prettier

**Zed Prettier Configuration**:
Zed uses a `formatter` configuration in `config.toml`:
```toml
# May need to add to languages/ripple/config.toml
[formatter]
command = "prettier"
arguments = ["--write", "--plugin", "@ripple-ts/prettier-plugin"]
```

**Success Criteria**:
- Manual formatting works
- Format on save works (if enabled)
- Formatting respects Ripple syntax rules

**Defer If**: Formatting is not critical for initial release (can format manually with CLI)

---

### 4.2 Configure Format on Save (Optional)

**Priority**: Medium | **Effort**: 15 minutes | **Blocker**: No

**Description**: Enable automatic formatting on file save

**Implementation Options**:

**Option A: Global Zed Setting**
Users can enable in their Zed settings:
```json
{
  "format_on_save": "on",
  "formatter": "prettier"
}
```

**Option B: Per-Language Setting in config.toml**
Add to `languages/ripple/config.toml`:
```toml
format_on_save = true
```

**Option C: Documentation Only**
Document that users should enable format-on-save in their Zed settings

**Recommendation**: Option C (document for users) - keep extension simple

**Success Criteria**:
- Formatting behavior is documented in README
- Users can enable if desired
- No conflicts with other formatters

---

## Phase 5: Enhanced Configuration

### 5.1 Review and Enhance Language Configuration

**Priority**: Medium | **Effort**: 30 minutes | **Blocker**: No

**Description**: Compare Zed's `config.toml` with VSCode's `language-configuration.json` and add missing features

**VSCode Features** (from `language-configuration.json`):
- `autoClosingPairs` with `notIn` contexts
- `surroundingPairs`
- `folding` markers (region/endregion)
- `wordPattern` for word selection
- `indentationRules` (increase/decrease/unIndent patterns)
- `onEnterRules` for smart Enter key behavior

**Zed Current Features** (from `config.toml`):
- ✅ `line_comments` and `block_comment`
- ✅ Basic `brackets` with auto-close
- ✅ `tab_size` and `hard_tabs`
- ✅ `autoclose_before` characters
- ❌ Missing contextual auto-closing (like VSCode's `notIn`)
- ❌ Missing advanced indentation rules
- ❌ Missing smart Enter behavior

**Zed Limitations**:
Zed's TOML config is simpler than VSCode's JSON - some features may not be directly portable

**Enhancement Tasks**:
- [ ] Review `brackets` array - ensure all Ripple brackets are included
- [ ] Add `autoclose_before` characters appropriate for Ripple
- [ ] Consider `first_line_pattern` for shebang-like detection (probably not needed)
- [ ] Document any limitations vs VSCode

**Example Enhanced config.toml**:
```toml
name = "Ripple"
grammar = "ripple"
path_suffixes = ["ripple"]
line_comments = ["//"]
block_comment = ["/*", "*/"]
tab_size = 2
hard_tabs = true  # Ripple uses tabs per .prettierrc

autoclose_before = ";:.,=}])>\" \n\t"

brackets = [
  { start = "{", end = "}", close = true, newline = true },
  { start = "[", end = "]", close = true, newline = true },
  { start = "(", end = ")", close = true, newline = true },
  { start = "<", end = ">", close = true, newline = false },
  { start = "\"", end = "\"", close = true, newline = false },
  { start = "'", end = "'", close = true, newline = false },
  { start = "`", end = "`", close = true, newline = false },
]

# Note: Zed doesn't support VSCode's advanced indentation rules
# These are handled by tree-sitter indents.scm instead
```

**Success Criteria**:
- `config.toml` includes all relevant Ripple brackets
- Settings match Ripple's conventions (tabs, etc.)
- Configuration is well-documented

---

### 5.2 Add File Icon (Optional)

**Priority**: Low | **Effort**: 15 minutes | **Blocker**: No

**Description**: Add custom icon for `.ripple` files in Zed's file tree

**Current Status**: VSCode extension has icons at `packages/vscode-plugin/icons/logo.png`

**Zed Icon Support**:
Zed extensions can provide file icons, but this requires additional configuration and icon assets

**Implementation**:
1. Decide if worth the effort (purely visual enhancement)
2. If yes: Export icon in format Zed expects
3. Reference icon in `extension.toml` or `config.toml`

**Recommendation**: **Defer to Phase 6 (Polish)** - not critical for functionality

**Success Criteria** (if implemented):
- `.ripple` files show Ripple icon in file tree
- Icon looks good at different sizes

---

## Phase 6: Documentation & Testing

### 6.1 Update Extension README

**Priority**: High | **Effort**: 30 minutes | **Blocker**: No

**Description**: Ensure `packages/zed-plugin/README.md` is accurate and complete

**Sections to Include**:
1. **Overview**: What is Ripple, what does this extension do
2. **Features**: List all supported features
   - Syntax highlighting (via tree-sitter)
   - IntelliSense (autocomplete, hover, diagnostics)
   - Go to Definition / Find References
   - Code folding, outline view
   - Formatting (with Prettier)
   - Bracket matching
3. **Installation**: How to install from Zed's extension registry
4. **Development Installation**: How to install as dev extension
5. **Requirements**:
   - Zed editor
   - Node.js (for language server)
   - Ripple project with proper configuration
6. **Language Server**: How it's installed automatically
7. **Formatting**: How to set up Prettier
8. **Troubleshooting**:
   - Language server not starting
   - Syntax highlighting not working
   - Formatting not working
9. **Contributing**: Link to main CONTRIBUTING.md
10. **License**: MIT

**Reference**: Look at `packages/zed-plugin/README.md` and update based on actual features

**Success Criteria**:
- README is clear and comprehensive
- Installation instructions work
- Troubleshooting section covers common issues

---

### 6.2 Update DEVELOPMENT.md

**Priority**: Medium | **Effort**: 20 minutes | **Blocker**: No

**Description**: Ensure development guide is accurate and complete

**Current Status**: `packages/zed-plugin/DEVELOPMENT.md` exists with good content

**Sections to Review/Update**:
1. **Building**: Ensure build instructions are current
2. **Testing**: Add test cases from Phase 3
3. **File Structure**: Verify all files are listed
4. **Publishing**: Update with any new requirements
5. **Updating**: Add info about syncing tree-sitter queries

**New Content to Add**:
- Script to copy tree-sitter queries: `pnpm run copy-scm`
- How to update `extension.toml` rev when grammar changes
- How to test all language features systematically

**Success Criteria**:
- Developers can follow DEVELOPMENT.md and build extension
- All scripts and commands work
- File structure is accurate

---

### 6.3 Create Test Matrix

**Priority**: Medium | **Effort**: 1 hour | **Blocker**: No

**Description**: Create comprehensive test checklist for regression testing

**Test Matrix Format** (Markdown table or checklist):

**Feature** | **Test Case** | **Expected Result** | **Pass/Fail** | **Notes**
- Syntax Highlighting | Open .ripple file | All syntax colored | ☐ |
- Language Server | Open file | "Ripple LS" in status | ☐ |
- Autocomplete | Type `track(` | Parameter hints | ☐ |
- Go to Definition | F12 on component | Jumps to def | ☐ |
- Hover | Hover over function | Shows signature | ☐ |
- Diagnostics | Add type error | Red squiggle | ☐ |
- Find References | Right-click var | Shows all uses | ☐ |
- Bracket Match | Click `{` | Highlights `}` | ☐ |
- Code Folding | Click fold icon | Collapses block | ☐ |
- Outline | Open outline | Shows structure | ☐ |
- Comment Toggle | Cmd+/ | Toggles comment | ☐ |
- Auto-close | Type `{` | Inserts `}` | ☐ |
- Format Document | Cmd+Shift+I | Formats code | ☐ |
- Format on Save | Save file | Auto-formats | ☐ |

**Deliverable**: Create `packages/zed-plugin/TEST_MATRIX.md` with full checklist

**Success Criteria**:
- All features have test cases
- Tests are reproducible
- Pass/fail can be easily checked

---

### 6.4 Perform End-to-End Testing

**Priority**: High | **Effort**: 1-2 hours | **Blocker**: No

**Description**: Run through entire test matrix in a real project

**Test Environment**:
- Fresh Zed installation (or clean profile)
- Real Ripple project (use `templates/basic` or create new one)
- Extension installed as dev extension

**Test Process**:
1. Create new Ripple project: `npx create-ripple test-zed-extension`
2. Open in Zed
3. Install extension as dev extension
4. Run through test matrix (from 6.3)
5. Document any failures or issues
6. Fix issues and re-test

**Edge Cases to Test**:
- Large `.ripple` files (performance)
- Files with syntax errors (error recovery)
- Multiple `.ripple` files open (no conflicts)
- Switching between `.ripple` and `.ts` files
- Project without `node_modules` (language server install)

**Success Criteria**:
- 100% of test matrix passes
- No crashes or hangs
- Performance is acceptable
- Extension works in real-world project

---

## Phase 7: Optimization & Polish

### 7.1 Optimize Language Server Startup

**Priority**: Medium | **Effort**: 1 hour | **Blocker**: No

**Description**: Ensure language server starts quickly and efficiently

**Current Implementation** (from `lib.rs`):
- Checks system path first (fastest)
- Falls back to worktree `node_modules/.bin` (fast)
- Downloads from npm if needed (slow, first-time only)

**Optimization Opportunities**:
1. **Caching**: Current code already caches binary path
2. **Lazy Installation**: Could defer download until first `.ripple` file is opened
3. **Background Download**: Install language server in background on extension load

**Measurements to Take**:
- Time from opening `.ripple` file to language server ready
- Time for first-time installation
- Time for cached startup

**Target Metrics**:
- Cached startup: < 1 second
- First-time install: < 30 seconds
- No blocking UI during install

**If Optimization Needed**:
- Add progress indicators
- Improve error messages
- Cache more aggressively

**Success Criteria**:
- Language server feels fast
- First-time install is smooth
- No user confusion during install

---

### 7.2 Add Error Handling & User Feedback

**Priority**: Medium | **Effort**: 1 hour | **Blocker**: No

**Description**: Improve error messages and user communication

**Current Error Handling** (from `lib.rs`):
- Returns `Result<String, String>` for errors
- Errors propagate to Zed's logging

**Improvement Areas**:

**Better Error Messages**:
```rust
// Current:
Err("Failed to locate language server binary")

// Better:
Err("Ripple language server not found. Install with: npm install -g @ripple-ts/language-server")
```

**Installation Status**:
- Already uses `LanguageServerInstallationStatus::Downloading`
- Could add more granular status updates

**User-Facing Messages**:
- Show notification when installation fails
- Provide actionable next steps
- Link to documentation

**Tasks**:
- [ ] Review all error messages in `lib.rs`
- [ ] Make errors more descriptive and actionable
- [ ] Add comments explaining error handling
- [ ] Test error scenarios (no npm, network failure, etc.)

**Success Criteria**:
- Error messages are clear and helpful
- Users know what to do when something fails
- No cryptic Rust errors exposed to users

---

### 7.3 Performance Testing

**Priority**: Low | **Effort**: 1 hour | **Blocker**: No

**Description**: Test extension performance with large files and projects

**Test Cases**:

**TC1: Large File**
- Create `.ripple` file with 1000+ lines
- Open in Zed
- Measure syntax highlighting time
- Test scrolling performance
- Test IntelliSense response time

**TC2: Many Files**
- Project with 50+ `.ripple` files
- Open multiple files simultaneously
- Test switching between files
- Measure memory usage

**TC3: Complex Syntax**
- File with deeply nested components
- Lots of reactive expressions (`@`, `#[]`, `#{}`)
- Measure parsing and highlighting time

**Performance Targets**:
- Syntax highlight any file < 500ms
- IntelliSense response < 1 second
- No memory leaks over extended use
- Smooth scrolling even in large files

**If Performance Issues Found**:
- May be tree-sitter grammar issue (upstream)
- May be language server issue (upstream)
- May be Zed issue (report to Zed)
- May be query file issue (optimize queries)

**Success Criteria**:
- No noticeable lag or slowdown
- Performance matches or exceeds VSCode extension
- No crashes with large files

---

### 7.4 Add Extension Icon

**Priority**: Low | **Effort**: 30 minutes | **Blocker**: No

**Description**: Add Ripple logo as extension icon

**Current Status**: VSCode has icon at `packages/vscode-plugin/icons/logo.png`

**Zed Icon Requirements**:
- Extensions can have icons shown in extension list
- Format and size requirements from Zed docs

**Tasks**:
1. Export icon in correct format/size for Zed
2. Add to extension directory
3. Reference in `extension.toml`

**Example `extension.toml`**:
```toml
id = "ripple"
name = "Ripple"
icon = "icon.png"  # or icon.svg
```

**Success Criteria**:
- Icon displays in Zed's extension list
- Icon is clear and recognizable
- Matches Ripple branding

---

## Phase 8: Publishing Preparation

### 8.1 Verify License Compliance

**Priority**: High | **Effort**: 15 minutes | **Blocker**: Yes

**Description**: Ensure extension meets Zed's licensing requirements

**Zed Requirements** (as of Oct 1, 2025):
Extensions must use one of:
- Apache 2.0
- BSD 3-Clause
- GNU GPLv3
- MIT

**Current License**: Check `packages/zed-plugin/LICENSE`

**Ripple Project License**: MIT (from root LICENSE)

**Tasks**:
- [ ] Verify `packages/zed-plugin/LICENSE` exists
- [ ] Verify it's MIT (matching main project)
- [ ] Verify LICENSE is at repository root (Zed requirement)
- [ ] Check `extension.toml` doesn't specify conflicting license

**If License Missing or Wrong**:
```bash
cp LICENSE packages/zed-plugin/LICENSE  # If missing
```

**Success Criteria**:
- LICENSE file exists at root
- LICENSE is one of the approved licenses
- License is consistent across project

---

### 8.2 Update Extension Metadata

**Priority**: High | **Effort**: 20 minutes | **Blocker**: No

**Description**: Ensure all metadata in `extension.toml` is accurate

**Current Metadata** (from `extension.toml`):
```toml
id = "ripple"
name = "Ripple"
description = "Ripple language support with LSP and Tree-sitter syntax highlighting"
version = "0.0.46"
schema_version = 1
authors = ["Dominic Gannaway <trueadm@users.noreply.github.com>"]
repository = "https://github.com/trueadm/ripple"  # Check if correct URL
```

**Fields to Verify/Update**:
- [x] `id` - Must be unique (keep "ripple")
- [ ] `name` - User-facing name (keep "Ripple")
- [ ] `description` - Accurate and complete?
- [ ] `version` - Should match package.json? (currently 0.0.46)
- [ ] `authors` - Is email correct?
- [ ] `repository` - Correct GitHub URL? (should it be Ripple-TS/ripple?)

**Update Repository URL If Needed**:
```toml
repository = "https://github.com/Ripple-TS/ripple"
```

**Grammar Reference**:
```toml
[grammars.ripple]
repository = "https://github.com/Ripple-TS/ripple"
rev = "1b35809e5d18aab72f6c0aff82e228981662d315"  # Update to latest
path = "packages/tree-sitter"
```

**Update `rev` to Latest Commit**:
```bash
git rev-parse HEAD  # Get current commit hash
# Update extension.toml with new hash
```

**Success Criteria**:
- All metadata is accurate
- URLs point to correct repository
- Version is consistent with package.json
- Grammar rev is current

---

### 8.3 Create Publishing Checklist

**Priority**: High | **Effort**: 30 minutes | **Blocker**: No

**Description**: Document exact steps to publish extension

**Publishing Process** (from Zed docs):
1. Fork `zed-industries/extensions` repo (personal account, not org)
2. Add Ripple repo as git submodule (HTTPS URL required)
3. Update `extensions.toml` with Ripple entry
4. Create pull request
5. Wait for review and merge
6. Extension auto-publishes after merge

**Detailed Checklist** (add to DEVELOPMENT.md or separate doc):

**Pre-Publishing**:
- [ ] All tests pass (Phase 6.4)
- [ ] Documentation is complete and accurate
- [ ] Version number is updated
- [ ] Grammar rev is current
- [ ] LICENSE file exists
- [ ] Repository URL is correct
- [ ] Extension builds without errors

**Publishing Steps**:
```bash
# 1. Fork zed-industries/extensions on GitHub (use personal account)

# 2. Clone your fork
git clone https://github.com/YOUR-USERNAME/extensions.git
cd extensions

# 3. Add Ripple as submodule (MUST use HTTPS, not SSH)
git submodule add https://github.com/Ripple-TS/ripple.git extensions/ripple

# 4. Configure submodule to use specific directory
git config -f .gitmodules submodule.extensions/ripple.path extensions/ripple
git config -f .gitmodules submodule.extensions/ripple.branch main

# 5. Edit extensions.toml
# Add:
[ripple]
submodule = "extensions/ripple"
version = "0.0.46"  # Match extension.toml version

# 6. Commit changes
git add .
git commit -m "Add Ripple language extension"

# 7. Push to your fork
git push origin main

# 8. Create PR to zed-industries/extensions on GitHub

# 9. Wait for review and merge

# 10. Once merged, extension appears in Zed's extension registry
```

**Post-Publishing**:
- [ ] Test installation from Zed's extension registry
- [ ] Monitor for issues reported by users
- [ ] Announce on Ripple Discord/community

**Success Criteria**:
- Checklist is complete and accurate
- Publishing steps are clear
- No steps missing

---

### 8.4 Test Pre-Publishing Build

**Priority**: High | **Effort**: 30 minutes | **Blocker**: Yes

**Description**: Final build and test before publishing

**Steps**:
1. Clean build from scratch:
   ```bash
   cd packages/zed-plugin
   cargo clean
   cargo build --target wasm32-wasip1 --release
   ```

2. Install in fresh Zed profile:
   - Create new Zed settings profile
   - Install extension as dev
   - Test all features

3. Verify no local dependencies:
   - Extension should work without project's `node_modules`
   - Language server should auto-install

4. Test on Different OS** (if possible):
   - macOS
   - Linux
   - Windows

**Success Criteria**:
- Clean build succeeds
- Extension works in fresh environment
- No errors in Zed logs
- All tests pass (Phase 6.4)

---

## Phase 9: Publishing & Maintenance

### 9.1 Publish to Zed Extensions Registry

**Priority**: High | **Effort**: 1 hour | **Blocker**: No

**Description**: Actually publish the extension

**Prerequisites** (all must be complete):
- ✅ All tests pass
- ✅ Documentation complete
- ✅ License compliant
- ✅ Metadata accurate
- ✅ Clean build successful

**Steps**: Follow checklist from Phase 8.3

**Common Issues**:
- **PR rejected**: Missing license, wrong URL format, etc.
- **Submodule issues**: Must use HTTPS, not SSH
- **Build fails in CI**: May need to fix Cargo.toml or Rust code

**Timeline**:
- PR review: Usually 1-3 days
- After merge: Available immediately

**Success Criteria**:
- PR is merged
- Extension appears in Zed registry
- Users can search and install

---

### 9.2 Announce Release

**Priority**: Medium | **Effort**: 30 minutes | **Blocker**: No

**Description**: Let Ripple community know about Zed extension

**Channels**:
1. **Ripple Discord**: Announce in #announcements or relevant channel
2. **GitHub Discussions**: Create announcement post
3. **Twitter/X**: Tweet from Ripple account (if applicable)
4. **README**: Update main Ripple README to mention Zed support

**Announcement Content**:
- Extension is now available in Zed
- How to install (search "Ripple" in Zed extensions)
- Features included
- Link to documentation
- Invite feedback

**Success Criteria**:
- Community is aware of Zed extension
- Install instructions are public
- Feedback mechanism exists

---

### 9.3 Monitor for Issues

**Priority**: High | **Effort**: Ongoing | **Blocker**: No

**Description**: Watch for bug reports and user issues

**Monitoring Channels**:
- Zed extension issues (via GitHub)
- Ripple GitHub issues (users may report there)
- Ripple Discord (users asking for help)
- Zed Discord (issues may be reported there)

**Response Plan**:
- Acknowledge issues within 24-48 hours
- Triage: Is it extension bug, language server bug, tree-sitter bug, Zed bug?
- Fix extension bugs quickly
- Report upstream bugs to appropriate repos
- Document workarounds

**Success Criteria**:
- Issues are tracked and responded to
- Critical bugs fixed quickly
- Users feel supported

---

### 9.4 Plan for Updates

**Priority**: Medium | **Effort**: 1 hour | **Blocker**: No

**Description**: Establish process for keeping extension up-to-date

**Update Triggers**:
1. **Tree-sitter grammar changes**: Update `extension.toml` rev
2. **Language server updates**: Update `package.json` version
3. **Zed API changes**: Update `zed_extension_api` dependency
4. **Bug fixes**: Fix and release new version

**Update Process**:
1. Make changes in `packages/zed-plugin/`
2. Update version in `extension.toml`
3. Test thoroughly
4. Commit and push to Ripple repo
5. Update submodule in zed-extensions repo:
   ```bash
   cd extensions/ripple
   git pull
   cd ../..
   git add extensions/ripple
   git commit -m "Update Ripple extension to vX.X.X"
   git push
   ```
6. Create PR to zed-extensions
7. After merge, new version available

**Automation Opportunities**:
- GitHub Actions to remind to update grammar rev
- Script to bump version and update metadata
- Automated testing before release

**Success Criteria**:
- Update process is documented
- Updates can be released smoothly
- No breaking changes for users

---

## Appendix A: Feature Comparison Matrix

| Feature | VSCode Extension | Zed Extension | Implementation | Priority |
|---------|------------------|---------------|----------------|----------|
| Syntax Highlighting | ✅ TextMate | ✅ Tree-sitter | Phase 2 | Critical |
| Language Server | ✅ Volar LSP | ✅ Same LSP | Phase 1 | Critical |
| Autocomplete | ✅ | ✅ | Via LSP | Critical |
| Go to Definition | ✅ | ✅ | Via LSP | Critical |
| Find References | ✅ | ✅ | Via LSP | Critical |
| Hover Info | ✅ | ✅ | Via LSP | Critical |
| Diagnostics | ✅ | ✅ | Via LSP | Critical |
| Code Folding | ✅ | ✅ | folds.scm | High |
| Outline View | ✅ | ✅ | outline.scm | High |
| Bracket Matching | ✅ | ✅ | brackets.scm | High |
| Comment Toggle | ✅ | ✅ | config.toml | High |
| Auto-close Pairs | ✅ | ✅ | config.toml | High |
| Formatting (Prettier) | ✅ | ⚠️ | Phase 4 | High |
| Format on Save | ✅ | ⚠️ | Phase 4 | Medium |
| Smart Indentation | ✅ | ⚠️ | indents.scm | Medium |
| Language Injection | ✅ | ✅ | injections.scm | Medium |
| TypeScript Integration | ✅ Custom | ✅ Via LSP | Built-in | N/A |
| Custom Commands | ✅ | ❌ | Not applicable | Low |
| File Icon | ✅ | ⚠️ | Phase 7 | Low |
| Vim Text Objects | ❌ | ⚠️ | textobjects.scm | Low |

**Legend**:
- ✅ Implemented
- ⚠️ Needs work/testing
- ❌ Not implemented/needed
- N/A Not applicable

---

## Appendix B: File Locations Quick Reference

```
packages/zed-plugin/
├── extension.toml              # Extension metadata
├── Cargo.toml                  # Rust dependencies
├── Cargo.lock                  # Dependency lock file
├── package.json                # NPM scripts & LS version
├── LICENSE                     # MIT License
├── README.md                   # User documentation
├── DEVELOPMENT.md              # Developer guide
├── .gitignore                  # Git ignore rules
│
├── src/
│   └── lib.rs                  # Main extension code (language server integration)
│
└── languages/
    └── ripple/
        ├── config.toml         # Language configuration
        ├── highlights.scm      # ⚠️ TO ADD: Syntax highlighting
        ├── brackets.scm        # ⚠️ TO ADD: Bracket matching
        ├── folds.scm           # ⚠️ TO ADD: Code folding
        ├── outline.scm         # ⚠️ TO ADD: Outline view
        ├── injections.scm      # ⚠️ TO ADD: Language injection
        ├── locals.scm          # ⚠️ TO ADD: Local scoping
        ├── indents.scm         # ⚠️ OPTIONAL: Smart indentation
        └── textobjects.scm     # ⚠️ OPTIONAL: Vim text objects
```

---

## Appendix C: Useful Commands

**Development**:
```bash
# Build extension
cd packages/zed-plugin
cargo build --target wasm32-wasip1 --release

# Copy tree-sitter queries
pnpm run copy-scm

# Install language server globally
npm install -g @ripple-ts/language-server

# Check language server version
ripple-language-server --version
```

**Testing**:
```bash
# Open Zed with logging
zed --foreground

# Reload Zed extensions
# In Zed: Cmd+Shift+P → "zed: reload extensions"

# View Zed logs
# In Zed: Cmd+Shift+P → "zed: open log"
```

**Git**:
```bash
# Get current commit hash (for extension.toml rev)
git rev-parse HEAD

# Update grammar reference
# Edit extension.toml, update rev field
```

---

## Appendix D: Troubleshooting Guide

### Issue: Syntax highlighting not working

**Symptoms**: `.ripple` files have no colors or wrong colors

**Possible Causes**:
1. Tree-sitter query files not copied
2. Grammar not compiled
3. Grammar rev incorrect in extension.toml
4. Extension not loaded

**Solutions**:
1. Run `pnpm run copy-scm` to copy query files
2. Check Zed logs for grammar compilation errors
3. Update `rev` in `extension.toml` to current commit: `git rev-parse HEAD`
4. Reload extensions: `Cmd+Shift+P` → "zed: reload extensions"

---

### Issue: Language server not starting

**Symptoms**: No IntelliSense, status bar doesn't show "Ripple Language Server"

**Possible Causes**:
1. Language server not installed
2. Binary not in PATH
3. npm not available
4. Binary path detection failure

**Solutions**:
1. Install manually: `npm install -g @ripple-ts/language-server`
2. Check PATH: `which ripple-language-server`
3. Install Node.js and npm
4. Check Zed logs for error messages
5. Review `lib.rs` binary detection logic

---

### Issue: Formatting not working

**Symptoms**: Format command doesn't work or formats incorrectly

**Possible Causes**:
1. Prettier not installed
2. Prettier plugin not installed
3. No `.prettierrc` configuration
4. Zed formatter not configured

**Solutions**:
1. Install: `pnpm install prettier @ripple-ts/prettier-plugin`
2. Create `.prettierrc` with Ripple plugin
3. Test CLI: `prettier --write file.ripple`
4. Check Zed settings for formatter configuration

---

### Issue: Extension won't build

**Symptoms**: `cargo build` fails

**Possible Causes**:
1. Rust not installed
2. wasm32-wasip1 target missing
3. Cargo.toml syntax error
4. API version mismatch

**Solutions**:
1. Install Rust via rustup
2. Add target: `rustup target add wasm32-wasip1`
3. Check TOML syntax
4. Update `zed_extension_api` version in Cargo.toml

---

### Issue: Extension not loading in Zed

**Symptoms**: Extension doesn't appear after installing

**Possible Causes**:
1. Wrong directory selected
2. extension.toml not found
3. Build artifacts missing
4. Zed bug

**Solutions**:
1. Select directory containing `extension.toml`
2. Verify file exists
3. Build first: `cargo build --target wasm32-wasip1 --release`
4. Check Zed logs, restart Zed

---

## Appendix E: Success Criteria Checklist

Use this final checklist before considering the extension complete:

### Core Functionality
- [ ] Syntax highlighting works for all Ripple syntax
- [ ] Language server starts automatically
- [ ] Autocomplete works (functions, variables, components)
- [ ] Go to Definition works
- [ ] Find References works
- [ ] Hover shows type information
- [ ] Diagnostics show TypeScript errors
- [ ] Code folding works
- [ ] Outline view shows structure
- [ ] Bracket matching works
- [ ] Comment toggling works (Cmd+/)
- [ ] Auto-closing pairs work

### Optional Features
- [ ] Formatting works (manual and on-save)
- [ ] Smart indentation works
- [ ] Language injection works (CSS in `<style>`)

### Quality
- [ ] No crashes or freezes
- [ ] Performance is good (< 1s for IntelliSense)
- [ ] Extension works in real-world projects
- [ ] Error messages are clear and helpful

### Documentation
- [ ] README is complete and accurate
- [ ] DEVELOPMENT.md is up-to-date
- [ ] Installation instructions work
- [ ] Troubleshooting guide exists

### Publishing
- [ ] License is compliant (MIT)
- [ ] Metadata is accurate
- [ ] Version is consistent
- [ ] Grammar rev is current
- [ ] Clean build succeeds

### Post-Publishing
- [ ] Extension appears in Zed registry
- [ ] Users can search and install
- [ ] Community is notified
- [ ] Monitoring is in place

---

## Conclusion

This plan provides a comprehensive, step-by-step approach to completing the Ripple Zed extension. Each phase builds on the previous one, with clear priorities, effort estimates, and success criteria.

**Estimated Total Effort**: 15-25 hours (depending on optional features)

**Critical Path**: Phases 1-3 and 6.4 (foundation, tree-sitter queries, testing)

**Can Be Deferred**: Formatting (Phase 4), optimization (Phase 7), some polish items

**Quick Start Path** (minimal viable extension):
1. Phase 1.1-1.2: Setup (45 min)
2. Phase 2.1-2.2: Tree-sitter queries (30 min)
3. Phase 3.1-3.4: Build and test (1.5 hours)
4. Phase 6.1-6.2: Documentation (50 min)
5. Phase 8.1-8.4: Pre-publishing (1.5 hours)

**Total Quick Start**: ~5 hours to a publishable extension

Good luck with the implementation! 🚀
