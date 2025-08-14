In **Spring Batch**, an **ItemProcessor** is used to transform, filter, or enrich the data between reading and writing.
The best example is one that demonstrates **all three** capabilities—filtering, transformation, and enrichment—without being overcomplicated.

---

## Example: Processing Customer Data

Imagine you are importing customer records from a CSV file, but:

* You **only** want active customers (filtering).
* You need to **capitalize their names** (transformation).
* You want to **add a generated loyalty tier** based on purchase amount (enrichment).

### Code Example

```java
import org.springframework.batch.item.ItemProcessor;

public class CustomerItemProcessor implements ItemProcessor<Customer, Customer> {

    @Override
    public Customer process(Customer customer) throws Exception {
        // 1. Filtering: Skip inactive customers
        if (!"ACTIVE".equalsIgnoreCase(customer.getStatus())) {
            return null; // Returning null means "skip" this record
        }

        // 2. Transformation: Capitalize names
        customer.setFirstName(capitalize(customer.getFirstName()));
        customer.setLastName(capitalize(customer.getLastName()));

        // 3. Enrichment: Add loyalty tier
        if (customer.getTotalPurchase() > 10000) {
            customer.setLoyaltyTier("GOLD");
        } else if (customer.getTotalPurchase() > 5000) {
            customer.setLoyaltyTier("SILVER");
        } else {
            customer.setLoyaltyTier("BRONZE");
        }

        return customer; // Pass processed item to the writer
    }

    private String capitalize(String str) {
        if (str == null || str.isEmpty()) return str;
        return str.substring(0, 1).toUpperCase() + str.substring(1).toLowerCase();
    }
}
```

---

### How It Works in a Job

```java
@Bean
public Step step1(ItemReader<Customer> reader,
                  ItemProcessor<Customer, Customer> processor,
                  ItemWriter<Customer> writer) {
    return stepBuilderFactory.get("step1")
        .<Customer, Customer>chunk(10)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .build();
}
```

---

✅ **Why this is a “best” example**

* Shows **filtering** (return `null` to skip an item)
* Shows **transformation** (modify existing fields)
* Shows **enrichment** (add new data based on business logic)
* Keeps the logic **business-focused** and **easy to test**

---

If you want, I can also give you a **Java 21 + Spring Batch 5.2** functional-style `ItemProcessor` example that avoids boilerplate class creation. That’s the modern approach. Would you like that next?



Alright — let’s modernize the example for **Java 21** and **Spring Batch 5.2** with the functional style `ItemProcessor` so you don’t need to create a separate class unless you want to.

---

## **Functional-Style ItemProcessor Example (Java 21 + Spring Batch 5.2)**

### Configuration

```java
import org.springframework.batch.item.ItemProcessor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class BatchConfig {

    @Bean
    public ItemProcessor<Customer, Customer> customerProcessor() {
        return customer -> {
            // 1. Filtering
            if (!"ACTIVE".equalsIgnoreCase(customer.getStatus())) {
                return null; // Skip inactive customers
            }

            // 2. Transformation
            customer.setFirstName(capitalize(customer.getFirstName()));
            customer.setLastName(capitalize(customer.getLastName()));

            // 3. Enrichment
            customer.setLoyaltyTier(
                switch ((int) customer.getTotalPurchase()) {
                    case int amount when amount > 10_000 -> "GOLD";
                    case int amount when amount > 5_000 -> "SILVER";
                    default -> "BRONZE";
                }
            );

            return customer;
        };
    }

    private String capitalize(String str) {
        if (str == null || str.isEmpty()) return str;
        return str.substring(0, 1).toUpperCase() + str.substring(1).toLowerCase();
    }
}
```

---

### Step Definition with Functional Processor

```java
@Bean
public Step customerStep(ItemReader<Customer> reader,
                         ItemProcessor<Customer, Customer> customerProcessor,
                         ItemWriter<Customer> writer) {

    return stepBuilderFactory.get("customerStep")
        .<Customer, Customer>chunk(10)
        .reader(reader)
        .processor(customerProcessor)
        .writer(writer)
        .build();
}
```

---

### Why this is great for Spring Batch 5.2 + Java 21

* **No boilerplate class** — uses a lambda bean definition.
* Uses **Java 21 `switch` with pattern matching** for cleaner logic.
* Works perfectly with Spring Batch's built-in `null` skip behavior.
* Keeps **business rules directly in the configuration** when logic is short.

---

If you like, I can also give you a **variant that chains multiple processors together** using `CompositeItemProcessor` — very handy when your ETL has multiple transformation steps. That’s actually how many production Spring Batch ETL jobs are structured.

Do you want me to give that composite processor example next?




Perfect — here’s how you’d do a **CompositeItemProcessor** in Spring Batch 5.2 with Java 21.

This is useful when you want to break transformation logic into **small, reusable processors** instead of one big monolith.

---

## **CompositeItemProcessor Example**

We’ll split the logic into 3 processors:

1. **FilterInactiveCustomerProcessor** → Skips inactive customers.
2. **NameCapitalizationProcessor** → Capitalizes first/last name.
3. **LoyaltyTierProcessor** → Adds loyalty tier based on purchases.

---

### 1️⃣ Individual Processors

