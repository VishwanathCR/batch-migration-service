Here’s a clean, production-ready way to PGP-encrypt the file your Spring Batch job writes with `FlatFileItemWriter`, using Java 21, Spring Batch 5.2, and Gradle.

I’ll show:

1. Gradle deps
2. A tiny PGP utility (Bouncy Castle)
3. A tasklet that encrypts a file (streaming; no big buffers)
4. A two-step job: write → encrypt
5. Minimal writer/reader beans + properties

---

# 1) Gradle (Java 21, Spring Batch 5.x, BC OpenPGP)

```gradle
plugins {
  id 'java'
  id 'org.springframework.boot' version '3.3.2'
  id 'io.spring.dependency-management' version '1.1.6'
}

java {
  toolchain { languageVersion = JavaLanguageVersion.of(21) }
}

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-batch'

  // Bouncy Castle (OpenPGP)
  implementation 'org.bouncycastle:bcprov-jdk18on:1.78.1'
  implementation 'org.bouncycastle:bcpg-jdk18on:1.78.1'
  implementation 'org.bouncycastle:bcpkix-jdk18on:1.78.1'

  // Demo/local dev
  runtimeOnly 'com.h2database:h2'
  testImplementation 'org.springframework.batch:spring-batch-test'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

> Use the latest BC versions your org approves; the above works on JDK 21.

---

# 2) PGP utility (read public key + encrypt stream)

```java
// src/main/java/com/example/pgp/PgpUtils.java
package com.example.pgp;

import org.bouncycastle.bcpg.ArmoredOutputStream;
import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.bouncycastle.openpgp.*;
import org.bouncycastle.openpgp.operator.jcajce.*;
import org.bouncycastle.openpgp.operator.*;
import org.bouncycastle.openpgp.jcajce.JcaPGPPublicKeyRingCollection;

import java.io.*;
import java.nio.file.*;
import java.security.Security;
import java.security.SecureRandom;
import java.util.Date;

public final class PgpUtils {

    static {
        if (Security.getProvider(BouncyCastleProvider.PROVIDER_NAME) == null) {
            Security.addProvider(new BouncyCastleProvider());
        }
    }

    private PgpUtils() {}

    public static PGPPublicKey readEncryptionKey(InputStream in) throws IOException, PGPException {
        try (InputStream decoder = PGPUtil.getDecoderStream(in)) {
            PGPPublicKeyRingCollection rings = new JcaPGPPublicKeyRingCollection(decoder);
            for (PGPPublicKeyRing ring : rings) {
                for (PGPPublicKey key : ring) {
                    // Prefer keys flagged for encryption; fall back to isEncryptionKey
                    if (key.isEncryptionKey()) {
                        return key;
                    }
                }
            }
            throw new IllegalArgumentException("No encryption-capable public key found.");
        }
    }

    /**
     * Streamed encryption: input file -> .pgp output (binary or armored).
     */
    public static void encryptFile(Path inputFile,
                                   Path outputFile,
                                   PGPPublicKey publicKey,
                                   boolean armor,
                                   boolean withIntegrityCheck) throws Exception {
        Files.createDirectories(outputFile.getParent());

        OutputStream fileOut = Files.newOutputStream(outputFile);
        OutputStream maybeArmored = armor ? new ArmoredOutputStream(fileOut) : fileOut;

        JcePGPDataEncryptorBuilder dataEncryptor =
                new JcePGPDataEncryptorBuilder(PGPEncryptedData.AES_256)
                        .setWithIntegrityPacket(withIntegrityCheck)
                        .setSecureRandom(new SecureRandom())
                        .setProvider(BouncyCastleProvider.PROVIDER_NAME);

        PGPEncryptedDataGenerator encGen = new PGPEncryptedDataGenerator(dataEncryptor);
        encGen.addMethod(new JcePublicKeyKeyEncryptionMethodGenerator(publicKey)
                .setProvider(BouncyCastleProvider.PROVIDER_NAME));

        try (OutputStream encryptedOut = encGen.open(maybeArmored, new byte[1 << 16])) {
            PGPCompressedDataGenerator comGen =
                    new PGPCompressedDataGenerator(PGPCompressedData.ZIP);
            try (OutputStream compressedOut = comGen.open(encryptedOut, new byte[1 << 16])) {
                PGPLiteralDataGenerator litGen = new PGPLiteralDataGenerator();
                try (OutputStream literalOut = litGen.open(
                        compressedOut,
                        PGPLiteralData.BINARY,
                        inputFile.getFileName().toString(),   // stored filename inside the PGP payload
                        new Date(),
                        new byte[1 << 16])) {
                    Files.copy(inputFile, literalOut);       // fully streamed, no large memory spikes
                } finally {
                    litGen.close();
                }
            } finally {
                comGen.close();
            }
        } finally {
            encGen.close();
            if (armor) {
                maybeArmored.close();
            } else {
                fileOut.close();
            }
        }
    }
}
```

---

# 3) Encrypt tasklet (post-writer)

```java
// src/main/java/com/example/batch/EncryptFileTasklet.java
package com.example.batch;

import com.example.pgp.PgpUtils;
import org.bouncycastle.openpgp.PGPPublicKey;
import org.springframework.batch.core.StepContribution;
import org.springframework.batch.core.scope.context.ChunkContext;
import org.springframework.batch.core.step.tasklet.Tasklet;
import org.springframework.batch.repeat.RepeatStatus;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Component;

import java.io.InputStream;
import java.nio.file.*;

@Component
public class EncryptFileTasklet implements Tasklet {

    @Value("${app.output.plain}")
    private String plainFile;

    @Value("${app.output.encrypted}")
    private String encryptedFile;

