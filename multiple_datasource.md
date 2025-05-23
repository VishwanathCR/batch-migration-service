To configure **multiple data sources** in a **Spring Boot** application—one for **H2** and another for **DB2**—follow these steps:

### **1. Define DataSource Properties**
In `application.yml` or `application.properties`, specify configurations for both databases:

```yaml
spring:
  datasource:
    h2:
      url: jdbc:h2:mem:testdb
      username: sa
      password:
      driver-class-name: org.h2.Driver
    db2:
      url: jdbc:db2://your-db2-host:50000/yourDB
      username: db2user
      password: db2password
      driver-class-name: com.ibm.db2.jcc.DB2Driver
```

### **2. Create Configuration Classes**
Define separate configuration classes for **H2** and **DB2** data sources.

#### **H2 DataSource Configuration**
```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    basePackages = "com.example.h2.repository",
    entityManagerFactoryRef = "h2EntityManagerFactory",
    transactionManagerRef = "h2TransactionManager"
)
public class H2DataSourceConfig {

    @Primary
    @Bean(name = "h2DataSource")
    @ConfigurationProperties(prefix = "spring.datasource.h2")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "h2EntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            @Qualifier("h2DataSource") DataSource dataSource) {
        return new LocalContainerEntityManagerFactoryBean();
    }

    @Primary
    @Bean(name = "h2TransactionManager")
    public PlatformTransactionManager transactionManager(
            @Qualifier("h2EntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

#### **DB2 DataSource Configuration**
```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    basePackages = "com.example.db2.repository",
    entityManagerFactoryRef = "db2EntityManagerFactory",
    transactionManagerRef = "db2TransactionManager"
)
public class DB2DataSourceConfig {

    @Bean(name = "db2DataSource")
    @ConfigurationProperties(prefix = "spring.datasource.db2")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "db2EntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            @Qualifier("db2DataSource") DataSource dataSource) {
        return new LocalContainerEntityManagerFactoryBean();
    }

    @Bean(name = "db2TransactionManager")
    public PlatformTransactionManager transactionManager(
            @Qualifier("db2EntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

### **3. Define Entity Classes**
Ensure that entities for **H2** and **DB2** are stored in separate packages (`com.example.h2.entity` and `com.example.db2.entity`).

### **4. Use Repositories**
Each repository should be mapped to the correct data source using the appropriate package.

### **Resources**
For more details, check out [this guide](https://www.baeldung.com/spring-boot-configure-multiple-datasources) on configuring multiple data sources in Spring Boot.

Let me know if you need further clarification! 🚀
