# Technical Advisory - Code Examples & Patterns

This document contains real-world examples of technical advisory scenarios with complete analysis and recommendations.

## Example 1: Caching Strategy Decision

**Question**: Where should we implement caching for our API?

**Options**:
1. Application-level caching (in-memory)
2. Redis distributed cache
3. CDN for static responses
4. Database query cache

**Analysis**:

**Application-level**:
- Pros: Fast, simple, no external dependency
- Cons: Not shared across instances, lost on restart
- Best for: Session data, computed values

**Redis**:
- Pros: Shared across instances, persistent, flexible
- Cons: Network latency, another service to manage
- Best for: User sessions, frequently accessed data

**CDN**:
- Pros: Globally distributed, reduces server load
- Cons: Only for GET requests, cache invalidation complexity
- Best for: Static assets, public API responses

**Database query cache**:
- Pros: Automatic, database-managed
- Cons: Limited control, shared across queries
- Best for: Read-heavy queries with similar patterns

**Recommendation**:
Use a layered approach:
1. CDN for static/public responses (30% load reduction)
2. Redis for user sessions and hot data (50% database load reduction)
3. Application cache for expensive computations (100ms → 1ms)
4. Database query cache as baseline

**Rationale**:
- Each layer handles what it's best at
- Significant performance gains at each level
- Manageable complexity with team's skills
- Can implement incrementally (CDN → Redis → App cache)

**Trade-offs**:
- More moving parts (accepting complexity for performance)
- Cache invalidation coordination (mitigated by TTL strategy)
- Additional infrastructure costs (justified by user experience improvement)

---

## Example 2: Authentication Strategy

**Question**: How should we implement authentication?

**Context**:
- B2B SaaS application
- Need SSO for enterprise customers
- Also support email/password for small teams
- Mobile app planned

**Options**:
1. Build custom auth system
2. Use Auth0/Okta
3. Use OAuth2 + JWT with open-source library
4. Combination approach

**Analysis**:

**Custom Auth**:
- Pros: Full control, no external dependencies, no per-user costs
- Cons: Security risk, maintenance burden, feature parity challenge
- Risks: Security vulnerabilities, compliance issues

**Auth0/Okta**:
- Pros: Battle-tested, SSO built-in, compliance handled, rapid implementation
- Cons: Ongoing costs scale with users, vendor lock-in, limited customization
- Risks: Cost escalation, service outages

**OAuth2 + JWT (e.g., Passport.js)**:
- Pros: Open standard, flexible, community support, no vendor lock-in
- Cons: Need to implement SSO connectors, security responsibility
- Risks: Implementation errors, maintenance overhead

**Recommendation**:
Use Auth0 initially, with abstraction layer for future flexibility

**Rationale**:
1. **Time-to-market**: SSO support critical for enterprise deals (3-month faster)
2. **Security**: Authentication is too critical to get wrong
3. **Compliance**: SOC2 requirement easier with proven solution
4. **Cost**: At current scale (<1000 users), monthly cost acceptable ($500/mo)
5. **Flexibility**: Abstract behind our own auth interface, can migrate later

**Implementation**:
```typescript
// Abstraction layer
interface AuthProvider {
  login(credentials): Promise<User>;
  logout(): Promise<void>;
  verifyToken(token): Promise<User>;
  refreshToken(token): Promise<string>;
}

// Auth0 implementation
class Auth0Provider implements AuthProvider {
  // Auth0-specific implementation
}

// Our auth service
class AuthService {
  constructor(private provider: AuthProvider) {}
  // Business logic using provider
}
```

**Migration Path**:
- Monitor costs and scaling
- If costs become prohibitive, implement OAuth2Provider
- Swap implementations without changing business logic

**Trade-offs**:
- Accepting vendor dependence for speed (mitigated by abstraction)
- Accepting ongoing costs for security (justified by compliance needs)
- Accepting less customization for reliability (acceptable for core auth)

---

## Example 3: Database Migration Strategy

**Question**: Should we migrate from PostgreSQL to a distributed database?

**Context**:
- Current PostgreSQL handling 10K writes/sec
- Growing 50% annually
- Starting to see lock contention
- Team of 8 engineers

**Options**:
1. Vertical scaling (bigger PostgreSQL instance)
2. Read replicas + connection pooling
3. Sharding PostgreSQL
4. Migrate to CockroachDB/TiDB
5. Migrate to DynamoDB

**Analysis Matrix**:

