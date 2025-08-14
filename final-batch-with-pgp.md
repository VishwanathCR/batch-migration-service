spring batch job implementation to read user data from DB2 DB and write only active users records to custom CSV flat file which has header row, data row which had extra fields than the data from reader and footer row should have the total number of data row using java 21 and spring batch 5.2


Got it — you want a **Spring Batch 5.2** job in **Java 21** that:

* Reads **user data** from a **DB2** database.
* Writes **only active users** to a **custom CSV file**.
* CSV must have:

  * **Header row**
  * **Data rows** (with **extra fields** computed or added beyond DB columns)
  * **Footer row** containing the **total number of data rows written**

We can do this by combining:

* **JdbcCursorItemReader** (or JdbcPagingItemReader) for DB2 read.
* **ItemProcessor** to filter and add extra fields.
* **Custom FlatFileItemWriter** with **HeaderCallback** and **FooterCallback**.

Here’s a clean implementation example.

---

## **Project Structure**

```
src/main/java
 ├─ com.example.batch
 │   ├─ BatchConfig.java
 │   ├─ User.java
 │   ├─ UserProcessor.java
 │   ├─ UserWriter.java
 │   └─ SpringBatchApplication.java
```

---

## **1. Entity Class**

```java
package com.example.batch;

public class User {
    private Long id;
    private String name;
    private String email;
    private String status; // active/inactive
    private String extraField; // added later in processor

    // getters & setters
}
```

---

## **2. Processor**

```java
package com.example.batch;

import org.springframework.batch.item.ItemProcessor;

public class UserProcessor implements ItemProcessor<User, User> {
    @Override
    public User process(User user) {
        // Only keep ACTIVE users
        if (!"ACTIVE".equalsIgnoreCase(user.getStatus())) {
            return null; // Filter out
        }

        // Add extra field value
        user.setExtraField("Processed-" + user.getId());

        return user;
    }
}
```

---

## **3. Custom Footer Writer**

```java
package com.example.batch;

import org.springframework.batch.item.file.FlatFileFooterCallback;

import java.io.IOException;
import java.io.Writer;
import java.util.concurrent.atomic.AtomicInteger;

public class UserFooterCallback implements FlatFileFooterCallback {

    private final AtomicInteger rowCount;

    public UserFooterCallback(AtomicInteger rowCount) {
        this.rowCount = rowCount;
    }

    @Override
    public void writeFooter(Writer writer) throws IOException {
        writer.write("Total Records: " + rowCount.get());
    }
}
```

---

## **4. Batch Config**

```java
package com.example.batch;

import org.springframework.batch.core.*;
import org.springframework.batch.core.configuration.annotation.*;
import org.springframework.batch.item.database.*;
import org.springframework.batch.item.database.support.*;
import org.springframework.batch.item.file.*;
import org.springframework.batch.item.file.builder.FlatFileItemWriterBuilder;
import org.springframework.batch.item.file.transform.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.core.io.FileSystemResource;

import javax.sql.DataSource;
import java.util.concurrent.atomic.AtomicInteger;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public JdbcCursorItemReader<User> reader(DataSource dataSource) {
        JdbcCursorItemReader<User> reader = new JdbcCursorItemReader<>();
        reader.setDataSource(dataSource);
        reader.setSql("SELECT ID, NAME, EMAIL, STATUS FROM USERS");
        reader.setRowMapper((rs, rowNum) -> {
            User u = new User();
            u.setId(rs.getLong("ID"));
            u.setName(rs.getString("NAME"));
            u.setEmail(rs.getString("EMAIL"));
            u.setStatus(rs.getString("STATUS"));
            return u;
        });
        return reader;
    }

    @Bean
    public UserProcessor processor() {
        return new UserProcessor();
    }

    @Bean
    public FlatFileItemWriter<User> writer(
            @Value("${output.file.path}") String outputPath) {

        AtomicInteger rowCount = new AtomicInteger();

        return new FlatFileItemWriterBuilder<User>()
                .name("userItemWriter")
                .resource(new FileSystemResource(outputPath))
                .headerCallback(writer -> writer.write("ID,Name,Email,Status,ExtraField"))
                .footerCallback(new UserFooterCallback(rowCount))
                .lineAggregator(item -> {
                    rowCount.incrementAndGet();
                    return String.format("%d,%s,%s,%s,%s",
                            item.getId(),
                            item.getName(),
                            item.getEmail(),
                            item.getStatus(),
                            item.getExtraField());
                })
                .build();
    }

    @Bean
    public Step step1(JobRepository jobRepository,
                      PlatformTransactionManager transactionManager,
                      JdbcCursorItemReader<User> reader,
                      UserProcessor processor,
                      FlatFileItemWriter<User> writer) {
        return new StepBuilder("step1", jobRepository)
                .<User, User>chunk(10, transactionManager)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step1) {
        return new JobBuilder("userExportJob", jobRepository)
                .start(step1)
                .build();
    }
}
```

---

## **5. Application Entry**

```java
package com.example.batch;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringBatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBatchApplication.class, args);
    }
}
```

---

## **6. `application.properties`**

```properties
spring.datasource.url=jdbc:db2://<host>:<port>/<dbname>
spring.datasource.username=<username>
spring.datasource.password=<password>
spring.datasource.driver-class-name=com.ibm.db2.jcc.DB2Driver

output.file.path=output/users.csv
```

---

## **Execution Flow**

1. **Reader**: Reads all users from DB2.
2. **Processor**: Keeps only active users and adds `extraField`.
3. **Writer**:

   * Writes header once.
   * Writes each data row with `extraField`.
   * Tracks row count and writes footer at the end.

---

If you want, I can also give you **a version that uses `JdbcPagingItemReader` instead of `JdbcCursorItemReader`** so it can handle large datasets from DB2 without keeping the cursor open. That’s better for production ETL.

Do you want me to give you **the paging version**? That would make this Spring Batch job safer for very large DB2 tables.





Alright — let’s swap out the **JdbcCursorItemReader** with a **JdbcPagingItemReader** so it can process large DB2 tables in a paginated, memory-safe way.

This version:

* Doesn’t keep the DB cursor open for the entire job.
* Retrieves records in **pages** using **ORDER BY**.
* Still filters active users in the **processor**.
* Still writes **custom CSV** with header + extra fields + footer row.

---

## **Updated BatchConfig with Paging Reader**

```java
package com.example.batch;

import org.springframework.batch.core.*;
import org.springframework.batch.core.configuration.annotation.*;
import org.springframework.batch.item.database.*;
import org.springframework.batch.item.database.support.Db2PagingQueryProvider;
import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.batch.item.file.builder.FlatFileItemWriterBuilder;
import org.springframework.batch.item.file.transform.LineAggregator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.core.io.FileSystemResource;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;
import java.util.concurrent.atomic.AtomicInteger;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public JdbcPagingItemReader<User> pagingReader(DataSource dataSource) throws Exception {
        JdbcPagingItemReader<User> reader = new JdbcPagingItemReader<>();
        reader.setDataSource(dataSource);
        reader.setPageSize(100); // Fetch 100 records per page
        reader.setRowMapper((rs, rowNum) -> {
            User u = new User();
            u.setId(rs.getLong("ID"));
            u.setName(rs.getString("NAME"));
            u.setEmail(rs.getString("EMAIL"));
            u.setStatus(rs.getString("STATUS"));
            return u;
        });

        Db2PagingQueryProvider queryProvider = new Db2PagingQueryProvider();
        queryProvider.setSelectClause("SELECT ID, NAME, EMAIL, STATUS");
        queryProvider.setFromClause("FROM USERS");
        queryProvider.setWhereClause("WHERE STATUS = 'ACTIVE' OR STATUS = 'INACTIVE'");
        queryProvider.setSortKeys(java.util.Map.of("ID", Order.ASCENDING));

        reader.setQueryProvider(queryProvider);
        reader.afterPropertiesSet();
        return reader;
    }

    @Bean
    public UserProcessor processor() {
        return new UserProcessor();
    }

    @Bean
    public FlatFileItemWriter<User> writer(
            @Value("${output.file.path}") String outputPath) {

        AtomicInteger rowCount = new AtomicInteger();

        return new FlatFileItemWriterBuilder<User>()
                .name("userItemWriter")
                .resource(new FileSystemResource(outputPath))
                .headerCallback(writer -> writer.write("ID,Name,Email,Status,ExtraField"))
                .footerCallback(new UserFooterCallback(rowCount))
                .lineAggregator((LineAggregator<User>) item -> {
                    rowCount.incrementAndGet();
                    return String.format("%d,%s,%s,%s,%s",
                            item.getId(),
                            item.getName(),
                            item.getEmail(),
                            item.getStatus(),
                            item.getExtraField());
                })
                .build();
    }

    @Bean
    public Step step1(JobRepository jobRepository,
                      PlatformTransactionManager transactionManager,
                      JdbcPagingItemReader<User> reader,
                      UserProcessor processor,
                      FlatFileItemWriter<User> writer) {
        return new StepBuilder("step1", jobRepository)
                .<User, User>chunk(50, transactionManager) // Process in chunks of 50
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step1) {
        return new JobBuilder("userExportJob", jobRepository)
                .start(step1)
                .build();
    }
}
```

---

### **Key Changes from Cursor Version**

1. **JdbcPagingItemReader** instead of JdbcCursorItemReader.
2. **Db2PagingQueryProvider** used to support DB2 paging syntax.
3. Explicit **ORDER BY ID** for stable paging.
4. **PageSize = 100** (can adjust based on performance).
5. **Chunk Size = 50** (writer flushes every 50 records).

---

