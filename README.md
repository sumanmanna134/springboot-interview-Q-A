# Spring Boot and RESTful Web Services Overview

This document provides a comprehensive overview of Spring Boot, RESTful services, and related concepts with code examples and best practices.

---

# Spring Framework and Spring Boot Interview Preparation

## Spring Framework Core Concepts

### 1. What is Loose Coupling?

Loose coupling is a design principle where components are independent and interact with each other through interfaces, ensuring minimal dependencies and changes.

### 2. What is a Dependency?

A dependency is an object that another object requires in order to function correctly. Dependencies are often passed into a class via constructors or methods.

### 3. What is IOC (Inversion of Control)?

IOC is a design principle in which the control of object creation and dependency injection is transferred from the application to a framework.

### 4. What is Dependency Injection?

Dependency Injection is a form of IOC, where an object's dependencies are provided by an external source rather than the object creating them itself.

### 5. Can you give a few examples of Dependency Injection?

- Constructor Injection
- Setter Injection
- Field Injection

### 6. What is Autowiring?

Autowiring is a feature in Spring that allows the automatic injection of dependencies by the framework without explicitly defining them.

### 7. What are the important roles of an IOC Container?

- Manages the lifecycle and dependencies of beans.
- Handles the configuration and wiring of dependencies.

### 8. What are Bean Factory and Application Context?

- **Bean Factory**: It is the simplest container in Spring for managing beans.
- **Application Context**: A more advanced container that provides more features like event propagation, declarative mechanisms, and more.

### 9. Can you compare Bean Factory with Application Context?

Bean Factory is a simpler container, while Application Context includes more functionality like event handling, AOP support, and easier integration with Spring's more complex features.
- Bean Factory Preferred Usage	- Lightweight Applications, it supports LAZY initialization
- Application Context Use for Enterprise Applications , LAZY Initialization - NO

### 10. How do you create an application context with Spring?

You can create it using `ClassPathXmlApplicationContext` for XML configuration or `AnnotationConfigApplicationContext` for annotation-based configuration.
```java
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

```

### 11. How does Spring know where to search for Components or Beans?

Spring uses component scanning to find beans, either through annotations (`@Component`, `@Service`, etc.) or through XML configuration.

### 12. What is a Component Scan?

It is a feature that automatically scans a specified package or location for annotated classes to register as beans.

### 13. How do you define a component scan in XML and Java Configurations?

- **XML**: `<context:component-scan base-package="com.example"/>`
- **Java Config**: `@ComponentScan("com.example")`

### 14. How is it done with Spring Boot?

Spring Boot does component scanning automatically based on the application's base package.

### 15. What does @Component signify?

It marks a class as a Spring bean for component scanning.

### 16. What does @Autowired signify?

It signifies automatic injection of a dependency into a Spring-managed bean.

### 17. What’s the difference Between @Controller, @Component, @Repository, and @Service Annotations in Spring?

- **@Controller**: Used in Spring MVC to mark a class as a controller.
- **@Component**: A generic annotation for marking a bean.
- **@Repository**: Used to mark a class as a data access object (DAO).
- **@Service**: Used to mark a service class.

### 18. What is the default scope of a bean?

The default scope is **Singleton**, meaning only one instance of the bean is created for the Spring container.

### 19. Are Spring beans thread safe?

No, by default Spring beans are not thread-safe.

### 20. What are the other scopes available?

- **Prototype**: A new instance is created every time the bean is requested.
- **Request**: A new bean instance is created for each HTTP request.
- **Session**: A new bean instance is created for each HTTP session.

### 21. How is Spring’s singleton bean different from Gang of Four Singleton Pattern?

Spring's singleton bean scope ensures only one instance per Spring container, while the Gang of Four Singleton pattern ensures a single instance across the entire application.

### 22. What are the different types of dependency injections?

- Constructor Injection
- Setter Injection
- Field Injection

### 23. What is setter injection?

Setter injection is where dependencies are provided through setter methods.

### 24. What is constructor injection?

Constructor injection is where dependencies are provided through constructor parameters.

### 25. How do you choose between setter and constructor injections?

Constructor injection is preferred for mandatory dependencies, while setter injection is used for optional ones.

### 26. What are the different options available to create Application Contexts for Spring?

- `ClassPathXmlApplicationContext`
- `AnnotationConfigApplicationContext`

