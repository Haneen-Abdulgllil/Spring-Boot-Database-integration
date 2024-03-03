## Spring Boot + Database integration

Before you start download and set up the base project and make sure it builds and runs successfully.

### 1. Add required maven dependencies

Add the following as a maven dependency (under dependencies tag)

i. `spring-boot-starter-data-jpa` reduces the code required for querying the database.

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

ii. We will use a postgres database, so we will need java libraries for interfacing our java application
to postgres.

```xml

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

iii. We need an **O**bject/**R**elational **M**apper (ORM) that converts from/to java objects to relational rows.
See more details on ORM in [FAQ#1](FAQS).

```xml

<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
</dependency>
```

iv. For writing **integration tests** required to verify interactions between our service and the database,
we need 2 dependencies, `spring-boot-starter-test` which simplifies writing integration tests involving spring and `h2`
which is an in-memory database that does not require any further setup.

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```xml

<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

### 2. Spring datasource configuration

We need to instruct and configure Spring on to how to connect to the database,

- is it on our localhost?
- what is the name of the database?
- credentials (username/password)
- which engine to use? postgres, mysql, etc

This is done through the `application.yaml` file, found under `src/main/resources`

Inline the following into your `application.yaml`.

Make sure you follow the indentation/tabs correctly.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/currency_exchange_dev
    username: postgres
    password: postgres
  jpa.hibernate.ddl-auto: create
```

We use the `datasource.url` to specify where to find the database, the
uri `jdbc:postgresql://localhost:5432/currency_exchange_dev`
specifies multiple information.

1. **jdbc**: `Java Database Connectivity` is the java API for SQL.
2. **postgresql**: specifies that the connection is to a postgres database.
3. **localhost:5432**: the address (IP+port) to find out database, 5432 is default postgres port.
4. **currency_exchange_dev**: the name of the database we need to access, **this has to be already created**.

`datasource.username/password` specifies the credentials to access the database, make sure you have those matching your
setup.

`jpa.hibernate.ddl-auto` instructs the behaviour of jpa/hibernate to create the tables on initialization of the
application.
In other words it will detect how our java classes are related and create the database table and relations for us, so we
do not write any SQL statements.
We use **create** here for simplicity, but there are other cases more suitable for production, see [FAQ](FAQS).

### 3. Checkpoint 1

Let's make sure everything is working, if you just run your application now.
First let's get a clean build of our project.
Also make sure your postgres instance is up and running.

```shell
mvn clean install
```

Then let's try to run our application either through IDE's run button (green triangle).
Or you can use the command line

```shell
mvn spring-boot:run
```