    @Value("${app.pgp.publicKey}")          // e.g., classpath:keys/receiver_public.asc or file:/...
    private Resource publicKey;

    @Value("${app.pgp.asciiArmor:true}")    // true => .asc; false => .pgp
    private boolean asciiArmor;

    @Value("${app.pgp.integrity:true}")
    private boolean withIntegrity;

    @Value("${app.output.deletePlainAfterEncrypt:true}")
    private boolean deletePlain;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        Path in = Path.of(plainFile);
        Path out = Path.of(encryptedFile);

        try (InputStream keyIn = publicKey.getInputStream()) {
            PGPPublicKey key = PgpUtils.readEncryptionKey(keyIn);
            PgpUtils.encryptFile(in, out, key, asciiArmor, withIntegrity);
        }

        if (deletePlain) {
            Files.deleteIfExists(in);
        }
        return RepeatStatus.FINISHED;
    }
}
```

---

# 4) Batch job config (write → encrypt)

```java
// src/main/java/com/example/batch/BatchConfig.java
package com.example.batch;

import com.example.model.Person;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.*;
import org.springframework.batch.item.file.FlatFileItemWriter;
import org.springframework.batch.item.file.transform.*;
import org.springframework.batch.item.support.ListItemReader;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;
import org.springframework.transaction.PlatformTransactionManager;

import java.util.List;

@Configuration
public class BatchConfig {

    @Bean
    public Job exportAndEncryptJob(JobRepository jobRepository,
                                   Step writeCsvStep,
                                   Step encryptStep) {
        return new JobBuilder("exportAndEncryptJob", jobRepository)
                .start(writeCsvStep)
                .next(encryptStep)
                .build();
    }

    @Bean
    public Step writeCsvStep(JobRepository jobRepository,
                             PlatformTransactionManager tx,
                             ItemReader<Person> personReader,
                             ItemWriter<Person> personWriter) {
        return new StepBuilder("writeCsvStep", jobRepository)
                .<Person, Person>chunk(100, tx)
                .reader(personReader)
                .writer(personWriter)
                .build();
    }

    @Bean
    public Step encryptStep(JobRepository jobRepository,
                            PlatformTransactionManager tx,
                            EncryptFileTasklet tasklet) {
        return new StepBuilder("encryptStep", jobRepository)
                .tasklet(tasklet, tx)
                .build();
    }

    // --- Demo reader + writer ---

    @Bean
    public ItemReader<Person> personReader() {
        return new ListItemReader<>(List.of(
                new Person(1L, "Alice"),
                new Person(2L, "Bob"),
                new Person(3L, "Charlie")
        ));
    }

    @Bean
    public FlatFileItemWriter<Person> personWriter(@Value("${app.output.plain}") String outPath) {
        FlatFileItemWriter<Person> writer = new FlatFileItemWriter<>();
        writer.setResource(new FileSystemResource(outPath));
        writer.setAppendAllowed(false);
        writer.setHeaderCallback(w -> w.write("id,name"));

        BeanWrapperFieldExtractor<Person> extractor = new BeanWrapperFieldExtractor<>();
        extractor.setNames(new String[]{"id", "name"});

        DelimitedLineAggregator<Person> agg = new DelimitedLineAggregator<>();
        agg.setDelimiter(",");
        agg.setFieldExtractor(extractor);

        writer.setLineAggregator(agg);
        return writer;
    }
}
```

Model for the demo (adjust to your item type):

```java
// src/main/java/com/example/model/Person.java
package com.example.model;

public record Person(Long id, String name) {}
```

---

# 5) `application.properties`

```properties
# Where the CSV is written (step 1)
app.output.plain=build/out/people.csv

# Where the encrypted file goes (step 2)
app.output.encrypted=build/out/people.csv.pgp

# Public key to encrypt with (armored .asc or binary .gpg)
app.pgp.publicKey=classpath:keys/receiver_public.asc

# Options
app.pgp.asciiArmor=false
app.pgp.integrity=true
app.output.deletePlainAfterEncrypt=true

# Spring Batch (demo)
spring.batch.jdbc.initialize-schema=always
```

Put your recipient’s public key at `src/main/resources/keys/receiver_public.asc` (ASCII-armored) or point to a filesystem location.

---

## How it runs

* Step 1 writes `people.csv` via `FlatFileItemWriter`.
* Step 2 reads the recipient public key and **streams** `people.csv` → `people.csv.pgp` (AES-256, ZIP compression, integrity packet), then deletes the plaintext (configurable).

---

## Notes & pragmatics

* **Why a separate step?** It’s simple, reliable, and keeps your writer pure (no crypto concerns inside chunk logic). It also lets you retry just the encrypt step if needed.
* **Large files**: Everything is streamed; no loading the whole file into memory.
* **Armor vs binary**: Set `app.pgp.asciiArmor=true` if your partner expects `.asc`; otherwise `.pgp` (binary) is smaller and faster.
* **Key selection**: `readEncryptionKey` picks the first encryption-capable subkey. If you need a specific key ID, extend that method to filter by fingerprint.

---

### Optional: encrypt **in place of** the writer step

If you absolutely want to write the `.pgp` directly (no plaintext file on disk), you can implement a custom `ItemStreamWriter<T>` that opens a PGP literal stream in `open()`, writes each line’s bytes in `write()`, and closes the generators in `close()`. It’s a bit more code and trickier to test; the two-step approach above is usually the sweet spot.

---

If you paste the snippets as-is, you’ll have a working job that writes a CSV and emits its encrypted counterpart. If you want me to adapt this to your concrete item type/fields or to use job parameters for the filenames, say the word and I’ll wire it up.