### 27. What is the difference between XML and Java Configurations for Spring?

XML is declarative configuration, while Java configuration provides more flexibility with annotations and Java-based configuration.

### 28. How do you choose between XML and Java Configurations for Spring?

Java configuration is more flexible and easier to maintain, while XML configuration might be suitable for legacy systems.

### 29. How does Spring do Autowiring?

Spring autowires dependencies based on the type of the bean, matching them automatically.

### 30. What are the different kinds of matching used by Spring for Autowiring?

- By Type
- By Name
- Qualifiers

### 31. How do you debug problems with Spring Framework?

- Use logging frameworks like Logback or Log4j.
- Use the `@Profile` annotation for environment-specific configurations.

### 32. How do you solve NoUniqueBeanDefinitionException?

You can solve it by qualifying the autowire with the `@Qualifier` annotation.

### 33. How do you solve NoSuchBeanDefinitionException?

This is solved by ensuring the required bean is correctly declared and scanned by Spring.

### 34. What is @Primary?

`@Primary` is used to mark a bean as the preferred candidate when multiple beans match the autowiring requirement.

### 35. What is @Qualifier?

`@Qualifier` is used to specify which bean to inject when there are multiple beans of the same type.

### 36. What is CDI (Contexts and Dependency Injection)?

CDI is a specification for dependency injection in Java EE, similar to Spring's DI.

### 37. Does Spring Support CDI?

Yes, Spring supports CDI via annotations like `@Inject`.

### 38. Would you recommend to use CDI or Spring Annotations?

Spring annotations offer more flexibility and integration with the Spring ecosystem, so they are typically preferred over CDI.

### 39. What are the major features in different versions of Spring?

- Spring 4.0 introduced enhanced support for REST and WebSockets.
- Spring 5.0 added reactive programming support with WebFlux.
- Spring 6 focuses on improvements in Spring Boot and Jakarta EE 9 migration.

### 40. What are important Spring Modules?

- Spring Core
- Spring AOP
- Spring Web
- Spring Data
- Spring Security

### 41. What are important Spring Projects?

- Spring Boot
- Spring Cloud
- Spring Batch
- Spring Data

### 42. What is the simplest way of ensuring that we are using a single version of all Spring-related dependencies?

Use Spring Boot's dependency management system to manage all dependencies.

### 43. Name some of the design patterns used in Spring Framework?

- Singleton Pattern
- Factory Pattern
- Proxy Pattern
- Template Method Pattern

### 44. What do you think about Spring Framework?

Spring is a robust, scalable, and flexible framework that supports various programming models, including Java EE, AOP, and cloud-native development.

### 45. Why is Spring Popular?

Spring is popular because it simplifies enterprise application development, supports dependency injection, and offers a wide range of modules for different use cases.

### 46. Can you give a big picture of the Spring Framework?

Spring is a comprehensive framework that provides infrastructure support for developing Java applications, offering features like dependency injection, transaction management, and web development.

## Spring Boot Concepts

### 1. What is Spring Boot?

Spring Boot is a framework that simplifies the setup and development of Spring applications by providing production-ready defaults and reducing the need for boilerplate configuration.

### 2. What are the important goals of Spring Boot?

- Simplify Spring application setup.
- Provide embedded servers for easier deployment.
- Automatically configure Spring components.
- Reduce the need for boilerplate code.

### 3. What are the important features of Spring Boot?

- **Embedded Server**: It includes embedded web servers like Tomcat, Jetty, and Undertow.
- **Auto Configuration**: Automatically configures components based on the classpath.
- **Spring Boot Starters**: Pre-configured sets of dependencies for common tasks.
- **Spring Boot Actuator**: Provides production-ready features like monitoring and health checks.
- **CommandLineRunner**: Enables running code at the application startup.
- **No code generation**: Does not generate any code but rather makes it easier to configure existing code.

### 4. Compare Spring Boot vs Spring Framework

- **Spring Boot**: Provides out-of-the-box configurations, embedded servers, and streamlined project setups.
- **Spring Framework**: Requires more setup, manual configuration, and does not include embedded servers by default.

### 5. Compare Spring Boot vs Spring MVC

- **Spring Boot**: A framework that streamlines Spring application development and simplifies configuration.
- **Spring MVC**: A part of the Spring framework focused on web applications, providing model-view-controller architecture.

