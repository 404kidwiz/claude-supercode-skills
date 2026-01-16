# GraphQL Architect - Examples & Patterns

## Anti-Patterns

### Anti-Pattern: Not Handling N+1 Queries

**What it looks like:**

```typescript
// Resolver without DataLoader
const resolvers = {
  Post: {
    author: async (post, _, { db }) => {
      // Executed once per post (N+1 problem!)
      return db.User.findByPk(post.userId);
    }
  }
};

// Query for 100 posts triggers 101 DB queries:
// 1 query for posts + 100 queries for authors
```

**Why it fails:**
- **Performance**: Linear scaling (100 posts = 101 queries, 1000 posts = 1001 queries)
- **Database load**: Overwhelming DB with small queries
- **Latency**: Sequential queries instead of batching

**Correct approach:**

```typescript
// Use DataLoader for batching
import DataLoader from 'dataloader';

// Create per-request loader
const createLoaders = (db) => ({
  userLoader: new DataLoader(async (userIds) => {
    const users = await db.User.findAll({
      where: { id: userIds }
    });
    const userMap = new Map(users.map(u => [u.id, u]));
    return userIds.map(id => userMap.get(id));
  })
});

const resolvers = {
  Post: {
    author: (post, _, { loaders }) => {
      // DataLoader batches all author requests
      return loaders.userLoader.load(post.userId);
    }
  }
};

// Same query now triggers 2 queries:
// 1 query for posts + 1 batched query for all authors
```

---

### Anti-Pattern: No Query Depth/Complexity Limits

**What it looks like:**

```graphql
# Malicious query - no limits
query DeepNested {
  users {
    posts {
      comments {
        author {
          posts {
            comments {
              author {
                # ... infinitely deep
              }
            }
          }
        }
      }
    }
  }
}
```

**Why it fails:**
- DoS vulnerability
- Memory exhaustion
- Database overload

**Correct approach:**

```typescript
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(10),  // Max 10 levels deep
    createComplexityLimitRule(1000)  // Max complexity score
  ]
});
```

---

### Anti-Pattern: Exposing Internal IDs

**What it looks like:**

```graphql
type User {
  id: Int!  # Database auto-increment ID exposed
  email: String!
}
```

**Why it fails:**
- Reveals database structure
- Enables enumeration attacks
- Breaks when migrating databases

**Correct approach:**

```graphql
type User {
  id: ID!  # Opaque identifier (base64 encoded or UUID)
  email: String!
}
```

```typescript
// Encode/decode IDs
const toGlobalId = (type: string, id: number) => 
  Buffer.from(`${type}:${id}`).toString('base64');

const fromGlobalId = (globalId: string) => {
  const decoded = Buffer.from(globalId, 'base64').toString();
  const [type, id] = decoded.split(':');
  return { type, id: parseInt(id) };
};
```

---

### Anti-Pattern: Nullable Everything

**What it looks like:**

```graphql
type User {
  id: ID
  email: String
  name: String
  posts: [Post]
}
```

**Why it fails:**
- Forces clients to handle null everywhere
- Hides required fields
- Makes schema less useful for code generation

**Correct approach:**

```graphql
type User {
  id: ID!                    # Always present
  email: String!             # Required field
  name: String               # Optional (nullable is intentional)
  posts: [Post!]!            # Non-null list with non-null items
}
```

---

## Integration Patterns

### backend-developer
- **Handoff**: Backend implements business logic → GraphQL architect exposes via schema
- **Collaboration**: Shared resolver implementation, authentication/authorization
- **Tools**: TypeScript, Node.js, ORM (Prisma, TypeORM), DataLoader
- **Example**: Backend provides user service → GraphQL creates User type with DataLoader

### api-designer
- **Handoff**: API-designer defines REST endpoints → GraphQL creates wrapper or migration plan
- **Collaboration**: API versioning strategy, REST→GraphQL migration
- **Tools**: OpenAPI specs, GraphQL schema, gateway patterns
- **Example**: REST API for products → GraphQL creates unified schema

### frontend-developer
- **Handoff**: Frontend needs data → GraphQL provides typed schema and queries
- **Collaboration**: Fragment colocation, Apollo Client cache configuration
- **Tools**: GraphQL Code Generator, Apollo Client, Relay
- **Example**: Frontend needs user profile → GraphQL provides typed query

