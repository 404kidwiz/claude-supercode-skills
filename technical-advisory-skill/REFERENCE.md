# Technical Advisory - Technical Reference

This document contains detailed analysis frameworks, evaluation matrices, and decision support tools for technical advisory.

## Analysis Frameworks

### Architecture Decision Record (ADR)

```markdown
# Title: [Decision being made]

## Status
[Proposed | Accepted | Deprecated | Superseded]

## Context
What is the issue that we're seeing that is motivating this decision or change?

## Decision
What is the change that we're proposing and/or doing?

## Consequences
What becomes easier or more difficult to do because of this change?

### Positive
- Benefit 1
- Benefit 2

### Negative
- Cost 1
- Cost 2

### Risks
- Risk 1 (Mitigation: ...)
- Risk 2 (Mitigation: ...)
```

### Technology Evaluation Matrix

| Criterion | Weight | Option A | Option B | Option C |
|-----------|--------|----------|----------|----------|
| Performance | 0.3 | 8/10 (2.4) | 6/10 (1.8) | 9/10 (2.7) |
| Maintainability | 0.25 | 7/10 (1.75) | 9/10 (2.25) | 6/10 (1.5) |
| Team Familiarity | 0.2 | 9/10 (1.8) | 5/10 (1.0) | 7/10 (1.4) |
| Cost | 0.15 | 6/10 (0.9) | 8/10 (1.2) | 7/10 (1.05) |
| Scalability | 0.1 | 8/10 (0.8) | 7/10 (0.7) | 9/10 (0.9) |
| **Total** | 1.0 | **7.65** | **6.95** | **7.55** |

### Risk Assessment Matrix

| Risk | Likelihood | Impact | Score | Mitigation |
|------|------------|--------|-------|------------|
| Data loss | Low | High | Medium | Regular backups, replication |
| Performance degradation | Medium | Medium | Medium | Load testing, caching |
| Security breach | Low | Critical | High | Security audit, penetration testing |
| Team knowledge gap | High | Low | Medium | Training, documentation |

## Common Scenarios

### Scenario 1: Monolith vs. Microservices

**Context**: Growing application, team discussing microservices

**Analysis**:

**Monolith Pros**:
- Simpler deployment
- Easier to develop locally
- Lower infrastructure costs
- Simpler data consistency
- Less operational complexity

**Monolith Cons**:
- Tight coupling
- Difficult to scale independently
- Longer deployment cycles
- Team coordination challenges

**Microservices Pros**:
- Independent scaling
- Technology flexibility
- Team autonomy
- Fault isolation
- Faster deployment per service

**Microservices Cons**:
- Distributed system complexity
- Network latency
- Data consistency challenges
- Higher infrastructure costs
- Operational overhead

**Recommendation Framework**:

Stay with monolith if:
- Team < 15 people
- Application load is manageable
- Deployment frequency < weekly
- Data consistency is critical
- Team lacks distributed systems experience

Consider microservices if:
- Team > 30 people
- Clear bounded contexts exist
- Need independent scaling
- Different performance requirements per module
- Team has microservices expertise

### Scenario 2: SQL vs. NoSQL

**Context**: Choosing database for new feature

**Analysis**:

**SQL (Relational)**:
- Strong ACID guarantees
- Complex queries with JOINs
- Structured schema
- Mature tooling
- Well-understood patterns

**NoSQL (Document/Key-Value)**:
- Flexible schema
- Horizontal scalability
- High write throughput
- Simpler data models
- Eventually consistent (often)

**Recommendation Framework**:

Choose SQL if:
- Data has clear relationships
- Need complex queries
- ACID compliance is critical
- Schema is relatively stable
- Team is familiar with SQL

Choose NoSQL if:
- Data is document-oriented
- Schema evolves frequently
- Extreme scalability needed
- Simple query patterns
- Need high write throughput

### Scenario 3: REST vs. GraphQL

**Context**: API design for new service

**Analysis**:

**REST Pros**:
- Simple, well-understood
- HTTP caching works well
- Easy to debug
- Wide tool support
- Stateless by design

