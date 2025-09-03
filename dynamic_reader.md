In **Spring Batch**, the usual way to switch between multiple `ItemReader`s (DB, REST API, file, etc.) is to make the reader bean conditional based on a **JobParameter** (or CLI argument).

Here’s a clean approach:

---

### 1. Pass CLI argument as a JobParameter

When starting your job, pass something like:

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob sourceType=DB
```

or

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob sourceType=REST
```

---

### 2. Configure multiple readers

Define separate readers for **DB** and **REST API**:

```java
@Configuration
public class ReaderConfig {

    @Bean
    @StepScope
    public JdbcCursorItemReader<MyEntity> dbReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<MyEntity>()
                .name("dbReader")
                .dataSource(dataSource)
                .sql("SELECT id, name, status FROM users WHERE status = 'ACTIVE'")
                .rowMapper(new BeanPropertyRowMapper<>(MyEntity.class))
                .build();
    }

    @Bean
    @StepScope
    public ItemReader<MyEntity> restReader(RestTemplate restTemplate) {
        return () -> {
            ResponseEntity<MyEntity[]> response =
                    restTemplate.getForEntity("https://api.example.com/users/active", MyEntity[].class);
            return Arrays.asList(response.getBody()).iterator().hasNext()
                    ? Arrays.asList(response.getBody()).iterator().next()
                    : null;
        };
    }
}
```

---

### 3. Create a delegating reader bean that picks based on JobParameter

```java
@Configuration
public class JobReaderSelector {

    @Bean
    @StepScope
    public ItemReader<MyEntity> dynamicReader(
            @Value("#{jobParameters['sourceType']}") String sourceType,
            JdbcCursorItemReader<MyEntity> dbReader,
            ItemReader<MyEntity> restReader) {

        if ("DB".equalsIgnoreCase(sourceType)) {
            return dbReader;
        } else if ("REST".equalsIgnoreCase(sourceType)) {
            return restReader;
        } else {
            throw new IllegalArgumentException("Invalid sourceType: " + sourceType);
        }
    }
}
```

---

### 4. Use `dynamicReader` in your Step

```java
@Bean
public Step myStep(StepBuilderFactory stepBuilderFactory,
                   ItemReader<MyEntity> dynamicReader,
                   ItemProcessor<MyEntity, MyEntity> processor,
                   ItemWriter<MyEntity> writer) {
    return stepBuilderFactory.get("myStep")
            .<MyEntity, MyEntity>chunk(10)
            .reader(dynamicReader)
            .processor(processor)
            .writer(writer)
            .build();
}
```

---

✅ This way, when you run:

* `sourceType=DB` → Reads from database.
* `sourceType=REST` → Reads from REST API.

---

Would you like me to also show you a **JobParametersValidator** to ensure only valid values (`DB`, `REST`) are accepted at startup, so the job fails fast on bad input?









Perfect 👍 Adding a **JobParametersValidator** is a good practice so your job fails fast if the user passes an invalid `sourceType`.

Here’s how you can implement it:

---

### 1. Create a custom validator

```java
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersInvalidException;
import org.springframework.batch.core.JobParametersValidator;

public class SourceTypeValidator implements JobParametersValidator {

    @Override
    public void validate(JobParameters parameters) throws JobParametersInvalidException {
        String sourceType = parameters.getString("sourceType");

        if (sourceType == null) {
            throw new JobParametersInvalidException("Missing required job parameter: sourceType");
        }

        if (!("DB".equalsIgnoreCase(sourceType) || "REST".equalsIgnoreCase(sourceType))) {
            throw new JobParametersInvalidException(
                    "Invalid sourceType: " + sourceType + ". Allowed values are DB or REST");
        }
    }
}
```

---

### 2. Register validator with your job

```java
@Configuration
public class JobConfig {

    @Bean
    public Job myJob(JobBuilderFactory jobBuilderFactory,
                     Step myStep) {
        return jobBuilderFactory.get("myJob")
                .validator(new SourceTypeValidator()) // <-- validator here
                .start(myStep)
                .build();
    }
}
```

---

