# Adding Multiple Data Sources in Spring Boot (H2 and DB2)

To configure multiple data sources in a Spring Boot application (one for H2 and another for DB2), you'll need to create separate configuration classes for each datasource. Here's how to do it:

## 1. Add Dependencies to pom.xml

First, ensure you have the required dependencies:

```xml
<dependencies>
    <!-- Spring Boot Starter JDBC -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    
    <!-- H2 Database -->
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- DB2 JDBC Driver -->
    <dependency>
        <groupId>com.ibm.db2</groupId>
        <artifactId>jcc</artifactId>
        <version>11.5.8.0</version> <!-- Use appropriate version -->
    </dependency>
    
    <!-- Spring Boot Configuration Processor -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

## 2. Configure application.properties

Add configuration for both datasources:

```properties
# H2 Datasource Configuration
spring.datasource.h2.url=jdbc:h2:mem:testdb
spring.datasource.h2.username=sa
spring.datasource.h2.password=
spring.datasource.h2.driver-class-name=org.h2.Driver

# DB2 Datasource Configuration
spring.datasource.db2.url=jdbc:db2://your-db2-server:50000/your-database
spring.datasource.db2.username=db2user
spring.datasource.db2.password=db2password
spring.datasource.db2.driver-class-name=com.ibm.db2.jcc.DB2Driver
```

## 3. Create Configuration Classes

### H2 Datasource Configuration

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    basePackages = "com.yourpackage.repository.h2",
    entityManagerFactoryRef = "h2EntityManagerFactory",
    transactionManagerRef = "h2TransactionManager"
)
public class H2DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.h2")
    public DataSource h2DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean h2EntityManagerFactory(
            EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(h2DataSource())
                .packages("com.yourpackage.model.h2")
                .persistenceUnit("h2")
                .build();
    }

    @Bean
    public PlatformTransactionManager h2TransactionManager(
            @Qualifier("h2EntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

### DB2 Datasource Configuration

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    basePackages = "com.yourpackage.repository.db2",
    entityManagerFactoryRef = "db2EntityManagerFactory",
    transactionManagerRef = "db2TransactionManager"
)
public class DB2DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.db2")
    public DataSource db2DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean db2EntityManagerFactory(
            EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(db2DataSource())
                .packages("com.yourpackage.model.db2")
                .persistenceUnit("db2")
                .build();
    }

    @Bean
    public PlatformTransactionManager db2TransactionManager(
            @Qualifier("db2EntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

## 4. Pure JDBC Configuration (Alternative)

If you prefer using JDBC without JPA, here's an alternative approach:

```java
@Configuration
public class JdbcDataSourcesConfig {

    // H2 Datasource
    @Bean(name = "h2DataSource")
    @ConfigurationProperties(prefix = "spring.datasource.h2")
    public DataSource h2DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "h2JdbcTemplate")
    public JdbcTemplate h2JdbcTemplate(@Qualifier("h2DataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    // DB2 Datasource
    @Bean(name = "db2DataSource")
    @ConfigurationProperties(prefix = "spring.datasource.db2")
    public DataSource db2DataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "db2JdbcTemplate")
    public JdbcTemplate db2JdbcTemplate(@Qualifier("db2DataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

## 5. Using the DataSources

### With JPA Repositories

```java
// For H2
@Repository
public interface H2EntityRepository extends JpaRepository<H2Entity, Long> {
    // repository methods
}

// For DB2
@Repository
public interface DB2EntityRepository extends JpaRepository<DB2Entity, Long> {
    // repository methods
}
```

### With JDBC Templates

```java
@Service
public class SomeService {
    
    private final JdbcTemplate h2JdbcTemplate;
    private final JdbcTemplate db2JdbcTemplate;
    
    public SomeService(
            @Qualifier("h2JdbcTemplate") JdbcTemplate h2JdbcTemplate,
            @Qualifier("db2JdbcTemplate") JdbcTemplate db2JdbcTemplate) {
        this.h2JdbcTemplate = h2JdbcTemplate;
        this.db2JdbcTemplate = db2JdbcTemplate;
    }
    
    public void someMethod() {
        // Use h2JdbcTemplate for H2 operations
        h2JdbcTemplate.query("SELECT * FROM h2_table", ...);
        
        // Use db2JdbcTemplate for DB2 operations
        db2JdbcTemplate.query("SELECT * FROM db2_table", ...);
    }
}
```

## Important Notes

1. Make sure to place your entity classes and repositories in the correct packages as specified in the configuration (`com.yourpackage.model.h2`, `com.yourpackage.repository.h2`, etc.)

2. For DB2, you might need additional connection properties in your configuration for optimal performance.

3. If you're using Spring Boot 2.x or later, the configuration is slightly different from Spring Boot 1.x.

4. For production environments, consider using connection pooling (like HikariCP) by adding the dependency and configuring pool properties.

This configuration provides a clean separation between the two datasources while allowing you to use either JPA or plain JDBC as needed.
