# Структура проекта Bank Demo

## 📁 Общая структура

```
demo1/
├── pom.xml                              # Maven конфигурация
├── src/
│   ├── main/
│   │   ├── java/com/example/demo/
│   │   │   ├── Demo1Application.java   # Точка входа
│   │   │   ├── controllers/            # Web контроллеры
│   │   │   ├── entities/               # JPA сущности
│   │   │   ├── repositories/           # Spring Data репозитории
│   │   │   └── services/               # Бизнес-логика
│   │   └── resources/
│   │       ├── application.properties  # Конфигурация приложения
│   │       └── templates/              # Thymeleaf шаблоны
│   └── test/
│       ├── java/                        # Тестовые классы
│       └── resources/features/          # Cucumber сценарии
└── target/                              # Скомпилированные файлы
```

---

## 🛠️ Технологический стек

| Технология | Версия | Назначение |
|------------|--------|------------|
| **Spring Boot** | 4.0.0 | Основной фреймворк |
| **Spring Data JPA** | - | ORM для работы с БД |
| **Spring Web** | - | REST/MVC контроллеры |
| **Thymeleaf** | - | Шаблонизатор HTML |
| **PostgreSQL** | - | Основная база данных |
| **H2 Database** | - | Тестовая in-memory БД |
| **Lombok** | - | Генерация boilerplate кода |
| **Cucumber** | 7.15.0 | BDD тестирование |
| **JUnit 5** | - | Unit тестирование |
| **Jakarta Validation** | - | Валидация данных |
| **Bootstrap** | 5.3.3 | CSS фреймворк (CDN) |
| **Java** | 17 | Язык программирования |

---

## 📦 Зависимости (pom.xml)

```xml
<!-- Spring Boot Starter для JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Spring Boot Starter для Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Thymeleaf для шаблонов -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<!-- PostgreSQL драйвер -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Lombok для уменьшения boilerplate -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>

<!-- Cucumber для BDD тестов -->
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-java</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>

<!-- H2 для тестовой БД -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 🏗️ Слои архитектуры

### 1. Entities (Сущности) — `entities/`

JPA сущности с аннотациями Hibernate и Jakarta Validation.

**Clients.java** — Клиенты банка:
```java
@Entity
@Table(name = "clients")
public class Clients {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Getter @Setter
    @Column(name = "full_name", nullable = false)
    @jakarta.validation.constraints.NotBlank(message = "Name cannot be empty")
    private String fullName;

    @Getter @Setter
    @Column(name = "phone_number", unique = true, nullable = false)
    @jakarta.validation.constraints.NotBlank(message = "Phone number cannot be empty")
    private String phoneNumber;

    public Clients(String fullName, String phoneNumber) {
        this.fullName = fullName;
        this.phoneNumber = phoneNumber;
    }
}
```

**Loans.java** — Кредиты:
```java
@Entity
@Table(name = "loans")
public class Loans {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "client_id", nullable = false)
    private Clients client;

    @Setter
    @Column(name = "amount")
    private double amount;        // Остаток долга

    @Setter
    @Column(name = "interest_rate")
    private double interestRate;  // Процентная ставка

    @Column(name = "start_date")
    private LocalDate startDate;

    @Setter
    @Column(name = "due_date")
    private LocalDate dueDate;    // Дата погашения
}
```

**LoanPayments.java** — История платежей:
```java
@Entity
@Table(name = "loan_payments")
public class LoanPayments {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(optional = false)
    @JoinColumn(name = "loan_id", nullable = false)
    private Loans loan;

    @Setter
    @Column(name = "amount_paid")
    private double amountPaid;

    @Column(name = "payment_date")
    private LocalDate paymentDate;
}
```

---

### 2. Repositories (Репозитории) — `repositories/`

Интерфейсы Spring Data JPA для доступа к данным.

**ClientRepository.java**:
```java
public interface ClientRepository extends JpaRepository<Clients, Long> {
    Optional<Clients> findByPhoneNumber(String phoneNumber);
}
```

**LoanRepository.java**:
```java
public interface LoanRepository extends JpaRepository<Loans, Long> {
    List<Loans> findByClient(Clients client);
}
```

**LoanPaymentRepository.java**:
```java
public interface LoanPaymentRepository extends JpaRepository<LoanPayments, Long> {
    List<LoanPayments> findByLoan(Loans loan);
}
```

---

### 3. Services (Сервисы) — `services/`

Бизнес-логика приложения с транзакционной обработкой.

**LoanService.java** — Логика кредитования:
```java
@Service
@RequiredArgsConstructor
public class LoanService {

    private final LoanRepository loanRepository;
    private final LoanPaymentRepository loanPaymentRepository;
    private final ClientRepository clientRepository;
    private final AccountRepository accountRepository;