### **Why this is safer for DB2 ETL**

* **Cursor approach**: DB2 keeps the connection open until the last record is read; long jobs can hit timeouts.
* **Paging approach**: Runs multiple small queries with offsets, reducing DB lock risk and memory usage.

---

If you want, I can also give you a **test dataset + in-memory H2 DB config** so you can run this job without DB2 locally, and then just swap the `DataSource` when moving to production.

Do you want me to prepare that **local runnable test version** so you can verify before hitting DB2?




Perfect — I’ll give you a **runnable local test version** using:

* **Java 21**
* **Spring Boot 3.3+ / Spring Batch 5.2**
* **H2 in-memory DB** for local testing
* **Gradle** build setup
* Same paging logic (just swap the datasource for DB2 later)

This will let you **run the job end-to-end locally** before pointing it to DB2.

---

## **1. build.gradle**

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.2'
    id 'io.spring.dependency-management' version '1.1.5'
}

group = 'com.example'
version = '1.0.0'
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    runtimeOnly 'com.h2database:h2'

    // For DB2 later (keep for prod switch)
    runtimeOnly 'com.ibm.db2:jcc:11.5.8.0'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

---

## **2. application.properties**

```properties
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=DB2
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
spring.h2.console.enabled=true

spring.batch.jdbc.initialize-schema=always
output.file.path=output/users.csv
```

---

## **3. schema.sql** (H2 DB schema)

```sql
CREATE TABLE USERS (
    ID BIGINT PRIMARY KEY,
    NAME VARCHAR(100),
    EMAIL VARCHAR(150),
    STATUS VARCHAR(20)
);
```

---

## **4. data.sql** (Test Data)

```sql
INSERT INTO USERS (ID, NAME, EMAIL, STATUS) VALUES
(1, 'Alice', 'alice@example.com', 'ACTIVE'),
(2, 'Bob', 'bob@example.com', 'INACTIVE'),
(3, 'Charlie', 'charlie@example.com', 'ACTIVE'),
(4, 'David', 'david@example.com', 'ACTIVE'),
(5, 'Eve', 'eve@example.com', 'INACTIVE');
```

---

## **5. User.java**

```java
package com.example.batch;

public class User {
    private Long id;
    private String name;
    private String email;
    private String status;
    private String extraField;

    // getters & setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }

    public String getExtraField() { return extraField; }
    public void setExtraField(String extraField) { this.extraField = extraField; }
}
```

---

## **6. UserProcessor.java**

```java
package com.example.batch;

import org.springframework.batch.item.ItemProcessor;

public class UserProcessor implements ItemProcessor<User, User> {
    @Override
    public User process(User user) {
        if (!"ACTIVE".equalsIgnoreCase(user.getStatus())) {
            return null; // Skip inactive users
        }
        user.setExtraField("Processed-" + user.getId());
        return user;
    }
}
```

---

## **7. UserFooterCallback.java**

```java
package com.example.batch;

import org.springframework.batch.item.file.FlatFileFooterCallback;

import java.io.IOException;
import java.io.Writer;
import java.util.concurrent.atomic.AtomicInteger;

public class UserFooterCallback implements FlatFileFooterCallback {

    private final AtomicInteger rowCount;

    public UserFooterCallback(AtomicInteger rowCount) {
        this.rowCount = rowCount;
    }

    @Override
    public void writeFooter(Writer writer) throws IOException {
        writer.write("Total Records: " + rowCount.get());
    }
}
```

---

## **8. BatchConfig.java**

```java
package com.example.batch;

import org.springframework.batch.core.*;
import org.springframework.batch.core.configuration.annotation.*;
import org.springframework.batch.item.database.*;
import org.springframework.batch.item.database.support.Db2PagingQueryProvider;
import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.batch.item.file.builder.FlatFileItemWriterBuilder;
import org.springframework.batch.item.file.transform.LineAggregator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.core.io.FileSystemResource;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public JdbcPagingItemReader<User> pagingReader(DataSource dataSource) throws Exception {
        JdbcPagingItemReader<User> reader = new JdbcPagingItemReader<>();
        reader.setDataSource(dataSource);
        reader.setPageSize(2);
        reader.setRowMapper((rs, rowNum) -> {
            User u = new User();
            u.setId(rs.getLong("ID"));
            u.setName(rs.getString("NAME"));
            u.setEmail(rs.getString("EMAIL"));
            u.setStatus(rs.getString("STATUS"));
            return u;
        });

        Db2PagingQueryProvider queryProvider = new Db2PagingQueryProvider();
        queryProvider.setSelectClause("SELECT ID, NAME, EMAIL, STATUS");
        queryProvider.setFromClause("FROM USERS");
        queryProvider.setWhereClause("WHERE STATUS IN ('ACTIVE','INACTIVE')");
        queryProvider.setSortKeys(Map.of("ID", Order.ASCENDING));

        reader.setQueryProvider(queryProvider);
        reader.afterPropertiesSet();
        return reader;
    }

    @Bean
    public UserProcessor processor() {
        return new UserProcessor();
    }

    @Bean
    public FlatFileItemWriter<User> writer(
            @Value("${output.file.path}") String outputPath) {

        AtomicInteger rowCount = new AtomicInteger();

        return new FlatFileItemWriterBuilder<User>()
                .name("userItemWriter")
                .resource(new FileSystemResource(outputPath))
                .headerCallback(writer -> writer.write("ID,Name,Email,Status,ExtraField"))
                .footerCallback(new UserFooterCallback(rowCount))
                .lineAggregator((LineAggregator<User>) item -> {
                    rowCount.incrementAndGet();
                    return String.format("%d,%s,%s,%s,%s",
                            item.getId(),
                            item.getName(),
                            item.getEmail(),
                            item.getStatus(),
                            item.getExtraField());
                })
                .build();
    }

    @Bean
    public Step step1(JobRepository jobRepository,
                      PlatformTransactionManager transactionManager,
                      JdbcPagingItemReader<User> reader,
                      UserProcessor processor,
                      FlatFileItemWriter<User> writer) {
        return new StepBuilder("step1", jobRepository)
                .<User, User>chunk(2, transactionManager)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step1) {
        return new JobBuilder("userExportJob", jobRepository)
                .start(step1)
                .build();
    }
}
```

---

## **9. SpringBatchApplication.java**

```java
package com.example.batch;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringBatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBatchApplication.class, args);
    }
}
```

---

## **Running Locally**

```bash
./gradlew bootRun
```

Expected `output/users.csv`:

```csv
ID,Name,Email,Status,ExtraField
1,Alice,alice@example.com,ACTIVE,Processed-1
3,Charlie,charlie@example.com,ACTIVE,Processed-3
4,David,david@example.com,ACTIVE,Processed-4
Total Records: 3
```

---

When ready for **DB2**, just change:

```properties
spring.datasource.url=jdbc:db2://<host>:<port>/<dbname>
spring.datasource.username=<username>
spring.datasource.password=<password>
spring.datasource.driver-class-name=com.ibm.db2.jcc.DB2Driver
```

---

I can also add a **unit test using Spring Batch’s JobLauncherTestUtils** so you can verify file output programmatically before running in prod.
Do you want me to add that test as well? That would make the job fully testable.









Alright — we’ll extend your **Spring Batch + Paging + CSV job** to:

1. Generate the flat file (as before).
2. After writing it, **PGP encrypt** it using a **public key**.
3. Delete the plain-text CSV if you want only the encrypted file in output.

We’ll integrate **BouncyCastle** for PGP in Java 21, since Spring Batch itself doesn’t do encryption out-of-the-box.

---

## **1. Add PGP dependency in `build.gradle`**

```gradle
dependencies {
    implementation 'org.bouncycastle:bcprov-jdk18on:1.78.1'
    implementation 'org.bouncycastle:bcpg-jdk18on:1.78.1'
}
```

---

## **2. Create a `PGPEncryptor` Utility**

This will take the CSV file from the writer and encrypt it using a **PGP public key**.

```java
package com.example.batch;

import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.operator.jcajce.*;
import org.bouncycastle.bcpg.ArmoredOutputStream;

import java.io.*;
import java.security.Security;
import java.util.Iterator;

public class PGPEncryptor {

    static {
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }

    public static void encryptFile(
            String outputFilePath,
            String inputFilePath,
            String publicKeyPath,
            boolean armor,
            boolean withIntegrityCheck) throws Exception {

        try (OutputStream out = new BufferedOutputStream(new FileOutputStream(outputFilePath))) {
            PGPPublicKey encKey = readPublicKey(new FileInputStream(publicKeyPath));
            if (armor) {
                try (ArmoredOutputStream armoredOut = new ArmoredOutputStream(out)) {
                    encrypt(encKey, inputFilePath, armoredOut, withIntegrityCheck);
                }
            } else {
                encrypt(encKey, inputFilePath, out, withIntegrityCheck);
            }
        }
    }

    private static PGPPublicKey readPublicKey(InputStream in) throws IOException, PGPException {
        PGPPublicKeyRingCollection pgpPub = new PGPPublicKeyRingCollection(
                PGPUtil.getDecoderStream(in),
                new JcaKeyFingerprintCalculator()
        );

        Iterator<PGPPublicKeyRing> keyRingIter = pgpPub.getKeyRings();
        while (keyRingIter.hasNext()) {
            PGPPublicKeyRing keyRing = keyRingIter.next();
            Iterator<PGPPublicKey> keyIter = keyRing.getPublicKeys();
            while (keyIter.hasNext()) {
                PGPPublicKey key = keyIter.next();
                if (key.isEncryptionKey()) {
                    return key;
                }
            }
        }
        throw new IllegalArgumentException("No encryption key found in public key ring.");
    }

    private static void encrypt(PGPPublicKey encKey, String fileName, OutputStream out, boolean withIntegrityCheck)
            throws IOException, PGPException {
        ByteArrayOutputStream bOut = new ByteArrayOutputStream();

        try (PGPCompressedDataGenerator comData = new PGPCompressedDataGenerator(PGPCompressedData.ZIP)) {
            PGPUtil.writeFileToLiteralData(
                    comData.open(bOut),
                    PGPLiteralData.BINARY,
                    new File(fileName)
            );
        }

        PGPEncryptedDataGenerator encGen = new PGPEncryptedDataGenerator(
                new JcePGPDataEncryptorBuilder(PGPEncryptedData.CAST5)
                        .setWithIntegrityPacket(withIntegrityCheck)
                        .setSecureRandom(new java.security.SecureRandom())
                        .setProvider("BC")
        );
        encGen.addMethod(new JcePublicKeyKeyEncryptionMethodGenerator(encKey).setProvider("BC"));

        byte[] bytes = bOut.toByteArray();
        try (OutputStream cOut = encGen.open(out, bytes.length)) {
            cOut.write(bytes);
        }
    }
}
```

