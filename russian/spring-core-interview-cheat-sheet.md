# Spring Core — Шпаргалка для собеседования (Middle/Senior)

Глубокий разбор IoC-контейнера, внедрения зависимостей, жизненного цикла бинов и внутренних механизмов.

---

## 1) Что такое Spring Core

### Концепция IoC-контейнера

**Inversion of Control (IoC)** означает, что фреймворк контролирует создание объектов и wiring зависимостей, а не ваш код. Вместо этого:

```java
// Без IoC — ручное создание объектов
OrderService orderService = new OrderService();
orderService.setPaymentService(new PayPalPaymentService());
orderService.setOrderRepository(new JpaOrderRepository());
// А если нужен Stripe вместо PayPal? Придётся менять все места создания!
```

Spring берёт это на себя:

```java
// С IoC — вы описываете, что вам нужно
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;  // Spring предоставляет это
    }
}
```

**Контейнер:** `ApplicationContext` в Spring — это сложная фабрика, которая:
- создаёт объекты (бины),
- связывает зависимости,
- управляет жизненным циклом,
- предоставляет точки расширения.

### Зачем это нужно

**Проблемы, которые решает Spring:**
1. **Жёсткая связанность** — классы жёстко зашивали свои зависимости.
2. **Сложное тестирование** — зависимости трудно подменять моками.
3. **Конфигурационный хаос** — смена реализаций требовала изменения кода.
4. **Управление жизненным циклом** — кто освобождает ресурсы и управляет транзакциями?

**Решение Spring:**
- слабая связанность через интерфейсы,
- удобное тестирование через dependency injection,
- внешняя конфигурация (аннотации/XML),
- автоматическое управление жизненным циклом.

### Высокоуровневая архитектура

```
ApplicationContext (IoC Container)
    ├── BeanFactory (создание бинов)
    ├── ResourceLoader (загрузка конфигов)
    ├── EventPublisher (события приложения)
    └── MessageSource (i18n)

Ваш код
    ← зависит от ← ApplicationContext
         создаёт и связывает
    ←────── Бины (ваши сервисы, репозитории)
```

**Ответ на собеседовании:**
Spring Core — это IoC-контейнер, который берёт на себя создание объектов и wiring зависимостей. Вместо того чтобы классы создавали зависимости сами, Spring создаёт бины и внедряет всё необходимое. Это ослабляет связанность, упрощает тестирование с моками и выносит конфигурацию наружу, чтобы можно было менять реализации без изменения кода.

---

## 2) IoC (Inversion of Control)

### Как Spring берёт управление на себя

**Обычный поток управления:**
```
Ваш код → создаёт объекты → вызывает методы
```

**Поток управления при IoC:**
```
Конфигурация Spring → ApplicationContext → создаёт бины → связывает зависимости
                                         ↓
                                   Ваш код получает готовые бины
```

**Механизм:**
1. Вы задаёте, **какие** бины должны существовать (`@Component`, `@Bean`).
2. Вы задаёте, **как** они связаны (`@Autowired`, параметры конструктора).
3. Spring решает, **когда** создавать и **как** связывать.

### ApplicationContext vs BeanFactory

| Feature | BeanFactory | ApplicationContext |
|---------|-------------|---------------------|
| **Что это** | Базовый IoC-контейнер | Продвинутый контейнер с enterprise-возможностями |
| **Создание бинов** | Lazy (по требованию) | Eager (предсоздаёт singleton'ы) |
| **Internationalization** | Нет | Да (`MessageSource`) |
| **Event propagation** | Нет | Да (`ApplicationEvent`) |
| **AOP integration** | Вручную | Встроено |
| **Message resources** | Нет | Да |
| **Bean post-processing** | Ручная регистрация | Автоматическая |

**Разница в коде:**
```java
// BeanFactory — низкий уровень, редко используется напрямую
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions("beans.xml");
MyService service = factory.getBean(MyService.class);  // Бин создаётся прямо сейчас

// ApplicationContext — то, что используют почти все
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
// Все singleton'ы уже созданы и связаны!
MyService service = context.getBean(MyService.class);
```

**Глубокое понимание:**
`ApplicationContext` расширяет `BeanFactory` и добавляет:
- автоматическую регистрацию `BeanPostProcessor`,
- автоматическую регистрацию `BeanFactoryPostProcessor`,
- загрузку ресурсов (classpath, file, URL),
- публикацию событий,
- разрешение сообщений (i18n).

**Ответ на собеседовании:**
`BeanFactory` — базовый IoC-контейнер, который создаёт бины по требованию. `ApplicationContext` — полнофункциональный контейнер, который предсоздаёт singleton'ы на старте, умеет internationalization, публикует события и автоматически регистрирует post-processors. На практике все используют `ApplicationContext` или его реализации вроде `AnnotationConfigApplicationContext`.

---

## 3) Dependency Injection (DI)

### Constructor Injection (предпочтительно)

```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final OrderRepository orderRepository;

    // Зависимости внедряются при создании — неизменяемо и удобно для тестов
    public OrderService(PaymentService paymentService,
                        OrderRepository orderRepository) {
        this.paymentService = paymentService;
        this.orderRepository = orderRepository;
    }
}
```

**Почему это предпочтительно:**
1. **Неизменяемые зависимости** — поля можно сделать `final`.
2. **Обязательные зависимости** — объект нельзя создать без них.
3. **Удобно тестировать** — легко передавать моки в конструктор.
4. **Чистый код** — зависимости явно видны.

### Field Injection (почему это плохо)

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;  // Скрытая зависимость!

    @Autowired
    private OrderRepository orderRepository;  // Нельзя сделать final
}
```

**Проблемы:**
1. **Скрытые зависимости** — их не видно в API.
2. **Нельзя сделать неизменяемым** — нет `final`-полей.
3. **Тяжело тестировать** — нужны reflection или Spring для инъекции.
4. **Нарушает принципы ООП** — объект создаётся в неполном состоянии.
5. **Привязка к фреймворку** — без Spring экземпляр создать нельзя.

**Частая ловушка:**
Использовать `@Autowired` на полях потому, что это «проще» — в итоге получаются плохо спроектированные классы, которые сложно тестировать.

### Setter Injection

```java
@Service
public class OrderService {
    private OptionalService optionalService;