**REST Cons**:
- Over/under-fetching data
- Multiple requests needed
- Version management challenges
- Client needs to know endpoints

**GraphQL Pros**:
- Clients specify exact data needed
- Single request for complex queries
- Strong typing
- Introspection
- Real-time with subscriptions

**GraphQL Cons**:
- Caching is complex
- Query complexity attacks
- Learning curve
- Tooling overhead
- Harder to debug

**Recommendation Framework**:

Choose REST if:
- Simple, stable API
- Caching is critical
- Team lacks GraphQL experience
- Mobile/bandwidth not an issue
- Need simple debugging

Choose GraphQL if:
- Complex data requirements
- Multiple clients with different needs
- Mobile/bandwidth constrained
- Need real-time updates
- Team has GraphQL expertise

## Decision Support Tools

### The Five Whys

Keep asking "why" to get to root cause:
1. Why do we need this? → To improve performance
2. Why is performance poor? → Database queries are slow
3. Why are queries slow? → No indexes on foreign keys
4. Why no indexes? → Migrations didn't include them
5. Why didn't migrations include them? → No review process for DB changes

**Real problem**: Missing DB change review process

### Pre-Mortem Analysis

Imagine the decision failed. Why did it fail?
- "We chose microservices, and it failed because..."
  - Team lacked distributed systems experience
  - Operational complexity overwhelmed us
  - Cost exceeded budget
  - Data consistency issues caused bugs

Now mitigate those risks before implementing.

### Reversibility Assessment

- **Easily Reversible** (Type 1): Feature flags, configuration changes
- **Moderately Reversible** (Type 2): Library changes, API designs
- **Difficult to Reverse** (Type 3): Database schemas, external contracts
- **Irreversible** (Type 4): Data formats, published APIs

Type 3-4 decisions need more analysis and consensus.

## Evaluation Criteria Deep Dive

### 1. Alignment with Requirements

Questions to ask:
- Does it solve the stated problem?
- Does it meet all functional requirements?
- Does it satisfy non-functional requirements (performance, security, etc.)?
- Does it support anticipated future needs?
- Are there any requirement gaps?

### 2. Technical Soundness

Questions to ask:
- Is the architecture coherent and consistent?
- Does it follow established design principles (SOLID, DRY, etc.)?
- Are the abstractions appropriate?
- Is it testable and maintainable?
- Does it avoid known anti-patterns?

### 3. Risk Profile

Questions to ask:
- What are the technical risks?
- What are the operational risks?
- What are the business risks?
- How likely are the risks?
- What's the impact if they occur?
- Can they be mitigated?

### 4. Implementation Feasibility

Questions to ask:
- Does the team have required expertise?
- Is the timeline realistic?
- Are dependencies available and stable?
- What's the learning curve?
- Are there sufficient resources?

### 5. Operational Impact

Questions to ask:
- How complex is deployment?
- How easy is it to monitor?
- How do we debug issues in production?
- What's the rollback strategy?
- How does it affect on-call burden?

### 6. Long-term Considerations

Questions to ask:
- How will it age?
- Is it extensible?
- What's the migration path if we need to change?
- What's the total cost of ownership?
- Does it create technical debt?

## Stakeholder Communication Templates

### Decision Summary Template

```markdown
## Decision: [Title]

### Recommendation
[One sentence recommendation]

### Context
[2-3 sentences describing the situation]

### Key Factors
1. [Factor 1]: [Brief explanation]
2. [Factor 2]: [Brief explanation]
3. [Factor 3]: [Brief explanation]

### Trade-offs
- Accepting [X] in exchange for [Y]
- Accepting risk of [Z] because [mitigation]

### Next Steps
1. [Immediate action]
2. [Follow-up action]
3. [Timeline/milestone]
```

### Risk Communication Template

```markdown
## Risk: [Title]

### Description
[What could go wrong]

### Likelihood: [Low/Medium/High]
### Impact: [Low/Medium/High/Critical]

### Mitigation Strategy
[How we're addressing this risk]

### Contingency Plan
[What we'll do if the risk materializes]

### Owner
[Who is responsible for monitoring]
```