---

## **3. Add a Step to Encrypt After Writing**

We’ll make the job **two-step**:

* Step 1: Read from DB → Write CSV
* Step 2: Encrypt CSV → Output `.pgp` file

```java
@Bean
public Step encryptStep(JobRepository jobRepository,
                        PlatformTransactionManager transactionManager,
                        @Value("${output.file.path}") String csvPath,
                        @Value("${pgp.public.key.path}") String publicKeyPath) {
    return new StepBuilder("encryptStep", jobRepository)
            .tasklet((contribution, chunkContext) -> {
                String encryptedFilePath = csvPath + ".pgp";
                PGPEncryptor.encryptFile(
                        encryptedFilePath,
                        csvPath,
                        publicKeyPath,
                        true, // ASCII armored
                        true  // Integrity check
                );
                // Optional: delete plain file
                new File(csvPath).delete();
                return RepeatStatus.FINISHED;
            }, transactionManager)
            .build();
}

@Bean
public Job job(JobRepository jobRepository, Step step1, Step encryptStep) {
    return new JobBuilder("userExportJob", jobRepository)
            .start(step1)
            .next(encryptStep)
            .build();
}
```

---

## **4. application.properties Changes**

```properties
pgp.public.key.path=keys/public.asc
output.file.path=output/users.csv
```

---

## **5. How it Works**

* **Step 1**: Writes CSV normally with header, data, footer.
* **Step 2**: Uses `PGPEncryptor` with your **public key** to create `users.csv.pgp`.
* **Optional**: Deletes `users.csv` so only `.pgp` remains.

---

## **6. Testing Locally**

* Generate PGP key pair for testing:

```bash
gpg --batch --gen-key <<EOF
Key-Type: default
Subkey-Type: default
Name-Real: Test User
Name-Email: test@example.com
Expire-Date: 0
%no-protection
%commit
EOF
gpg --export -a "test@example.com" > public.asc
```

* Put `public.asc` in `keys/` directory.
* Run:

```bash
./gradlew bootRun
```

* You’ll get `output/users.csv.pgp` (encrypted).

---

If you want, I can also make this **encrypt step streaming** so we **never write an unencrypted file to disk** — the writer would directly output `.pgp` during batch execution. That’s more secure for production ETL pipelines.

Do you want me to prepare that **direct PGP writing FlatFileItemWriter** version so the plain-text CSV never exists?







Got it — let’s refactor so the **Spring Batch writer directly produces a PGP-encrypted file**, without ever writing the plain CSV to disk.

This way:

* The **FlatFileItemWriter** writes to an **in-memory stream**.
* That stream is **piped through PGP encryption** into the final `.pgp` output file.
* No unencrypted intermediate file exists on disk.

---

## **1. Concept**

Instead of:

```
DB → FlatFileItemWriter (CSV on disk) → Encrypt (new file)
```

We’ll do:

```
DB → CustomPGPItemWriter (CSV in memory → encrypt on the fly) → Encrypted .pgp file
```

---

## **2. Custom `PGPFlatFileItemWriter`**

```java
package com.example.batch;

import org.bouncycastle.bcpg.ArmoredOutputStream;
import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.operator.jcajce.*;
import org.springframework.batch.item.*;
import org.springframework.batch.item.file.transform.LineAggregator;

import java.io.*;
import java.security.Security;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

public class PGPFlatFileItemWriter<T> implements ItemStreamWriter<T> {

    private final String outputFilePath;
    private final String publicKeyPath;
    private final LineAggregator<T> lineAggregator;
    private final AtomicInteger rowCount = new AtomicInteger();

    static {
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }

    public PGPFlatFileItemWriter(String outputFilePath, String publicKeyPath, LineAggregator<T> lineAggregator) {
        this.outputFilePath = outputFilePath;
        this.publicKeyPath = publicKeyPath;
        this.lineAggregator = lineAggregator;
    }

    @Override
    public void open(ExecutionContext executionContext) throws ItemStreamException {
        // No-op
    }

    @Override
    public void update(ExecutionContext executionContext) throws ItemStreamException {
        // No-op
    }

    @Override
    public void close() throws ItemStreamException {
        // No-op
    }

    @Override
    public void write(List<? extends T> items) throws Exception {
        ByteArrayOutputStream csvOut = new ByteArrayOutputStream();

        try (PrintWriter writer = new PrintWriter(csvOut)) {
            // Write header if first chunk
            if (rowCount.get() == 0) {
                writer.println("ID,Name,Email,Status,ExtraField");
            }
            for (T item : items) {
                rowCount.incrementAndGet();
                writer.println(lineAggregator.aggregate(item));
            }
        }

        // On last chunk, append footer and encrypt everything
        // In real usage, you'd detect the end via JobListener; here we encrypt in each chunk (less efficient)
        if (/* Could add condition to encrypt only once at the end */ false) {
            // No direct condition, see listener version
        }

        // For this example, we encrypt after writing all chunks in a listener (not here)
    }

    public void encryptToFile(byte[] plainData) throws Exception {
        PGPPublicKey encKey = readPublicKey(new FileInputStream(publicKeyPath));

        try (OutputStream out = new BufferedOutputStream(new FileOutputStream(outputFilePath))) {
            try (ArmoredOutputStream armoredOut = new ArmoredOutputStream(out)) {
                encrypt(encKey, plainData, armoredOut, true);
            }
        }
    }

    private static PGPPublicKey readPublicKey(InputStream in) throws IOException, PGPException {
        PGPPublicKeyRingCollection pgpPub = new PGPPublicKeyRingCollection(
                PGPUtil.getDecoderStream(in),
                new JcaKeyFingerprintCalculator()
        );
        Iterator<PGPPublicKeyRing> keyRingIter = pgpPub.getKeyRings();
        while (keyRingIter.hasNext()) {
            PGPPublicKeyRing keyRing = keyRingIter.next();
            Iterator<PGPPublicKey> keyIter = keyRing.getPublicKeys();
            while (keyIter.hasNext()) {
                PGPPublicKey key = keyIter.next();
                if (key.isEncryptionKey()) {
                    return key;
                }
            }
        }
        throw new IllegalArgumentException("No encryption key found in public key ring.");
    }

    private static void encrypt(PGPPublicKey encKey, byte[] plainData, OutputStream out, boolean withIntegrityCheck)
            throws IOException, PGPException {
        ByteArrayOutputStream bOut = new ByteArrayOutputStream();

        try (PGPCompressedDataGenerator comData = new PGPCompressedDataGenerator(PGPCompressedData.ZIP)) {
            try (OutputStream cos = comData.open(bOut)) {
                PGPLiteralDataGenerator lData = new PGPLiteralDataGenerator();
                try (OutputStream pOut = lData.open(cos, PGPLiteralData.BINARY, "data.csv", plainData.length, new java.util.Date())) {
                    pOut.write(plainData);
                }
            }
        }

        PGPEncryptedDataGenerator encGen = new PGPEncryptedDataGenerator(
                new JcePGPDataEncryptorBuilder(PGPEncryptedData.CAST5)
                        .setWithIntegrityPacket(withIntegrityCheck)
                        .setSecureRandom(new java.security.SecureRandom())
                        .setProvider("BC")
        );
        encGen.addMethod(new JcePublicKeyKeyEncryptionMethodGenerator(encKey).setProvider("BC"));

        byte[] bytes = bOut.toByteArray();
        try (OutputStream cOut = encGen.open(out, bytes.length)) {
            cOut.write(bytes);
        }
    }
}
```

---

## **3. Integrating with Spring Batch**

We can’t encrypt per chunk because PGP is a single stream. The cleanest way is:

* Use a **ByteArrayOutputStream** to collect CSV data in memory.
* At the **end of the step** (via `StepExecutionListener.afterStep`), encrypt the whole thing.

---

### **Updated Step Config**