    // Для действительно необязательных зависимостей
    @Autowired(required = false)
    public void setOptionalService(OptionalService service) {
        this.optionalService = service;
    }
}
```

**Когда использовать:** для optional dependencies, у которых есть разумное значение по умолчанию (например, `Cache`, который по умолчанию делает no-op, если не предоставлен).

### Как Spring внедряет зависимости

**Процесс разрешения:**
1. Spring определяет нужный тип (тип параметра конструктора).
2. Ищет в контейнере бины этого типа.
3. Если совпадение одно — внедряет его.
4. Если совпадений несколько — ищет `@Primary` или сопоставляет по имени.
5. Если совпадений нет — падает, если только не указан `@Autowired(required=false)`.

**Сопоставление по имени:**
```java
public OrderService(PaymentService stripePaymentService) {
    // Spring ищет бин с именем "stripePaymentService"
}
```

**Ответ на собеседовании:**
Я использую только constructor injection, потому что он делает зависимости явными и позволяет использовать immutable-поля. Field injection скрывает зависимости и усложняет unit-тесты — для проверки нужен reflection или Spring context. Setter injection использую только для действительно необязательных зависимостей с разумным дефолтом.

---

## 4) Bean Definition & Creation

### Что такое BeanDefinition

Прежде чем Spring создаёт бин, он создаёт **BeanDefinition** — рецепт создания бина:

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    String getBeanClassName();           // Какой класс создавать
    String getScope();                   // singleton, prototype и т.д.
    ConstructorArgumentValues getConstructorArgumentValues();  // Аргументы конструктора
    MutablePropertyValues getPropertyValues();  // Какие свойства установить
    String getInitMethodName();          // @PostConstruct
    String getDestroyMethodName();       // @PreDestroy
    boolean isLazyInit();                // Создавать по требованию?
}
```

**Источники BeanDefinition:**
- классы с `@Component` → `AnnotatedGenericBeanDefinition`,
- методы `@Bean` → `ConfigurationClassBeanDefinition`,
- XML `<bean>` → `GenericBeanDefinition`,
- auto-configuration → разные реализации.

### Как Spring сканирует классы

**Механика `@ComponentScan`:**

```java
@ComponentScan(basePackages = "com.example")
public class AppConfig { }
```

**Пошагово:**
1. `ClassPathBeanDefinitionScanner` сканирует classpath.
2. Читает ресурсы, подходящие под `com/example/**/*.class`.
3. Для каждого класса проверяет stereotype-аннотации `@Component`.
4. Если находит — создаёт `ScannedGenericBeanDefinition`.
5. Регистрирует его в `BeanDefinitionRegistry`.

**Stereotype-аннотации (все включают `@Component`):**
- `@Component` — обычный бин,
- `@Service` — слой бизнес-логики (специальной обработки нет, это семантика),
- `@Repository` — слой доступа к данным (включает перевод исключений),
- `@Controller` / `@RestController` — веб-слой.

**Глубокое понимание:**
`@Repository` запускает `PersistenceExceptionTranslationPostProcessor`, который автоматически переводит специфичные для БД исключения, например `SQLException`, в иерархию `DataAccessException` Spring.
