# GraphQL Architect - Technical Reference

## Apollo Federation Setup

**Scenario**: Split monolithic GraphQL into federated services (Users, Posts, Products)

### Step 1: Define federated subgraphs

```graphql
# users-service/schema.graphql

extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key", "@shareable"])

type User @key(fields: "id") {
  id: ID!
  email: String!
  username: String!
  profile: UserProfile
  createdAt: DateTime!
}

type UserProfile {
  bio: String
  avatarUrl: String
}

type Query {
  user(id: ID!): User
  me: User
}
```

```graphql
# posts-service/schema.graphql

extend schema
  @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key", "@external"])

# Reference User from users-service
type User @key(fields: "id") {
  id: ID! @external
  posts: [Post!]!  # Extend User with posts field
}

type Post @key(fields: "id") {
  id: ID!
  title: String!
  content: String!
  author: User!  # Reference to User
  createdAt: DateTime!
}

type Query {
  post(id: ID!): Post
  posts(first: Int, after: String): [Post!]!
}
```

```graphql
# products-service/schema.graphql

type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
  inventory: Int!
}

type Query {
  product(id: ID!): Product
  products: [Product!]!
}
```

### Step 2: Implement reference resolvers

```typescript
// users-service/resolvers.ts

export const resolvers = {
  User: {
    __resolveReference: async (ref, { loaders }) => {
      return loaders.userLoader.load(ref.id);
    }
  },
  
  Query: {
    user: (_, { id }, { loaders }) => loaders.userLoader.load(id),
    me: (_, __, { user }) => user
  }
};
```

```typescript
// posts-service/resolvers.ts

export const resolvers = {
  User: {
    // Extend User type with posts field
    posts: async (user, _, { db }) => {
      return db.Post.findAll({ where: { userId: user.id } });
    }
  },
  
  Post: {
    __resolveReference: async (ref, { loaders }) => {
      return loaders.postLoader.load(ref.id);
    },
    
    author: (post) => ({ __typename: 'User', id: post.userId })  // Return reference
  },
  
  Query: {
    post: (_, { id }, { loaders }) => loaders.postLoader.load(id),
    posts: async (_, { first, after }, { db }) => {
      return db.Post.findAll({ limit: first });
    }
  }
};
```

### Step 3: Set up Apollo Gateway

```typescript
// gateway/index.ts

import { ApolloGateway, IntrospectAndCompose } from '@apollo/gateway';
import { ApolloServer } from '@apollo/server';

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: 'http://localhost:4001/graphql' },
      { name: 'posts', url: 'http://localhost:4002/graphql' },
      { name: 'products', url: 'http://localhost:4003/graphql' }
    ]
  })
});

const server = new ApolloServer({ gateway });

await server.listen({ port: 4000 });
console.log('Gateway running at http://localhost:4000');
```

### Step 4: Query across federated services

```graphql
# Client query (gateway resolves across services)
query GetUserWithPosts {
  user(id: "1") {
    id
    username
    email
    posts {  # Resolved by posts-service
      id
      title
      content
    }
  }
}

# Gateway execution plan:
# 1. Query users-service for user(id: "1")
# 2. Query posts-service with user reference { __typename: "User", id: "1" }
# 3. Merge results
```

**Expected outcome**:
- Independent service deployment (users, posts, products)
- Type-safe cross-service references
- Single GraphQL endpoint for clients
- Distributed resolver execution

---

## Field-Level Authorization Directive

**Use case**: Protect sensitive fields with custom directive

```typescript
// directives.ts

import { mapSchema, getDirective, MapperKind } from '@graphql-tools/utils';
import { defaultFieldResolver, GraphQLSchema } from 'graphql';

export function authDirective(directiveName: string = 'auth') {
  return {
    authDirectiveTypeDefs: `
      directive @auth(requires: Role = USER) on FIELD_DEFINITION | OBJECT
      
      enum Role {
        ADMIN
        USER
        GUEST
      }
    `,
    
    authDirectiveTransformer: (schema: GraphQLSchema) =>
      mapSchema(schema, {
        [MapperKind.OBJECT_FIELD]: (fieldConfig) => {
          const authDirective = getDirective(schema, fieldConfig, directiveName)?.[0];
          
          if (authDirective) {
            const { requires } = authDirective;
            const { resolve = defaultFieldResolver } = fieldConfig;
            
            fieldConfig.resolve = async function (source, args, context, info) {
              const user = context.user;
              
              if (!user) {
                throw new Error('Unauthorized: Authentication required');
              }
              
              if (requires === 'ADMIN' && user.role !== 'ADMIN') {
                throw new Error('Forbidden: Admin access required');
              }
              
              return resolve(source, args, context, info);
            };
          }
          
          return fieldConfig;
        }
      })
  };
}

// Usage in schema:
/*
type User {
  id: ID!
  email: String! @auth(requires: ADMIN)  # Only admins can see emails
  username: String!
  salary: Float! @auth(requires: ADMIN)  # Sensitive field
}

type Query {
  users: [User!]! @auth  # Requires authentication (default USER role)
  adminDashboard: Dashboard! @auth(requires: ADMIN)  # Admin only
}
*/
```