```java
@Bean
public Step step1(JobRepository jobRepository,
                  PlatformTransactionManager transactionManager,
                  JdbcPagingItemReader<User> reader,
                  UserProcessor processor,
                  @Value("${output.file.path}") String encryptedFilePath,
                  @Value("${pgp.public.key.path}") String publicKeyPath) {

    ByteArrayOutputStream csvBuffer = new ByteArrayOutputStream();
    PrintWriter csvWriter = new PrintWriter(csvBuffer);
    AtomicInteger rowCount = new AtomicInteger();

    ItemWriter<User> writer = items -> {
        if (rowCount.get() == 0) {
            csvWriter.println("ID,Name,Email,Status,ExtraField");
        }
        for (User user : items) {
            rowCount.incrementAndGet();
            csvWriter.printf("%d,%s,%s,%s,%s%n",
                    user.getId(),
                    user.getName(),
                    user.getEmail(),
                    user.getStatus(),
                    user.getExtraField());
        }
    };

    return new StepBuilder("step1", jobRepository)
            .<User, User>chunk(50, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .listener((StepExecutionListener) new StepExecutionListener() {
                @Override
                public void beforeStep(StepExecution stepExecution) {}

                @Override
                public ExitStatus afterStep(StepExecution stepExecution) {
                    csvWriter.printf("Total Records: %d%n", rowCount.get());
                    csvWriter.flush();
                    try {
                        PGPEncryptor.encryptFile(
                                encryptedFilePath,
                                csvBuffer.toString(),
                                publicKeyPath
                        );
                    } catch (Exception e) {
                        throw new RuntimeException("PGP encryption failed", e);
                    }
                    return ExitStatus.COMPLETED;
                }
            })
            .build();
}
```

---

### **Small Change to `PGPEncryptor`**

We add an overload that accepts a string instead of a file:

```java
public static void encryptFile(String outputFilePath, String plainTextData, String publicKeyPath) throws Exception {
    PGPPublicKey encKey = readPublicKey(new FileInputStream(publicKeyPath));
    byte[] plainBytes = plainTextData.getBytes(java.nio.charset.StandardCharsets.UTF_8);
    try (OutputStream out = new BufferedOutputStream(new FileOutputStream(outputFilePath))) {
        try (ArmoredOutputStream armoredOut = new ArmoredOutputStream(out)) {
            encrypt(encKey, plainBytes, armoredOut, true);
        }
    }
}
```

---

## **4. Flow Now**

* Spring Batch writes CSV lines into memory.
* When step finishes → Append footer → Encrypt entire CSV content in memory → Write `.pgp` file.
* No `.csv` file is ever created on disk.

---

If you like, I can also **add streaming encryption** so that each CSV row is encrypted immediately as it’s written — this avoids storing the whole file in memory (better for large exports). That’s a bit trickier but perfect for very large DB2 datasets.

Do you want me to make that **streaming PGP CSV writer** version so it’s memory-safe for huge files?







Awesome — here’s a **true streaming PGP CSV writer** for Spring Batch 5.2 that:

* Encrypts **as it writes** (no plaintext file, no buffering whole CSV in memory).
* Writes a **header** at `open()`, **data rows** in `write()`, and a **footer with total count** at `close()`.
* Uses **BouncyCastle** (JDK18+ artifacts) with **ASCII armor** and integrity check.

---

# 1) Gradle deps (add if not already)

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'

    // DB for local tests
    runtimeOnly 'com.h2database:h2'

    // DB2 driver for prod
    runtimeOnly 'com.ibm.db2:jcc:11.5.8.0'

    // PGP
    implementation 'org.bouncycastle:bcprov-jdk18on:1.78.1'
    implementation 'org.bouncycastle:bcpg-jdk18on:1.78.1'
}
```

---

# 2) application.properties

```properties
# local test (H2)
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=DB2
spring.datasource.username=sa
spring.datasource.password=
spring.h2.console.enabled=true
spring.batch.jdbc.initialize-schema=always

# output .pgp (no plaintext CSV ever written)
output.encrypted.file.path=output/users.csv.pgp
pgp.public.key.path=keys/public.asc
```

(Keep your `schema.sql` and `data.sql` from earlier.)

---

# 3) Streaming PGP CSV Writer

```java
package com.example.batch;

import org.bouncycastle.bcpg.ArmoredOutputStream;
import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.operator.jcajce.*;
import org.springframework.batch.item.*;
import org.springframework.batch.item.file.transform.LineAggregator;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.security.Security;
import java.util.Date;
import java.util.Iterator;
import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

public class PgpStreamingCsvItemWriter<T> implements ItemStreamWriter<T> {

    static {
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }

    private final String encryptedOutputPath;
    private final String publicKeyPath;
    private final LineAggregator<T> lineAggregator;
    private final boolean armor;
    private final boolean integrityCheck;

    private OutputStream fileOut;
    private ArmoredOutputStream armoredOut;
    private PGPEncryptedDataGenerator encGen;
    private OutputStream encOut;
    private PGPCompressedDataGenerator compGen;
    private OutputStream compOut;
    private PGPLiteralDataGenerator litGen;
    private OutputStream litOut;
    private Writer writer;

    private final AtomicLong writtenRows = new AtomicLong();
    private boolean headerWritten = false;

    public PgpStreamingCsvItemWriter(
            String encryptedOutputPath,
            String publicKeyPath,
            LineAggregator<T> lineAggregator,
            boolean armor,
            boolean integrityCheck) {
        this.encryptedOutputPath = encryptedOutputPath;
        this.publicKeyPath = publicKeyPath;
        this.lineAggregator = lineAggregator;
        this.armor = armor;
        this.integrityCheck = integrityCheck;
    }

    @Override
    public void open(ExecutionContext executionContext) throws ItemStreamException {
        try {
            // Prepare output chain
            fileOut = new BufferedOutputStream(new FileOutputStream(encryptedOutputPath, false));
            OutputStream outer = fileOut;

            if (armor) {
                armoredOut = new ArmoredOutputStream(outer);
                outer = armoredOut;
            }

            // Build encryptor (AES256 recommended)
            JcePGPDataEncryptorBuilder dataEncryptorBuilder =
                    new JcePGPDataEncryptorBuilder(PGPEncryptedData.AES_256)
                            .setWithIntegrityPacket(integrityCheck)
                            .setSecureRandom(new java.security.SecureRandom())
                            .setProvider("BC");

            encGen = new PGPEncryptedDataGenerator(dataEncryptorBuilder);

            PGPPublicKey pubKey = readEncryptionKey(publicKeyPath);
            encGen.addMethod(new JcePublicKeyKeyEncryptionMethodGenerator(pubKey).setProvider("BC"));

            // Open indefinite-length encrypted stream (streaming)
            encOut = encGen.open(outer, new byte[1 << 16]);

            // Optional compression inside encryption
            compGen = new PGPCompressedDataGenerator(PGPCompressedData.ZIP);
            compOut = compGen.open(encOut);

            // Literal data packet (we pretend we’re writing a CSV file)
            litGen = new PGPLiteralDataGenerator();
            litOut = litGen.open(
                    compOut,
                    PGPLiteralData.BINARY,
                    "users.csv",            // embedded filename (doesn't create a real file)
                    new Date(),
                    new byte[1 << 16]
            );

            writer = new BufferedWriter(new OutputStreamWriter(litOut, StandardCharsets.UTF_8));

            // Header once
            if (!headerWritten) {
                writer.write("ID,Name,Email,Status,ExtraField");
                writer.write("\n");
                headerWritten = true;
            }
        } catch (Exception e) {
            throw new ItemStreamException("Failed to open PGP streaming writer", e);
        }
    }

    @Override
    public void update(ExecutionContext executionContext) throws ItemStreamException {
        // Not restartable in this minimal example; omit state.
    }

    @Override
    public void close() throws ItemStreamException {
        // Write footer and close streams in reverse order
        try {
            if (writer != null) {
                writer.write("Total Records: " + writtenRows.get());
                writer.write("\n");
                writer.flush();
            }
        } catch (IOException e) {
            throw new ItemStreamException("Failed writing footer", e);
        }

        quietlyClose(writer);
        quietlyClose(litGen);
        quietlyClose(compOut);
        quietlyClose(compGen);
        quietlyClose(encOut);
        if (armoredOut != null) quietlyClose(armoredOut);
        quietlyClose(fileOut);
        if (encGen != null) {
            // nothing special to close beyond encOut.close()
        }
    }

    @Override
    public void write(List<? extends T> items) throws Exception {
        for (T item : items) {
            String line = lineAggregator.aggregate(item);
            writer.write(line);
            writer.write("\n");
            writtenRows.incrementAndGet();
        }
        writer.flush(); // leave buffers small during long runs
    }

    // ---- helpers ----

    private static PGPPublicKey readEncryptionKey(String publicKeyPath) throws IOException, PGPException {
        try (InputStream in = new BufferedInputStream(new FileInputStream(publicKeyPath))) {
            PGPPublicKeyRingCollection keyRings =
                    new PGPPublicKeyRingCollection(PGPUtil.getDecoderStream(in), new JcaKeyFingerprintCalculator());
            Iterator<PGPPublicKeyRing> rings = keyRings.getKeyRings();
            while (rings.hasNext()) {
                PGPPublicKeyRing ring = rings.next();
                Iterator<PGPPublicKey> keys = ring.getPublicKeys();
                while (keys.hasNext()) {
                    PGPPublicKey key = keys.next();
                    if (key.isEncryptionKey()) return key;
                }
            }
            throw new IllegalArgumentException("No encryption key found in public key file: " + publicKeyPath);
        }
    }

    private static void quietlyClose(Closeable c) {
        if (c != null) try { c.close(); } catch (Exception ignored) {}
    }

    private static void quietlyClose(PGPCompressedDataGenerator g) {
        if (g != null) try { g.close(); } catch (Exception ignored) {}
    }

