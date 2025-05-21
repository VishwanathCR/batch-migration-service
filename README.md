# batch-migration-service
DB2CSV batch migration service

# Production-Ready Spring Batch 5.2 Implementation for Multiple Migration Jobs

This implementation guide covers a robust Spring Batch 5.2 solution for migrating data from DB2 schema tables to CSV files across multiple environments (dev, stage, prod) with Jenkins pipeline integration.

## Architecture Overview

```
DB2 Database → Spring Batch Job → CSV Files
            |
            v
Environment-specific configuration (dev/stage/prod)
```

## Implementation Steps

### 1. Project Structure

```
src/
├── main/
│   ├── java/com/yourcompany/batch/
│   │   ├── config/
│   │   │   ├── BatchConfig.java          # Main batch configuration
│   │   │   ├── DataSourceConfig.java     # DB2 datasource configuration
│   │   │   ├── JobConfig.java            # Job definitions
│   │   │   └── EnvironmentConfig.java    # Environment-specific settings
│   │   ├── listener/
│   │   ├── model/                        # Domain objects
│   │   ├── processor/
│   │   ├── reader/
│   │   ├── writer/
│   │   └── BatchApplication.java        # Main application class
│   └── resources/
│       ├── application-dev.properties
│       ├── application-stage.properties
│       ├── application-prod.properties
│       └── application.properties        # Common properties
├── test/                                 # Test classes
└── Jenkinsfile                           # Pipeline definition
```

### 2. Core Configuration

**BatchConfig.java**
```java
@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public JobBuilderFactory jobBuilderFactory(JobRepository jobRepository) {
        return new JobBuilderFactory(jobRepository);
    }

    @Bean
    public StepBuilderFactory stepBuilderFactory(JobRepository jobRepository, 
                                               PlatformTransactionManager transactionManager) {
        return new StepBuilderFactory(jobRepository, transactionManager);
    }

    @Bean
    public JobLauncher jobLauncher(JobRepository jobRepository) throws Exception {
        SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
        jobLauncher.setJobRepository(jobRepository);
        jobLauncher.setTaskExecutor(new SimpleAsyncTaskExecutor());
        jobLauncher.afterPropertiesSet();
        return jobLauncher;
    }
}
```

**DataSourceConfig.java**
```java
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

### 3. Job Configuration

**JobConfig.java**
```java
@Configuration
public class JobConfig {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Value("${chunk.size:100}")
    private int chunkSize;

    @Bean
    public Job customerMigrationJob(Step customerMigrationStep) {
        return jobBuilderFactory.get("customerMigrationJob")
                .incrementer(new RunIdIncrementer())
                .start(customerMigrationStep)
                .build();
    }

    @Bean
    public Step customerMigrationStep(ItemReader<Customer> reader,
                                    ItemProcessor<Customer, Customer> processor,
                                    ItemWriter<Customer> writer) {
        return stepBuilderFactory.get("customerMigrationStep")
                .<Customer, Customer>chunk(chunkSize)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    // Additional job configurations for other tables
}
```

### 4. Reader, Processor, and Writer Components

**DB2ItemReader.java**
```java
@Configuration
public class DB2ItemReader {

    @Bean
    @StepScope
    public JdbcCursorItemReader<Customer> customerReader(
            DataSource dataSource,
            @Value("#{jobParameters[schema]}") String schema) {
        return new JdbcCursorItemReaderBuilder<Customer>()
                .name("customerReader")
                .dataSource(dataSource)
                .sql("SELECT * FROM " + schema + ".CUSTOMERS")
                .rowMapper(new BeanPropertyRowMapper<>(Customer.class))
                .saveState(false)
                .build();
    }
}
```

**CustomerItemProcessor.java**
```java
public class CustomerItemProcessor implements ItemProcessor<Customer, Customer> {

    @Override
    public Customer process(Customer customer) throws Exception {
        // Add any transformation logic here
        return customer;
    }
}
```

**CSVItemWriter.java**
```java
@Configuration
public class CSVItemWriter {