### 6. What is the importance of @SpringBootApplication?

The `@SpringBootApplication` annotation is a convenience annotation that combines three key annotations:

- `@Configuration`: Marks the class as a source of bean definitions.
- `@EnableAutoConfiguration`: Enables auto-configuration of Spring beans.
- `@ComponentScan`: Enables component scanning to discover beans.

### 7. What is Auto Configuration?

Auto Configuration is a mechanism that allows Spring Boot to automatically configure your application based on the libraries and classes available in the classpath.

### 8. How can we find more information about Auto Configuration?

You can find more information about Auto Configuration by checking the Spring Boot documentation or by using the `@EnableAutoConfiguration` and `@SpringBootApplication` annotations.

### 9. What is an embedded server? Why is it important?

An embedded server is a web server that is packaged within the application, allowing you to run the application without needing an external server. This makes the deployment process easier and reduces configuration overhead.

### 10. What is the default embedded server with Spring Boot?

The default embedded server is **Tomcat**.

### 11. What are the other embedded servers supported by Spring Boot?

- Jetty
- Undertow

### 12. What are Starter Projects?

Starter Projects are a set of pre-configured Maven/Gradle dependencies that make it easier to get started with Spring Boot applications. They group common dependencies for particular tasks.

### 13. Can you give examples of important starter projects?

- **spring-boot-starter-web**: For building web applications.
- **spring-boot-starter-data-jpa**: For JPA and database integration.
- **spring-boot-starter-security**: For adding security features.
- **spring-boot-starter-test**: For testing support.

### 14. What is Starter Parent?

`spring-boot-starter-parent` is a parent POM file that is designed to simplify the configuration of Spring Boot applications by managing common dependencies and providing default settings.

### 15. What are the different things that are defined in Starter Parent?

- Dependency versions
- Build configuration
- Plugin management

### 16. How does Spring Boot enforce common dependency management for all its Starter projects?

Spring Boot enforces common dependency management by using the `spring-boot-dependencies` BOM (Bill of Materials) in the parent project.

### 17. What is Spring Initializr?

Spring Initializr is a web-based tool to generate Spring Boot projects with your desired configurations (dependencies, packaging, Java version, etc.).

### 18. What is application.properties?

`application.properties` is a configuration file in Spring Boot that allows you to configure application settings, such as database connections, logging, and server settings.

### 19. What are some of the important things that can be customized in application.properties?

- Server port and host
- Database configurations
- Logging settings
- Custom application properties

### 20. How do you externalize configuration using Spring Boot?

Spring Boot supports externalized configuration via the `application.properties` file, environment variables, command-line arguments, or YAML files.

### 21. How can you add custom application properties using Spring Boot?

You can add custom properties by simply adding them in the `application.properties` file and then accessing them using `@Value` or `@ConfigurationProperties`.

### 22. What is @ConfigurationProperties?

`@ConfigurationProperties` is a Spring annotation that allows binding external configuration properties to a Java bean.

### 23. What is a profile?

A profile is a mechanism in Spring to define different configurations for different environments (e.g., dev, prod).

### 24. How do you define beans for a specific profile?

You can define beans for a specific profile using the `@Profile` annotation on a class or method.

### 25. How do you create application configuration for a specific profile?

You can create profile-specific configurations by using properties files like `application-dev.properties` or `application-prod.properties`.

### 26. How do you have different configuration for different environments?

You can use Spring profiles and external configuration files to define different settings for different environments.

### 27. What is Spring Boot Actuator?

Spring Boot Actuator is a set of built-in tools and features for monitoring and managing your Spring Boot application in production.

### 28. How do you monitor web services using Spring Boot Actuator?

Spring Boot Actuator provides endpoints for monitoring, such as `/actuator/health`, `/actuator/metrics`, and `/actuator/info`.

### 29. How do you find more information about your application environment using Spring Boot?

You can use Spring Boot's Actuator `/actuator/env` endpoint to get details about your application environment.

### 30. What is a CommandLineRunner?

`CommandLineRunner` is a Spring Boot interface that can be used to execute code at the startup of the application.

---

## Spring Data Concepts

### 1. What is Spring Data?

Spring Data is a collection of projects in the Spring ecosystem designed to simplify data access and interaction with data sources, such as databases, through various repository abstractions.

### 2. What is the need for Spring Data?