---

## Query Complexity Limiting

**Use case**: Prevent DoS from expensive nested queries

```typescript
// complexity.ts

import { GraphQLSchema } from 'graphql';
import { 
  getComplexity, 
  simpleEstimator, 
  fieldExtensionsEstimator 
} from 'graphql-query-complexity';

export const complexityPlugin = (
  schema: GraphQLSchema, 
  maxComplexity: number = 1000
) => ({
  async requestDidStart() {
    return {
      async didResolveOperation({ request, document }) {
        const complexity = getComplexity({
          schema,
          operationName: request.operationName,
          query: document,
          variables: request.variables,
          estimators: [
            fieldExtensionsEstimator(),
            simpleEstimator({ defaultComplexity: 1 })
          ]
        });
        
        if (complexity > maxComplexity) {
          throw new Error(
            `Query complexity ${complexity} exceeds max ${maxComplexity}. ` +
            `Simplify your query or request fewer nested fields.`
          );
        }
        
        console.log('Query complexity:', complexity);
      }
    };
  }
});

// Schema with complexity annotations:
/*
type Query {
  users(first: Int!): [User!]!
}

type User {
  id: ID!
  posts(first: Int!): [Post!]!  # Multiplies complexity by first arg
}

# Query complexity calculation:
# query {
#   users(first: 100) {     # 100 complexity
#     posts(first: 10) {    # 100 × 10 = 1000 complexity
#       title               # Total: 1000
#     }
#   }
# }
*/
```

---

## Real-Time Subscriptions

```typescript
// subscriptions.ts

import { PubSub } from 'graphql-subscriptions';
import { withFilter } from 'graphql-subscriptions';

const pubsub = new PubSub();

const resolvers = {
  Subscription: {
    messageAdded: {
      subscribe: withFilter(
        () => pubsub.asyncIterator(['MESSAGE_ADDED']),
        (payload, variables, context) => {
          // Filter: only receive messages for user's channels
          return payload.messageAdded.channelId === variables.channelId;
        }
      )
    },
    
    userStatusChanged: {
      subscribe: () => pubsub.asyncIterator(['USER_STATUS_CHANGED'])
    }
  },
  
  Mutation: {
    sendMessage: async (_, { input }, { db, user }) => {
      const message = await db.Message.create({
        content: input.content,
        channelId: input.channelId,
        authorId: user.id
      });
      
      // Publish to subscribers
      pubsub.publish('MESSAGE_ADDED', { messageAdded: message });
      
      return message;
    }
  }
};

// Schema
/*
type Subscription {
  messageAdded(channelId: ID!): Message!
  userStatusChanged: UserStatus!
}

type Message {
  id: ID!
  content: String!
  author: User!
  createdAt: DateTime!
}
*/
```

---

## Caching Strategies

### Response Caching

```typescript
// Apollo Server response cache
import responseCachePlugin from '@apollo/server-plugin-response-cache';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    responseCachePlugin({
      sessionId: (context) => context.user?.id || null,
    })
  ]
});

// Schema with cache hints
/*
type Query {
  products: [Product!]! @cacheControl(maxAge: 3600)  # Cache 1 hour
  me: User @cacheControl(maxAge: 0, scope: PRIVATE)  # No cache
}

type Product @cacheControl(maxAge: 86400) {  # Cache 24 hours
  id: ID!
  name: String!
  price: Float! @cacheControl(maxAge: 60)  # Price changes more often
}
*/
```

### DataLoader Caching

```typescript
// Per-request caching (automatic with DataLoader)
const userLoader = new DataLoader(async (ids) => {
  // This only runs once per unique ID per request
  const users = await db.User.findAll({ where: { id: ids } });
  return ids.map(id => users.find(u => u.id === id));
});

// Cross-request caching (Redis)
import Redis from 'ioredis';
const redis = new Redis();

const cachedUserLoader = new DataLoader(async (ids) => {
  // Check Redis first
  const cached = await redis.mget(ids.map(id => `user:${id}`));
  const missingIds = ids.filter((_, i) => !cached[i]);
  
  if (missingIds.length > 0) {
    const users = await db.User.findAll({ where: { id: missingIds } });
    // Cache results
    await Promise.all(users.map(u => 
      redis.setex(`user:${u.id}`, 3600, JSON.stringify(u))
    ));
  }
  
  return ids.map((id, i) => 
    cached[i] ? JSON.parse(cached[i]) : null
  );
});
```