    @Bean
    @StepScope
    public FlatFileItemWriter<Customer> customerWriter(
            @Value("#{jobParameters[outputFile]}") String outputFile) {
        
        return new FlatFileItemWriterBuilder<Customer>()
                .name("customerWriter")
                .resource(new FileSystemResource(outputFile))
                .delimited()
                .delimiter(",")
                .names(new String[]{"id", "name", "email", "createdDate"})
                .headerCallback(writer -> writer.write("ID,Name,Email,CreatedDate"))
                .build();
    }
}
```

### 5. Environment Configuration

**application-dev.properties**
```properties
spring.datasource.url=jdbc:db2://dev-db2-server:50000/DEVDB
spring.datasource.username=dev_user
spring.datasource.password=dev_password
spring.datasource.driver-class-name=com.ibm.db2.jcc.DB2Driver

batch.default.schema=DEV_SCHEMA
batch.output.dir=/data/dev/output
```

**application-stage.properties**
```properties
spring.datasource.url=jdbc:db2://stage-db2-server:50000/STAGEDB
spring.datasource.username=stage_user
spring.datasource.password=stage_password
spring.datasource.driver-class-name=com.ibm.db2.jcc.DB2Driver

batch.default.schema=STAGE_SCHEMA
batch.output.dir=/data/stage/output
```

**application-prod.properties**
```properties
spring.datasource.url=jdbc:db2://prod-db2-server:50000/PRODDB
spring.datasource.username=prod_user
spring.datasource.password=prod_password
spring.datasource.driver-class-name=com.ibm.db2.jcc.DB2Driver

batch.default.schema=PROD_SCHEMA
batch.output.dir=/data/prod/output
```

### 6. Jenkins Pipeline

**Jenkinsfile**
```groovy
pipeline {
    agent any
    
    environment {
        SPRING_PROFILES_ACTIVE = 'dev'
        VERSION = '1.0.0'
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Deploy to Dev') {
            steps {
                script {
                    env.SPRING_PROFILES_ACTIVE = 'dev'
                }
                sh "java -jar target/migration-batch-${VERSION}.jar --spring.profiles.active=${SPRING_PROFILES_ACTIVE} schema=${BATCH_DEFAULT_SCHEMA} outputFile=/data/dev/output/customers_${BUILD_NUMBER}.csv"
            }
        }
        
        stage('Deploy to Stage') {
            when {
                branch 'release/*'
            }
            steps {
                script {
                    env.SPRING_PROFILES_ACTIVE = 'stage'
                }
                sh "java -jar target/migration-batch-${VERSION}.jar --spring.profiles.active=${SPRING_PROFILES_ACTIVE} schema=${BATCH_DEFAULT_SCHEMA} outputFile=/data/stage/output/customers_${BUILD_NUMBER}.csv"
            }
        }
        
        stage('Deploy to Prod') {
            when {
                branch 'main'
            }
            steps {
                script {
                    env.SPRING_PROFILES_ACTIVE = 'prod'
                }
                sh "java -jar target/migration-batch-${VERSION}.jar --spring.profiles.active=${SPRING_PROFILES_ACTIVE} schema=${BATCH_DEFAULT_SCHEMA} outputFile=/data/prod/output/customers_${BUILD_NUMBER}.csv"
            }
        }
    }
    
    post {
        always {
            junit '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
        }
    }
}
```

### 7. Monitoring and Error Handling

```java
@Configuration
public class BatchListeners {

    @Bean
    public JobExecutionListener jobExecutionListener() {
        return new JobExecutionListener() {
            @Override
            public void beforeJob(JobExecution jobExecution) {
                // Log job start
            }

            @Override
            public void afterJob(JobExecution jobExecution) {
                if (jobExecution.getStatus() == BatchStatus.FAILED) {
                    // Handle job failure (send notification, etc.)
                }
            }
        };
    }

    @Bean
    public StepExecutionListener stepExecutionListener() {
        return new StepExecutionListener() {
            @Override
            public void beforeStep(StepExecution stepExecution) {
                // Step start logic
            }

            @Override
            public ExitStatus afterStep(StepExecution stepExecution) {
                if (stepExecution.getStatus() == BatchStatus.FAILED) {
                    // Handle step failure
                }
                return stepExecution.getExitStatus();
            }
        };
    }
}
```

### 8. Best Practices

1. **Chunk Size Optimization**: Configure optimal chunk size based on memory and performance testing
2. **Error Handling**: Implement skip and retry logic for transient errors
3. **Restartability**: Configure jobs to be restartable with proper job repository
4. **Monitoring**: Integrate with Spring Boot Actuator for monitoring
5. **Logging**: Implement comprehensive logging for troubleshooting
6. **Parameter Validation**: Validate job parameters before execution
7. **Resource Cleanup**: Ensure proper resource cleanup after job completion
8. **Testing**: Implement unit and integration tests for all components

### 9. Running the Jobs

```bash
# For development
java -jar target/migration-batch-1.0.0.jar --spring.profiles.active=dev schema=DEV_SCHEMA outputFile=/data/dev/output/customers.csv

# For production
java -jar target/migration-batch-1.0.0.jar --spring.profiles.active=prod schema=PROD_SCHEMA outputFile=/data/prod/output/customers.csv
```

This implementation provides a production-ready Spring Batch 5.2 solution that can be easily extended for additional migration jobs and scaled across different environments.