    @Transactional
    public Loans createLoan(Long clientId, Long accountId, double amount, int termInMonths) {
        // Определение процентной ставки по сроку
        double interestRate;
        switch (termInMonths) {
            case 3:  interestRate = 15.0; break;  // 3 мес = 15%
            case 6:  interestRate = 10.0; break;  // 6 мес = 10%
            case 12: interestRate = 5.0;  break;  // 12 мес = 5%
            default: interestRate = 10.0;
        }

        // Расчёт общей суммы долга
        double totalDebt = amount + (amount * interestRate / 100.0);
        LocalDate dueDate = LocalDate.now().plusMonths(termInMonths);

        Loans loan = new Loans(client, totalDebt, interestRate, dueDate);
        loanRepository.save(loan);

        // Зачисление суммы кредита на счёт
        account.setBalance(account.getBalance() + amount);
        return loan;
    }

    @Transactional
    public LoanPayments makePayment(Long loanId, Long accountId, double amount) {
        // Защита от переплаты
        double actualPayment = Math.min(amount, loan.getAmount());

        if (account.getBalance() < actualPayment) {
            throw new RuntimeException("Insufficient funds in account");
        }

        // Списание со счёта и уменьшение долга
        account.setBalance(account.getBalance() - actualPayment);
        loan.setAmount(loan.getAmount() - actualPayment);

        // Запись в историю платежей
        LoanPayments payment = new LoanPayments(loan, actualPayment);
        return loanPaymentRepository.save(payment);
    }
}
```

**ClientService.java** — Управление клиентами:
```java
@Service
@RequiredArgsConstructor
public class ClientService {

    private final ClientRepository clientRepository;
    private final LoanService loanService;
    private final AccountService accountService;

    @Transactional
    public Clients createClient(String fullName, String phoneNumber) {
        // Проверка уникальности телефона
        if (clientRepository.findByPhoneNumber(phoneNumber).isPresent()) {
            throw new RuntimeException("Phone number already exists");
        }
        Clients client = new Clients(fullName, phoneNumber);
        return clientRepository.saveAndFlush(client);
    }

    @Transactional
    public void deleteClient(Long clientId) {
        // Каскадное удаление: кредиты → счета → клиент
        loanService.deleteLoansForClient(clientId);
        accountService.deleteAccountsForClient(clientId);
        clientRepository.deleteById(clientId);
    }
}
```

---

### 4. Controllers (Контроллеры) — `controllers/`

Spring MVC контроллеры для обработки HTTP запросов.

**WebController.java**:
```java
@Controller
@RequiredArgsConstructor
public class WebController {

    private final ClientService clientService;
    private final AccountService accountService;
    private final LoanService loanService;
    private final ClientRepository clientRepository;
    private final LoanRepository loanRepository;
    private final LoanPaymentRepository loanPaymentRepository;

    // Главная страница со списком клиентов
    @GetMapping("/")
    public String index(Model model) {
        model.addAttribute("clients", clientRepository.findAll());
        model.addAttribute("client", new Clients());
        return "index";
    }

    // Страница клиента с его счетами и кредитами
    @GetMapping("/client/{id}")
    public String clientView(@PathVariable Long id, Model model) {
        Clients client = clientRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Client not found"));
        model.addAttribute("client", client);
        model.addAttribute("accounts", accountRepository.findByClient(client));
        model.addAttribute("loans", loanRepository.findByClient(client));
        return "client-view";
    }

    // История платежей по кредиту
    @GetMapping("/loan/{id}/history")
    public String loanHistory(@PathVariable Long id, Model model) {
        Loans loan = loanRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Loan not found"));
        var payments = loanPaymentRepository.findByLoan(loan);
        double totalPaid = payments.stream()
                .mapToDouble(p -> p.getAmountPaid())
                .sum();
        model.addAttribute("loan", loan);
        model.addAttribute("payments", payments);
        model.addAttribute("totalPaid", totalPaid);
        return "loan-history";
    }

    // Оплата кредита
    @PostMapping("/loan/pay")
    public String payLoan(@RequestParam Long loanId, 
                          @RequestParam Long accountId, 
                          @RequestParam double amount) {
        loanService.makePayment(loanId, accountId, amount);
        Long clientId = loanRepository.findById(loanId).get().getClient().getId();
        return "redirect:/client/" + clientId;
    }
}
```

---

### 5. Templates (Шаблоны) — `resources/templates/`

HTML шаблоны с Thymeleaf и Bootstrap 5.

**index.html** — Главная страница:
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Bank Demo</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <!-- Форма добавления клиента -->
    <form th:action="@{/save}" method="post" th:object="${client}">
        <input id="fullName" type="text" th:field="*{fullName}" required>
        <input id="phoneNumber" type="text" th:field="*{phoneNumber}" required>
        <button type="submit">Save</button>
    </form>

    <!-- Таблица клиентов -->
    <table>
        <tr th:each="c : ${clients}">
            <td th:text="${c.id}"></td>
            <td><a th:href="@{/client/{id}(id=${c.id})}" th:text="${c.fullName}"></a></td>
            <td th:text="${c.phoneNumber}"></td>
        </tr>
    </table>
</body>
</html>
```