If everything is working fine the application should start normally without any errors and see in the logs
> [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT
> mode.
>
> [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 7 ms. Found 0
> JPA repository interfaces.
>
> [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 6.4.1.Final
>
> [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence
> unit 'default'

At this point you can proceed to step 4 unless you have failures.

If it fails to start with the
error `org.postgresql.util.PSQLException: FATAL: database "currency_exchange_dev" does not exist
` that means you have not created the database so make sure to do so.
If you're using docker, login into the docker container.
Here my container is called `postgres-container` and my username is `postgres` so match this with your setup.

```shell
docker exec -it postgres-container psql -U postgres
```

then inside the container

```shell
create database currency_exchange_dev;
```

### 4. Build our entity

An entity represents a java object that we want to be mappable to the database.

In our setup we want to have a class `ExchangeRateEntity` which stores the information for conversion.

i. Create a new package in your project let's call it `repository`

ii. Create a new class within this package called `ExchangeRateEntity`

```java
public final class ExchangeRateEntity {
}
```

iii. Annotate the class with `@Entity` which specifies to jpa/hibernate that this class is to be represented in the
database. Also with `@Table(name = "exchange_rates")` which is specifies which table to connect to here we call it
`exchange rates`

```java

@Entity
@Table(name = "exchange_rates")
public final class ExchangeRateEntity {
}
```

iv. Let's add our fields while trying our best to honour immutablity by copying mutable data passed into the class.

```java

@Entity
@Table(name = "exchange_rates")
public final class ExchangeRateEntity {
    private final ZonedDateTime timeLastUpdateUTC;
    private final String sourceCurrency;
    private final Map<String, Double> conversionRates;

    public ExchangeRateEntity(ZonedDateTime timeLastUpdateUnix,
                              String sourceCurrency,
                              Map<String, Double> conversionRates) {
        this.timeLastUpdateUnix = ZonedDateTime.from(timeLastUpdateUnix);
        this.sourceCurrency = sourceCurrency;
        this.conversionRates = Map.copyOf(conversionRates);
    }

    //... Add getters, equals and hashcode
}
```

v. Add a no args constructor, which is required by jpa/hibernate as it uses Proxy and reflection to build perform the
mapping from the
database. [External link for more details](https://www.baeldung.com/jpa-no-argument-constructor-entity-class)

```java

@Entity
@Table(name = "exchange_rates")
public final class ExchangeRateEntity {
    //... Fields declared previously

    ExchangeRateEntity() {
        this(ZonedDateTime.now(), null, Collections.emptyMap());
    }

    //... Constructors & other methods declared previously
}
```

vi. In databases we often need Ids in the form of an auto-incremental number that is added upon each insert.
We could do that using the annotations as follows

```java

@Entity
@Table(name = "exchange_rates")
public final class ExchangeRateEntity {

    @Id
    @GeneratedValue
    private long id;
//... Fields declared previously
//... Constructors & other methods declared previously    
}

```

v. The field `Map<String, Double> conversionRates;` is special in how it would be represented in the database as it will
be a separate table. Therefore, we need to give some instructions on how to map it.
Add the following above the field.

```java

@Entity
@Table(name = "exchange_rates")
public final class ExchangeRateEntity {
    //... Fields declared previously
    @ElementCollection
    @CollectionTable(name = "exchange_rate_mapping",
            joinColumns = {@JoinColumn(name = "exchange_rate_record_id", referencedColumnName = "id")})
    @MapKeyColumn(name = "target_currency")
    @Column(name = "rate")
    private final Map<String, Double> conversionRates;

    //... Constructors & other methods declared previously   
}
```

- `` @ElementCollection`` specifies that we're having a collection, here it is a Map but can be used with List or Set,
  etc.
- ```@CollectionTable(name = "exchange_rate_mapping",
            joinColumns = {@JoinColumn(name = "exchange_rate_record_id", referencedColumnName = "id")})
  ```
  Specifies that we will model this collection as a table called `exchange_rate_mapping`, that is joint to the original
  table `exchange_rates` on the `id` field with the foreign key `exchange_rate_record_id`. So every time we will query
  for an `ExchangeRateEntity` a join would be performed to bring the nested `conversionRates`.
- ```@MapKeyColumn(name = "target_currency") @Column(name = "rate")``` as we mentioned we're dealing with a Map, so
  these annotations specify that the Map key is going to be a column called `target_currency` and the value is a column
  called `rate`.

### 5. Checkpoint 2

Let's make sure everything is working, if you just run your application now.
First let's get a clean build of our project.
Also make sure your postgres instance is up and running.

```shell
mvn clean install
```

Then let's try to run our application either through IDE's run button (green triangle).
Or you can use the command line

```shell
mvn spring-boot:run
```

If the application starts with no errors then we should start seeing the tables created for us.

**Verify Table creations on your postgres**

I use PGAdmin as a UI interface, I navigate to `currency_exchange_dev>schemas>public>Tables`

I see already 2 tables created with the relation matching exactly the naming we specified.
![](DB-SC.png)

### 6. Add a repository interface

We need an interface to query and insert into the database from java.
Spring data + JPA simplifies this a lot such that we'd write minimum code by providing the following levels of
interfaces

1. [CRUDRepository](https://docs.spring.io/spring-data/data-commons/docs/2.7.9/api/org/springframework/data/repository/CrudRepository.html)
   an interface from Spring Data providing basic **C**reate **R**ead **U**pdate **D**elete (CRUD) functions.
    1. save/saveAll to create or update 1 or more objects
    2. findById/findAll,ect to read
    3. delete/deleteAll to delete 1 or more objects
2. [PagingAndSortingRepository](https://docs.spring.io/spring-data/data-commons/docs/2.7.9/api/org/springframework/data/repository/PagingAndSortingRepository.html)
   an extension of **CRUDRepository** providing extra functionality for pagination, useful when we don't want to load
   all the data at once.
3. [JpaRepository](https://docs.spring.io/spring-data/jpa/docs/2.7.9/api/org/springframework/data/jpa/repository/JpaRepository.html)
   a more advanced interface incorporating **PagingAndSortingRepository**, therefore also **CRUDRepository** with extra
   functionalities for flushing. Flushing ensure that the data is saved to the database, while save would cache
   internally and insert to the database later.

It's often the recommendation to use JpaRepository as we would see together.

Create a new **interface** `ExchangeRateEntityRepository` within the repository package.

Add the `@Repository` annotation which serves as a Spring stereotype similar to others `@Service` `@Component`

We extend the JPARepository as mentioned it depends on 2 class type arguments the first is the type of object
we will be querying here it's `ExchangeRateEntity` and the second argument is the type of the Id column and our `id`
had a long type.

```java

@Repository
public interface ExchangeRateEntityRepository extends JpaRepository<ExchangeRateEntity, Long> {
}
```

### 7. Writing our first query

As we said JPA simplifies querying a lot to the extent one won't write any code.

In our example we want to find the latest record given a source currency.
In SQL terms it'd be something like

```sql
SELECT *
FROM exchange_rates rates join exchange_rate_mapping mappings on
rates.id = mappings.exchange_rate_record_id
WHERE rates.source_currency = 'USD'
ORDER BY rates.time_last_update_unix DESC
LIMIT 1
```

In java with JPA it's just adding the following to our interface

```java

@Repository
public interface ExchangeRateEntityRepository extends JpaRepository<ExchangeRateEntity, Long> {

    Optional<ExchangeRateEntity> findFirstBySourceCurrencyOrderByTimeLastUpdateUnixDesc(String sourceCurrency);

}
```

This uses the JPQL notation some notes on how this works

1. We return an `Optional<ExchangeRateEntity>` in case the database returns null, for example if the database is empty.
2. Method starts with `find` which is like a get or select
3. `First` means find the first element, which is similar to LIMIT 1
4. `BySourceCurrency` means that we will pass the source currency to select with, similar
   to `WHERE rates.source_currency = 'USD'`
5. `OrderByTimeLastUpdateUnixDesc` is to sort by the time last update field desc.

Overall it read like
> Find the elements with given source_currency, sort them by time_last_update_unix in descending order
>
> get the first element or return empty.

JPA would implement and translate this function to SQL.

### 8. Integration test

We need an integration test to verify that our code will function as expected.
Let's add a test, if you're in the `ExchangeRateEntityRepository` we created just click
`⇧⌘T (macOS) / Ctrl+Shift+T (Windows/Linux)` to create a new test class.

1. Since we agreed it is an integration test, let's rename it to be suffixed with IT or `ExchangeRateEntityRepositoryIT`
2. Setup the test class

```java

@ExtendWith(SpringExtension.class)
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class ExchangeRateEntityRepositoryIT {

}
```

`@ExtendWith(SpringExtension.class)` Signals to Junit to interact with Spring, which is required for an integration
test.
`@DataJpaTest` and `@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)`Marks the test as a
database test which would use test configuration for testing.

3. In order to test the repository we need to inject it into the test.

````java
class ExchangeRateEntityRepositoryIT {
    @Autowired
    private ExchangeRateEntityRepository rateRepository;
}
````

4. Add a simple test to check if it returns an empty optional if nothing is in database

```java

@Test
void testReturnsEmptyForNonPersistedRates() {
    Optional<ExchangeRateEntity> test =
            rateRepository.findFirstBySourceCurrencyOrderByTimeLastUpdateUnixDesc("EUR");

    assertThat(test).isEmpty();
}
```

* Notice the queries in console log

5. Add another test to check that if we have 2 instances of the same currency when we call
   `findFirstBySourceCurrencyOrderByTimeLastUpdateUnixDesc` it always returns the latest. **TIP** you can save into the
   database using `rateRepository.save`

### Exercise

1. Add another method to `ExchangeRateEntityRepository` that finds all `ExchangeRateEntity` with a given source currency
   and between 2 given timestamps. Use an integration test to verify.
2. Wire the repository in the service to act as a cache. Can we use a design pattern?

### FAQs

1. What is an **O**bject/**R**elational **M**apper?
   An **O**bject/**R**elational **M**apper ORM converts from/to java objects to relational rows.
   For example assume the following java classes

```java
public class Person {
    private String firstName;
    private String lastName;
    private Address addresses;
}

public class Address {
    private String street;
    private int number;
    private int zipcode;
    private String city;
}
```

That could be represented in a relational database as 2 tables.

**Person**

| id | first_name | last_name |
|----|------------|-----------|
| 1  | John       | Doe       |

**Addresses**

| person_id | street      | number | zipcode | city        |
|-----------|-------------|--------|---------|-------------|
| 1         | Thiel Burgs | 2246   | 73728   | Port Marian |

* Notice in the java (object oriented) a Person has an Address which translates into a join relationship.
* Basically the ORM handles the conversion between both representations.


2. What other options are there for `jpa.hibernate.ddl-auto`?
    1. **none**
    2. **validate** option won't make any changes to your database. It will validate that your entity models match the
       existing schema. Use this in production environments to ensure schema consistency without any automatic
       modifications
    3. **update** option dynamically updates your schema based on your entity models. It's perfect for
       development and testing phases when you frequently modify your entity classes. This approach ensures that your
       schema stays in sync with your code without losing existing data.
    4. **create** will drop and recreate the schema every time your application starts. This is ideal for a clean slate
       during development and testing. However, use caution in production as it can result in data loss
    5. **create-drop** (default)  This option combines the power of create and drop. It creates a new schema on
       application startup and drops it when the application shuts down. It's useful for integration testing but, like
       create, should be used cautiously in production.
3. Can we reverse engineer, as in create Java classes for entities from an existing database? </br>Yes, there are
   plugins to help with that or an ide can support it,
   see [intellij](https://www.jetbrains.com/help/idea/persistence-tool-window.html#jump-to-source)
4. What other technologies are useful for
   production? </br> [Flyway](https://www.baeldung.com/database-migrations-with-flyway) is crucial to simplify database
   migrations, as in adding tables, columns or dropping them.  