Spring Data eliminates the boilerplate code and provides a higher-level abstraction for data access, making it easier to integrate with databases and reduce the complexity of the data layer.

### 3. What is Spring Data JPA?

Spring Data JPA is a part of Spring Data that provides easy integration with JPA-based data access frameworks. It simplifies working with relational databases using the Java Persistence API (JPA).

### 4. What is a CrudRepository?

`CrudRepository` is an interface in Spring Data that provides CRUD (Create, Read, Update, Delete) operations for entities, with minimal boilerplate code.

### 5. What is a PagingAndSortingRepository?

`PagingAndSortingRepository` is an extension of `CrudRepository` that adds methods for pagination and sorting.

---

Let me know if you'd like me to continue further or adjust the content!

## Spring Data Concepts

### 6. What is Spring Data JPA?

Spring Data JPA simplifies the development of Java applications using the Java Persistence API (JPA). It provides a repository-based approach to interact with relational databases, abstracting away much of the boilerplate code related to data access. It enables you to define repositories and perform operations like saving, deleting, and querying data with minimal effort.

### 7. What is a CrudRepository?

`CrudRepository` is an interface in Spring Data that provides methods to perform basic CRUD operations on an entity. It includes methods like `save()`, `findById()`, `findAll()`, and `deleteById()`.

```java
public interface EmployeeRepository extends CrudRepository<Employee, Long> {
    // You can add custom queries here
}
```

### What is a PagingAndSortingRepository?

- PagingAndSortingRepository is an extension of CrudRepository that provides additional methods for pagination and sorting.

- Example usage:

```java
public interface EmployeeRepository extends PagingAndSortingRepository<Employee, Long> {
}

```

- Common methods in PagingAndSortingRepository:
  - `findAll(Pageable pageable):` Retrieves paginated results.
  - `findAll(Sort sort):` Retrieves all entities sorted by specified fields.

```java
Pageable pageable = PageRequest.of(0, 10, Sort.by("name").ascending());
Page<Employee> employees = employeeRepository.findAll(pageable);

```

`Why Use It?` Enables efficient handling of large datasets by fetching only a subset of data at a time.

### What is Spring Data MongoDB?

Spring Data MongoDB provides easy integration with MongoDB databases, enabling you to work with MongoDB in a Spring-friendly way. It extends the concepts of Spring Data repositories, offering the same repository-based programming model for MongoDB operations.

### What is Spring JDBC? How is it different from JDBC?

Spring JDBC is a module of the Spring Framework that simplifies database access by providing a higher-level abstraction over standard JDBC.

Differences:

- Error Handling: Spring JDBC converts checked exceptions into runtime exceptions.
- Template-Based Programming: Spring JDBC provides JdbcTemplate, which simplifies code by eliminating boilerplate (e.g., opening/closing connections).
- Declarative Transaction Management: Spring supports declarative transactions, unlike standard JDBC.

### What is a RowMapper?

- `RowMapper` is an interface used by JdbcTemplate to map rows of a ResultSet to Java objects.
- Example

  ```java
  public class EmployeeRowMapper implements RowMapper<Employee> {
    @Override
    public Employee mapRow(ResultSet rs, int rowNum) throws SQLException {
        Employee employee = new Employee();
        employee.setId(rs.getInt("id"));
        employee.setName(rs.getString("name"));
        return employee;
    }
  }
  ```



## JPA and Hibernate Concepts

### 1. What is JPA?

JPA (Java Persistence API) is a specification for managing relational data in Java applications. It allows developers to work with relational databases using object-oriented programming concepts, eliminating the need for manual SQL code in most cases.

### 2. What is Hibernate?

Hibernate is an implementation of the JPA specification. It is an Object-Relational Mapping (ORM) framework that provides a framework for mapping Java objects to database tables. Hibernate handles the persistence of Java objects and offers a wide range of functionality for database operations.

### How do you define an entity in JPA?

In JPA, an entity is a Java class that is mapped to a database table. You define an entity using the @Entity annotation, and each instance of the entity corresponds to a row in the table.

Example:

