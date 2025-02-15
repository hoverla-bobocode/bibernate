# Bibernate

## Java ORM framework

---

## What is Bibernate?

Bibernate is a Java Object-Relational Mapping (ORM) framework which can help you <b>work with a
database in an object-oriented manner</b>

---

## Quick Start

---
To install Bibernate locally in your project you should:

* clone repo ```https://github.com/hoverla-bobocode/Bibernate.git```
* go to the root of Bibernate project ```cd <path_to_bring>/Bibernate```
* build jar with ```mvn clean install -DskipTests```
* add jar to your maven project:

```
<dependency>
    <groupId>com.bobocode.hoverla</groupId>
    <artifactId>bibernate</artifactId>
    <version>1.0-SNAPSHOT</version> 
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.1.214</version>
</dependency>
```

## Supported Databases
* H2
* PostgreSQL

## How to use

There are multiple steps you need to perform to correctly use Bibernate:

1. Create `src/main/resources/persistence.yml` [resource file](#persistenceyml-file-example) with information about your
   database.
2. Create [entities](#entity-declaration-example), that represent table from the database.

* Mark them with the [`@Entity`](src/main/java/com/bobocode/bibernate/annotation/Entity.java) annotation.
* Add field that represents primary key of relevant table and annotated with
  the [`@Id`](src/main/java/com/bobocode/bibernate/annotation/Id.java)
  * (Please Note! we don't support automatic ID generation yet, so you have to manually set up ID to your entity before saving, but if you stay with
  us and donate to us, we promise to improve and develop our product.) * 
3. Create [`SessionFactory`](src/main/java/com/bobocode/bibernate/session/SessionFactory.java) that would represent your
   persistence unit using [`Persistence`](src/main/java/com/bobocode/bibernate/Persistence.java) class
4. Create [`Session`](src/main/java/com/bobocode/bibernate/session/Session.java) object to interact with persistence context
   using [`SessionFactory`](src/main/java/com/bobocode/bibernate/session/SessionFactory.java). Look at
   the [example](#sessionsrcmainjavacombobocodebibernatesessionsessionjava-creation-example) below.

#### `persistence.yml` file example:

```yaml
logLevel: TRACE
persistenceUnit:
  name: default
  dataSource:
    jdbcUrl: jdbc:h2:mem:testdb
    user: sa
    password: password
  dialect: h2
```

#### Entity declaration example:

```java
import com.bobocode.bibernate.annotation.Column;
import com.bobocode.bibernate.annotation.Entity;
import com.bobocode.bibernate.annotation.Id;
import com.bobocode.bibernate.annotation.Table;
import lombok.Data;

// Entity is required to be marked with @Entity
@Entity
public class Product {

    // Entity is required to have public non-arg constructor
    public Product() {
    }

    // Entity is required to have @Id field
    @Id
    private Long id;

    private String name;

    private Double price;

    // getters and setters are omitted for brevity
}
```

#### [Session](src/main/java/com/bobocode/bibernate/session/Session.java) creation example:

```java
import com.bobocode.bibernate.Persistence;
import com.bobocode.bibernate.session.Session;
import com.bobocode.bibernate.session.SessionFactory;

import java.util.Optional;

public class SessionCreationExample {

    public static void main(String[] args) {
        SessionFactory sessionFactory = Persistence.createSessionFactory(
                "default"); // creates session factory from persistence.yml properties
        Session session = sessionFactory.openSession(); // opens Session that corresponds to DB Connection
        try {
            session.begin(); // starts transaction
            Optional<Product> product = session.find(Product.class, 1L); // finds product by ID
            product.ifPresent(p -> p.setName("new name")); // updates product and defers update query execution
            session.commit(); // flushed all deferred actions (only 'update' in this case) and commits transaction 
        } catch (Exception e) {
            session.rollback(); // rollbacks transaction
            throw e;
        } finally {
            session.close(); // under the hood flushes all deferred actions and closes JDBC Connection
        }
    }
}
```
The same construction can be simplified with convenient API to wrap your code in transaction
```java
import com.bobocode.bibernate.Persistence;
import com.bobocode.bibernate.session.Session;
import com.bobocode.bibernate.session.SessionFactory;

import java.util.Optional;

public class SessionCreationExample {

    public static void main(String[] args) {
        SessionFactory sessionFactory = Persistence.createSessionFactory("default");
        // wraps the code in transaction 
        QueryHelper.runWithinTx(sessionFactory, session -> {
            Optional<Product> product = session.find(Product.class, 1L);
            product.ifPresent(p -> p.setName("new name"));
        });
    }
}
```

## Dive into Bibernate

---
Bibernate is inspired by the project called [Hibernate ORM](https://hibernate.org/) -
object/relational mapping (ORM) framework. Although, it does not implement Java
Persistence API (JPA) specification, its API is based on object names of Hibernate.

### Major classes and interfaces structure

---
![](Major%20Classes%20Structure.png)

* [`Persistence`](src/main/java/com/bobocode/bibernate/Persistence.java) class contains a static
  method to obtain [`SessionFactory`](src/main/java/com/bobocode/bibernate/session/SessionFactory.java)
* [`SessionFactory`](src/main/java/com/bobocode/bibernate/session/SessionFactory.java) is a
  factory for [`Session`](src/main/java/com/bobocode/bibernate/session/Session.java).
  It can create and manage multiple Session instances
* [`Session`](src/main/java/com/bobocode/bibernate/session/Session.java)
  manages the persistence operations on [`Entities`](src/main/java/com/bobocode/bibernate/annotation/Entity.java)
* [`Entity`](src/main/java/com/bobocode/bibernate/annotation/Entity.java) is a persistent object.
  It corresponds to records inside a database table.
* [`Transaction`](src/main/java/com/bobocode/bibernate/transaction/Transaction.java) maintains
  operations for each Session. It can rather perform `commit` or `rollback` operations.

### What is persistence unit

---
A persistence unit defines a set of all `Entity` classes that are managed
by `Session` instances in an application.
This set of entity classes represents the data contained within a single data
store.

Persistence units are defined by the `persistence.yml` configuration file. Here is an example `persistence.yml` file:

```yaml
persistenceUnit:
  name: mysql
  dataSource:
    jdbcUrl: jdbc:mysql://127.0.0.1:3306/test_database
    user: root
    password: root
  dialect: h2
```

### What is an entity

---

Entity is a class that represents table in the database. Each entity instance corresponds to a row in that table.
Entity fields or properties represent columns of the related table.
Class is defined as entity if it's annotated with `@Entity` and has field annotated with `@Id`, which represents primary
key. Also, an entity is required to have a public no-arg constructor and standard setters/getters

You can use `@Table` annotation to specify table name. This annotation is optional. The lowercase class name is used
as table name if annotation is omitted.

```java
import com.bobocode.bibernate.annotation.Entity;
import com.bobocode.bibernate.annotation.Id;
import com.bobocode.bibernate.annotation.Table;

@Entity
@Table(value = "products")
public class Product {

    public Product() {
    }

    @Id
    private Long id;

    public void setId(Long id) {
        this.id = id;
    }

    public Long getId() {
        return id;
    }
}
```

You can use `@Column` annotation to specify column name. This annotation is optional. The lowercase field name is used
as column name if annotation is omitted.

```java
import com.bobocode.bibernate.annotation.Column;
import com.bobocode.bibernate.annotation.Entity;
import com.bobocode.bibernate.annotation.Id;
import com.bobocode.bibernate.annotation.Table;

@Entity
@Table("products")
public class Product {

    public Product() {
    }
    
    @Id
    private Long id;

    @Column("name")
    private String productName;

    public void setId(Long id) {
        this.id = id;
    }

    public Long getId() {
        return id;
    }

    public void setProductName(String productName) {
        this.productName = productName;
    }

    public String getProductName() {
        return productName;
    }
}
```

Every entity naturally has a lifecycle within the framework – it's either in a transient, persistent (or managed) or
detached state.

* **Transient entity** has neither any representation in the datastore nor in the current `Session`.
  A transient entity is simply a POJO without any identifier.
* **Persistent entity** exist in the database, and persistent context tracks all the changes done
  on the persistent entity by the client code. A persistent entity is mapped to a specific database row,
  identified by the ID field. `Session` is responsible for tracking all changes done to a managed entity
  and propagating these changes to database.
* **Detached entity** has a representation in the database, but it is not managed by the `Session`.
  Any changes to a detached entity will not be reflected in the database, and vice-versa.

### What is [Session](src/main/java/com/bobocode/bibernate/session/Session.java)

[`Session`](src/main/java/com/bobocode/bibernate/session/Session.java) offers create, read and delete operations for
instances of mapped entity classes. The lifecycle of a `Session`is bounded by the beginning and end of a
logical [`Transaction`](src/main/java/com/bobocode/bibernate/transaction/Transaction.java)

Main [`Session`](src/main/java/com/bobocode/bibernate/session/Session.java) methods definition:

| Method                    |                                                                     Description                                                                      |
|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| `find(class, primaryKey)` |                            find an entity by primary key<br/>the returned entity will be contained in a persistent entity                            |
| `save(entity)`            |                            save an entity into the database <br/>the entity state is changed from transient to persistent                            |
| `delete(entity)`          |                            remove an entity from the database <br/>the entity state is changed from persistent to removed                            |
| `merge(entity)`           | copy the state of the given object onto the persistent object with the same identifier. <br/>the entity state is changed from detached to persistent |
| `detach(entity)`          |                                                remove a provided entity from the persistence context                                                 |
| `begin`                   |                                                             start a resource transaction                                                             |
| `commit`                  |                                commit the current resource transaction, writing any unflushed changes to the database                                |
| `rollbalck`               |                                                      roll back the current resource transaction                                                      |

## Supported Date types:
| Java type     | JDBC type       |
|---------------|-----------------|
| LocalDate     | Date            |
| LocalDateTime | Timestamp       |
| LocalTime     | Timestamp       |
| ZonedDateTime | Zoned Timestamp |


