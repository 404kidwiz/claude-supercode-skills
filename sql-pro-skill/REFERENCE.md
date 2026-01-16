# SQL Pro - Technical Reference

## What This Skill Does

The SQL pro analyzes database schemas, optimizes queries for performance, designs efficient database structures, and implements data management solutions that ensure scalability and data integrity across multiple database platforms.

### Schema Analysis
- Review database design and normalization
- Analyze index usage and effectiveness
- Identify query patterns and bottlenecks
- Assess data distribution and growth
- Evaluate constraint design and referential integrity
- Check statistics accuracy and maintenance

### Implementation Phase
- Design set-based operations avoiding row-by-row processing
- Optimize subqueries and CTEs for performance
- Apply appropriate join strategies and algorithms
- Implement proper indexing strategies
- Leverage platform-specific features and optimizations
- Document query intent and logic

### Performance Verification
- Analyze execution plans and identify bottlenecks
- Confirm index usage and scan reduction
- Eliminate table scans and key lookups
- Update statistics for accurate plans
- Eliminate deadlocks and blocking
- Test scalability with production data volumes

## Advanced Query Patterns

- Common Table Expressions (CTEs) and recursive queries
- Window functions: ROW_NUMBER, RANK, LEAD, LAG, aggregate windows
- PIVOT/UNPIVOT operations for data transformation
- Hierarchical queries for tree/graph structures
- Graph traversal patterns and recursive CTEs
- Temporal queries for time-based analysis
- Geospatial operations and spatial indexing

## Query Optimization Techniques

- Execution plan analysis and interpretation
- Index selection strategies and covering indexes
- Statistics management and maintenance
- Query hints and plan guides (when necessary)
- Parallel query execution tuning
- Partition pruning and partitioning strategies
- Join algorithm selection (hash, merge, nested loop)
- Subquery optimization and EXISTS vs. IN patterns

## Index Design Patterns

- Clustered vs. non-clustered indexes
- Covering indexes for query optimization
- Filtered/partial indexes for selective queries
- Function-based/indexes on expressions
- Composite index column ordering
- Index intersection and union strategies
- Missing index analysis and recommendations
- Index maintenance and fragmentation control

## Transaction Management

- Isolation level selection and implications
- Deadlock prevention and resolution
- Lock escalation and escalation control
- Optimistic concurrency with row versioning
- Savepoint usage for nested transactions
- Distributed transactions and two-phase commit
- Transaction log optimization and sizing

## Performance Tuning

- Query plan caching and parameter sniffing solutions
- Statistics updates and auto-stats configuration
- Table partitioning for large tables
- Materialized views and indexed views
- Query rewriting and optimization patterns
- Resource governor/workload management
- Wait statistics analysis and resolution
- Connection pooling optimization

## Data Warehousing

- Star schema and snowflake schema design
- Slowly changing dimensions (SCD types 1-4)
- Fact table optimization and compression
- ETL/ELT pattern design and optimization
- Aggregate tables and pre-computed summaries
- Columnstore indexes and analytical queries
- Data compression and storage optimization
- Incremental loading strategies

## Database-Specific Features

### PostgreSQL
- JSONB handling and indexing
- Array types and operations
- Advanced CTEs and materialized views
- Full-text search and trigram matching
- Table partitioning and declarative partitioning
- Custom functions and extensions

### MySQL
- Storage engines (InnoDB, MyISAM) selection
- Replication and master-slave configurations
- Query cache analysis and optimization
- Partitioning and sharding strategies

### SQL Server
- Columnstore indexes and analytical workloads
- In-Memory OLTP and memory-optimized tables
- Query Store and plan regression monitoring
- Temporal tables and system-versioned tables

### Oracle
- Partitioning strategies (range, list, hash, composite)
- Real Application Clusters (RAC)
- Materialized views and query rewrite
- Advanced queuing and CDC

## Security Implementation

- Row-level security (RLS) policies
- Dynamic data masking for sensitive data
- Encryption at rest (TDE) and in transit
- Column-level encryption and key management
- Audit trail design and implementation
- Permission management and role-based access
- SQL injection prevention and parameterized queries
- Data anonymization and redaction techniques

## Best Practices

### Query Development
- Start with understanding the data model and relationships
- Write readable CTEs to break complex queries into logical steps
- Apply filtering early in the query to reduce row count
- Use EXISTS instead of COUNT(*) for existence checks
- Avoid SELECT * - specify only needed columns
- Implement pagination using OFFSET-FETCH or LIMIT-OFFSET
- Handle NULL values explicitly with IS NULL/IS NOT NULL
- Test queries with production data volumes

### Indexing Strategy
- Create indexes on columns used in WHERE, JOIN, and ORDER BY
- Use covering indexes to eliminate key lookups
- Consider filtered/partial indexes for selective queries
- Order columns in composite indexes by selectivity
- Monitor index usage and remove unused indexes
- Rebuild/reorganize fragmented indexes regularly
- Balance read vs. write performance for index selection

### Performance Tuning
- Always analyze execution plans before optimization
- Update statistics for accurate query plans
- Use parameterized queries for plan reuse
- Consider partitioning for large tables
- Implement query hints only as last resort
- Monitor wait statistics for bottleneck identification
- Set appropriate isolation levels for concurrency
- Use proper data types to minimize storage and improve performance

### Database Design
- Normalize to reduce data redundancy (usually 3NF)
- Denormalize for read-heavy workloads and performance
- Use appropriate data types for columns
- Define constraints for data integrity
- Plan for growth and scalability from the start
- Document schema decisions and business rules
- Implement proper naming conventions
- Design for query patterns, not just data storage

## Integration with Other Skills

- **backend-engineer**: Optimize application queries and design efficient data access patterns
- **database-administrator**: Collaborate on schema design, maintenance, and backup strategies
- **data-engineer**: Support on ETL/ELT pipeline optimization and data transformation
- **python-developer**: Help with ORM query optimization and SQLAlchemy patterns
- **java-architect**: Collaborate on JPA/Hibernate query optimization and JPQL
- **performance-engineer**: Work on database performance tuning and monitoring
- **devops-engineer**: Assist on database deployment, migrations, and CI/CD
- **data-scientist**: Help optimize analytical queries and data extraction