```java
@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String role;

    // Getters and Setters
}
````

### What is an Entity Manager?

The EntityManager is the interface used to interact with the persistence context in JPA. It manages the lifecycle of entity instances, including operations like persisting, updating, and deleting entities.

### 5. What is a Persistence Context?

The persistence context is a set of entities that are managed by an EntityManager. It ensures that entities are loaded into memory in a way that allows changes to be tracked, and it manages their lifecycle.

### 6. How do you map relationships in JPA?

In JPA, relationships between entities can be mapped using annotations like @OneToOne, @OneToMany, @ManyToOne, and @ManyToMany. These annotations define the nature of the relationships between entities.

### What are the different types of relationships in JPA?

One-to-One: Each entity is associated with exactly one other entity.
One-to-Many: One entity is associated with multiple other entities.
Many-to-One: Multiple entities are associated with one other entity.
Many-to-Many: Multiple entities are associated with multiple other entities.

### 8. How do you define One to One Mapping in JPA?

One-to-One mapping can be defined using the @OneToOne annotation. It is used when one entity is associated with exactly one other entity.

Example:

```java
@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    private Address address;

    // Getters and Setters
}
```

### 9. How do you define One to Many Mapping in JPA?

One-to-Many mapping is defined using the @OneToMany annotation. It is used when one entity is associated with multiple other entities.

Example:

```java
@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "department")
    private List<Employee> employees;

    // Getters and Setters
}
```

### 10. How do you define Many to Many Mapping in JPA?

Many-to-Many mapping is defined using the @ManyToMany annotation. It is used when multiple entities are associated with multiple other entities.

Example:

```java
@Entity
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany
    private List<Student> students;

    // Getters and Setters
}
```

## Datasource and Persistence Context Configuration

## 10.How do you define a datasource in a Spring Context?

Using the DataSource bean in Java configuration:

```java
@Bean
public DataSource dataSource() {
    HikariDataSource dataSource = new HikariDataSource();
    dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/testdb");
    dataSource.setUsername("root");
    dataSource.setPassword("password");
    return dataSource;
}
```

### 11. What is the use of persistence.xml?

- Defines persistence units and configurations for JPA.
- Contains details like database URL, provider, and JPA properties.

### 12. How do you configure Entity Manager Factory and Transaction Manager?

Configure LocalContainerEntityManagerFactoryBean for the Entity Manager Factory.
Configure JpaTransactionManager for transaction management.
Example:

```java
@Bean
public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
    LocalContainerEntityManagerFactoryBean factory = new LocalContainerEntityManagerFactoryBean();
    factory.setDataSource(dataSource());
    factory.setPackagesToScan("com.example.entity");
    factory.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
    return factory;
}

@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
    return new JpaTransactionManager(emf);
}
```

## Transaction Management

### 1. How do you define transaction management for Spring – Hibernate integration?

- Add @EnableTransactionManagement in the configuration class.
- Use @Transactional on methods to enable transaction management.
  Example:

```java
@Service
public class EmployeeService {
    @Transactional
    public void saveEmployee(Employee employee) {
        employeeRepository.save(employee);
    }
}
```

## Basics of REST

### What is REST?

REST (Representational State Transfer) is an architectural style for designing networked applications. It relies on stateless communication and uses standard HTTP methods like GET, POST, PUT, DELETE to interact with resources.

### What are the key concepts in designing RESTful API?

- **Resources:** Represent data entities and are identified using URIs.
- **HTTP Methods:** Standard methods to perform actions (GET for read, POST for create, etc.).
- **Statelessness:** Each request should contain all the information needed to process it.
- **Representation:** Data can be represented in different formats like JSON, XML, etc.
- **HATEOAS:** Links within responses guide the client.
- **Content Negotiation:** Enables clients to specify the desired format.
- **Versioning:** Ensures backward compatibility.

### What are the Best Practices of RESTful Services?

- Use nouns for URIs (`/users` instead of `/getUsers`).
- Use standard HTTP methods appropriately.
- Return proper HTTP status codes.
- Keep responses simple and consistent.
- Implement pagination for large datasets.
- Use HATEOAS to include links in responses.
- Secure APIs using authentication and authorization.

---

## Example Methods in Spring REST

### Example Get Resource Method

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }
}
```

### What happens when we return a bean from a Request Mapping Method?

Spring automatically converts the bean to the desired response format (e.g., JSON or XML) using **HttpMessageConverters**.

### What is GetMapping and related methods available in Spring MVC?

- `@GetMapping` is a shortcut for `@RequestMapping(method = RequestMethod.GET)`.
- Related methods: `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`.

### Example Post Resource Method