| Factor | Vertical | Replicas | Sharding | CockroachDB | DynamoDB |
|--------|----------|----------|----------|-------------|----------|
| Complexity | Low | Medium | High | Medium | Medium |
| Cost | Medium | Medium | High | High | Variable |
| Team learning | None | Low | High | High | High |
| Time to implement | Days | Weeks | Months | Months | Months |
| Scalability ceiling | 50K w/s | 50K w/s | 500K+ w/s | 500K+ w/s | Unlimited |
| Risk | Low | Low | High | Medium | Medium |

**Recommendation**:
Implement read replicas + connection pooling now; evaluate CockroachDB in 12 months

**Rationale**:
1. **Current bottleneck is reads**: 80% of queries are reads that can use replicas
2. **Quick win**: PgBouncer can be deployed in days
3. **Buy time**: This solution handles 3x current load
4. **Learn distributed patterns**: Start designing for eventual sharding
5. **Team readiness**: 12 months to build distributed systems expertise

**Implementation Plan**:
```
Month 1: Deploy PgBouncer connection pooling
Month 2: Add 2 read replicas, route read traffic
Month 3-4: Optimize slow queries identified during migration
Month 6: Evaluate progress, begin CockroachDB POC if needed
Month 12: Decision point for distributed database
```

**Trade-offs**:
- Not solving "ultimate" scalability now (acceptable given growth rate)
- Some complexity with read replica lag (mitigated by careful query routing)
- Delaying distributed database learning (offset by focused POC work)

---

## Example 4: Microservices Extraction

**Question**: Should we extract the payment system into a microservice?

**Context**:
- Monolithic e-commerce application
- Payment code is 15% of codebase
- 3 developers work on payments
- Need PCI compliance audit
- Want faster payment feature releases

**Analysis**:

**Benefits of Extraction**:
- Isolated PCI compliance scope (significant audit cost reduction)
- Independent deployment (weekly → daily releases possible)
- Team autonomy (payment team can choose optimal tech)
- Fault isolation (payment failures don't crash main app)

**Costs of Extraction**:
- Network boundary introduction (latency, failure modes)
- Data consistency challenges (distributed transactions)
- Operational overhead (another service to monitor)
- Migration effort (3-6 months estimated)

**Risk Analysis**:
- **Data consistency**: High risk - orders and payments must be consistent
  - Mitigation: Saga pattern with compensating transactions
- **Performance**: Medium risk - network calls add latency
  - Mitigation: Async processing, caching
- **Partial failure**: High risk - what if payment service is down?
  - Mitigation: Circuit breakers, retry logic, graceful degradation

**Recommendation**:
Extract payment service with careful preparation

**Rationale**:
1. PCI compliance scope reduction justifies effort alone
2. Clear bounded context (payment is well-defined domain)
3. Dedicated team available for ownership
4. Accept increased operational complexity for compliance benefits

**Pre-Extraction Checklist**:
```markdown
Before extracting:
- [ ] Define clear API contract (OpenAPI spec)
- [ ] Implement feature flags for gradual rollout
- [ ] Set up distributed tracing (Jaeger/Zipkin)
- [ ] Establish SLOs for payment service
- [ ] Design saga for order-payment consistency
- [ ] Create runbook for payment service incidents
- [ ] Train team on microservices debugging
```

---

## Anti-Pattern Examples

### Anti-Pattern: Resume-Driven Development

**Scenario**: Team wants to use Kubernetes for a small application with 100 users

**Analysis**:
- Application runs fine on single VM
- Team has no K8s experience
- No requirements for auto-scaling
- DevOps bandwidth is limited

**Red Flags**:
- "It's what everyone is using"
- "We should learn it anyway"
- "It will be easier when we scale"

**Better Approach**:
- Use managed platform (Heroku, Railway, Render)
- Focus engineering effort on product
- Revisit when scaling is actually needed

### Anti-Pattern: Premature Optimization

**Scenario**: Team wants to implement event sourcing for simple CRUD app

**Analysis**:
- Current data model is straightforward
- No audit requirements
- No need for temporal queries
- Team hasn't built event-sourced systems

**Red Flags**:
- "It's the right pattern for this domain"
- "We might need history later"
- "Event sourcing is best practice"

**Better Approach**:
- Start with simple CRUD
- Add audit logging if needed
- Migrate to event sourcing only when benefits are clear

### Anti-Pattern: Analysis Paralysis

**Scenario**: Team has spent 3 weeks evaluating 8 different frontend frameworks

**Analysis**:
- All frameworks are production-ready
- Team has React experience
- Timeline is slipping

**Red Flags**:
- Endless POCs and benchmarks
- New options keep emerging
- Perfect solution doesn't exist

**Better Approach**:
- Set evaluation deadline
- Define must-have criteria
- Choose based on team expertise
- Accept that good enough is good enough
