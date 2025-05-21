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















# Production-Ready Spring Batch 5.2 Implementation for DB2 to CSV Migration

This implementation provides a robust, production-ready solution for migrating data from DB2 tables to CSV files using Spring Batch 5.2, with support for multiple environments (dev, stage, prod).

## Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                     Spring Batch Application                   │
├───────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐      ┌────────────────────┐  │
│  │ JobLauncher │  │JobOperator  │      │   JobRepository    │  │
│  └─────────────┘  └─────────────┘      └────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                      Job Registry                      │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│  │  Job Config │  │  Job Config │  │  Job Config │  ...      │
│  │ (Job1)      │  │ (Job2)      │  │ (Job3)      │           │
│  └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                 Environment Configuration                │  │
│  │ (Profiles: dev, stage, prod)                            │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

## Core Implementation

### 1. Database Configuration (DB2)

```java
@Configuration
public class Db2DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.db2")
    public DataSourceProperties db2DataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean(name = "db2DataSource")
    @Profile("!test")
    public DataSource db2DataSource() {
        return db2DataSourceProperties()
            .initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();
    }

    @Bean(name = "db2JdbcTemplate")
    public JdbcTemplate db2JdbcTemplate(@Qualifier("db2DataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### 2. Batch Job Configuration

```java
@Configuration
@RequiredArgsConstructor
public class MigrationJobConfig {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final Db2ToCsvItemReaderBuilder db2ToCsvItemReaderBuilder;
    private final CsvFileItemWriterBuilder csvFileItemWriterBuilder;
    
    @Value("${batch.chunk.size:100}")
    private int chunkSize;
    
    @Value("${batch.retry.limit:3}")
    private int retryLimit;

    @Bean
    public Job migrationJob(@Qualifier("migrationStep") Step migrationStep) {
        return jobBuilderFactory.get("db2ToCsvMigrationJob")
            .incrementer(new RunIdIncrementer())
            .start(migrationStep)
            .listener(jobExecutionListener())
            .build();
    }

    @Bean
    public Step migrationStep(
        ItemReader<Map<String, Object>> db2ItemReader,
        ItemWriter<Map<String, Object>> csvItemWriter,
        PlatformTransactionManager transactionManager) {
        
        return stepBuilderFactory.get("db2ToCsvMigrationStep")
            .<Map<String, Object>, Map<String, Object>>chunk(chunkSize)
            .reader(db2ItemReader)
            .processor(itemProcessor())
            .writer(csvItemWriter)
            .faultTolerant()
            .retryLimit(retryLimit)
            .retry(DataAccessException.class)
            .skipLimit(100)
            .skip(DataAccessException.class)
            .transactionManager(transactionManager)
            .listener(stepExecutionListener())
            .build();
    }

    @Bean
    @StepScope
    public ItemReader<Map<String, Object>> db2ItemReader(
        @Value("#{jobParameters['tableName']}") String tableName,
        @Value("#{jobParameters['query']}") String query) {
        
        return db2ToCsvItemReaderBuilder.buildReader(tableName, query);
    }

    @Bean
    @StepScope
    public ItemWriter<Map<String, Object>> csvItemWriter(
        @Value("#{jobParameters['outputFile']}") String outputFile) {
        
        return csvFileItemWriterBuilder.buildWriter(outputFile);
    }

    @Bean
    public ItemProcessor<Map<String, Object>, Map<String, Object>> itemProcessor() {
        return new DataTransformationProcessor();
    }

    @Bean
    public JobExecutionListener jobExecutionListener() {
        return new MigrationJobListener();
    }

    @Bean
    public StepExecutionListener stepExecutionListener() {
        return new MigrationStepListener();
    }
}
```

### 3. Custom Reader Builder

```java
@Component
@RequiredArgsConstructor
public class Db2ToCsvItemReaderBuilder {

    private final JdbcTemplate db2JdbcTemplate;

    public ItemReader<Map<String, Object>> buildReader(String tableName, String query) {
        String finalQuery = StringUtils.isNotBlank(query) ? query : 
            String.format("SELECT * FROM %s", tableName);

        return new JdbcCursorItemReaderBuilder<Map<String, Object>>()
            .name(tableName + "Reader")
            .dataSource(db2JdbcTemplate.getDataSource())
            .sql(finalQuery)
            .rowMapper(new ColumnMapRowMapper())
            .saveState(false) // For restartability
            .verifyCursorPosition(false)
            .build();
    }
}
```

### 4. Custom Writer Builder

```java
@Component
public class CsvFileItemWriterBuilder {