### 3. Run job with correct parameter

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob sourceType=DB
```

✅ If you pass `sourceType=FILE` or forget it entirely, the job will immediately fail before even starting.

---

Would you like me to also extend this setup so you can **add new readers easily in the future** (e.g., FILE, KAFKA) without modifying the core job config every time?






Nice 👌 Future-proofing is important in batch jobs. Instead of hardcoding `"DB"` or `"REST"`, you can build a **pluggable reader registry**. That way, if you later add `FILE`, `KAFKA`, etc., you just drop in a new reader bean and update the validator list.

Here’s how you can do it step by step:

---

## 1. Create a registry for readers

```java
@Component
@StepScope
public class ItemReaderRegistry<T> {

    private final Map<String, ItemReader<T>> readers = new HashMap<>();

    public void register(String sourceType, ItemReader<T> reader) {
        readers.put(sourceType.toUpperCase(), reader);
    }

    public ItemReader<T> getReader(String sourceType) {
        ItemReader<T> reader = readers.get(sourceType.toUpperCase());
        if (reader == null) {
            throw new IllegalArgumentException("No reader found for sourceType: " + sourceType);
        }
        return reader;
    }

    public Set<String> getSupportedTypes() {
        return readers.keySet();
    }
}
```

---

## 2. Register each reader with the registry

Instead of hardcoding in the job config:

```java
@Configuration
public class ReaderConfig {

    @Bean
    @StepScope
    public JdbcCursorItemReader<MyEntity> dbReader(DataSource dataSource, ItemReaderRegistry<MyEntity> registry) {
        JdbcCursorItemReader<MyEntity> reader = new JdbcCursorItemReaderBuilder<MyEntity>()
                .name("dbReader")
                .dataSource(dataSource)
                .sql("SELECT id, name, status FROM users WHERE status = 'ACTIVE'")
                .rowMapper(new BeanPropertyRowMapper<>(MyEntity.class))
                .build();
        registry.register("DB", reader);
        return reader;
    }

    @Bean
    @StepScope
    public ItemReader<MyEntity> restReader(RestTemplate restTemplate, ItemReaderRegistry<MyEntity> registry) {
        ItemReader<MyEntity> reader = new IteratorItemReader<>(
                Arrays.asList(restTemplate.getForObject("https://api.example.com/users/active", MyEntity[].class))
        );
        registry.register("REST", reader);
        return reader;
    }
}
```

➡️ Later, you can add:

```java
registry.register("FILE", fileReader);
registry.register("KAFKA", kafkaReader);
```

---

## 3. Dynamic reader bean

```java
@Configuration
public class JobReaderSelector {

    @Bean
    @StepScope
    public ItemReader<MyEntity> dynamicReader(
            @Value("#{jobParameters['sourceType']}") String sourceType,
            ItemReaderRegistry<MyEntity> registry) {
        return registry.getReader(sourceType);
    }
}
```

---

## 4. Validator that adapts automatically

```java
@Configuration
public class JobConfig {

    @Bean
    public Job myJob(JobBuilderFactory jobBuilderFactory,
                     Step myStep,
                     ItemReaderRegistry<MyEntity> registry) {
        return jobBuilderFactory.get("myJob")
                .validator(parameters -> {
                    String sourceType = parameters.getString("sourceType");
                    if (sourceType == null) {
                        throw new JobParametersInvalidException("Missing required job parameter: sourceType");
                    }
                    if (!registry.getSupportedTypes().contains(sourceType.toUpperCase())) {
                        throw new JobParametersInvalidException(
                                "Invalid sourceType: " + sourceType +
                                        ". Allowed values: " + registry.getSupportedTypes());
                    }
                })
                .start(myStep)
                .build();
    }
}
```

---

✅ Now the system is **extensible**:

* Adding a new reader (FILE, KAFKA, etc.) = just register it in `ReaderConfig`.
* Validator automatically picks up new types without any extra changes.

---

Do you also want me to show how to make this **completely annotation-driven** (using `@ConditionalOnProperty` or `@Profile` instead of a registry), so you can even swap readers without touching Java code?






Perfect ⚡ Let's make it **completely annotation-driven**, so you don’t even need a registry. Instead, Spring will auto-pick the right `ItemReader` bean based on your CLI/job parameter.

We’ll use `@ConditionalOnProperty` (or profiles) to make this super clean.

---

## 🔹 Option 1: Using `@ConditionalOnProperty`

Spring Boot will activate only the reader whose property matches the CLI argument.

### 1. Pass CLI argument as a property

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob --reader.type=DB
```