```java
@PostMapping
public ResponseEntity<User> createUser(@RequestBody User user) {
    User savedUser = userService.saveUser(user);
    return ResponseEntity.status(HttpStatus.CREATED).body(savedUser);
}
```

### Appropriate HTTP Response Status for Resource Creation

`201 Created`.

### Why do we use ResponseEntity in RESTful Service?

It allows greater control over the HTTP response, including the status code, headers, and body.

---

## HATEOAS and Documentation

### What is HATEOAS?

HATEOAS (Hypermedia as the Engine of Application State) is a constraint of REST where the client interacts with the application entirely through hypermedia provided dynamically by application servers.

### Example Response for HATEOAS

```json
{
  "id": 1,
  "name": "John Doe",
  "links": [
    { "rel": "self", "href": "/users/1" },
    { "rel": "orders", "href": "/users/1/orders" }
  ]
}
```

### How to Implement HATEOAS using Spring?

```java
@GetMapping("/{id}")
public EntityModel<User> getUser(@PathVariable Long id) {
    User user = userService.getUserById(id);
    EntityModel<User> resource = EntityModel.of(user);
    resource.add(Link.of("/users/" + id).withSelfRel());
    resource.add(Link.of("/users/" + id + "/orders").withRel("orders"));
    return resource;
}
```

### How to Document RESTful Web Services?

Use tools like **Swagger** or **OpenAPI**.

---

## Content Negotiation and Exception Handling

### What is Content Negotiation?

Content negotiation allows the client to specify the desired representation format using HTTP headers like `Accept`.

### How to Implement Content Negotiation in Spring Boot?

Configure `HttpMessageConverters` to support formats like JSON and XML.

### Example of Exception Handling in Spring REST

```java
@ControllerAdvice
public class CustomExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse("NOT_FOUND", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

---

## Versioning

### Why is Versioning Important?

To ensure backward compatibility and allow evolution of APIs.

### Versioning Options

- **URI Versioning:** `/v1/resource`
- **Query Parameter Versioning:** `/resource?version=1`
- **Header Versioning:** Use custom headers like `X-API-VERSION`.
- **Content Negotiation:** Based on MIME types like `application/vnd.company.resource-v1+json`.

### Example of Implementing Versioning

```java
@RestController
@RequestMapping("/v1/users")
public class UserV1Controller {
    @GetMapping
    public String getUsersV1() {
        return "Version 1";
    }
}

@RestController
@RequestMapping("/v2/users")
public class UserV2Controller {
    @GetMapping
    public String getUsersV2() {
        return "Version 2";
    }
}
```

## Spring Framework and Unit Testing

### 1.How does Spring Framework Make Unit Testing Easy?

Spring simplifies unit testing through:

- Dependency Injection (DI): Enables easy mocking and replacing of dependencies.
- Annotations: Reduces boilerplate code with annotations like @MockBean, @Autowired, etc.
- Test Context Framework: Manages application context specifically for testing, ensuring consistent configuration.
- Integration with Tools: Seamlessly integrates with testing frameworks like JUnit and Mockito.

### 2. What is Mockito?

Mockito is a popular Java mocking framework used for creating and managing mock objects in tests. It allows testing the behavior of a class independently by simulating its dependencies.

### 3. What is your favorite mocking framework?

My favorite is Mockito, because:

It’s easy to use with simple API.
Supports annotations like `@Mock`, `@InjectMocks`, and `@Captor`.
Integrates well with JUnit and Spring.

### 4. How do you do mock data with Mockito?

Using `@Mock` annotation or `Mockito.mock()` to create mock objects:

```java
@Mock
private SomeService someService;

@BeforeEach
void setUp() {
    MockitoAnnotations.openMocks(this);
}

```

Defining behavior using `when` and `thenReturn`

```java
when(someService.getData()).thenReturn(mockData);