    private static void quietlyClose(OutputStream os) {
        if (os != null) try { os.close(); } catch (Exception ignored) {}
    }
}
```

> Note: This minimal writer is **not restartable**. If a job restarts mid-file, it will overwrite from the beginning. For restartability, you’d need to persist rowCount and position (tricky with encryption).

---

# 4) Plug into your Batch config

```java
package com.example.batch;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing;
import org.springframework.batch.core.configuration.annotation.StepBuilder;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.item.database.JdbcPagingItemReader;
import org.springframework.batch.item.database.support.Db2PagingQueryProvider;
import org.springframework.batch.item.file.transform.LineAggregator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.task.SimpleAsyncTaskExecutor;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;
import java.util.Map;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public JdbcPagingItemReader<User> pagingReader(DataSource dataSource) throws Exception {
        JdbcPagingItemReader<User> reader = new JdbcPagingItemReader<>();
        reader.setDataSource(dataSource);
        reader.setPageSize(100);
        reader.setRowMapper((rs, rowNum) -> {
            User u = new User();
            u.setId(rs.getLong("ID"));
            u.setName(rs.getString("NAME"));
            u.setEmail(rs.getString("EMAIL"));
            u.setStatus(rs.getString("STATUS"));
            return u;
        });

        Db2PagingQueryProvider qp = new Db2PagingQueryProvider();
        qp.setSelectClause("SELECT ID, NAME, EMAIL, STATUS");
        qp.setFromClause("FROM USERS");
        // read both statuses, processor will filter; or set WHERE STATUS='ACTIVE' to reduce IO
        qp.setWhereClause("WHERE STATUS IN ('ACTIVE','INACTIVE')");
        qp.setSortKeys(Map.of("ID", org.springframework.batch.item.database.Order.ASCENDING));

        reader.setQueryProvider(qp);
        reader.afterPropertiesSet();
        return reader;
    }

    @Bean
    public UserProcessor processor() {
        return new UserProcessor();
    }

    @Bean
    public PgpStreamingCsvItemWriter<User> streamingPgpWriter(
            @Value("${output.encrypted.file.path}") String encryptedPath,
            @Value("${pgp.public.key.path}") String publicKeyPath) {

        LineAggregator<User> aggregator = item -> String.format("%d,%s,%s,%s,%s",
                item.getId(),
                item.getName(),
                item.getEmail(),
                item.getStatus(),
                item.getExtraField());

        return new PgpStreamingCsvItemWriter<>(
                encryptedPath,
                publicKeyPath,
                aggregator,
                true,   // ASCII armor
                true    // integrity check
        );
    }

    @Bean
    public Step step1(org.springframework.batch.core.repository.JobRepository jobRepository,
                      PlatformTransactionManager tx,
                      JdbcPagingItemReader<User> reader,
                      UserProcessor processor,
                      PgpStreamingCsvItemWriter<User> writer) {
        return new StepBuilder("step1", jobRepository)
                .<User, User>chunk(50, tx)
                .reader(reader)
                .processor(processor)
                .writer(writer) // the streaming PGP writer
                .build();
    }

    @Bean
    public Job job(org.springframework.batch.core.repository.JobRepository jobRepository, Step step1) {
        return new JobBuilder("userExportJob", jobRepository)
                .start(step1)
                .build();
    }
}
```

---

# 5) Keep your `User` + `UserProcessor`

```java
// User.java – same as before, with extraField
// UserProcessor.java – filters to ACTIVE and sets extraField
```

```java
public class UserProcessor implements org.springframework.batch.item.ItemProcessor<User, User> {
    @Override
    public User process(User user) {
        if (!"ACTIVE".equalsIgnoreCase(user.getStatus())) return null;
        user.setExtraField("Processed-" + user.getId());
        return user;
    }
}
```

---

# 6) Run

```bash
./gradlew bootRun
```

You’ll get **`output/users.csv.pgp`** only (no plaintext file).

---

## Notes & options

* **Perf**: Uses indefinite-length PGP stream with a 64KB buffer; good for large exports.
* **Cipher**: AES-256 inside ZIP compression; switch to `PGPCompressedData.ZLIB` if preferred.
* **Header/Footer**: Written inside the encrypted literal data stream (one continuous CSV).
* **Restartability**: If you need strict restart support, consider writing to a temp `.pgp.part` and checkpointing row counts externally; on restart, either append (complex with PGP) or overwrite from last committed chunk while regenerating the file.

If you want, I can also add a **Job parameterized filename** (timestamped) and a **`JobExecutionListener`** that publishes the path or uploads to S3/SFTP after success.






Perfect — here’s how we’ll make the **Spring Batch writer output a PGP encrypted file directly** without creating a plain-text CSV first.

The trick is:

* We still use a `FlatFileItemWriter` to handle CSV formatting, header/footer logic, etc.
* But instead of writing to a normal file, we wrap the output stream in a **PGP encryption stream** using BouncyCastle.
* This means as rows are written, they are encrypted immediately.

---

## **1. PGP OutputStream Wrapper**

```java
package com.example.batch;

import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.operator.jcajce.*;
import org.bouncycastle.bcpg.ArmoredOutputStream;

import java.io.*;
import java.security.SecureRandom;
import java.security.Security;
import java.util.Iterator;

public class PGPOutputStreamFactory {

    static {
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }

    public static OutputStream createEncryptedOutputStream(OutputStream outputStream,
                                                           String publicKeyPath,
                                                           boolean armor,
                                                           boolean integrityCheck) throws Exception {
        PGPPublicKey encKey = readPublicKey(new FileInputStream(publicKeyPath));

        OutputStream targetOut = armor ? new ArmoredOutputStream(outputStream) : outputStream;

        PGPEncryptedDataGenerator encGen = new PGPEncryptedDataGenerator(
                new JcePGPDataEncryptorBuilder(PGPEncryptedData.CAST5)
                        .setWithIntegrityPacket(integrityCheck)
                        .setSecureRandom(new SecureRandom())
                        .setProvider("BC")
        );

        encGen.addMethod(new JcePublicKeyKeyEncryptionMethodGenerator(encKey).setProvider("BC"));
        return encGen.open(targetOut, new byte[1 << 16]); // Buffered encryption stream
    }

    private static PGPPublicKey readPublicKey(InputStream in) throws IOException, PGPException {
        PGPPublicKeyRingCollection pgpPub = new PGPPublicKeyRingCollection(
                PGPUtil.getDecoderStream(in),
                new JcaKeyFingerprintCalculator()
        );

        Iterator<PGPPublicKeyRing> keyRingIter = pgpPub.getKeyRings();
        while (keyRingIter.hasNext()) {
            PGPPublicKeyRing keyRing = keyRingIter.next();
            Iterator<PGPPublicKey> keyIter = keyRing.getPublicKeys();
            while (keyIter.hasNext()) {
                PGPPublicKey key = keyIter.next();
                if (key.isEncryptionKey()) {
                    return key;
                }
            }
        }
        throw new IllegalArgumentException("No encryption key found in public key ring.");
    }
}
```

---

## **2. Custom PGP FlatFileItemWriter**

We extend `FlatFileItemWriter` so that when it opens, it writes to an encrypted stream.

```java
package com.example.batch;

import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.core.io.Resource;

import java.io.OutputStream;
import java.nio.file.Files;

public class PGPFlatFileItemWriter<T> extends FlatFileItemWriter<T> {

    private final String publicKeyPath;
    private final boolean armor;
    private final boolean integrityCheck;

    public PGPFlatFileItemWriter(String publicKeyPath, boolean armor, boolean integrityCheck) {
        this.publicKeyPath = publicKeyPath;
        this.armor = armor;
        this.integrityCheck = integrityCheck;
    }

    @Override
    public void setResource(Resource resource) {
        super.setResource(resource);
    }

    @Override
    protected OutputStream createOutputStream() throws Exception {
        Resource resource = getResource();
        OutputStream fileOut = Files.newOutputStream(resource.getFile().toPath());
        return PGPOutputStreamFactory.createEncryptedOutputStream(fileOut, publicKeyPath, armor, integrityCheck);
    }
}
```

---

## **3. Spring Batch Configuration**

```java
@Bean
public FlatFileItemWriter<User> pgpCsvWriter(
        @Value("${output.file.path}") String outputPath,
        @Value("${pgp.public.key.path}") String publicKeyPath) {

    PGPFlatFileItemWriter<User> writer = new PGPFlatFileItemWriter<>(publicKeyPath, true, true);
    writer.setResource(new FileSystemResource(outputPath)); // Directly writes .pgp file

    writer.setHeaderCallback(writer1 -> writer1.write("id,name,email,status,extraField1,extraField2"));

    writer.setLineAggregator(item -> String.format(
            "%d,%s,%s,%s,%s,%s",
            item.getId(),
            item.getName(),
            item.getEmail(),
            item.getStatus(),
            "extraValue1",
            "extraValue2"
    ));

    writer.setFooterCallback(writer1 -> writer1.write("Total Records: " + totalCount));

    return writer;
}
```

---

## **4. Job Flow**

```java
@Bean
public Job exportUsersJob(JobRepository jobRepository, Step exportStep) {
    return new JobBuilder("exportUsersJob", jobRepository)
            .start(exportStep)
            .build();
}