### database-optimizer
- **Handoff**: GraphQL identifies slow resolvers → Database optimizer creates indexes
- **Collaboration**: Query optimization, connection pooling
- **Tools**: EXPLAIN ANALYZE, DataLoader batching, pg_stat_statements
- **Example**: DataLoader batching 100 users → Create covering index

### devops-engineer
- **Handoff**: GraphQL defines gateway architecture → DevOps deploys federated services
- **Collaboration**: Service mesh, load balancing, monitoring
- **Tools**: Kubernetes, Istio, Apollo Router, Grafana
- **Example**: Apollo Federation with 5 subgraphs → Deploy with service discovery

---

## Complete Resolver Example

```typescript
import { GraphQLResolveInfo } from 'graphql';
import DataLoader from 'dataloader';

interface Context {
  user: User | null;
  db: Database;
  loaders: {
    userLoader: DataLoader<string, User>;
    postLoader: DataLoader<string, Post>;
    commentCountLoader: DataLoader<string, number>;
  };
}

export const resolvers = {
  Query: {
    me: (_, __, { user }: Context) => user,
    
    user: async (_, { id }, { loaders }: Context) => {
      return loaders.userLoader.load(id);
    },
    
    posts: async (_, { first, after }, { db }: Context) => {
      const cursor = after ? decodeCursor(after) : null;
      const posts = await db.Post.findAll({
        where: cursor ? { id: { [Op.gt]: cursor } } : {},
        limit: first + 1,
        order: [['id', 'ASC']]
      });
      
      const hasNextPage = posts.length > first;
      const edges = posts.slice(0, first).map(post => ({
        node: post,
        cursor: encodeCursor(post.id)
      }));
      
      return {
        edges,
        pageInfo: {
          hasNextPage,
          hasPreviousPage: !!after,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor
        }
      };
    }
  },
  
  User: {
    posts: async (user, { first = 10 }, { db }: Context) => {
      return db.Post.findAll({
        where: { authorId: user.id },
        limit: first
      });
    },
    
    postCount: (user, _, { loaders }: Context) => {
      return loaders.postCountLoader.load(user.id);
    }
  },
  
  Post: {
    author: (post, _, { loaders }: Context) => {
      return loaders.userLoader.load(post.authorId);
    },
    
    commentCount: (post, _, { loaders }: Context) => {
      return loaders.commentCountLoader.load(post.id);
    }
  },
  
  Mutation: {
    createPost: async (_, { input }, { user, db }: Context) => {
      if (!user) {
        throw new Error('Authentication required');
      }
      
      const post = await db.Post.create({
        title: input.title,
        content: input.content,
        authorId: user.id
      });
      
      return {
        post,
        errors: []
      };
    }
  }
};
```

---

## DataLoader Factory

```typescript
// loaders.ts

import DataLoader from 'dataloader';
import { Database } from './db';

export function createLoaders(db: Database) {
  return {
    userLoader: new DataLoader<string, User>(async (ids) => {
      const users = await db.User.findAll({ where: { id: ids } });
      const userMap = new Map(users.map(u => [u.id, u]));
      return ids.map(id => userMap.get(id) || null);
    }),
    
    postLoader: new DataLoader<string, Post>(async (ids) => {
      const posts = await db.Post.findAll({ where: { id: ids } });
      const postMap = new Map(posts.map(p => [p.id, p]));
      return ids.map(id => postMap.get(id) || null);
    }),
    
    postCountLoader: new DataLoader<string, number>(async (userIds) => {
      const counts = await db.Post.findAll({
        attributes: ['authorId', [db.fn('COUNT', '*'), 'count']],
        where: { authorId: userIds },
        group: ['authorId']
      });
      const countMap = new Map(counts.map(c => [c.authorId, c.get('count')]));
      return userIds.map(id => countMap.get(id) || 0);
    }),
    
    commentCountLoader: new DataLoader<string, number>(async (postIds) => {
      const counts = await db.Comment.findAll({
        attributes: ['postId', [db.fn('COUNT', '*'), 'count']],
        where: { postId: postIds },
        group: ['postId']
      });
      const countMap = new Map(counts.map(c => [c.postId, c.get('count')]));
      return postIds.map(id => countMap.get(id) || 0);
    })
  };
}
```
