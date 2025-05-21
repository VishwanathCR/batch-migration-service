# In-Memory Database Implementation for Spring Batch 5.2 Metadata

When implementing an in-memory database for Spring Batch metadata tables in a production-ready application with multiple environments, you need to consider several factors. Here's a comprehensive approach:

## Recommended Implementation Options

### 1. H2 Database (Recommended for non-production environments)

```java
@Configuration
@Profile({"dev", "sit"}) // Only active in dev and sit environments
public class H2BatchConfig {

    @Bean
    public DataSource batchDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:org/springframework/batch/core/schema-h2.sql")
            .generateUniqueName(true)
            .build();
    }
}
```

### 2. Production Configuration (e.g., PostgreSQL, MySQL)

```java
@Configuration
@Profile("prod")
public class ProductionBatchConfig {

    @Bean
    @ConfigurationProperties(prefix="spring.datasource")
    public DataSource batchDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

## Multi-Environment Setup

### application-dev.properties
```properties
spring.batch.jdbc.initialize-schema=always
spring.datasource.url=jdbc:h2:mem:batchdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
```

### application-sit.properties
```properties
spring.batch.jdbc.initialize-schema=always
spring.datasource.url=jdbc:h2:mem:batchdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
```

### application-prod.properties
```properties
spring.batch.jdbc.initialize-schema=never
spring.datasource.url=jdbc:postgresql://prod-db:5432/batchdb
spring.datasource.driver-class-name=org.postgresql.Driver
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

## Jenkins Pipeline Considerations

```groovy
pipeline {
    agent any
    
    environment {
        // Common environment variables
        SPRING_PROFILES_ACTIVE = ''
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Dev Deployment') {
            when {
                branch 'dev'
            }
            environment {
                SPRING_PROFILES_ACTIVE = 'dev'
            }
            steps {
                sh 'java -jar -Dspring.profiles.active=dev your-application.jar'
            }
        }
        
        stage('SIT Deployment') {
            when {
                branch 'sit'
            }
            environment {
                SPRING_PROFILES_ACTIVE = 'sit'
            }
            steps {
                sh 'java -jar -Dspring.profiles.active=sit your-application.jar'
            }
        }
        
        stage('Prod Deployment') {
            when {
                branch 'main'
            }
            environment {
                SPRING_PROFILES_ACTIVE = 'prod'
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'prod-db-creds', 
                               passwordVariable: 'DB_PASSWORD', 
                               usernameVariable: 'DB_USERNAME')]) {
                    sh 'java -jar -Dspring.profiles.active=prod your-application.jar'
                }
            }
        }
    }
}
```

## Best Practices

1. **Schema Initialization**:
   - Use `spring.batch.jdbc.initialize-schema=always` for in-memory databases
   - Use `spring.batch.jdbc.initialize-schema=never` for production

2. **Transaction Management**:
   ```java
   @Configuration
   @EnableBatchProcessing
   public class BatchConfiguration {
       @Bean
       public PlatformTransactionManager transactionManager(DataSource dataSource) {
           return new DataSourceTransactionManager(dataSource);
       }
   }
   ```

3. **Table Prefixes** (if needed):
   ```properties
   spring.batch.jdbc.table-prefix=BATCH_
   ```

4. **Monitoring**:
   - For H2, enable console in dev:
     ```properties
     spring.h2.console.enabled=true
     spring.h2.console.path=/h2-console
     ```

5. **Job Repository Configuration**:
   ```java
   @Bean
   public JobRepository jobRepository(DataSource dataSource, PlatformTransactionManager transactionManager) 
       throws Exception {
       return new JdbcJobRepositoryBuilder()
           .dataSource(dataSource)
           .transactionManager(transactionManager)
           .databaseType("H2") // or appropriate for your environment
           .build();
   }
   ```

This setup provides a flexible configuration that uses in-memory databases for development and testing environments while maintaining a production-grade database for your live environment, all managed through your Jenkins pipeline.