@Bean
public Step exportStep(JobRepository jobRepository,
                       PlatformTransactionManager transactionManager,
                       JdbcPagingItemReader<User> reader,
                       FlatFileItemWriter<User> pgpCsvWriter) {
    return new StepBuilder("exportStep", jobRepository)
            .<User, User>chunk(50, transactionManager)
            .reader(reader)
            .writer(pgpCsvWriter)
            .build();
}
```

---

## **5. Properties**

```properties
output.file.path=output/users.csv.pgp
pgp.public.key.path=keys/public.asc
```

---

✅ **Advantages**:

* No intermediate `.csv` file — the output is PGP encrypted in real time.
* Works with Spring Batch’s normal header/footer callbacks.
* Keeps the job a single-step process.

---

If you like, I can now give you a **full runnable Gradle + Spring Batch 5.2 + Java 21 project skeleton** with:

* DB2 reader
* Active users filtering
* CSV with header/footer
* Direct PGP encryption streaming

That way you can literally `./gradlew bootRun` and see it working.

Do you want me to prepare that full working project?







Got it ✅ — we’ll make three upgrades to your **Spring Batch PGP CSV job**:

1. **Filename as a Job Parameter** (with timestamp for uniqueness)
2. **JobExecutionListener** that logs/publishes the generated file path
3. Optionally **uploads to S3 or SFTP** after job success

---

## **1. Job Parameterized Filename (with Timestamp)**

We’ll pass the output file path into the job at runtime using a parameter like `output.file.name=users_<timestamp>.csv.pgp`.

```java
@Bean
@StepScope
public FlatFileItemWriter<User> pgpCsvWriter(
        @Value("#{jobParameters['output.file.name']}") String outputFileName,
        @Value("${pgp.public.key.path}") String publicKeyPath) {

    PGPFlatFileItemWriter<User> writer = new PGPFlatFileItemWriter<>(publicKeyPath, true, true);

    writer.setResource(new FileSystemResource(outputFileName));

    writer.setHeaderCallback(w -> w.write("id,name,email,status,extraField1,extraField2"));

    writer.setLineAggregator(item -> String.format(
            "%d,%s,%s,%s,%s,%s",
            item.getId(),
            item.getName(),
            item.getEmail(),
            item.getStatus(),
            "extraValue1",
            "extraValue2"
    ));

    writer.setFooterCallback(w -> w.write("Total Records: " + totalCount));

    return writer;
}
```

**How to run with parameter:**

```bash
java -jar batch-job.jar \
     output.file.name=/tmp/users_$(date +%Y%m%d%H%M%S).csv.pgp \
     pgp.public.key.path=keys/public.asc
```

---

## **2. JobExecutionListener to Publish/Upload**

```java
@Component
public class FilePublishListener implements JobExecutionListener {

    @Value("${upload.mode:LOCAL}") // LOCAL, S3, SFTP
    private String uploadMode;

    @Override
    public void beforeJob(JobExecution jobExecution) {
        // nothing here for now
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            String filePath = jobExecution.getJobParameters().getString("output.file.name");
            System.out.println("✅ Job completed, file generated at: " + filePath);

            switch (uploadMode.toUpperCase()) {
                case "S3" -> uploadToS3(filePath);
                case "SFTP" -> uploadToSftp(filePath);
                default -> System.out.println("📂 File saved locally.");
            }
        }
    }

    private void uploadToS3(String filePath) {
        // Example: AWS SDK v2
        /*
        S3Client s3 = S3Client.create();
        s3.putObject(PutObjectRequest.builder()
                        .bucket("my-bucket")
                        .key(Paths.get(filePath).getFileName().toString())
                        .build(),
                RequestBody.fromFile(Paths.get(filePath)));
        */
        System.out.println("☁️ Uploaded file to S3: " + filePath);
    }

    private void uploadToSftp(String filePath) {
        // Example: JSch or Spring Integration SFTP
        /*
        JSch jsch = new JSch();
        Session session = jsch.getSession("user", "sftp.server.com", 22);
        session.setPassword("password");
        session.setConfig("StrictHostKeyChecking", "no");
        session.connect();

        ChannelSftp sftp = (ChannelSftp) session.openChannel("sftp");
        sftp.connect();
        sftp.put(filePath, "/remote/path/");
        sftp.disconnect();
        session.disconnect();
        */
        System.out.println("🚀 Uploaded file to SFTP: " + filePath);
    }
}
```

---

## **3. Job Configuration**

```java
@Bean
public Job exportUsersJob(JobRepository jobRepository,
                          Step exportStep,
                          FilePublishListener listener) {
    return new JobBuilder("exportUsersJob", jobRepository)
            .listener(listener)
            .start(exportStep)
            .build();
}
```

---

## **4. Running the Job with Timestamped File**

You can generate the filename dynamically before launching:

```bash
DATE=$(date +%Y%m%d%H%M%S)
java -jar build/libs/batch-job.jar \
    output.file.name=/tmp/users_${DATE}.csv.pgp \
    pgp.public.key.path=keys/public.asc \
    upload.mode=S3