```

### 5. What are the different mocking annotations that you worked with?

- `@Mock`: Creates a mock object.
- `@InjectMocks`: Injects mock dependencies into the object under test
- `@Captor`: Captures argument values passed to mocks.
- `@Spy`: Partially mocks a real object.

### 6. What is MockMvc?

MockMvc is a Spring utility for testing Spring MVC controllers. It allows simulating HTTP requests and verifying responses without needing a full web server.

### 7. What is @WebMvcTest?

`@WebMvcTest` is used to test Spring MVC components like controllers in isolation. It configures only the web layer, excluding services or repositories.

Example:

```java
@WebMvcTest(MyController.class)
public class MyControllerTest {
}
```

### 8. What is @MockBean?

`@MockBean` is used in Spring tests to create mock implementations of beans. It replaces the actual bean with a mock in the application context.

### 9. How do you write a unit test with MockMVC?

```java
@WebMvcTest(MyController.class)
public class MyControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @Test
    void testGetEndpoint() throws Exception {
        mockMvc.perform(get("/endpoint"))
               .andExpect(status().isOk())
               .andExpect(content().string("Hello, World!"));
    }
}
```

### 10. hat is JSONAssert?

JSONAssert is a library for comparing JSON strings in unit tests. It allows strict and non-strict comparison.

Example:

```java

JSONAssert.assertEquals(expectedJson, actualJson, false);

```

### 11. How do you write an integration test with Spring Boot?

Use `@SpringBootTest` to load the complete application context.
Use TestRestTemplate to simulate HTTP requests.

Example:

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class MyIntegrationTest {
    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void testEndpoint() {
        ResponseEntity<String> response = restTemplate.getForEntity("/api/test", String.class);
        assertEquals(HttpStatus.OK, response.getStatusCode());
    }
}

```

### 12. What is @SpringBootTest?

`@SpringBootTest` is an annotation used to load the entire Spring application context for testing. It is commonly used for integration tests.

### 13. What is @LocalServerPort?

`@LocalServerPort` is used in tests to inject the randomly assigned port of an embedded web server.

Example:

```java
@LocalServerPort
private int port;

```

### 14. What is TestRestTemplate?

TestRestTemplate is a Spring utility for testing RESTful services. It simplifies HTTP request execution in tests.

## AOP

### 1. What are cross-cutting concerns?

Cross-cutting concerns are functionalities that affect multiple modules of an application but are not specific to any single module. Common examples include logging, security, transaction management, and performance monitoring.

### 2. How do you implement cross cutting concerns in a web application?

- Manually: Adding the functionality directly in multiple places, which is not ideal and leads to code duplication.
- Using Interceptors: Middleware intercepts requests and responses for reusable logic.
- Using AOP (Aspect-Oriented Programming): Implements cross-cutting concerns declaratively without code duplication.

### 3. If you would want to log every request to a web application, what are the options you can think of?

- Servlet Filters: Use filters to log requests.
  Interceptors: Use Spring MVC HandlerInterceptor.
- AOP (Aspects): Use an AOP advice to log requests.
  Middleware: Implement a centralized logging mechanism at the server or application layer.

### 4. If you would want to track performance of every request, what options can you think of?

- AOP: Define an aspect to calculate request execution time.
- Interceptors: Use Spring interceptors to measure request time.
- Filters: Use servlet filters to log start and end times.
- Monitoring Tools: Use tools like Prometheus, Grafana, or Spring Boot Actuator for performance tracking.

### 5. What is an Aspect and Pointcut in AOP?

- Aspect: A modularization of cross-cutting concerns, such as logging or security. It defines the behavior to be applied.
- Pointcut: A predicate that matches specific join points (e.g., method execution) where the aspect is applied.

### 6. What are the different types of AOP advices?

- Before Advice: Runs before the join point.
- After Advice: Runs after the join point, regardless of its outcome.
- After Returning Advice: Runs after the join point only if it completes successfully.
- After Throwing Advice: Runs if the join point throws an exception.
- Around Advice: Wraps around the join point, providing control before and after its execution.

### 7. What is weaving?

Weaving is the process of linking aspects with the target object to create an advised object. It can be done at:

- Compile-time: Aspects are woven into the code during compilation.
- Load-time: Aspects are woven as the classes are loaded into the JVM.
- Runtime: Aspects are woven during the execution of the application.

### 8. Compare Spring AOP vs AspectJ?

- Spring AOP:

  - #### Proxy-based implementation.

    - Supports only method-level join points.

    - Simple and easy to integrate with Spring applications.
    - Weaving is done at runtime.

  - ### AspectJ:
    - Full-fledged AOP framework with support for all join points.
    - Requires a separate AspectJ compiler.
    - Weaving can be done at compile-time, load-time, or runtime.
    - More powerful but more complex to set up compared to Spring AOP.

---
