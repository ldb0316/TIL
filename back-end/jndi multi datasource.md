standalone.xml (ha구성 시 standalone-ha.xml)에 driver 및 데이터소스 추가

```xml title="standalone.xml"
<datasources>

	...

	<datasource jndi-name="java:jboss/postgresDS" pool-name="postgresDS" enabled="true">
		<connection-url>jdbc:postgresql://[호스트]:5432/ws?targetServerType=master</connection-url>
		<driver-class>org.postgresql.Driver</driver-class>
		<driver>postgres</driver>
		<pool>
			<min-pool-size>10</min-pool-size>
			<max-pool-size>300</max-pool-size>
			<prefill>true</prefill>
			<use-strict-min>true</use-strict-min>
		</pool>
		<security>
			<user-name>[사용자명]</user-name>
			<password>[비밀번호]</password>
		</security>
		<validation>
			<valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLValidConnectionChecker"/>
			<check-valid-connection-sql>select 1</check-valid-connection-sql>
			<validate-on-match>true</validate-on-match>
			<background-validation>false</background-validation>
			<exception-sorter class-name="org.jboss.jca.adapters.jdbc.extensions.postgres.PostgreSQLExceptionSorter"/>
		</validation>
		<timeout>
			<blocking-timeout-millis>300000</blocking-timeout-millis>
			<idle-timeout-minutes>10</idle-timeout-minutes>
		</timeout>
		<statement>
			<share-prepared-statements>false</share-prepared-statements>
		</statement>
	</datasource>

	...

	<drivers>

		...

		<driver name="postgres" module="postgres">
			<driver-class>org.postgresql.Driver</driver-class>
		</driver>

		...

	</drivers>
</datasources>
```

jboss module에 JDBC 추가 및 module.xml 추가

```xml title="module.xml"
<?xml version="1.0" encoding="UTF-8"?>
<!--
  ~ 작업일시       작업자  작업내용
  ~ 2022.12.27  이다빈  최초생성
  -->
<module xmlns="urn:jboss:module:1.1" name="postgres">
    <resources>
        <resource-root path="postgresql-42.6.0.jar"/>
    </resources>

    <dependencies>
        <module name="javax.api"/>
        <module name="javax.transaction.api"/>
    </dependencies>
</module>

```

jndi name을 프로젝트 설정 파일 (application.yml 등)에 추가

```yml

spring:
  application:
    name: temp-project

  # 기존 부분
  datasource:
    jndi-name: java:jboss/OracleDS
  # 추가된 부분
  sub-datasource:
    jndi-name: java:jboss/postgresDS

	...
```

DataSourceConfig.java 추가

```java
@Configuration
public class DataSourceConfig {

  @Value("${spring.datasource.jndi-name}")
  private String jndiName;

  @Value("${spring.sub-datasource.jndi-name}")
  private String subJndiName;

  @Bean(name = "dataSource")
  public DataSource dataSource() {
    JndiDataSourceLookup dsLookup = new JndiDataSourceLookup();
    return dsLookup.getDataSource(jndiName); // default
  }

  @Bean(name = "subDataSource")
  public DataSource subDataSource() {
    JndiDataSourceLookup dsLookup = new JndiDataSourceLookup();
    return dsLookup.getDataSource(subJndiName); // append
  }

}
```

bean을 사용할 때는 Qualifier 어노테이션을 붙여서 사용하는게 안전하다.

```java
	  @Autowired
	  @Qualifier("subDataSource")
	  private final DataSource subDataSource;
```

JPA환경에서 multi datasource를 활용하려면 datasource 마다 스캔할 대상 엔티티 패키지 경로를 나누어 구성해야 하며
별도의 EntityConfig를 설정하고 스캔 경로를 지정한다.
해당 설정이 없으면 모든 엔티티에 대해서 두 datasource마다 양쪽 데이터베이스를 전부 스캔 진행하기 때문에 테이블이 없다는 예외가 발생한다.
패키지를 나누지 않고 커스텀 어노테이션을 만들어 exclude filter나 include filter에 지정하여 repository는 분리할 수도 있으나,
entity는 LocalContainerEntityManagerFactoryBean를 통해 패키지 스캔을 지정해야 하기 때문에 근본적으로 패키지를 나누는 것이 깔끔하다

1. main datasource에 대한 설정 예시 (Primary 어노테이션 지정 권장)

```java

@Configuration
// 다른 DB source 바라볼 component는 스캔하지 않도록 한다.(bean 충돌 방지)
@EnableJpaRepositories(basePackages = {"스캔 대상 repository 패키지 루트"}, excludeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Postgresql.class),
    entityManagerFactoryRef = "entityManagerFactory", transactionManagerRef = "transactionManager")
@EnableTransactionManagement
@RequiredArgsConstructor
public class JpaConfig {

  @Qualifier("dataSource")
  private final DataSource dataSource;

  // JPA 설정
  @Bean(name = "entityManagerFactory")
  @Primary // main datasource는 primary 로 지정 권장
  public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
    LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
    em.setDataSource(dataSource);
    em.setPackagesToScan("스캔 대상 엔티티 패키지 루트");

    JpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
    em.setJpaVendorAdapter(vendorAdapter);

    return em;
  }

}

```

2. sub datasource 에 대한 설정 + 추가적인 jpa 설정 예시

```java

@Configuration
// 다른 DB source 바라볼 component는 스캔하지 않도록 한다.(bean 충돌 방지)
@EnableJpaRepositories(basePackages = {"스캔 대상 repository 패키지 루트"}, includeFilters = @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Postgresql.class),
    entityManagerFactoryRef = "subEntityManagerFactory", transactionManagerRef = "subTransactionManager")
@EnableTransactionManagement
@RequiredArgsConstructor
public class SubJpaConfig {

  @Qualifier("subDataSource")
  private final DataSource subDataSource;

  @Value("${database.jpa.subdialect}")
  private String dialect;

  @Value("${database.jpa.show_sql}")
  private String showSql;

  @Value("${database.jpa.format_sql}")
  private String formatSql;

  @Value("${database.jpa.use_sql_comments}")
  private String useSqlComments;

  @Value("${database.jpa.physical_naming_strategy}")
  private String physicalNamingStrategy;

  @Value("${database.jpa.implicit_naming_strategy}")
  private String implicitNamingStrategy;

  @Value("${database.jpa.sub_schema}")
  private String defaultSchema;

  // JPA 설정
  @Bean(name = "subEntityManagerFactory")
  public LocalContainerEntityManagerFactoryBean subEntityManagerFactory() {
    LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
    em.setDataSource(subDataSource);
    em.setPackagesToScan("스캔 대상 엔티티 패키지 루트");

    JpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
    em.setJpaVendorAdapter(vendorAdapter);


    // 추가적인 JPA 속성 설정(선택)
	Properties properties = new Properties();
    properties.setProperty("hibernate.dialect", dialect);
    properties.setProperty("hibernate.show_sql", "true");
    properties.setProperty("hibernate.format_sql", formatSql);
    properties.setProperty("hibernate.use_sql_comments", useSqlComments);

    // 네이밍 전략 설정
    properties.setProperty("hibernate.physical_naming_strategy", physicalNamingStrategy);
    properties.setProperty("hibernate.implicit_naming_strategy", implicitNamingStrategy);

    // default schema 설정
    properties.setProperty("hibernate.default_schema", defaultSchema); // 적용안되는 것 같음

    em.setJpaProperties(properties);

    return em;
  }

}

```