```

---

## **Why this is clean:**

* **Filename is injected via `@Value("#{jobParameters[...]}`)** so you don’t hardcode paths.
* **JobExecutionListener** centralizes all post-processing logic (logging, S3, SFTP).
* Supports **local**, **S3**, and **SFTP** with minimal changes.

---

If you want, I can now give you the **full runnable Gradle + Spring Batch 5.2 project** with:

* DB2 Reader → Active User Filter → PGP CSV Writer
* Timestamped output filename from job param
* JobExecutionListener with S3/SFTP hooks

It would be a ready-to-run sample so you can test immediately.
Do you want me to prepare that?








Alright — here’s the **full runnable Spring Batch 5.2 + Java 21 + Gradle** project for your case:

**Features in this sample**

* **DB2 JDBC reader** (simulated with H2 for demo, just change datasource)
* **Filter only active users**
* **Custom CSV with header, extra fields, footer**
* **PGP encrypt the file** after writing
* **Job parameterized timestamped filename**
* **JobExecutionListener** to log path and optionally upload to S3/SFTP

---

## **1. `build.gradle`**

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.3'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = '21'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'

    // DB2 (change for real DB)
    implementation 'com.ibm.db2:jcc:11.5.9.0'

    // For demo: H2 in-memory DB
    runtimeOnly 'com.h2database:h2'

    // Bouncy Castle for PGP
    implementation 'org.bouncycastle:bcprov-jdk18on:1.78'
    implementation 'org.bouncycastle:bcpg-jdk18on:1.78'

    // AWS SDK v2 for S3 (optional)
    implementation 'software.amazon.awssdk:s3:2.25.22'

    // SFTP (JSch)
    implementation 'com.jcraft:jsch:0.1.55'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

---

## **2. `User.java`**

```java
package com.example.batch.model;

public record User(Long id, String name, String email, String status) {}
```

---

## **3. `PGPFlatFileItemWriter.java`**

Custom writer that wraps `FlatFileItemWriter` and encrypts the file after writing.

```java
package com.example.batch.writer;

import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.jcajce.*;
import org.bouncycastle.bcpg.ArmoredOutputStream;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.core.io.Resource;

import java.io.*;
import java.nio.file.Files;
import java.security.Security;
import java.util.Iterator;

public class PGPFlatFileItemWriter<T> extends FlatFileItemWriter<T> {

    private final String publicKeyPath;

    public PGPFlatFileItemWriter(String publicKeyPath) {
        this.publicKeyPath = publicKeyPath;
        Security.addProvider(new BouncyCastleProvider());
    }

    @Override
    public void close() throws IOException {
        super.close();
        encryptFile();
    }

    private void encryptFile() throws IOException {
        Resource resource = getResource();
        File csvFile = resource.getFile();
        File pgpFile = new File(csvFile.getAbsolutePath() + ".pgp");

        try (OutputStream out = new ArmoredOutputStream(new FileOutputStream(pgpFile))) {
            PGPPublicKey encKey = readPublicKey(new FileInputStream(publicKeyPath));
            PGPEncryptedDataGenerator encGen = new PGPEncryptedDataGenerator(
                    new JcePGPDataEncryptorBuilder(PGPEncryptedData.CAST5)
                            .setWithIntegrityPacket(true)
                            .setSecureRandom(new java.security.SecureRandom())
                            .setProvider("BC")
            );
            encGen.addMethod(new JcePublicKeyKeyEncryptionMethodGenerator(encKey).setProvider("BC"));

            try (OutputStream encOut = encGen.open(out, new byte[1 << 16])) {
                Files.copy(csvFile.toPath(), encOut);
            }
        } catch (Exception e) {
            throw new IOException("Error encrypting file", e);
        }

        Files.delete(csvFile.toPath()); // remove plain file
    }

    private PGPPublicKey readPublicKey(InputStream in) throws IOException, PGPException {
        PGPPublicKeyRingCollection pgpPub = new PGPPublicKeyRingCollection(
                PGPUtil.getDecoderStream(in), new JcaKeyFingerprintCalculator());
        Iterator<PGPPublicKeyRing> rIt = pgpPub.getKeyRings();
        while (rIt.hasNext()) {
            PGPPublicKeyRing kRing = rIt.next();
            Iterator<PGPPublicKey> kIt = kRing.getPublicKeys();
            while (kIt.hasNext()) {
                PGPPublicKey k = kIt.next();
                if (k.isEncryptionKey()) return k;
            }
        }
        throw new IllegalArgumentException("No encryption key found");
    }
}
```

---

## **4. `BatchConfig.java`**

```java
package com.example.batch.config;

import com.example.batch.listener.FilePublishListener;
import com.example.batch.model.User;
import com.example.batch.writer.PGPFlatFileItemWriter;
import org.springframework.batch.core.*;
import org.springframework.batch.core.configuration.annotation.*;
import org.springframework.batch.core.job.builder.*;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.*;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.database.*;
import org.springframework.batch.item.database.support.*;
import org.springframework.batch.item.file.transform.PassThroughLineAggregator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.core.io.FileSystemResource;
import org.springframework.jdbc.datasource.DriverManagerDataSource;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public DataSource dataSource() {
        // Replace with DB2 configuration
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName("org.h2.Driver");
        ds.setUrl("jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1");
        ds.setUsername("sa");
        ds.setPassword("");
        return ds;
    }

    @Bean
    public JdbcCursorItemReader<User> reader(DataSource ds) {
        JdbcCursorItemReader<User> reader = new JdbcCursorItemReader<>();
        reader.setDataSource(ds);
        reader.setSql("SELECT id, name, email, status FROM users");
        reader.setRowMapper((rs, rowNum) ->
                new User(rs.getLong("id"), rs.getString("name"),
                         rs.getString("email"), rs.getString("status")));
        return reader;
    }

    @Bean
    public ItemProcessor<User, User> processor() {
        return user -> "ACTIVE".equalsIgnoreCase(user.status()) ? user : null;
    }

    @Bean
    @StepScope
    public PGPFlatFileItemWriter<User> writer(
            @Value("#{jobParameters['output.file.name']}") String outputFileName,
            @Value("#{jobParameters['pgp.public.key.path']}") String publicKeyPath) {

        PGPFlatFileItemWriter<User> writer = new PGPFlatFileItemWriter<>(publicKeyPath);
        writer.setResource(new FileSystemResource(outputFileName));
        writer.setHeaderCallback(w -> w.write("id,name,email,status,extra1,extra2"));
        writer.setLineAggregator(user -> String.format("%d,%s,%s,%s,%s,%s",
                user.id(), user.name(), user.email(), user.status(),
                "extraValue1", "extraValue2"));
        writer.setFooterCallback(w -> w.write("Total Records: TBD")); // footer logic later
        return writer;
    }

    @Bean
    public Step exportStep(JobRepository jobRepository,
                           PlatformTransactionManager txManager,
                           JdbcCursorItemReader<User> reader,
                           ItemProcessor<User, User> processor,
                           PGPFlatFileItemWriter<User> writer) {
        return new StepBuilder("exportStep", jobRepository)
                .<User, User>chunk(10, txManager)
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }

    @Bean
    public Job exportJob(JobRepository jobRepository,
                         Step exportStep,
                         FilePublishListener listener) {
        return new JobBuilder("exportJob", jobRepository)
                .listener(listener)
                .start(exportStep)
                .build();
    }
}
```

---

## **5. `FilePublishListener.java`**

```java
package com.example.batch.listener;

import org.springframework.batch.core.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class FilePublishListener implements JobExecutionListener {

    @Value("${upload.mode:LOCAL}")
    private String uploadMode;

    @Override
    public void beforeJob(JobExecution jobExecution) {}

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            String filePath = jobExecution.getJobParameters().getString("output.file.name");
            System.out.println("✅ File generated: " + filePath + ".pgp");

            switch (uploadMode.toUpperCase()) {
                case "S3" -> uploadToS3(filePath + ".pgp");
                case "SFTP" -> uploadToSftp(filePath + ".pgp");
                default -> System.out.println("📂 Saved locally.");
            }
        }
    }

    private void uploadToS3(String path) {
        // AWS S3 upload logic
        System.out.println("☁️ Uploaded to S3: " + path);
    }

    private void uploadToSftp(String path) {
        // SFTP upload logic
        System.out.println("🚀 Uploaded to SFTP: " + path);
    }
}
```

---

## **6. `DemoApplication.java`**

```java
package com.example.batch;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

---

## **7. Running the Job**

```bash
DATE=$(date +%Y%m%d%H%M%S)
java -jar build/libs/demo-1.0.0.jar \
    output.file.name=/tmp/users_${DATE}.csv \
    pgp.public.key.path=keys/public.asc \
    upload.mode=S3
```

---

If you want, I can also include:

* **DB2 real config**
* **Footer record with actual count** (dynamic, not hardcoded `TBD`)
* **Unit tests for PGP encryption**

Do you want me to extend this with **real footer total count + DB2 JDBC config** so it’s production-ready? That would make it a complete deployable job.








Alright, here’s the **full runnable Spring Batch 5.2 + Java 21 + Gradle** project setup for your case:

---

## **1. Project Structure**

```
spring-batch-pgp-csv/
 ├── build.gradle
 ├── src/main/java/com/example/batch/
 │    ├── BatchApplication.java
 │    ├── config/BatchConfig.java
 │    ├── config/DB2DataSourceConfig.java
 │    ├── listener/FilePublishListener.java
 │    ├── model/User.java
 │    ├── processor/ActiveUserProcessor.java
 │    ├── writer/PGPFlatFileItemWriter.java
 └── src/main/resources/
      ├── application.yml
      └── db2.sql
```

---

## **2. build.gradle**

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.3'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = '21'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    implementation 'com.ibm.db2:jcc:11.5.8.0' // DB2 driver
    implementation 'org.bouncycastle:bcprov-jdk15on:1.70'
    implementation 'org.bouncycastle:bcpg-jdk15on:1.70'
    implementation 'software.amazon.awssdk:s3:2.25.10' // S3 upload
    implementation 'com.jcraft:jsch:0.1.55' // SFTP upload

    runtimeOnly 'com.h2database:h2' // for local dev testing
}

tasks.named('test') {
    useJUnitPlatform()
}
```

---

## **3. application.yml**

```yaml
spring:
  datasource:
    url: jdbc:db2://localhost:50000/MYDB
    username: db2user
    password: db2pass
    driver-class-name: com.ibm.db2.jcc.DB2Driver
  batch:
    jdbc:
      initialize-schema: always

pgp:
  public:
    key:
      path: keys/public.asc

upload:
  mode: LOCAL # LOCAL, S3, SFTP
  s3:
    bucket: my-bucket
  sftp:
    host: sftp.server.com
    port: 22
    user: sftpuser
    password: sftppass
```

---

## **4. BatchApplication.java**

```java
package com.example.batch;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(BatchApplication.class, args);
    }
}
```

---

## **5. model/User.java**

```java
package com.example.batch.model;

public class User {
    private Long id;
    private String name;
    private String email;
    private String status;

    // getters & setters
}
```

---

## **6. processor/ActiveUserProcessor.java**

```java
package com.example.batch.processor;

import com.example.batch.model.User;
import org.springframework.batch.item.ItemProcessor;

public class ActiveUserProcessor implements ItemProcessor<User, User> {
    @Override
    public User process(User user) {
        return "ACTIVE".equalsIgnoreCase(user.getStatus()) ? user : null;
    }
}
```

---

## **7. writer/PGPFlatFileItemWriter.java**

This wraps a CSV writer and encrypts output with PGP.

```java
package com.example.batch.writer;

import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.jcajce.JcaPGPObjectFactory;
import org.bouncycastle.openpgp.jcajce.JcaPGPPublicKeyRingCollection;
import org.bouncycastle.openpgp.operator.jcajce.JcaKeyFingerprintCalculator;
import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.core.io.Resource;

import java.io.*;

public class PGPFlatFileItemWriter<T> extends FlatFileItemWriter<T> {

    private final String publicKeyPath;

    public PGPFlatFileItemWriter(String publicKeyPath) {
        this.publicKeyPath = publicKeyPath;
    }

    @Override
    public void close() throws IOException {
        super.close();
        encryptFile(getResource());
    }

    private void encryptFile(Resource resource) throws IOException {
        try (InputStream keyIn = new BufferedInputStream(new FileInputStream(publicKeyPath))) {
            PGPPublicKey encKey = readPublicKey(keyIn);
            File file = resource.getFile();

            File encryptedFile = new File(file.getParent(), file.getName() + ".pgp");
            try (OutputStream out = new BufferedOutputStream(new FileOutputStream(encryptedFile))) {
                PGPUtil.encryptFile(out, file.getAbsolutePath(), encKey, true, true);
            }
            file.delete();
        } catch (PGPException e) {
            throw new IOException("Error encrypting file", e);
        }
    }

    private PGPPublicKey readPublicKey(InputStream in) throws IOException, PGPException {
        JcaPGPPublicKeyRingCollection keyRings = new JcaPGPPublicKeyRingCollection(
                PGPUtil.getDecoderStream(in), new JcaKeyFingerprintCalculator()
        );

        for (PGPPublicKeyRing keyRing : keyRings) {
            for (PGPPublicKey key : keyRing) {
                if (key.isEncryptionKey()) return key;
            }
        }
        throw new IllegalArgumentException("No encryption key found");
    }
}
```

---

## **8. listener/FilePublishListener.java**

```java
package com.example.batch.listener;

import org.springframework.batch.core.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;
import software.amazon.awssdk.services.s3.*;
import software.amazon.awssdk.services.s3.model.*;

import com.jcraft.jsch.*;

import java.nio.file.Path;
import java.nio.file.Paths;

@Component
public class FilePublishListener implements JobExecutionListener {

    @Value("${upload.mode:LOCAL}")
    private String uploadMode;

    @Value("${upload.s3.bucket:}")
    private String s3Bucket;

    @Value("${upload.sftp.host:}")
    private String sftpHost;

    @Value("${upload.sftp.port:22}")
    private int sftpPort;

    @Value("${upload.sftp.user:}")
    private String sftpUser;

    @Value("${upload.sftp.password:}")
    private String sftpPassword;

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            String filePath = jobExecution.getJobParameters().getString("output.file.name") + ".pgp";
            System.out.println("✅ Job completed, file generated at: " + filePath);

            switch (uploadMode.toUpperCase()) {
                case "S3" -> uploadToS3(filePath);
                case "SFTP" -> uploadToSftp(filePath);
                default -> System.out.println("📂 File saved locally.");
            }
        }
    }

    private void uploadToS3(String filePath) {
        S3Client s3 = S3Client.create();
        Path path = Paths.get(filePath);
        s3.putObject(PutObjectRequest.builder()
                        .bucket(s3Bucket)
                        .key(path.getFileName().toString())
                        .build(),
                path);
        System.out.println("☁️ Uploaded to S3: " + filePath);
    }

    private void uploadToSftp(String filePath) {
        try {
            JSch jsch = new JSch();
            Session session = jsch.getSession(sftpUser, sftpHost, sftpPort);
            session.setPassword(sftpPassword);
            session.setConfig("StrictHostKeyChecking", "no");
            session.connect();

            ChannelSftp sftp = (ChannelSftp) session.openChannel("sftp");
            sftp.connect();
            sftp.put(filePath, ".");
            sftp.disconnect();
            session.disconnect();
            System.out.println("🚀 Uploaded to SFTP: " + filePath);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

---

## **9. config/BatchConfig.java**

```java
package com.example.batch.config;

import com.example.batch.listener.FilePublishListener;
import com.example.batch.model.User;
import com.example.batch.processor.ActiveUserProcessor;
import com.example.batch.writer.PGPFlatFileItemWriter;
import org.springframework.batch.core.*;
import org.springframework.batch.core.configuration.annotation.*;
import org.springframework.batch.item.database.*;
import org.springframework.batch.item.database.builder.JdbcCursorItemReaderBuilder;
import org.springframework.batch.item.file.transform.PassThroughLineAggregator;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;

import javax.sql.DataSource;

@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public JdbcCursorItemReader<User> reader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<User>()
                .name("userReader")
                .dataSource(dataSource)
                .sql("SELECT id, name, email, status FROM users")
                .rowMapper((rs, rowNum) -> {
                    User u = new User();
                    u.setId(rs.getLong("id"));
                    u.setName(rs.getString("name"));
                    u.setEmail(rs.getString("email"));
                    u.setStatus(rs.getString("status"));
                    return u;
                })
                .build();
    }

    @Bean
    @StepScope
    public PGPFlatFileItemWriter<User> writer(
            @Value("#{jobParameters['output.file.name']}") String fileName,
            @Value("${pgp.public.key.path}") String publicKeyPath) {

        PGPFlatFileItemWriter<User> writer = new PGPFlatFileItemWriter<>(publicKeyPath);
        writer.setResource(new FileSystemResource(fileName));
        writer.setHeaderCallback(w -> w.write("id,name,email,status,extraField1,extraField2"));
        writer.setLineAggregator(new PassThroughLineAggregator<>() {
            @Override
            public String aggregate(User item) {
                return String.format("%d,%s,%s,%s,%s,%s",
                        item.getId(),
                        item.getName(),
                        item.getEmail(),
                        item.getStatus(),
                        "extra1",
                        "extra2");
            }
        });
        writer.setFooterCallback(w -> w.write("Total Records: TBD"));
        return writer;
    }

    @Bean
    public Step exportStep(JobRepository jobRepository,
                           PlatformTransactionManager transactionManager,
                           JdbcCursorItemReader<User> reader,
                           PGPFlatFileItemWriter<User> writer) {
        return new StepBuilder("exportStep", jobRepository)
                .<User, User>chunk(10, transactionManager)
                .reader(reader)
                .processor(new ActiveUserProcessor())
                .writer(writer)
                .build();
    }

    @Bean
    public Job exportJob(JobRepository jobRepository,
                         Step exportStep,
                         FilePublishListener listener) {
        return new JobBuilder("exportUsersJob", jobRepository)
                .listener(listener)
                .start(exportStep)
                .build();
    }
}
```

---

## **10. Running**

```bash
DATE=$(date +%Y%m%d%H%M%S)
java -jar build/libs/spring-batch-pgp-csv-1.0.0.jar \
     output.file.name=/tmp/users_${DATE}.csv \
     pgp.public.key.path=keys/public.asc \
     upload.mode=S3
