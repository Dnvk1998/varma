<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
    driverClassName: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true
  jpa:
    hibernate:
      ddl-auto: update
    database-platform: org.hibernate.dialect.H2Dialect
    show-sql: true
package com.midas.core.model;

import javax.persistence.Entity;
import javax.persistence.Id;

@Entity
public class User {

    @Id
    private String id;
    private double balance;

    // Getters and setters
    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public double getBalance() {
        return balance;
    }

    public void setBalance(double balance) {
        this.balance = balance;
    }
}
package com.midas.core.model;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.ManyToOne;

@Entity
public class TransactionRecord {

    @Id
    private String transactionId;
    
    @ManyToOne
    private User sender;

    @ManyToOne
    private User recipient;

    private double amount;

    // Getters and setters
    public String getTransactionId() {
        return transactionId;
    }

    public void setTransactionId(String transactionId) {
        this.transactionId = transactionId;
    }

    public User getSender() {
        return sender;
    }

    public void setSender(User sender) {
        this.sender = sender;
    }

    public User getRecipient() {
        return recipient;
    }

    public void setRecipient(User recipient) {
        this.recipient = recipient;
    }

    public double getAmount() {
        return amount;
    }

    public void setAmount(double amount) {
        this.amount = amount;
    }
}
package com.midas.core.repository;

import com.midas.core.model.User;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, String> {
}
package com.midas.core.repository;

import com.midas.core.model.TransactionRecord;
import org.springframework.data.jpa.repository.JpaRepository;

public interface TransactionRecordRepository extends JpaRepository<TransactionRecord, String> {
}
package com.midas.core.listener;

import com.midas.core.model.TransactionRecord;
import com.midas.core.model.User;
import com.midas.core.repository.TransactionRecordRepository;
import com.midas.core.repository.UserRepository;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class TransactionKafkaListener {

    @Value("${kafka.topic}")
    private String topic;  // Topic name from application.yml

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TransactionRecordRepository transactionRecordRepository;

    // Listen to the Kafka topic and deserialize the incoming messages
    @KafkaListener(topics = "${kafka.topic}", groupId = "transaction-group")
    public void listenToTransactions(ConsumerRecord<String, TransactionRecord> record) {
        TransactionRecord transaction = record.value();

        // Check if sender and recipient are valid
        User sender = userRepository.findById(transaction.getSender().getId()).orElse(null);
        User recipient = userRepository.findById(transaction.getRecipient().getId()).orElse(null);

        if (sender == null || recipient == null || sender.getBalance() < transaction.getAmount()) {
            // Invalid transaction, discard it
            System.out.println("Transaction invalid, discarding: " + transaction);
            return;
        }

        // Update sender and recipient balances
        sender.setBalance(sender.getBalance() - transaction.getAmount());
        recipient.setBalance(recipient.getBalance() + transaction.getAmount());

        userRepository.save(sender);
        userRepository.save(recipient);

        // Save the transaction record
        transactionRecordRepository.save(transaction);

        // Log successful transaction
        System.out.println("Transaction successful: " + transaction);
    }
}
package com.midas.core.listener;

import com.midas.core.model.TransactionRecord;
import com.midas.core.model.User;
import com.midas.core.repository.TransactionRecordRepository;
import com.midas.core.repository.UserRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.junit.jupiter.api.Test;

@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = "transactions")
public class TaskThreeTests {

    @Autowired
    private KafkaTemplate<String, TransactionRecord> kafkaTemplate;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TransactionRecordRepository transactionRecordRepository;

    @Test
    public void testTransactionProcessing() {
        // Create and save users
        User sender = new User();
        sender.setId("waldorf");
        sender.setBalance(1000);  // Example balance

        User recipient = new User();
        recipient.setId("statler");
        recipient.setBalance(500);

        userRepository.save(sender);
        userRepository.save(recipient);

        // Send 4 transactions
        for (int i = 0; i < 4; i++) {
            TransactionRecord transaction = new TransactionRecord();
            transaction.setTransactionId("txn" + i);
            transaction.setSender(sender);
            transaction.setRecipient(recipient);
            transaction.setAmount(100 + i * 50);  // Example amounts: 100, 150, 200, 250

            kafkaTemplate.send("transactions", transaction);
        }

        // Retrieve and check the balance of "waldorf" user
        User waldorf = userRepository.findById("waldorf").orElse(null);
        System.out.println("Final balance of waldorf: " + waldorf.getBalance());
    }
}