---

### 2. Define conditional readers

```java
@Configuration
public class ReaderConfig {

    @Bean
    @StepScope
    @ConditionalOnProperty(name = "reader.type", havingValue = "DB")
    public ItemReader<MyEntity> dbReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<MyEntity>()
                .name("dbReader")
                .dataSource(dataSource)
                .sql("SELECT id, name, status FROM users WHERE status = 'ACTIVE'")
                .rowMapper(new BeanPropertyRowMapper<>(MyEntity.class))
                .build();
    }

    @Bean
    @StepScope
    @ConditionalOnProperty(name = "reader.type", havingValue = "REST")
    public ItemReader<MyEntity> restReader(RestTemplate restTemplate) {
        return new IteratorItemReader<>(
                Arrays.asList(restTemplate.getForObject("https://api.example.com/users/active", MyEntity[].class))
        );
    }
}
```

➡️ If you run with `--reader.type=DB`, only the `dbReader` bean exists.
➡️ If you run with `--reader.type=REST`, only the `restReader` bean exists.

---

### 3. Inject the selected reader directly

```java
@Bean
public Step myStep(StepBuilderFactory stepBuilderFactory,
                   ItemReader<MyEntity> reader,
                   ItemProcessor<MyEntity, MyEntity> processor,
                   ItemWriter<MyEntity> writer) {
    return stepBuilderFactory.get("myStep")
            .<MyEntity, MyEntity>chunk(10)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
}
```

👉 No dynamic switch needed. Spring ensures **only one reader** is active.

---

## 🔹 Option 2: Using Spring Profiles

Instead of a property, you activate a **profile**:

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob --spring.profiles.active=db
```

### Example

```java
@Configuration
@Profile("db")
public class DbReaderConfig {
    @Bean
    @StepScope
    public ItemReader<MyEntity> dbReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<MyEntity>()
                .name("dbReader")
                .dataSource(dataSource)
                .sql("SELECT id, name, status FROM users WHERE status = 'ACTIVE'")
                .rowMapper(new BeanPropertyRowMapper<>(MyEntity.class))
                .build();
    }
}

@Configuration
@Profile("rest")
public class RestReaderConfig {
    @Bean
    @StepScope
    public ItemReader<MyEntity> restReader(RestTemplate restTemplate) {
        return new IteratorItemReader<>(
                Arrays.asList(restTemplate.getForObject("https://api.example.com/users/active", MyEntity[].class))
        );
    }
}
```

---

✅ Differences:

* **`@ConditionalOnProperty`** → Flexible, you can pass any property at runtime.
* **Profiles** → Cleaner for environments (e.g., dev=DB, prod=REST).

---

👉 Since you mentioned CLI argument (`sourceType`), the **`@ConditionalOnProperty`** solution is probably better for you.

Do you want me to also show how to **combine this with JobParameters** (instead of Spring properties), so you can still use `sourceType=DB` like in your original setup?








Got it ✅ You want to keep using **Spring Batch JobParameters** (like `sourceType=DB`) instead of Spring Boot app properties. Since `@ConditionalOnProperty` works on **application properties**, not JobParameters, the trick is:

➡️ **Copy the JobParameter into a Spring property** before the job starts.

That way, you can still run with:

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob sourceType=DB
```

…and Spring will behave as if you had `--reader.type=DB`.

---

## 🔹 Step 1: Listener to map JobParameter → Spring property

```java
@Component
public class JobParameterToPropertyListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        String sourceType = jobExecution.getJobParameters().getString("sourceType");
        if (sourceType != null) {
            System.setProperty("reader.type", sourceType); // map job param to property
        }
    }
}
```

Now, when your job runs with `sourceType=DB`, Spring sees it as if you set `reader.type=DB`.

