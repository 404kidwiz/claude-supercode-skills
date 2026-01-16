# Codebase Exploration - Code Examples & Patterns

## Example 1: Finding Authentication in Express App

**Question**: "Where is user authentication handled?"

### Quick Search:
```bash
$ rg -l "auth" src/
src/middleware/auth.ts
src/routes/auth.routes.ts  
src/services/auth.service.ts
src/models/user.model.ts
```

### Medium Depth:
```bash
$ rg -A 5 "middleware.*auth"
src/routes/protected.routes.ts:
router.use(authMiddleware);  # Applied to routes

$ rg "class.*Auth|function.*auth"
src/middleware/auth.ts:
export const authMiddleware = async (req, res, next) => {
  // JWT validation
}
```

**Finding**: Authentication is in `src/middleware/auth.ts`, uses JWT, applied in route files

## Example 2: Understanding Data Flow in React App

**Question**: "How does data flow from API to UI?"

**Thoroughness**: Medium

### Search Process:
```bash
# Find API calls
$ rg "fetch|axios|api\." src/
src/services/api.service.ts
src/hooks/useData.ts

# Find state management
$ rg "useState|useQuery|createSlice"
src/hooks/useData.ts  # React Query
src/store/dataSlice.ts  # Redux

# Find components
$ fd -e tsx . src/components
```

**Finding**: API → Service → React Query Hook → Component

## Example 3: Locating Database Queries

**Question**: "Find all database queries"

**Thoroughness**: Very Thorough

### Search Process:
```bash
# Find ORM usage
$ rg "from.*Model.*import|Model\.|@Entity"

# Find raw SQL
$ rg "SELECT|INSERT|UPDATE|DELETE" --type ts

# Find query builders
$ rg "query|find|where\(|create\("

# Analyze patterns
$ rg -c "\.find\(|\.findOne\(|\.create\(" src/**/*.ts | sort -t: -k2 -rn
```

**Finding**: Queries concentrated in `src/repositories/`, using TypeORM

## Example 4: Mapping Component Hierarchy

**Question**: "What's the component structure?"

```bash
# Find all components
$ fd -e tsx src/components

# Find component imports
$ rg "import.*from.*components" src/

# Find component usage patterns
$ ast-grep --pattern "<$COMPONENT $$$>"
```

## Example 5: Finding Error Handling Patterns

```bash
# Find try-catch blocks
$ rg -A 3 "try {"

# Find error classes
$ rg "class.*Error|extends Error"

# Find error middleware
$ rg "error.*middleware|middleware.*error"

# Find error logging
$ rg "console.error|logger.error|log.error"
```

## Common Search Commands Reference

### File Discovery
```bash
# Find by name pattern
fd "auth" src/
fd -e ts -e tsx src/

# Find by content
rg -l "authentication" src/

# Find by structure
tree -L 3 src/
```

### Pattern Matching
```bash
# Function definitions
rg "function \w+\(" src/
rg "const \w+ = \(" src/
rg "async function" src/

# Class definitions
rg "class \w+" src/
ast-grep --pattern "class $NAME { $$$ }"

# Interface/Type definitions
rg "interface \w+|type \w+ =" src/
```

### Dependency Analysis
```bash
# What imports this file?
rg "from.*filename"

# What does this file import?
rg "^import" src/path/to/file.ts

# External dependencies
rg "from ['\"][^.]" src/
```

### Git History Analysis
```bash
# Recently changed files
git log --name-only --since="1 week ago" --pretty=format: | sort | uniq

# Most changed files (hotspots)
git log --name-only --pretty=format: | sort | uniq -c | sort -rn | head -20

# Who worked on this area
git log --format="%an" -- src/auth/ | sort | uniq -c | sort -rn
```

## Quick Reference: Search by Question Type

| Question | Command |
|----------|---------|
| "Where is X defined?" | `rg "function X\|class X\|const X"` |
| "Where is X used?" | `lsp_find_references` or `rg "X\("` |
| "What files contain X?" | `rg -l "X"` |
| "What's the structure?" | `tree -L 2` |
| "What changed recently?" | `git log --name-only --since="1 week"` |
| "Who owns this code?" | `git log --format="%an" -- path/` |
| "What imports X?" | `rg "import.*X\|from.*X"` |
| "What does X import?" | `rg "^import" path/to/X.ts` |