    public ItemWriter<Map<String, Object>> buildWriter(String outputFile) {
        FlatFileItemWriter<Map<String, Object>> writer = new FlatFileItemWriter<>();
        writer.setResource(new FileSystemResource(outputFile));
        
        // Header callback to write column names
        writer.setHeaderCallback(w -> {
            if (!writer.getExecutionContext().containsKey("headersWritten")) {
                Map<String, Object> firstItem = (Map<String, Object>) writer.getExecutionContext().get("firstItem");
                if (firstItem != null) {
                    w.write(String.join(",", firstItem.keySet()));
                }
            }
        });

        writer.setLineAggregator(new DelimitedLineAggregator<Map<String, Object>>() {
            {
                setDelimiter(",");
                setFieldExtractor(new BeanWrapperFieldExtractor<Map<String, Object>>() {
                    {
                        setNames(new String[]{}); // Will be set dynamically
                    }
                });
            }
        });

        writer.setExecutionContextName("csvWriterContext");
        
        return new ItemWriter<Map<String, Object>>() {
            @Override
            public void write(List<? extends Map<String, Object>> items) throws Exception {
                if (!writer.getExecutionContext().containsKey("headersWritten")) {
                    if (!items.isEmpty()) {
                        writer.getExecutionContext().put("firstItem", items.get(0));
                        String[] headers = items.get(0).keySet().toArray(new String[0]);
                        ((BeanWrapperFieldExtractor<Map<String, Object>>) 
                            ((DelimitedLineAggregator<Map<String, Object>>) writer.getLineAggregator())
                                .getFieldExtractor()).setNames(headers);
                    }
                }
                writer.write(items);
                writer.getExecutionContext().put("headersWritten", true);
            }
        };
    }
}
```

### 5. Environment Configuration

```yaml
# application-dev.yml
spring:
  datasource:
    db2:
      url: jdbc:db2://dev-db2-server:50000/DEVDB
      username: dev_user
      password: ${DB2_DEV_PASSWORD}
      driver-class-name: com.ibm.db2.jcc.DB2Driver
      hikari:
        maximum-pool-size: 5

batch:
  output:
    base-path: /mnt/data/dev/migrations
  chunk:
    size: 100
  retry:
    limit: 3
```

```yaml
# application-prod.yml
spring:
  datasource:
    db2:
      url: jdbc:db2://prod-db2-server:50000/PRODDB
      username: prod_user
      password: ${DB2_PROD_PASSWORD}
      driver-class-name: com.ibm.db2.jcc.DB2Driver
      hikari:
        maximum-pool-size: 10
        connection-timeout: 30000
        idle-timeout: 600000
        max-lifetime: 1800000

batch:
  output:
    base-path: /mnt/data/prod/migrations
  chunk:
    size: 500
  retry:
    limit: 5
```

### 6. Job Parameters Validation

```java
@Component
public class JobParameterValidator implements JobParametersValidator {

    @Override
    public void validate(JobParameters parameters) throws JobParametersInvalidException {
        if (parameters == null) {
            throw new JobParametersInvalidException("Job parameters cannot be null");
        }

        String tableName = parameters.getString("tableName");
        String outputFile = parameters.getString("outputFile");
        String query = parameters.getString("query");

        if (StringUtils.isBlank(tableName) && StringUtils.isBlank(query)) {
            throw new JobParametersInvalidException(
                "Either tableName or query parameter must be provided");
        }

        if (StringUtils.isBlank(outputFile)) {
            throw new JobParametersInvalidException("outputFile parameter is required");
        }
    }
}
```

### 7. Monitoring and Metrics

```java
@Configuration
@EnableBatchMetrics
public class MetricsConfig {

    @Bean
    public MeterRegistry meterRegistry() {
        return new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
    }

    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}

@Aspect
@Component
public class BatchMonitoringAspect {

    private static final Logger logger = LoggerFactory.getLogger(BatchMonitoringAspect.class);
    
    @AfterReturning(
        pointcut = "execution(* org.springframework.batch.core.JobExecutionListener+.afterJob(..))",
        returning = "jobExecution")
    public void afterJobMonitoring(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            logger.info("Job {} completed successfully", jobExecution.getJobInstance().getJobName());
            // Additional monitoring logic
        } else if (jobExecution.getStatus() == BatchStatus.FAILED) {
            logger.error("Job {} failed", jobExecution.getJobInstance().getJobName());
            // Error handling and alerting
        }
    }
}
```

## Production Considerations

### 1. Error Handling and Retry

```java
@Configuration
public class RetryConfig {

    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate retryTemplate = new RetryTemplate();
        
        FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy();
        backOffPolicy.setBackOffPeriod(2000); // 2 seconds
        
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(3);
        
        retryTemplate.setBackOffPolicy(backOffPolicy);
        retryTemplate.setRetryPolicy(retryPolicy);
        
        return retryTemplate;
    }
}
```

### 2. Parallel Processing

```java
@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(10);
    executor.setQueueCapacity(25);
    executor.setThreadNamePrefix("migration-job-");
    executor.initialize();
    return executor;
}

@Bean
public Step partitionedStep(
    @Qualifier("migrationStep") Step migrationStep,
    TaskExecutor taskExecutor) {
    
    return stepBuilderFactory.get("partitionedMigrationStep")
        .partitioner("migrationStep", new ColumnRangePartitioner())
        .step(migrationStep)
        .taskExecutor(taskExecutor)
        .gridSize(4)
        .build();
}
```

### 3. Job Scheduling

```java
@Configuration
@EnableScheduling
public class JobSchedulerConfig {

    @Autowired
    private JobLauncher jobLauncher;
    
    @Autowired
    private Job migrationJob;
    
    @Autowired
    private JobOperator jobOperator;
    
    @Value("${migration.job.cron}")
    private String cronExpression;

    @Scheduled(cron = "${migration.job.cron}")
    public void runMigrationJob() throws Exception {
        JobParameters jobParameters = new JobParametersBuilder()
            .addString("tableName", "CUSTOMER")
            .addString("outputFile", "/output/customer_data_" + System.currentTimeMillis() + ".csv")
            .addLong("time", System.currentTimeMillis())
            .toJobParameters();
            
        jobLauncher.run(migrationJob, jobParameters);
    }
}
```

## Deployment Strategy

1. **Containerization**:
```dockerfile
FROM eclipse-temurin:17-jdk-jammy

VOLUME /tmp
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar

ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

2. **Kubernetes Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-migration
spec:
  replicas: 2
  selector:
    matchLabels:
      app: batch-migration
  template:
    metadata:
      labels:
        app: batch-migration
    spec:
      containers:
      - name: migration-app
        image: your-registry/batch-migration:1.0.0
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB2_PROD_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db2-credentials
              key: password
        resources:
          limits:
            memory: "1Gi"
            cpu: "500m"
```

## Best Practices

1. **Job Repository Configuration**:
```java
@Configuration
@EnableBatchProcessing
public class BatchRepositoryConfig {

    @Bean
    public JobRepository jobRepository(
        DataSource dataSource,
        PlatformTransactionManager transactionManager) throws Exception {
        
        JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
        factory.setDataSource(dataSource);
        factory.setTransactionManager(transactionManager);
        factory.setIsolationLevelForCreate("ISOLATION_READ_COMMITTED");
        factory.setTablePrefix("BATCH_");
        factory.setMaxVarCharLength(1000);
        factory.afterPropertiesSet();
        return factory.getObject();
    }
}
```

2. **Database Schema Initialization**:
```sql
-- DB2-specific schema for Spring Batch tables
CREATE TABLE BATCH_JOB_INSTANCE (
  JOB_INSTANCE_ID BIGINT NOT NULL PRIMARY KEY,
  VERSION BIGINT,
  JOB_NAME VARCHAR(100) NOT NULL,
  JOB_KEY VARCHAR(32) NOT NULL,
  CONSTRAINT JOB_INST_UN UNIQUE (JOB_NAME, JOB_KEY)
);

-- Other Spring Batch tables...
```

3. **Logging Configuration**:
```xml
<!-- logback-spring.xml -->
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    
    <springProfile name="dev">
        <logger name="org.springframework.batch" level="DEBUG"/>
        <logger name="org.springframework.jdbc" level="DEBUG"/>
    </springProfile>
    
    <springProfile name="prod">
        <logger name="org.springframework.batch" level="INFO"/>
        <logger name="org.springframework.jdbc" level="WARN"/>
    </springProfile>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

This implementation provides a comprehensive, production-ready solution for DB2 to CSV migrations using Spring Batch 5.2, with support for multiple environments, robust error handling, monitoring, and scalability features.
