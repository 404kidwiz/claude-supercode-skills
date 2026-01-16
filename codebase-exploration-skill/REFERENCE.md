# Codebase Exploration - Technical Reference

## Search Patterns by Use Case

### Finding Authentication

**Quick** (`grep/rg`):
```bash
# Find auth-related files
rg -l "auth|login|session" --type ts

# Find middleware
rg "middleware.*auth|auth.*middleware"

# Find route guards
rg "guard|protect|require.*auth"
```

**Medium** (deeper context):
```bash
# Find auth implementations
rg -A 5 "class.*Auth|interface.*Auth"

# Find password handling
rg "password|hash|bcrypt|argon"

# Find session management
rg "session|token|jwt"
```

**Very Thorough** (comprehensive):
```bash
# Trace full auth flow
rg -C 10 "login.*request"
ast-grep --pattern "async function login($$$) { $$$ }"
lsp_find_references on auth symbols
```

### Finding Data Models

**Quick**:
```bash
# Locate model files
fd model
fd -e ts -e js . src/models

# Find class definitions
rg "class.*Model|interface.*Model"
```

**Medium**:
```bash
# Find schemas/types
rg "type.*Schema|interface.*Type|class.*Entity"

# Find database decorators
rg "@Entity|@Table|@Model"

# Find validation rules
rg "validator|validate|schema|zod|yup"
```

**Very Thorough**:
```bash
# Map all data structures
ast-grep --pattern "interface $NAME { $$$ }"
ast-grep --pattern "class $NAME { $$$ }"
lsp_document_symbols on model files
```

### Finding API Endpoints

**Quick**:
```bash
# Find route definitions
rg "router\.|route|@Get|@Post|@Put|@Delete"

# Find controller files
fd controller
rg "class.*Controller"
```

**Medium**:
```bash
# Find endpoint handlers
rg -A 10 "@Get\(|@Post\(|router.get|router.post"

# Find middleware chain
rg "use\(|middleware|guard"

# Find request/response types
rg "Request|Response|Dto|Input|Output"
```

**Very Thorough**:
```bash
# Trace full request flow
rg -C 15 "router\.(get|post|put|delete)"
ast-grep --pattern "router.$METHOD('$PATH', $$$)"
# Find all references to route handlers
```

### Finding State Management

**Quick**:
```bash
# Find store/state files
fd store state
rg "createStore|createSlice|useState|Vuex|Redux"
```

**Medium**:
```bash
# Find actions and mutations
rg "action|mutation|dispatch|commit"

# Find selectors
rg "selector|useSelector|mapState"

# Find context providers
rg "Context|Provider|createContext"
```

**Very Thorough**:
```bash
# Map state architecture
ast-grep --pattern "const $NAME = createSlice({ $$$ })"
rg -C 20 "combineReducers|configureStore"
# Trace state flow from components to store
```

## Exploration Techniques

### File Structure Analysis

```bash
# Quick overview
tree -L 2 src/

# Count by type
find src/ -type f | sed 's/.*\.//' | sort | uniq -c | sort -nr

# Find largest files (potential hotspots)
find . -type f -exec du -h {} + | sort -rh | head -20

# Find recently changed files
git log --name-only --pretty=format: --since="1 month ago" | sort | uniq -c | sort -nr
```

### Pattern Discovery

```bash
# Find common patterns
rg "^import.*from" | cut -d"'" -f2 | sort | uniq -c | sort -nr

# Find naming conventions
find src/ -name "*.ts" | sed 's/.*\///' | sed 's/\..*//' | sort

# Find architectural patterns
rg "class.*Service|class.*Controller|class.*Repository" | wc -l
```

### Dependency Mapping

```bash
# Find imports
rg "^import.*from ['\"]\.\.?/"  # Internal imports
rg "^import.*from ['\"][^.]"    # External imports

# Find common dependencies
rg "^import.*from" | cut -d"'" -f2 | grep -v "^\." | sort | uniq -c | sort -nr
```

### Code Hotspots

```bash
# Find files with most changes
git log --format=format: --name-only | grep -v '^$' | sort | uniq -c | sort -rn | head -20

# Find files with most lines
find . -name "*.ts" -exec wc -l {} + | sort -rn | head -20

# Find complex files (many functions)
rg -c "function|const.*= \(" src/**/*.ts | sort -t: -k2 -rn
```

## Exploration Reports

After exploration, summarize findings:

### Format

```markdown
# Codebase Exploration: [Topic]

## Question
[What we were looking for]

## Findings

### Primary Implementation
**File**: src/path/to/main.ts
**Purpose**: [What it does]
**Key Functions**:
- `functionA()`: [Description]
- `functionB()`: [Description]

### Supporting Files
**File**: src/path/to/helper.ts
**Purpose**: [Supporting role]

### Related Patterns
- Pattern A found in: file1, file2, file3
- Pattern B found in: file4, file5

## Architecture Notes
[How things connect, patterns observed]

## Next Steps
[Suggested further exploration or actions]
```
