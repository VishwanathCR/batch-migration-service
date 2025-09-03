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