```java
import org.springframework.batch.item.ItemProcessor;

public class FilterInactiveCustomerProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer customer) {
        return "ACTIVE".equalsIgnoreCase(customer.getStatus()) ? customer : null;
    }
}

public class NameCapitalizationProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer customer) {
        customer.setFirstName(capitalize(customer.getFirstName()));
        customer.setLastName(capitalize(customer.getLastName()));
        return customer;
    }

    private String capitalize(String str) {
        if (str == null || str.isEmpty()) return str;
        return str.substring(0, 1).toUpperCase() + str.substring(1).toLowerCase();
    }
}

public class LoyaltyTierProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer customer) {
        customer.setLoyaltyTier(
            switch ((int) customer.getTotalPurchase()) {
                case int amount when amount > 10_000 -> "GOLD";
                case int amount when amount > 5_000 -> "SILVER";
                default -> "BRONZE";
            }
        );
        return customer;
    }
}
```

---

### 2️⃣ Composite Processor Bean

```java
import org.springframework.batch.item.support.CompositeItemProcessor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class BatchConfig {

    @Bean
    public CompositeItemProcessor<Customer, Customer> customerCompositeProcessor() {
        CompositeItemProcessor<Customer, Customer> processor = new CompositeItemProcessor<>();
        processor.setDelegates(List.of(
            new FilterInactiveCustomerProcessor(),
            new NameCapitalizationProcessor(),
            new LoyaltyTierProcessor()
        ));
        return processor;
    }
}
```

---

### 3️⃣ Step Configuration

```java
@Bean
public Step customerStep(ItemReader<Customer> reader,
                         CompositeItemProcessor<Customer, Customer> customerCompositeProcessor,
                         ItemWriter<Customer> writer) {

    return stepBuilderFactory.get("customerStep")
        .<Customer, Customer>chunk(10)
        .reader(reader)
        .processor(customerCompositeProcessor)
        .writer(writer)
        .build();
}
```

---

### **Advantages**

* **Separation of concerns** → Each processor has a single responsibility.
* **Reusability** → Same small processors can be reused in other jobs.
* **Testability** → You can unit-test each transformation step independently.
* **Clean chaining** → Spring Batch automatically passes the output of one processor into the next.

---

If you want, I can make you a **Spring Batch 5.2 + Java 21** example where the `CompositeItemProcessor` is fully **functional-style (lambda)** without defining these separate classes. That way you get both modularity *and* minimal code.

Do you want me to do that version?






Got it — let’s make the **CompositeItemProcessor** in **Spring Batch 5.2 + Java 21** completely **functional style** so there’s **no separate processor classes** — everything stays clean in your config.

---

## **Functional CompositeItemProcessor Example**

Here we’ll define **three processors as lambdas** and chain them together with `CompositeItemProcessor`.

### Configuration

```java
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.support.CompositeItemProcessor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class BatchConfig {

    @Bean
    public CompositeItemProcessor<Customer, Customer> customerCompositeProcessor() {
        // Processor 1: Filter inactive customers
        ItemProcessor<Customer, Customer> filterInactive = customer ->
            "ACTIVE".equalsIgnoreCase(customer.getStatus()) ? customer : null;

        // Processor 2: Capitalize names
        ItemProcessor<Customer, Customer> capitalizeNames = customer -> {
            customer.setFirstName(capitalize(customer.getFirstName()));
            customer.setLastName(capitalize(customer.getLastName()));
            return customer;
        };

        // Processor 3: Add loyalty tier
        ItemProcessor<Customer, Customer> addLoyaltyTier = customer -> {
            customer.setLoyaltyTier(
                switch ((int) customer.getTotalPurchase()) {
                    case int amount when amount > 10_000 -> "GOLD";
                    case int amount when amount > 5_000 -> "SILVER";
                    default -> "BRONZE";
                }
            );
            return customer;
        };

        CompositeItemProcessor<Customer, Customer> compositeProcessor = new CompositeItemProcessor<>();
        compositeProcessor.setDelegates(List.of(filterInactive, capitalizeNames, addLoyaltyTier));
        return compositeProcessor;
    }

    private String capitalize(String str) {
        if (str == null || str.isEmpty()) return str;
        return str.substring(0, 1).toUpperCase() + str.substring(1).toLowerCase();
    }
}
```

---

### Step Configuration

```java
@Bean
public Step customerStep(ItemReader<Customer> reader,
                         CompositeItemProcessor<Customer, Customer> customerCompositeProcessor,
                         ItemWriter<Customer> writer) {

    return stepBuilderFactory.get("customerStep")
        .<Customer, Customer>chunk(10)
        .reader(reader)
        .processor(customerCompositeProcessor)
        .writer(writer)
        .build();
}
```

---

### **Why This Version is Great**

* **Zero boilerplate** — no separate classes for each processor.
* **Still modular** — each lambda is its own transformation/filter step.
* **Easy to read** — processors are in one place but logically separated.
* **Java 21 switch patterns** — clean business rules without if/else nesting.

---

If you want, I can make a **visual flow diagram** showing how Spring Batch moves data from **ItemReader → CompositeItemProcessor → ItemWriter** so it’s crystal clear how the chaining works. That would make it easier to explain in interviews or documentation.

Do you want me to prepare that diagram?