---

## 🔹 Step 2: Conditional readers

```java
@Configuration
public class ReaderConfig {

    @Bean
    @StepScope
    @ConditionalOnProperty(name = "reader.type", havingValue = "DB")
    public ItemReader<MyEntity> dbReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<MyEntity>()
                .name("dbReader")
                .dataSource(dataSource)
                .sql("SELECT id, name, status FROM users WHERE status = 'ACTIVE'")
                .rowMapper(new BeanPropertyRowMapper<>(MyEntity.class))
                .build();
    }

    @Bean
    @StepScope
    @ConditionalOnProperty(name = "reader.type", havingValue = "REST")
    public ItemReader<MyEntity> restReader(RestTemplate restTemplate) {
        return new IteratorItemReader<>(
                Arrays.asList(restTemplate.getForObject("https://api.example.com/users/active", MyEntity[].class))
        );
    }
}
```

---

## 🔹 Step 3: Inject the active reader

```java
@Bean
public Step myStep(StepBuilderFactory stepBuilderFactory,
                   ItemReader<MyEntity> reader,
                   ItemProcessor<MyEntity, MyEntity> processor,
                   ItemWriter<MyEntity> writer) {
    return stepBuilderFactory.get("myStep")
            .<MyEntity, MyEntity>chunk(10)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
}
```

---

✅ Now you can launch the job with:

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob sourceType=DB
```

or

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob sourceType=REST
```

and the correct reader is auto-wired without any `if/else` logic or registry.

---

👉 Question for you: Do you also want me to show how to **fail fast with validation** in this setup (so if someone passes `sourceType=XYZ`, the job aborts at startup)?








Great 👍 Let’s extend the **JobParametersValidator** idea to work with this **`@ConditionalOnProperty` + JobParameter mapping** setup.

That way, your job will **fail fast** if someone passes `sourceType=XYZ`.

---

## 🔹 Step 1: Validator

```java
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersInvalidException;
import org.springframework.batch.core.JobParametersValidator;

import java.util.Set;

public class SourceTypeValidator implements JobParametersValidator {

    private final Set<String> allowedValues = Set.of("DB", "REST");

    @Override
    public void validate(JobParameters parameters) throws JobParametersInvalidException {
        String sourceType = parameters.getString("sourceType");

        if (sourceType == null) {
            throw new JobParametersInvalidException("Missing required job parameter: sourceType");
        }

        if (!allowedValues.contains(sourceType.toUpperCase())) {
            throw new JobParametersInvalidException(
                    "Invalid sourceType: " + sourceType +
                    ". Allowed values: " + allowedValues);
        }
    }
}
```

---

## 🔹 Step 2: Register validator with the Job

```java
@Configuration
public class JobConfig {

    @Bean
    public Job myJob(JobBuilderFactory jobBuilderFactory,
                     Step myStep) {
        return jobBuilderFactory.get("myJob")
                .validator(new SourceTypeValidator()) // ✅ fail fast here
                .start(myStep)
                .build();
    }
}
```

---

## 🔹 Step 3: Combine with the parameter → property listener

```java
@Component
public class JobParameterToPropertyListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        String sourceType = jobExecution.getJobParameters().getString("sourceType");
        if (sourceType != null) {
            System.setProperty("reader.type", sourceType);
        }
    }
}
```

---

## 🔹 Behavior

✅ Valid run:

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob sourceType=DB
```

→ Passes validator, sets `reader.type=DB`, Spring creates `dbReader`.

❌ Invalid run:

```bash
java -jar mybatch.jar --spring.batch.job.name=myJob sourceType=XYZ
```

→ Validator throws `JobParametersInvalidException` **before job starts**.

---

⚡ This gives you:

1. Clean selection with `@ConditionalOnProperty` (no if/else).
2. Runtime mapping from `JobParameter → property`.
3. Fail-fast validation before execution.

---

Would you like me to extend this so the **list of allowed sourceTypes comes automatically** from the active readers (instead of being hardcoded `"DB", "REST"` in the validator)? That way, adding a new reader doesn’t require touching the validator at all.










Perfect 🚀 Let’s make the **validator self-updating** so you don’t hardcode `"DB", "REST"`.
Instead, it will **discover allowed `sourceType` values automatically** from the readers you’ve defined with `@ConditionalOnProperty`.

---

## 🔹 Step 1: Create a custom annotation for readers

This lets us tag each reader with its `sourceType`.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ReaderType {
    String value();
}
```