**loan-history.html** — История платежей:
```html
<!-- Таблица платежей -->
<table>
    <tr th:each="payment : ${payments}">
        <td th:text="${payment.id}"></td>
        <td th:text="${payment.paymentDate}"></td>
        <td th:text="'- ' + ${payment.amountPaid} + ' $'"></td>
    </tr>
    <tfoot>
        <tr>
            <td colspan="2">Total Paid:</td>
            <td th:text="${totalPaid} + ' $'"></td>  <!-- Итоговая сумма -->
        </tr>
    </tfoot>
</table>
```

---

## 🧪 Тестирование (Cucumber BDD)

### Структура тестов

```
src/test/
├── java/com/example/demo/
│   ├── RunCucumberTest.java    # Точка запуска тестов
│   └── steps/
│       └── BankingSteps.java   # Step definitions
└── resources/
    ├── features/
    │   └── banking.feature     # Gherkin сценарии
    └── application-test.properties
```

### Пример сценария (banking.feature):

```gherkin
Feature: Banking System Functionality
  As a bank manager
  I want to manage clients, accounts, and loans
  So that I can track the bank's business and risk

  Scenario: Create a client with valid data
    Given I have a client data with name "Arseni Kiyko" and phone "123456789"
    When I request to create the client
    Then The client "Arseni Kiyko" should be saved in the system

  Scenario: Take 12-month loan (5% Interest)
    Given A client "Hlib Filobok" exists
    And "Hlib Filobok" has an account with balance 100.00
    When "Hlib Filobok" takes a loan of 1000.00 for 12 months
    Then The loan interest rate should be 5.0%
    And The loan due date should be 12 months from today
    And The total debt should be 1050.00

  Scenario: Verify Payment History accuracy
    Given A client "Me" exists
    And "Me" has an account with balance 2000.00
    And "Me" has an active loan of 1000.00 for 12 months
    When "Me" pays 100.00 towards the loan
    And "Me" pays 200.00 towards the loan
    And "Me" pays 50.00 towards the loan
    Then The history for this loan should show exactly 3 payments
    And The Total Paid in history should sum to 350.00
```

---

## ⚙️ Конфигурация

### application.properties (Production)

```properties
# PostgreSQL подключение
spring.datasource.url=jdbc:postgresql://localhost:5432/postgres
spring.datasource.username=postgres
spring.datasource.password=1502
spring.datasource.driver-class-name=org.postgresql.Driver

# Hibernate настройки
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# Сервер
server.port=8080

# Логирование
logging.level.org.hibernate=TRACE
```

### application-test.properties (Testing)

```properties
# H2 in-memory база для тестов
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
```

---

## 🚀 Запуск приложения

```bash
# Запуск приложения
mvn spring-boot:run

# Запуск тестов
mvn test

# Сборка JAR
mvn clean package
```

---

## 📊 Диаграмма сущностей

```
┌─────────────┐       ┌─────────────┐       ┌───────────────┐
│   Clients   │──1:N──│   Accounts  │       │  LoanPayments │
│─────────────│       │─────────────│       │───────────────│
│ id          │       │ id          │       │ id            │
│ fullName    │       │ balance     │       │ amountPaid    │
│ phoneNumber │       │ client_id   │       │ paymentDate   │
└─────────────┘       └─────────────┘       │ loan_id       │
       │                                    └───────────────┘
       │                                           │
       │1:N                                       N:1
       │                                           │
       ▼                                           ▼
┌─────────────┐                            ┌─────────────┐
│    Loans    │────────────1:N─────────────│             │
│─────────────│                            └─────────────┘
│ id          │
│ amount      │
│ interestRate│
│ startDate   │
│ dueDate     │
│ client_id   │
└─────────────┘
```

---

## 📝 Бизнес-правила

1. **Процентные ставки по срокам:**
   - 3 месяца → 15%
   - 6 месяцев → 10%
   - 12 месяцев → 5%

2. **Защита от переплаты:** система автоматически ограничивает платёж суммой остатка долга

3. **Каскадное удаление:** при удалении клиента удаляются все его счета и кредиты

4. **Уникальность телефона:** нельзя создать двух клиентов с одинаковым номером
