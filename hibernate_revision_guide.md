**Hibernate Revision Guide**

### 1. Overview & Architecture

- **ORM (Object–Relational Mapping)**: Maps Java objects to relational database tables. Each Java class becomes a table, and each instance becomes a row.
- **SessionFactory**: A heavyweight, thread-safe factory for creating `Session` objects. Created once per application lifetime. It caches metadata and is expensive to initialize.
- **Session**: A lightweight, single-threaded object used to interact with the database. It represents a unit of work and contains a first-level cache.
- **Transaction**: Defines atomic units of work. Transactions in Hibernate are abstracted over JDBC or JTA.
- **Connection Management**: Hibernate handles connection pooling and transaction demarcation through configuration (`hibernate.cfg.xml`, HikariCP, C3P0, etc.).

### 2. Configuration

- **XML Configuration**: Define `hibernate.cfg.xml` with database properties and entity mappings.
- **Programmatic Configuration**: Use `Configuration` class and `StandardServiceRegistryBuilder`.
- **Important Properties**:
  - `hibernate.dialect`: SQL dialect for the target DB (e.g., `MySQL5Dialect`).
  - `hibernate.hbm2ddl.auto`: Schema generation (`validate`, `update`, `create`, `create-drop`).
  - `hibernate.show_sql`, `hibernate.format_sql`: For logging SQL output.
  - Second-level cache settings (provider, region factory, etc.).

### 3. Entity Mapping

- Annotate POJOs with:
  - `@Entity`: Marks the class as a Hibernate entity.
  - `@Table`: Optional. Specifies the table name.
  - `@Id`: Declares the primary key.
  - `@GeneratedValue`: Specifies auto-generation strategy (`AUTO`, `IDENTITY`, `SEQUENCE`, `TABLE`).
- Data types and attributes:
  - `@Column`: Maps to a DB column with optional settings (length, nullable, etc.).
  - `@Temporal`: For `Date`/`Calendar` types (DATE, TIME, TIMESTAMP).
  - `@Enumerated`: For enums.
  - `@Lob`: For large objects like text/blob.
- **Embeddables**:
  - Use `@Embeddable` to create reusable value-type components.
  - Embed in entity using `@Embedded`.

### 4. Relationships & Associations

- **@OneToOne**: Typically joined on a primary or foreign key.
- **@ManyToOne**: Foreign key relationship to another entity.
- **@OneToMany**: Typically mapped by the child using `mappedBy`. Needs a collection type.
- **@ManyToMany**: Requires a join table with `@JoinTable`.
- **Cascading**:
  - `CascadeType.ALL` = all operations propagate.
  - Fine-tune with `PERSIST`, `MERGE`, `REMOVE`, `DETACH`, `REFRESH`.
- **Orphan Removal**: `orphanRemoval=true` deletes child records no longer associated.

### 5. Fetching Strategies

- **FetchType.LAZY (default for collections)**: Data is loaded only when accessed.
- **FetchType.EAGER (default for single-valued associations)**: Data is fetched immediately with the parent.
- Override defaults using `fetch = FetchType.LAZY` or `EAGER`.
- Avoid **N+1 Select Problem** using:
  - `JOIN FETCH` in HQL.
  - `@BatchSize(size = N)`.
  - `@Fetch(FetchMode.SUBSELECT)` for collection batching.

### 6. Caching

- **L1 Cache (First-Level)**:
  - Session-scoped. Active by default.
  - Repeated `session.get()` returns same instance.
- **L2 Cache (Second-Level)**:
  - Shared across sessions.
  - Configure using providers (Ehcache, Infinispan).
  - Annotate with `@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)`.
- **Query Cache**:
  - Use `setCacheable(true)` on queries.
  - Enable using `hibernate.cache.use_query_cache=true`.

### 7. Querying

- **HQL**:
  - Object-oriented SQL-like query language.
  - Example: `FROM Order o WHERE o.customer.name = :name`
- **Criteria API**:
  - Type-safe, dynamic query building.
  - Useful for complex, conditionally constructed queries.
- **Query by Example (QBE)**:
  - Create a template entity and Hibernate finds similar records.
- **Native SQL**:
  - Direct SQL using `createNativeQuery()`.
  - Must handle entity mapping manually.
- **Named Queries**:
  - Declared with `@NamedQuery` and `@NamedNativeQuery`.

### 8. Transactions & Concurrency

- Use `session.beginTransaction()` and `transaction.commit()`.
- Integration with Spring's `@Transactional` recommended.
- **Optimistic Locking**:
  - Use `@Version` to detect stale updates.
- **Pessimistic Locking**:
  - Explicit database-level locks using `LockMode.PESSIMISTIC_WRITE`.

### 9. Performance Tuning

- **Avoid N+1 queries** using:
  - Fetch joins, batch size tuning.
- **DTO Projection**:
  - Use constructor expressions to retrieve only necessary fields.
- **StatelessSession**:
  - Useful for batch processing; lacks L1 cache.
- **ScrollableResult**:
  - For large result sets with pagination or cursor-based iteration.

### 10. Advanced Features

- **Interceptors**:
  - Implement `EmptyInterceptor` to log or manipulate behavior.
- **Event Listeners**:
  - Hook into lifecycle events like `onLoad`, `onFlush`, `onSave`.
- **Custom Types**:
  - Implement `UserType` or use `@Type` for mapping custom classes.
- **Multi-Tenancy**:
  - Use discriminator or schema separation for tenant data.
- **Envers**:
  - Add `@Audited` to enable automatic entity versioning and audit history.

### 11. Testing & Best Practices

- Use in-memory DB (H2) for integration testing.
- Use Spring Boot’s `@DataJpaTest` for repository testing.
- Avoid returning JPA entities directly to clients (prefer DTOs).
- Prevent LazyInitializationException with Open Session in View pattern (use cautiously).
- Set `hbm2ddl.auto=validate` in production to ensure schema correctness.

---

> **Study Tips**
> 1. Draw session and transaction boundaries to visualize flow.
> 2. Practice HQL vs. Criteria queries with joins and projections.
> 3. Experiment with L1, L2, and query caching in a sample app.
> 4. Intentionally trigger and fix the N+1 problem.
> 5. Map and persist complex object graphs with cascading and orphan removal.

Good luck with your interview prep! Let me know if you'd like flashcards or a cheat sheet next.