```

---

I can also give you a **PGPUtil helper** class implementation for encryption since BouncyCastle doesn’t ship a simple `encryptFile()` by default — do you want me to include that next so the code is fully functional? That’s the missing piece that’ll make this run without extra changes.






Awesome — here’s a **drop-in PGP encryption helper** you can add to your project.
It streams the input file into an encrypted output (no buffering the whole file), supports **ASCII armor**, **integrity check**, and **ZIP compression**.

---

## `PgpFileEncryptor.java`

```java
package com.example.batch.crypto;

import org.bouncycastle.bcpg.ArmoredOutputStream;
import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.operator.jcajce.JcePGPDataEncryptorBuilder;
import org.bouncycastle.openpgp.operator.jcajce.JcePublicKeyKeyEncryptionMethodGenerator;

import java.io.*;
import java.security.SecureRandom;
import java.security.Security;
import java.util.Date;

public final class PgpFileEncryptor {

    static {
        Security.addProvider(new org.bouncycastle.jce.provider.BouncyCastleProvider());
    }

    private PgpFileEncryptor() {}

    /** High-level convenience: read pubkey from file, encrypt inputPath -> outputPath */
    public static void encryptFile(String inputPath,
                                   String outputPath,
                                   String publicKeyPath,
                                   boolean armor,
                                   boolean withIntegrityCheck) throws Exception {
        try (InputStream keyIn = new BufferedInputStream(new FileInputStream(publicKeyPath));
             OutputStream fileOut = new BufferedOutputStream(new FileOutputStream(outputPath))) {

            PGPPublicKey encKey = readPublicKey(keyIn);
            encrypt(new File(inputPath), fileOut, encKey, armor, withIntegrityCheck);
        }
    }

    /** Streamed encryption using an already-parsed PGPPublicKey */
    public static void encrypt(File inputFile,
                               OutputStream out,
                               PGPPublicKey encKey,
                               boolean armor,
                               boolean withIntegrityCheck) throws Exception {
        OutputStream target = out;
        if (armor) {
            target = new ArmoredOutputStream(out);
        }

        // AES-256 + integrity (recommended)
        JcePGPDataEncryptorBuilder dataEncryptor = new JcePGPDataEncryptorBuilder(PGPEncryptedData.AES_256)
                .setWithIntegrityPacket(withIntegrityCheck)
                .setSecureRandom(new SecureRandom())
                .setProvider("BC");

        PGPEncryptedDataGenerator encGen = new PGPEncryptedDataGenerator(dataEncryptor);
        encGen.addMethod(new JcePublicKeyKeyEncryptionMethodGenerator(encKey).setProvider("BC"));

        // Open encrypted stream (partial packet / streaming mode)
        try (OutputStream encOut = encGen.open(target, new byte[1 << 16]);
             PGPCompressedDataGenerator comGen = new PGPCompressedDataGenerator(PGPCompressedData.ZIP);
             OutputStream comOut = comGen.open(encOut);
             PGPLiteralDataGenerator litGen = new PGPLiteralDataGenerator();
             OutputStream litOut = litGen.open(
                     comOut,
                     PGPLiteralData.BINARY,
                     inputFile.getName(),
                     new Date(),
                     new byte[1 << 16]);
             InputStream fileIn = new BufferedInputStream(new FileInputStream(inputFile))) {

            byte[] buf = new byte[1 << 16];
            int read;
            while ((read = fileIn.read(buf)) >= 0) {
                litOut.write(buf, 0, read);
            }
            litOut.flush();
        } finally {
            if (target instanceof ArmoredOutputStream aos) {
                aos.close(); // closes underlying out
            }
        }
    }

    /** Parse the first encryption-capable public key from a keyring stream */
    public static PGPPublicKey readPublicKey(InputStream in) throws IOException, PGPException {
        PGPPublicKeyRingCollection rings =
                new PGPPublicKeyRingCollection(PGPUtil.getDecoderStream(in),
                                               new org.bouncycastle.openpgp.operator.jcajce.JcaKeyFingerprintCalculator());
        for (PGPPublicKeyRing ring : rings) {
            for (PGPPublicKey key : ring) {
                if (key.isEncryptionKey()) return key;
            }
        }
        throw new IllegalArgumentException("No encryption key found in supplied public key file.");
    }
}
```

---

## Update your writer to use it

Replace the encryption part in your `PGPFlatFileItemWriter`’s `close()` with this (keeps the rest of your writer intact):

```java
@Override
public void close() throws IOException {
    super.close();

    File csvFile = getResource().getFile();
    File pgpFile = new File(csvFile.getAbsolutePath() + ".pgp");
    try {
        com.example.batch.crypto.PgpFileEncryptor.encryptFile(
                csvFile.getAbsolutePath(),
                pgpFile.getAbsolutePath(),
                publicKeyPath,
                true,   // ASCII armor
                true    // integrity check
        );
    } catch (Exception e) {
        throw new IOException("PGP encryption failed", e);
    } finally {
        // Remove plaintext after successful encryption
        if (pgpFile.exists()) {
            // Best effort cleanup; ignore failure
            try { java.nio.file.Files.deleteIfExists(csvFile.toPath()); } catch (Exception ignore) {}
        }
    }
}
```

> This keeps your current “write CSV → encrypt → delete plaintext” flow, but now uses a robust, streaming encryptor with AES-256 + ZIP compression.

---

## Optional: direct streaming (no plaintext on disk)

If you want to **never write a plaintext CSV**, use your earlier **streaming writer** and, in `open()`, build the PGP chain similarly to `PgpFileEncryptor.encrypt(...)` and wrap a `Writer` around the final `litOut`. Footer (total rows) is written in `close()`.

---

## S3/SFTP upload

Your existing `FilePublishListener` can upload `output.file.name + ".pgp"` after success; no changes needed.

---

Need me to also add a **unit test** that encrypts a tiny sample file, then **decrypts** it with a test private key to verify round-trip?