---

## 🔹 Step 2: Annotate your reader beans

```java
@Configuration
public class ReaderConfig {

    @Bean
    @StepScope
    @ReaderType("DB")
    @ConditionalOnProperty(name = "reader.type", havingValue = "DB")
    public ItemReader<MyEntity> dbReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<MyEntity>()
                .name("dbReader")
                .dataSource(dataSource)
                .sql("SELECT id, name, status FROM users WHERE status = 'ACTIVE'")
                .rowMapper(new BeanPropertyRowMapper<>(MyEntity.class))
                .build();
    }

    @Bean
    @StepScope
    @ReaderType("REST")
    @ConditionalOnProperty(name = "reader.type", havingValue = "REST")
    public ItemReader<MyEntity> restReader(RestTemplate restTemplate) {
        return new IteratorItemReader<>(
                Arrays.asList(restTemplate.getForObject("https://api.example.com/users/active", MyEntity[].class))
        );
    }
}
```

Later, if you add a `FILE` reader, just annotate it with `@ReaderType("FILE")`. ✅

---

## 🔹 Step 3: Auto-discover allowed values

We can use Spring’s bean factory to scan for all beans annotated with `@ReaderType`.

```java
@Component
public class SourceTypeRegistry {

    private final Set<String> allowedTypes = new HashSet<>();

    public SourceTypeRegistry(ListableBeanFactory beanFactory) {
        String[] beanNames = beanFactory.getBeanDefinitionNames();
        for (String beanName : beanNames) {
            BeanDefinition def = ((ConfigurableListableBeanFactory) beanFactory).getBeanDefinition(beanName);
            if (def.getSource() instanceof StandardMethodMetadata metadata) {
                ReaderType annotation = metadata.getIntrospectedMethod().getAnnotation(ReaderType.class);
                if (annotation != null) {
                    allowedTypes.add(annotation.value().toUpperCase());
                }
            }
        }
    }

    public Set<String> getAllowedTypes() {
        return allowedTypes;
    }
}
```

---

## 🔹 Step 4: Validator uses the registry

```java
@Component
public class DynamicSourceTypeValidator implements JobParametersValidator {

    private final SourceTypeRegistry registry;

    public DynamicSourceTypeValidator(SourceTypeRegistry registry) {
        this.registry = registry;
    }

    @Override
    public void validate(JobParameters parameters) throws JobParametersInvalidException {
        String sourceType = parameters.getString("sourceType");

        if (sourceType == null) {
            throw new JobParametersInvalidException("Missing required job parameter: sourceType");
        }

        if (!registry.getAllowedTypes().contains(sourceType.toUpperCase())) {
            throw new JobParametersInvalidException(
                    "Invalid sourceType: " + sourceType +
                            ". Allowed values: " + registry.getAllowedTypes());
        }
    }
}
```

---

## 🔹 Step 5: Plug validator into your job

```java
@Configuration
public class JobConfig {

    @Bean
    public Job myJob(JobBuilderFactory jobBuilderFactory,
                     Step myStep,
                     DynamicSourceTypeValidator validator) {
        return jobBuilderFactory.get("myJob")
                .validator(validator) // ✅ dynamic validation
                .start(myStep)
                .build();
    }
}
```

---

## 🔹 Behavior

* Add a new reader with `@ReaderType("FILE")`.
* The `SourceTypeRegistry` picks it up automatically.
* The validator immediately recognizes `"FILE"` as valid.
* No changes required to validator code. 🎉

---

👉 This gives you:

* Clean conditional bean selection (`@ConditionalOnProperty`).
* Fail-fast validation.
* Auto-updating allowed values list.

---

Would you like me to also show you how to **unit test this setup** (e.g., test job startup with `sourceType=DB`, `sourceType=REST`, `sourceType=XYZ`)?

