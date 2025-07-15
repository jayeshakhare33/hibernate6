**Hibernate Revision Guide**

### 1. Overview & Architecture

```java
Configuration cfg = new Configuration().configure();
SessionFactory sessionFactory = cfg.buildSessionFactory();
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

// perform DB operations

tx.commit();
session.close();
```

* **ORM (Object–Relational Mapping)**: Maps Java objects to relational database tables. Each Java class becomes a table, and each instance becomes a row.
* **SessionFactory**: A heavyweight, thread-safe factory for creating `Session` objects. Created once per application lifetime. It caches metadata and is expensive to initialize.
* **Session**: A lightweight, single-threaded object used to interact with the database. It represents a unit of work and contains a first-level cache.
* **Transaction**: Defines atomic units of work. Transactions in Hibernate are abstracted over JDBC or JTA.
* **Connection Management**: Hibernate handles connection pooling and transaction demarcation through configuration (`hibernate.cfg.xml`, HikariCP, C3P0, etc.).

### 2. Configuration

**hibernate.cfg.xml**

```xml
<hibernate-configuration>
  <session-factory>
    <property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
    <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
    <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/testdb</property>
    <property name="hibernate.connection.username">root</property>
    <property name="hibernate.connection.password">password</property>
    <property name="hibernate.hbm2ddl.auto">update</property>
    <mapping class="com.example.User"/>
  </session-factory>
</hibernate-configuration>
```

**Programmatic Config Example**

```java
Configuration configuration = new Configuration();
configuration.configure();
SessionFactory sessionFactory = configuration.buildSessionFactory();
```

* **XML Configuration**: Define `hibernate.cfg.xml` with database properties and entity mappings.
* **Programmatic Configuration**: Use `Configuration` class and `StandardServiceRegistryBuilder`.
* **Important Properties**:

  * `hibernate.dialect`: SQL dialect for the target DB (e.g., `MySQL5Dialect`).
  * `hibernate.hbm2ddl.auto`: Schema generation (`validate`, `update`, `create`, `create-drop`).
  * `hibernate.show_sql`, `hibernate.format_sql`: For logging SQL output.
  * Second-level cache settings (provider, region factory, etc.).

### 3. Entity Mapping

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "username", nullable = false, unique = true)
    private String username;

    @Temporal(TemporalType.TIMESTAMP)
    private Date registeredAt;

    @Enumerated(EnumType.STRING)
    private Role role;

    @Lob
    private String profileDescription;

    @Embedded
    private Address address;
}

@Embeddable
public class Address {
    private String street;
    private String city;
}
```

* Annotate POJOs with:

  * `@Entity`: Marks the class as a Hibernate entity.
  * `@Table`: Optional. Specifies the table name.
  * `@Id`: Declares the primary key.
  * `@GeneratedValue`: Specifies auto-generation strategy (`AUTO`, `IDENTITY`, `SEQUENCE`, `TABLE`).
* Data types and attributes:

  * `@Column`: Maps to a DB column with optional settings (length, nullable, etc.).
  * `@Temporal`: For `Date`/`Calendar` types (DATE, TIME, TIMESTAMP).
  * `@Enumerated`: For enums.
  * `@Lob`: For large objects like text/blob.
* **Embeddables**:

  * Use `@Embeddable` to create reusable value-type components.
  * Embed in entity using `@Embedded`.

### 4. Relationships & Associations

* **@OneToOne**: Typically joined on a primary or foreign key.

```java
@Entity
public class Passport {
    @Id
    private Long id;

    @OneToOne
    @JoinColumn(name = "person_id")
    private Person person;
}
```

* **@ManyToOne**: Foreign key relationship to another entity.

```java
@Entity
public class Order {
    @Id
    private Long id;

    @ManyToOne
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

* **@OneToMany**: Mapped by the child using `mappedBy`. Requires a collection type.

```java
@Entity
public class Customer {
    @Id
    private Long id;

    @OneToMany(mappedBy = "customer")
    private List<Order> orders;
}
```

* **@ManyToMany**: Requires a join table with `@JoinTable`.

```java
@Entity
public class Student {
    @Id
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private List<Course> courses;
}
```

* **Cascading**:

  * `CascadeType.ALL` = all operations propagate.
  * Fine-tune with `PERSIST`, `MERGE`, `REMOVE`, `DETACH`, `REFRESH`.

* **Orphan Removal**: `orphanRemoval=true` deletes child records no longer associated.

* **@OneToOne**: Typically joined on a primary or foreign key.

* **@ManyToOne**: Foreign key relationship to another entity.

* **@OneToMany**: Typically mapped by the child using `mappedBy`. Needs a collection type.

* **@ManyToMany**: Requires a join table with `@JoinTable`.

* **Cascading**:

  * `CascadeType.ALL` = all operations propagate.
  * Fine-tune with `PERSIST`, `MERGE`, `REMOVE`, `DETACH`, `REFRESH`.

* **Orphan Removal**: `orphanRemoval=true` deletes child records no longer associated.

### 5. Fetching Strategies

```java
// Fetch join example (HQL)
List<Order> orders = session.createQuery(
    "SELECT o FROM Order o JOIN FETCH o.items", Order.class).getResultList();

// Batch fetch using annotation
@OneToMany(mappedBy = "order")
@BatchSize(size = 10)
private List<LineItem> items;

// Subselect fetch
@OneToMany(fetch = FetchType.LAZY)
@Fetch(FetchMode.SUBSELECT)
private List<LineItem> items;
```

* **FetchType.LAZY (default for collections)**: Data is loaded only when accessed.
* **FetchType.EAGER (default for single-valued associations)**: Data is fetched immediately with the parent.
* Override defaults using `fetch = FetchType.LAZY` or `EAGER`.
* Avoid **N+1 Select Problem** using:

  * `JOIN FETCH` in HQL.
  * `@BatchSize(size = N)`.
  * `@Fetch(FetchMode.SUBSELECT)` for collection batching.

### 6. Caching

```java
// Enable L2 cache for an entity
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
  @Id
  private Long id;
  private String name;
}

// Enable query cache
Query<Product> query = session.createQuery("from Product", Product.class);
query.setCacheable(true);
```

* **L1 Cache (First-Level)**:

  * Session-scoped. Active by default.
  * Repeated `session.get()` returns same instance.
* **L2 Cache (Second-Level)**:

  * Shared across sessions.
  * Configure using providers (Ehcache, Infinispan).
  * Annotate with `@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)`.
* **Query Cache**:

  * Use `setCacheable(true)` on queries.
  * Enable using `hibernate.cache.use_query_cache=true`.

### 7. Querying

```java
// HQL
Query<User> query = session.createQuery("FROM User WHERE username = :name", User.class);
query.setParameter("name", "john");
List<User> users = query.list();

// Criteria API
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.select(root).where(cb.equal(root.get("username"), "john"));
List<User> results = session.createQuery(cq).getResultList();

// Native query
List<Object[]> rows = session.createNativeQuery("SELECT * FROM users").getResultList();
```

* **HQL**:

  * Object-oriented SQL-like query language.
  * Example: `FROM Order o WHERE o.customer.name = :name`
* **Criteria API**:

  * Type-safe, dynamic query building.
  * Useful for complex, conditionally constructed queries.
* **Query by Example (QBE)**:

  * Create a template entity and Hibernate finds similar records.
* **Native SQL**:

  * Direct SQL using `createNativeQuery()`.
  * Must handle entity mapping manually.
* **Named Queries**:

  * Declared with `@NamedQuery` and `@NamedNativeQuery`.

### 8. Transactions & Concurrency

```java
// Basic transaction handling
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();

User user = session.get(User.class, 1L);
user.setUsername("updated");

tx.commit();
session.close();

// Optimistic locking with @Version
@Entity
public class Account {
  @Id
  private Long id;

  @Version
  private int version;
}
```

* Use `session.beginTransaction()` and `transaction.commit()`.
* Integration with Spring's `@Transactional` recommended.
* **Optimistic Locking**:

  * Use `@Version` to detect stale updates.
* **Pessimistic Locking**:

  * Explicit database-level locks using `LockMode.PESSIMISTIC_WRITE`.

### 9. Performance Tuning

```java
// Stateless session for bulk inserts
StatelessSession session = sessionFactory.openStatelessSession();
Transaction tx = session.beginTransaction();

for (int i = 0; i < 1000; i++) {
    Product product = new Product();
    product.setName("Product " + i);
    session.insert(product);
}

tx.commit();
session.close();
```

* **Avoid N+1 queries** using:

  * Fetch joins, batch size tuning.
* **DTO Projection**:

  * Use constructor expressions to retrieve only necessary fields.
* **StatelessSession**:

  * Useful for batch processing; lacks L1 cache.
* **ScrollableResult**:

  * For large result sets with pagination or cursor-based iteration.

### 10. Advanced Features

```java
// Interceptor
class AuditInterceptor extends EmptyInterceptor {
    public boolean onSave(Object entity, Serializable id, Object[] state, String[] propertyNames, Type[] types) {
        System.out.println("Saving: " + entity);
        return false;
    }
}

// Custom UserType
public class JsonStringType implements UserType {
    // implement nullSafeGet, nullSafeSet, returnedClass, etc.
}
```

* **Interceptors**:

  * Implement `EmptyInterceptor` to log or manipulate behavior.
* **Event Listeners**:

  * Hook into lifecycle events like `onLoad`, `onFlush`, `onSave`.
* **Custom Types**:

  * Implement `UserType` or use `@Type` for mapping custom classes.
* **Multi-Tenancy**:

  * Use discriminator or schema separation for tenant data.
* **Envers**:

  * Add `@Audited` to enable automatic entity versioning and audit history.

### 11. Testing & Best Practices

```java
// Using H2 and Spring Boot
@RunWith(SpringRunner.class)
@DataJpaTest
public class UserRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private UserRepository userRepository;

    @Test
    public void testFindByUsername() {
        User user = new User();
        user.setUsername("john");
        entityManager.persist(user);
        entityManager.flush();

        Optional<User> found = userRepository.findByUsername("john");
        assertTrue(found.isPresent());
    }
}
```

* Use in-memory DB (H2) for integration testing.
* Use Spring Boot’s `@DataJpaTest` for repository testing.
* Avoid returning JPA entities directly to clients (prefer DTOs).
* Prevent LazyInitializationException with Open Session in View pattern (use cautiously).
* Set `hbm2ddl.auto=validate` in production to ensure schema correctness.

---

### 12. Interview Questions

1. **What is the difference between `Session` and `SessionFactory`?**

   * `SessionFactory` is a heavyweight, thread-safe object used to create `Session` instances. `Session` is a lightweight, single-threaded object used to interact with the database.

2. **How does Hibernate handle caching? Explain L1 vs L2 cache.**

   * L1 Cache is session-scoped and always enabled. L2 Cache is session-factory scoped and needs explicit configuration and annotations.

3. **What are the different fetching strategies in Hibernate?**

   * `FetchType.LAZY` and `FetchType.EAGER`. Use `JOIN FETCH`, `@BatchSize`, and `@Fetch` to optimize loading.

4. **Explain cascading and orphan removal with examples.**

   * Cascading propagates operations from parent to child (e.g., persist, merge). Orphan removal deletes child when it's removed from the parent collection.

5. **How is `@OneToMany` different from `@ManyToOne`?**

   * `@OneToMany` is a collection on the parent side and often requires `mappedBy`. `@ManyToOne` is a single-valued association on the child side and usually owns the foreign key.

6. **What is the N+1 Select Problem and how do you resolve it?**

   * Occurs when fetching a list and then lazily loading children in a loop. Fix it using `JOIN FETCH` or batch fetching.

7. **Difference between `get()` and `load()` in Hibernate?**

   * `get()` fetches data immediately and returns null if not found. `load()` returns a proxy and throws `ObjectNotFoundException` if data is not found.

8. **How does optimistic locking differ from pessimistic locking?**

   * Optimistic uses `@Version` and checks versions before commit. Pessimistic uses database-level locks to prevent concurrent access.

9. **What are the benefits of using Hibernate over JDBC?**

   * Auto schema generation, caching, cleaner code, transaction management, and database-agnostic.

10. **How does Hibernate support transaction management?**

* Via programmatic transactions (`Transaction`), JTA, or integration with Spring's `@Transactional`.

11. **Explain `StatelessSession` and when you would use it.**

* It skips L1 cache and lifecycle events. Use it for batch inserts/updates for performance.

12. **What is Envers and how do you enable entity auditing?**

* A Hibernate module for auditing. Annotate entity with `@Audited` and add Envers dependency to track history.

### 13. Flashcards (Quick Recall)

* **Q:** What annotation marks a class as an entity?
  **A:** `@Entity`

* **Q:** What is the default fetch type for `@OneToMany`?
  **A:** `LAZY`

* **Q:** Which cache is enabled by default in Hibernate?
  **A:** First-Level Cache (L1)

* **Q:** What is the use of `@Version`?
  **A:** Enables optimistic locking

* **Q:** Which annotation is used for embedded value types?
  **A:** `@Embeddable`, `@Embedded`

* **Q:** How do you write a native SQL query in Hibernate?
  **A:** `session.createNativeQuery("SELECT * FROM ...")`

* **Q:** Which property controls schema generation?
  **A:** `hibernate.hbm2ddl.auto`

* **Q:** How to enable second-level cache?
  **A:** Annotate entity with `@Cache` and configure cache provider

* **Q:** Which API allows dynamic, type-safe queries?
  **A:** Criteria API

* **Q:** What is `@NamedQuery` used for?
  **A:** Declares reusable HQL queries

> **Study Tips**
>
> 1. Draw session and transaction boundaries to visualize flow.
> 2. Practice HQL vs. Criteria queries with joins and projections.
> 3. Experiment with L1, L2, and query caching in a sample app.
> 4. Intentionally trigger and fix the N+1 problem.
> 5. Map and persist complex object graphs with cascading and orphan removal.

Good luck with your interview prep! Let me know if you'd like flashcards or a cheat sheet next.

> 1. Draw session and transaction boundaries to visualize flow.
> 2. Practice HQL vs. Criteria queries with joins and projections.
> 3. Experiment with L1, L2, and query caching in a sample app.
> 4. Intentionally trigger and fix the N+1 problem.
> 5. Map and persist complex object graphs with cascading and orphan removal.

