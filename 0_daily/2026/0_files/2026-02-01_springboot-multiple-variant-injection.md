
# Multiple Variant Injection with SpringBoot

**TBD**

How to...

---

## Use Case 1: alternatives on equal level

### Approach 1 (recommended)

Create a qualifier annotations to distinguish between the different variants:

````java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Oob {}
````

````java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Evt {}
````

Create configuration classes for each variant and use the qualifier annotations to distinguish between the different beans:

````java
@Configuration
public class OobConfig {
    public static final String OOB_BASE_URL = "/demo/oob";

    @Bean
    @Oob
    public PeopleRepository oobPeopleRepository(
        JdbcClient jdbcClient,
        JdbcTemplate jdbcTemplate,
        @Oob RouteBuilder routeBuilder
    ) {
        return new HSQLPeopleRepository(jdbcClient, jdbcTemplate, routeBuilder);
    }

    @Bean
    @Oob
    public PeopleService oobPeopleService(@Oob PeopleRepository repo) {
        return new PeopleService(repo);
    }

    @Bean
    @Oob
    public RouteBuilder oobRouteBuilder() {
        return new ConfigurableRouteBuilder("/demo/oob");
    }
}
````

Usage:
````java
@Controller
@RequestMapping(OobConfig.OOB_BASE_URL)
public class PeopleController {

    private final PeopleService peopleService;
    private final RouteBuilder routeBuilder;

    public PeopleController(
        @Oob PeopleService peopleService,
        @Oob RouteBuilder routeBuilder

    ) {
        this.peopleService = peopleService;
    }
}
````

````java
@Configuration
public class EvtConfig {
    public static final String EVT_BASE_URL = "/demo/event";

    @Bean
    @Evt
    public PeopleRepository evtPeopleRepository(
        JdbcClient jdbcClient,
        JdbcTemplate jdbcTemplate,
        @Evt RouteBuilder routeBuilder
    ) {
        return new HSQLPeopleRepository(jdbcClient, jdbcTemplate, routeBuilder);
    }

    @Bean
    @Evt
    public PeopleService evtPeopleService(@Evt PeopleRepository repo) {
        return new PeopleService(repo);
    }

    @Bean
    @Evt
    public RouteBuilder evtRouteBuilder() {
        return new ConfigurableRouteBuilder(EVT_BASE_URL);
    }
}
````

NOTE: it is not possible to define the @Oob and @Evt annotations at the root of a bean structure and have it automatically injected into all sub-beans.
For example, if we define the @Oob annotation on the PeopleService bean,
it will not be automatically injected into the PeopleRepository bean that is injected into the PeopleService bean.
We need to explicitly define the @Oob annotation on each bean that we want to be part of the OOB variant.

### Approach 2

Just use @Qualifier("oob") and @Qualifier("evt") annotations on the beans and inject them using the same qualifiers.
This approach is less clean and more error-prone than the first approach,
as it relies on string literals to distinguish between the different variants.
It also does not provide any compile-time safety, as it is possible to misspell the qualifier name and have it injected into the wrong bean.

````java
@Configuration
public class OobConfig {
    public static final String OOB_BASE_URL = "/demo/oob";

    @Bean
    @Qualifier("Oob")
    public PeopleRepository oobPeopleRepository(
        JdbcClient jdbcClient,
        JdbcTemplate jdbcTemplate,
        @Oob RouteBuilder routeBuilder
    ) {
        return new HSQLPeopleRepository(jdbcClient, jdbcTemplate, routeBuilder);
    }

    @Bean
    @Qualifier("Oob")
    public PeopleService oobPeopleService(@Oob PeopleRepository repo) {
        return new PeopleService(repo);
    }

    @Bean
    @Qualifier("Oob")
    public RouteBuilder oobRouteBuilder() {
        return new ConfigurableRouteBuilder("/demo/oob");
    }
}
````

Usage:
````java
@Controller
@RequestMapping(OobConfig.OOB_BASE_URL)
public class PeopleController {

    private final PeopleService peopleService;
    private final RouteBuilder routeBuilder;

    public PeopleController(
        @Qualifier("Oob") PeopleService peopleService,
        @Qualifier("Oob") RouteBuilder routeBuilder

    ) {
        this.peopleService = peopleService;
    }
}
````

## Use Case 2: primary with fallback

For example: usually services are used with the security context of the logged in user,
but for some use cases we want to use a service with a fixed system user (e.g. for background jobs like in schedulers when no user is involved).

Standard service:
````java
@Service
public class MyService {
    private final OtherService otherService;
}
````

Configuration for scheduler variant:
````java
@Configuration
public class SchedulerConfiguration {
    @Bean
    @M2M
    @Fallback // do not use this bean (e.g. in Controller) if there is another bean of the same type without @Fallback annotation
    public MyService m2mMyService(@M2M OtherService otherService) {
        return new MyService(otherService);
    }
}
````

Scheduler usage:
````java
@Component
public class AppSchedulers {
    @M2M
    private final MyService myService;

    @Scheduled(cron = "0 00 06 * * MON-FRI") // automatically called if Application class is annotated with @EnableScheduling
    public void doSomething() {
        myService.doit();
    }
}
````

