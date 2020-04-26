---
layout: blog
title: Spring Boot/React - FullStack Project Template/Tutorial
date: '2020-04-12T15:58:27-04:00'
cover: /images/java-logo400.png
categories:
  - tutorial
  - Java
  - Spring Boot
---
Here's a Spring Boot project template/tutorial that I've put together to try to get some best practices codified on how to quickly throw together a service.

Full code here for the impatient: https://github.com/marksteele/SpringBootReactFullStackSample

Here's what it'll look like when we're done (note my amazing front-end skills)

![null](/images/app.jpg)

We'll cover: REST endpoints, scheduled tasks, thread-pools to parallelize work, React, MySQL, 12-factor app best practices, OAuth2 (google), monitoring, unit testing, code coverage, project mess detection, spotbugs, DevOps and more!

This blog post is going to be a bit of a beast, so before I dig too deep let's set the stage on what we're trying to accomplish. First, we'd like to build a modern front-end and back-end system. For the front-end, we'll use React. For the back-end, we'll use Java+Spring Boot.

Our fictitious project is called Visitors. We have been tasked to build a form where users can enter an IP address, and we are supposed to save it to a database. We'd also like to see the entries in the database as well.

The stakeholders in this type of project typically are: The product owner, front-end engineer, back-end engineer, QA, DevOps & Security. Hopefully by the time we're done, everyone's happy.

In this example, we want the front-end to be packaged and deployed inside the back-end application. For this reason, we're going to co-locate the front-end and back-end code in the same repo, and make it relatively easy for both front-end or back-end developers to work on.

On the back-end side, we'd like some APIs:

* List all IPs for which we don't have country information for.
* Lookup the location of an IP address in real-time.
* Trigger a background task that scans the database immediately and attempts to geo-locate all unknown IP locations.
* An API endpoint to add new IPs
* An API endpoint to view all the entries in the table.

In addition to the APIs, we'd also want the back-end to trigger resolving all unknown IP locations on a daily basis.

Furthermore, as the numbers of entries in the database might get large, we'd want the geo-lookups to be done in parallel, several simultaneously.

We're always security conscious, so we'll want the whole shebang to be protected via OAuth2, and authenticate via our google domain. We also want our backend/front-end protected with CSRF.

We'll use Maven and we want to create a fat jar that can run everything stand-alone. That'll make deploying it easy.

Our DevOps wants everything configured via environment variables.

To help with quality, consistency, and security we'll run the following checks against the code every time we build it: 

* Unit tests, 
* Unit test code coverage
* Code styling (google coding standard)
* Static code analysis
  * Spotbugs
  * Project mess detection
  * Copy paste detection
* Automatic detection of vulnerable dependancies (OWASP)

Also, it'd be nice if the APIs were documented somehow.

Phew! Let's get crackin.

# Backend

All rightie. First things first, it is assumed that you know your way around your IDE and have used maven before. You'll also need to have a MySQL server up and running with credentials, and a database called 'sample'.

## Config

Here's the `pom.xml` for this project. It's got all the goodies inside.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.3.0.M4</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>org.controlaltdel.sample</groupId>
	<artifactId>service</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>${project.artifactId}</name>
	<description>Sample Project</description>

	<properties>
		<java.version>11</java.version>
		<maven.compiler.source>${java.version}</maven.compiler.source>
		<maven.compiler.target>${java.version}</maven.compiler.target>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<!-- Dependencies -->
		<javax-xml-bind.version>2.3.1</javax-xml-bind.version>
		<logback-json.version>0.1.5</logback-json.version>
		<mockito.version>2.23.4</mockito.version>
		<!-- Build settings -->
		<dependency-check-maven.version>5.2.2</dependency-check-maven.version>
		<maven-surefire-plugin.version>2.22.2</maven-surefire-plugin.version>
		<jacoco-maven-plugin.version>0.8.5</jacoco-maven-plugin.version>
		<maven-checkstyle-plugin.version>3.1.1</maven-checkstyle-plugin.version>
		<maven-pmd-plugin.version>3.12.0</maven-pmd-plugin.version>
		<spotbugs-maven-plugin.version>3.1.12.2</spotbugs-maven-plugin.version>
		<build-tools.version>0.0.6</build-tools.version>
		<coverage.minimum>0.75</coverage.minimum>
		<cpd.tokens>150</cpd.tokens>
		<pmd.priority>4</pmd.priority>
	</properties>

	<dependencies>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.46</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
		</dependency>
		<dependency>
			<groupId>com.pivovarit</groupId>
			<artifactId>parallel-collectors</artifactId>
			<version>1.1.0</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-oauth2-client</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-oauth2-jose</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-configuration-processor</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-context</artifactId>
			<version>2.1.2.RELEASE</version>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger2</artifactId>
			<version>2.9.2</version>
		</dependency>
		<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger-ui</artifactId>
			<version>2.9.2</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
			<exclusions>
				<exclusion>
					<groupId>org.junit.vintage</groupId>
					<artifactId>junit-vintage-engine</artifactId>
				</exclusion>
			</exclusions>
		</dependency>

		<dependency>
			<groupId>org.springframework.security</groupId>
			<artifactId>spring-security-test</artifactId>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>org.mockito</groupId>
			<artifactId>mockito-core</artifactId>
			<version>${mockito.version}</version>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<finalName>service</finalName>
		<plugins>
			<plugin>
				<!-- To generate report, run: mvn dependency-check:aggregate -->
				<groupId>org.owasp</groupId>
				<artifactId>dependency-check-maven</artifactId>
				<version>${dependency-check-maven.version}</version>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-surefire-plugin</artifactId>
				<version>${maven-surefire-plugin.version}</version>

				<configuration>
					<useSystemClassLoader>false</useSystemClassLoader>
					<trimStackTrace>false</trimStackTrace>
				</configuration>
			</plugin>

			<plugin>
				<groupId>org.jacoco</groupId>
				<artifactId>jacoco-maven-plugin</artifactId>
				<version>${jacoco-maven-plugin.version}</version>

				<configuration>
					<excludes>
						<exclude>org/controlaltdel/**/model/**</exclude>
					</excludes>

					<rules>
						<rule>
							<element>BUNDLE</element>

							<limits>
								<limit>
									<counter>INSTRUCTION</counter>
									<value>COVEREDRATIO</value>
									<minimum>${coverage.minimum}</minimum>
								</limit>
							</limits>
						</rule>
					</rules>
				</configuration>

				<executions>
					<execution>
						<id>jacoco-initialize</id>

						<goals>
							<goal>prepare-agent</goal>
						</goals>
					</execution>

					<execution>
						<id>jacoco-check</id>
						<phase>test</phase>

						<goals>
							<goal>report</goal>
							<goal>check</goal>
						</goals>
					</execution>
				</executions>
			</plugin>

			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-checkstyle-plugin</artifactId>
				<version>${maven-checkstyle-plugin.version}</version>

				<configuration>
					<consoleOutput>true</consoleOutput>
					<failsOnError>true</failsOnError>
					<failOnViolation>true</failOnViolation>
					<violationSeverity>warning</violationSeverity>
					<linkXRef>false</linkXRef>
					<encoding>${project.build.sourceEncoding}</encoding>
					<configLocation>google_checks.xml</configLocation>
					<suppressionsFileExpression>checkstyle.suppressions.file</suppressionsFileExpression>
					<excludes>org/controlaltdel/**/model/**
					</excludes>
				</configuration>

				<executions>
					<execution>
						<id>checkstyle-check</id>
						<phase>test</phase>

						<goals>
							<goal>check</goal>
						</goals>
					</execution>
				</executions>
			</plugin>

			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-pmd-plugin</artifactId>
				<version>${maven-pmd-plugin.version}</version>

				<configuration>
					<linkXRef>false</linkXRef>
					<aggregate>true</aggregate>
					<printFailingErrors>true</printFailingErrors>
					<sourceEncoding>${project.build.sourceEncoding}</sourceEncoding>
					<minimumPriority>${pmd.priority}</minimumPriority>
					<minimumTokens>${cpd.tokens}</minimumTokens>
					<targetJdk>${java.version}</targetJdk>
					<rulesets>
						<ruleset>pmd.xml</ruleset>
					</rulesets>
				</configuration>

				<executions>
					<execution>
						<id>pmd-check</id>
						<phase>test</phase>

						<goals>
							<goal>pmd</goal>
							<goal>cpd</goal>
						</goals>
					</execution>
				</executions>
			</plugin>

			<plugin>
				<groupId>com.github.spotbugs</groupId>
				<artifactId>spotbugs-maven-plugin</artifactId>
				<version>${spotbugs-maven-plugin.version}</version>

				<configuration>
					<effort>Max</effort>
					<threshold>Low</threshold>
					<failOnError>true</failOnError>
				</configuration>

				<executions>
					<execution>
						<id>spotbugs-check</id>
						<phase>test</phase>

						<goals>
							<goal>spotbugs</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>

			<plugin>
				<groupId>com.github.eirslett</groupId>
				<artifactId>frontend-maven-plugin</artifactId>
				<version>1.9.1</version>
				<configuration>
					<workingDirectory>frontend</workingDirectory>
					<installDirectory>target</installDirectory>
				</configuration>
				<executions>
					<execution>
						<id>install node and npm</id>
						<goals>
							<goal>install-node-and-npm</goal>
						</goals>
						<configuration>
							<nodeVersion>v12.16.1</nodeVersion>
							<npmVersion>6.13.4</npmVersion>
						</configuration>
					</execution>
					<execution>
						<id>npm install</id>
						<goals>
							<goal>npm</goal>
						</goals>
						<configuration>
							<arguments>install</arguments>
						</configuration>
					</execution>
					<execution>
						<id>npm run build</id>
						<goals>
							<goal>npm</goal>
						</goals>
						<configuration>
							<arguments>run build</arguments>
						</configuration>
					</execution>
				</executions>
			</plugin>

			<plugin>
				<artifactId>maven-antrun-plugin</artifactId>
				<executions>
					<execution>
						<phase>generate-resources</phase>
						<configuration>
							<target>
								<copy todir="${project.build.directory}/classes/public">
									<fileset dir="${project.basedir}/frontend/build"/>
								</copy>
							</target>
						</configuration>
						<goals>
							<goal>run</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

	<repositories>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
		</repository>
	</repositories>
	<pluginRepositories>
		<pluginRepository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
		</pluginRepository>
	</pluginRepositories>

</project>
```

Next we want to create our app config file: `src\main\resources\bootstrap.yml`

```yaml
spring:
  main:
    banner-mode: "off" # Requires to be quoted
  application:
    name: service
  datasource.driverClassName: com.mysql.jdbc.Driver
  datasource:
    url: "jdbc:mysql://${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}?useSSL=false&useConfigs=maxPerformance"
    username: "${DATABASE_USER}"
    password: "${DATABASE_PASSWORD}"
    hikari:
      connectionTimeout: 3000
      maxLifetime: 60000
      prepStmtCacheSize: 250
      prepStmtCacheSqlLimit: 2048
      connectionTestQuery: "SELECT 1"
  http:
    log-request-details: "${LOG_HTTP_REQUEST_DETAILS:true}"
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: "${GOOGLE_OAUTH2_CLIENT_ID}"
            client-secret: "${GOOGLE_OAUTH2_CLIENT_SECRET}"

logging:
  level:
    org:
      springframework:
        web: "${LOG_LEVEL:DEBUG}"
      controlaltdel:
        sample:
          service: "${LOG_LEVEL:DEBUG}"

server:
  error:
    whitelabel:
      enabled: "false"
  port: 8080
  servlet:
    context-path: /

management:
  endpoint:
    metrics:
      enabled: true
  endpoints:
    web:
      exposure:
        include: "*"
  server:
    port: 8001
    servlet:
      context-path: /
springfox:
  documentation:
    swagger:
      v2:
        path: "/api-docs"
```

For logging, we also want a config: `src\main\resources\logback-spring.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="!json-output">
        <include resource="org/springframework/boot/logging/logback/base.xml"/>
    </springProfile>

    <springProfile name="json-output">
        <appender name="json" class="ch.qos.logback.core.ConsoleAppender">
            <layout class="ch.qos.logback.contrib.json.classic.JsonLayout">
                <jsonFormatter class="ch.qos.logback.contrib.jackson.JacksonJsonFormatter">
                    <prettyPrint>false</prettyPrint>
                </jsonFormatter>
                <timestampFormat>yyyy-MM-dd' 'HH:mm:ss.SSS</timestampFormat>
            </layout>
        </appender>

        <root level="all">
            <appender-ref ref="json" />
        </root>
    </springProfile>
</configuration>
```

That's it for config, now moving on to the actual code itself. Thankfully Spring Boot helps remove lots of boilerplate. Also there is very little business logic in this codebase, which should keep things easy to understand.

## Model

Models in this code is pretty straight forward. We'll use one model to represent the entries in the database, and one model to represent the values we'll receive from the geo-location API we'll be talking to.

`src\main\java\org\controlaltdel\sample\model\IpLookupResponse.java`:

```java
package org.controlaltdel.sample.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class IpLookupResponse {

  private String countryCode;
  private String countryCode3;
  private String countryName;
  private String countryEmoji;
}
```

`src\main\java\org\controlaltdel\sample\model\Visitor.java`:

```java
package org.controlaltdel.sample.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.NonNull;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Visitor {

  @NonNull
  private Integer id;
  @NonNull
  private String ip;
  private String countryCode;
}
```

## Database

To talk to the database, we'll also keep things super simple and use JdbcTemplate. We'll have one repository that deals with IP mapping related things, and one that deals with adding entries and viewing them.

`src\main\java\org\controlaltdel\sample\repository\VisitorRepository.java`:

```java
package org.controlaltdel.sample.repository;

import java.util.List;
import org.controlaltdel.sample.model.Visitor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class VisitorRepository {

  /**
   * Query to retrieve all visitors.
   */
  private static final String LIST_VISITORS_SQL = "SELECT * FROM tblVisitors";

  /**
   * Query to add a visitor.
   */
  private static final String INSERT_SQL = "INSERT INTO tblVisitors VALUES(NULL,?,NULL)";

  private JdbcTemplate mysqlJdbcTemplate;

  @Autowired
  public VisitorRepository(final JdbcTemplate mysqlJdbcTemplate) {
    this.mysqlJdbcTemplate = mysqlJdbcTemplate;
  }

  /**
   * Retrieves all visitors.
   *
   * @return A list of visitors.
   */
  public List<Visitor> listVisitors() {
    return this.mysqlJdbcTemplate
        .query(LIST_VISITORS_SQL, new BeanPropertyRowMapper(Visitor.class));
  }

  /**
   * Updates the country code for a given IP address.
   *
   * @param ip An IP address, as a string.
   */
  public void addVisitor(final String ip) {
    this.mysqlJdbcTemplate.update(
        INSERT_SQL,
        ip
    );
  }
}
```

`src\main\java\org\controlaltdel\sample\repository\IpRepository.java`:

```java
package org.controlaltdel.sample.repository;


import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class IpRepository {

  /**
   * Query to retrieve all unmapped IP addresses.
   */
  private static final String GRAB_UNMAPPED_IPS =
      "SELECT ip FROM tblVisitors WHERE countryCode IS NULL";

  /**
   * Query to update a IP's information.
   */
  private static final String UPDATE_SQL = "UPDATE tblVisitors SET countryCode = ? WHERE ip = ?";

  private JdbcTemplate mysqlJdbcTemplate;

  @Autowired
  public IpRepository(final JdbcTemplate mysqlJdbcTemplate) {
    this.mysqlJdbcTemplate = mysqlJdbcTemplate;
  }

  /**
   * Retrieves all ips that don't have country mapping.
   *
   * @return A list of ips.
   */
  public List<String> listUnmappedIps() {
    return this.mysqlJdbcTemplate.queryForList(String.format(GRAB_UNMAPPED_IPS), String.class);
  }

  /**
   * Updates the country code for a given IP address.
   *
   * @param ip          An IP address, as a string.
   * @param countryCode The country code.
   */
  public void updateIp(final String ip, final String countryCode) {
    this.mysqlJdbcTemplate.update(
        UPDATE_SQL,
        countryCode,
        ip
    );
  }
}
```

## App boilerplate

Nothing fancy here, just a regular Spring Boot App. Note that we do add the `@EnableScheduling` annotation, which will allow us to schedule tasks to run at regular/specific intervals. We'll need that later on for our daily task run.

`src\main\java\org\controlaltdel\sample\Application.java`

```java
package org.controlaltdel.sample;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}
```

## Web Security Configuration

Here we'll extend `WebSecurityConfigurerAdapter` and override the `configure` method to produce the configuration which will all unfettered access some paths, and require an OAuth2 login for everything else.

We'll configure CSRF to add a token to a cookie returned to the browser. Our front-end app will have to pass that back in an HTTP header in order for the backend to accept the requests. This will mitigate CSRF attacks.

For the purpose of brevity, we're going to disable CORS, although that's probably something you would want to configure for a real non-trivial service.

`src\main\java\org\controlaltdel\sample\configuration\SecurityConfiguration.java`:

```java
package org.controlaltdel.sample.configuration;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

  @Override
  protected void configure(final HttpSecurity http) throws Exception {
    http.csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());    http.cors().disable();
    http
        .antMatcher("/**")
        .authorizeRequests()
        .antMatchers("/login**", "/webjars/**", "/error**", "/actuator/**")
        .permitAll()
        .anyRequest()
        .authenticated().and()
        .oauth2Login();
  }
}
```

## Automatically documenting our APIs

To automatically generate API documentation for our APIs, let's setup Swagger.

`src\main\java\org\controlaltdel\sample\configuration\SwaggerConfiguration.java`:

```java
package org.controlaltdel.sample.configuration;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

@Configuration
@EnableSwagger2
public class SwaggerConfiguration {

  /**
   * Docket.
   *
   * @return A docket.
   */
  @Bean
  public Docket docket() {
    return new Docket(DocumentationType.SWAGGER_2)
        .select()
        .apis(RequestHandlerSelectors.basePackage("org.controlaltdel.sample.controllers.api"))
        .paths(PathSelectors.any())
        .build();
  }
}
```

Once our service is running, this will expose an HTTP endpoint that will display the API documentation. (http://localhost:8080/swagger-ui.html)

![null](/images/swagger.jpg)

![null](/images/swagger-1.jpg)

## Services

Our back-end code does two types of operations which we'll be building out API endpoints for. We'll define those in services, which we'll expose using the web controllers.

The first one, allows us to perform lookups on a public ip2country API (https://api.ip2country.info)

`org\controlaltdel\sample\service\IpLookupService.java`:

```java
package org.controlaltdel.sample.service;

import java.util.Optional;
import org.controlaltdel.sample.model.IpLookupResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClientException;
import org.springframework.web.client.RestTemplate;

@Component
public class IpLookupService {

  private static final Logger log = LoggerFactory.getLogger(IpLookupService.class);
  private static final String API_URL = "https://api.ip2country.info/ip?%s";

  /**
   * Query the IP2Country API.
   *
   * @param ip The IP address to lookup.
   * @return A string that represents the country code, or null.
   */
  public String lookupIp(final String ip) {
    String countryCode = "unknown";
    RestTemplate restTemplate = new RestTemplate();
    try {
      IpLookupResponse res = restTemplate
          .getForObject(String.format(API_URL, ip), IpLookupResponse.class);
      if (res != null) {
        countryCode = Optional.ofNullable(res.getCountryCode()).orElse("unknown");
      }
    } catch (RestClientException e) {
      log.debug(e.getMessage());
    }
    return countryCode;
  }
}
```

Our second service is concerned with updating all the unmapped IP addresses in our database. One thing to note here is that we are creating a thread pool and parallelizing the API calls. We use the `com.pivovarit.collectors.ParallelCollectors` library which greatly simplifies the logic in handling the Futures.

`src\main\java\org\controlaltdel\sample\service\IpUpdateService.java`:

```java
package org.controlaltdel.sample.service;

import static com.pivovarit.collectors.ParallelCollectors.parallelToMap;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import org.controlaltdel.sample.repository.IpRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class IpUpdateService {

  private static final Logger log = LoggerFactory.getLogger(IpUpdateService.class);
  private static final Integer PARALLEL_REQUESTS = 10;

  private IpRepository ipRepository;
  private IpLookupService ipLookupService;
  private ExecutorService executor;

  /**
   * Constructor.
   * @param ipRepository The IP repository.
   * @param ipLookupService The IP lookup service.
   */
  @Autowired
  public IpUpdateService(
      final IpRepository ipRepository,
      final IpLookupService ipLookupService) {
    this.ipRepository = ipRepository;
    this.ipLookupService = ipLookupService;
    this.executor = Executors.newFixedThreadPool(PARALLEL_REQUESTS);
  }

  /**
   * Updates all country codes that aren't set.
   */
  public void updateIps() {
    log.debug("Updating all ips");
    this.ipRepository
        .listUnmappedIps()
        .stream()
        .collect(parallelToMap(i -> i, i -> this.ipLookupService.lookupIp(i), this.executor,
            PARALLEL_REQUESTS))
        .join()
        .forEach((ip, countryCode) -> {
          if (!"unknown".equals(countryCode)) {
            this.ipRepository.updateIp(ip, countryCode);
          }
        });
  }
}
```

Here we can see 10 parallel requests being launched simultaneously:

![null](/images/parallel-requests.jpg)

## Web Controllers

We're going to create three web controllers. The first one is the controller that will handle redirecting URL paths to our single-page web app. This will allow folks to do browser refreshes and not get a 404.

`org\controlaltdel\sample\controllers\RedirectController.java`:

```java
package org.controlaltdel.sample.controllers;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class RedirectController {

  @GetMapping(value = {"/{regex:\\w+}", "/**/{regex:\\w+}"})
  public String forward404() {
    return "forward:/";
  }

}
```

The next two web controllers we'll need are the API controllers.

The IP lookup controller can list all unmapped IPs, perform a realtime IP geo-location lookup, and trigger a mass-update to lookup all unmapped IPs. 

`src\main\java\org\controlaltdel\sample\controllers\api\v1\IpLookupController.java`:

```java
package org.controlaltdel.sample.controllers.api.v1;

import java.io.IOException;
import java.util.List;
import org.controlaltdel.sample.repository.IpRepository;
import org.controlaltdel.sample.service.IpLookupService;
import org.controlaltdel.sample.service.IpUpdateService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
@RequestMapping("/api/v1")
public class IpLookupController {

  private final IpUpdateService ipUpdateService;
  private final IpLookupService ipLookupService;
  private final IpRepository ipRepository;

  /**
   * Constructor.
   * @param ipRepository The IP Repository.
   * @param ipUpdateService The update service.
   * @param ipLookupService The IP lookup service.
   */
  @Autowired
  public IpLookupController(final IpRepository ipRepository,
      final IpUpdateService ipUpdateService,
      final IpLookupService ipLookupService) {
    this.ipUpdateService = ipUpdateService;
    this.ipLookupService = ipLookupService;
    this.ipRepository = ipRepository;
  }

  /**
   * Retrieves all unmapped IPs.
   *
   * @return A list of ips as strings.
   * @throws IOException IO exception.
   */
  @RequestMapping(value = "ips", method = RequestMethod.GET)
  @ResponseBody
  public ResponseEntity<List<String>> listIps() throws IOException {
    return new ResponseEntity<List<String>>(this.ipRepository.listUnmappedIps(), HttpStatus.OK);
  }

  /**
   * Fetches country code for a given IP.
   *
   * @param ip The ip.
   * @return A string containing country code.
   */
  @RequestMapping(value = "lookup/{ip}", method = RequestMethod.GET)
  @ResponseBody
  public ResponseEntity<String> lookupIp(final @PathVariable("ip") String ip) {
    return new ResponseEntity<String>(this.ipLookupService.lookupIp(ip), HttpStatus.OK);
  }

  /**
   * Updates all ips that are unmapped.
   *
   * @return HTTP status code accepted.
   */
  @RequestMapping(value = "update", method = RequestMethod.GET)
  @ResponseBody
  public ResponseEntity updateAllIps() {
    this.ipUpdateService.updateIps();
    return new ResponseEntity<>(HttpStatus.ACCEPTED);
  }
}
```

The Visitor controller allows us to add new visitors to our database and list all the visitor ips and country codes.

`src\main\java\org\controlaltdel\sample\controllers\api\v1\VisitorController.java`:

```java
package org.controlaltdel.sample.controllers.api.v1;

import java.io.IOException;
import java.util.List;
import org.controlaltdel.sample.model.Visitor;
import org.controlaltdel.sample.repository.VisitorRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
@RequestMapping("/api/v1")
public class VisitorController {
  private VisitorRepository visitorRepository;

  @Autowired
  public VisitorController(final VisitorRepository visitorRepository) {
    this.visitorRepository = visitorRepository;
  }

  /**
   * Retrieves all visitors.
   * @return The list of visitors
   */
  @RequestMapping(value = "visitor", method = RequestMethod.GET)
  @ResponseBody
  public ResponseEntity<List<Visitor>> listVisitors() throws IOException {
    return new ResponseEntity<List<Visitor>>(this.visitorRepository.listVisitors(), HttpStatus.OK);
  }

  /**
   * Add a visitor.
   * @param ip An IP.
   * @return An HTTP status code.
   */
  @RequestMapping(value = "visitor/{ip}", method = RequestMethod.POST)
  @ResponseBody
  public ResponseEntity addVisitor(final @PathVariable("ip") String ip) throws IOException {
    this.visitorRepository.addVisitor(ip);
    return new ResponseEntity<>(HttpStatus.ACCEPTED);
  }

}
```

## Scheduled task

The last part of our back-end is the daily scheduled task. With the built-in scheduler that we added with the `@EnableScheduler` annotation on our Spring Boot Application, we can add a `@Scheduled` annotation with a cron expression to invoke a function on a regular interval.

`src\main\java\org\controlaltdel\sample\task\DailyIpCountryUpdate.java`:

```java
package org.controlaltdel.sample.task;

import org.controlaltdel.sample.service.IpUpdateService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class DailyIpCountryUpdate {

  private static final Logger log = LoggerFactory.getLogger(DailyIpCountryUpdate.class);
  private final IpUpdateService ipUpdateService;

  @Autowired
  public DailyIpCountryUpdate(final IpUpdateService ipUpdateService) {
    this.ipUpdateService = ipUpdateService;
  }

  /**
   * Scheduled task to update all ips that are unmapped. second, minute, hour, day of month, month,
   * day(s) of week
   */
  @Scheduled(cron = "0 0 2 * * ?")
  public void updateAllIps() {
    log.info("Running daily number update");
    this.ipUpdateService.updateIps();
  }
}
```

# Frontend

We're going to assume that we've got a modern tool chain ready to go.

To create our front-end, we'll use 

```
npx create-react-app frontend
```

Which will initialize an empty React app in our `frontend/` folder.

To keep this already long write-up a bit shorter, I'll omit some of the front-end files which are boilerplate.

## Setup

Nothing fancy here, just a few dependencies and a proxy setting that will allow us to keep our URI paths relative in the code.

`frontend\package.json`:

```json
{
  "name": "frontend",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "@material-ui/core": "^4.9.0",
    "@material-ui/icons": "^4.5.1",
    "@testing-library/jest-dom": "^4.2.4",
    "@testing-library/react": "^9.3.2",
    "@testing-library/user-event": "^7.1.2",
    "react": "^16.13.1",
    "react-dom": "^16.13.1",
    "react-router": "^5.1.2",
    "react-router-dom": "^5.1.2",
    "react-scripts": "3.4.1"
  },
  "proxy": "http://localhost:8080",
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": "react-app"
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  }
}
```

## Code

There are three files in this little app.

`frontend\src\App.js`:

```javascript
import React from 'react';
import { Route, BrowserRouter as Router } from "react-router-dom";
import AddVisitor from "./Components/AddVisitor";
import ListVisitors from "./Components/ListVisitors";

import './App.css';

function App() {
  return (
    <Router>
        <Route exact path="/" component={AddVisitor} />
        <Route exact path="/visitors" component={ListVisitors} />
      </Router>
  );
}

export default App;
```

`frontend\src\Components\AddVisitor.js`:

```javascript
import React from "react";
import TextField from "@material-ui/core/TextField";
import Button from "@material-ui/core/Button";
import { Link } from "react-router-dom";
import Typography from "@material-ui/core/Typography";

export default function AddVisitor() {
    const [ip, setIp] = React.useState("");
    const handleIpChange = event => setIp(event.target.value);

    async function sendRequest() {
        const response = await fetch(
            `/api/v1/visitor/${ip}`, 
            {
                method: "POST", 
                credentials: "include",
            }
        );
        let body = await response.body;
        console.log(body);
    }

    const handleSubmit = ip => {
        sendRequest(ip);
        setIp("");
    }
    return (
        <>
        <TextField
                id="ip"
                label="ip"
                type="string"
                onChange={handleIpChange}
              />
        <Button preventDefault onClick={handleSubmit}>
            Save
        </Button>
        <Link to="/visitors">
            <Typography align="left">
            List visitors
            </Typography>{" "}
        </Link>
        </>
    )
}
```

`frontend\src\Components\ListVisitors.js`:

```javascript
import React from "react";
import { makeStyles } from "@material-ui/core/styles";
import Table from "@material-ui/core/Table";
import TableBody from "@material-ui/core/TableBody";
import TableCell from "@material-ui/core/TableCell";
import TableContainer from "@material-ui/core/TableContainer";
import TableHead from "@material-ui/core/TableHead";
import TableRow from "@material-ui/core/TableRow";
import Paper from "@material-ui/core/Paper";
import Typography from "@material-ui/core/Typography";
import CircularProgress from "@material-ui/core/CircularProgress";
import { Link } from "react-router-dom";

const useStyles = makeStyles(theme => ({
  table: {
    minWidth: 600
  },
  paper: {
    display: "flex",
    flexDirection: "column",
    justifyContent: "center",
    alignItems: "center",
    margin: `10px`,
    height: "100%",
    width: "99%",
    marginTop: theme.spacing(7)
  }
}));

export default function SimpleTable() {
  const classes = useStyles();

  const [data, updateData] = React.useState([]);
  const [firstLoad, setLoad] = React.useState(true);
  let isLoading = true;

  async function listVisitors() {
    let response = await fetch("/api/v1/visitor");
    let body = await response.json();
    updateData(body);
  }

  if (firstLoad) {
    listVisitors();
    setLoad(false);
  }

  if (data.length > 0) isLoading = false;

  return (
    <div className={classes.paper}>
      <Typography component="h1" variant="h5">
        Visitor List
      </Typography>

      {isLoading ? (
        <CircularProgress />
      ) : (
        <TableContainer
          style={{ width: "80%", margin: "0 10px" }}
          component={Paper}
        >
          <Table className={classes.table} aria-label="simple table">
            <TableHead>
              <TableRow>
                <TableCell align="center">ID</TableCell>
                <TableCell align="center">IP Address</TableCell>
                <TableCell align="center">Country Code</TableCell>
              </TableRow>
            </TableHead>
            <TableBody>
              {data?.map(row => (
                <TableRow key={row.id}>
                  <TableCell align="center">{row.id}</TableCell>
                  <TableCell align="center">{row.ip}</TableCell>
                  <TableCell align="center">{row.countryCode}</TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </TableContainer>
      )}
              <Link to="/">

        <Typography align="left">
          &#x2190; Add another visitor
        </Typography>{" "}
        </Link>
    </div>
  );
}
```

# OAuth2 Setup

My little demo system uses OAuth2 to authenticate users against my domain (control-alt-del.org) which is hosted in Google. Heading on over to https://console.developers.google.com/

You'll need to have 'Google Cloud Platform' enabled in your Google Admin.

You'll need to create an application, then create OAuth2 credentials of type 'Web application'.

For this demo, I listed my `Authorized Javascript Origins` as:

* http://localhost:8080
* http://localhost:3000

And my `Authorised redirect URI` as "http://localhost:8080/login/oauth2/code/google"

For a real deployment, you'd want one set of credentials for local dev that would look like this, and one set for production which would have actual urls.

# IDE Setup

## Back-end

I'm using Intellij Idea, with pretty much stock settings. Before starting, I installed OpenJDK 11, and configured Intellij to use that installation.

I did import the google style 

`google_style.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<code_scheme name="GoogleStyle">
  <option name="OTHER_INDENT_OPTIONS">
    <value>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
      <option name="TAB_SIZE" value="2" />
      <option name="USE_TAB_CHARACTER" value="false" />
      <option name="SMART_TABS" value="false" />
      <option name="LABEL_INDENT_SIZE" value="0" />
      <option name="LABEL_INDENT_ABSOLUTE" value="false" />
      <option name="USE_RELATIVE_INDENTS" value="false" />
    </value>
  </option>
  <option name="INSERT_INNER_CLASS_IMPORTS" value="true" />
  <option name="CLASS_COUNT_TO_USE_IMPORT_ON_DEMAND" value="999" />
  <option name="NAMES_COUNT_TO_USE_IMPORT_ON_DEMAND" value="999" />
  <option name="PACKAGES_TO_USE_IMPORT_ON_DEMAND">
    <value />
  </option>
  <option name="IMPORT_LAYOUT_TABLE">
    <value>
      <package name="" withSubpackages="true" static="true" />
      <emptyLine />
      <package name="" withSubpackages="true" static="false" />
    </value>
  </option>
  <option name="RIGHT_MARGIN" value="100" />
  <option name="JD_ALIGN_PARAM_COMMENTS" value="false" />
  <option name="JD_ALIGN_EXCEPTION_COMMENTS" value="false" />
  <option name="JD_P_AT_EMPTY_LINES" value="false" />
  <option name="JD_KEEP_EMPTY_PARAMETER" value="false" />
  <option name="JD_KEEP_EMPTY_EXCEPTION" value="false" />
  <option name="JD_KEEP_EMPTY_RETURN" value="false" />
  <option name="KEEP_CONTROL_STATEMENT_IN_ONE_LINE" value="false" />
  <option name="KEEP_BLANK_LINES_BEFORE_RBRACE" value="0" />
  <option name="KEEP_BLANK_LINES_IN_CODE" value="1" />
  <option name="BLANK_LINES_AFTER_CLASS_HEADER" value="0" />
  <option name="ALIGN_MULTILINE_PARAMETERS" value="false" />
  <option name="ALIGN_MULTILINE_FOR" value="false" />
  <option name="CALL_PARAMETERS_WRAP" value="1" />
  <option name="METHOD_PARAMETERS_WRAP" value="1" />
  <option name="EXTENDS_LIST_WRAP" value="1" />
  <option name="THROWS_KEYWORD_WRAP" value="1" />
  <option name="METHOD_CALL_CHAIN_WRAP" value="1" />
  <option name="BINARY_OPERATION_WRAP" value="1" />
  <option name="BINARY_OPERATION_SIGN_ON_NEXT_LINE" value="true" />
  <option name="TERNARY_OPERATION_WRAP" value="1" />
  <option name="TERNARY_OPERATION_SIGNS_ON_NEXT_LINE" value="true" />
  <option name="FOR_STATEMENT_WRAP" value="1" />
  <option name="ARRAY_INITIALIZER_WRAP" value="1" />
  <option name="WRAP_COMMENTS" value="true" />
  <option name="IF_BRACE_FORCE" value="3" />
  <option name="DOWHILE_BRACE_FORCE" value="3" />
  <option name="WHILE_BRACE_FORCE" value="3" />
  <option name="FOR_BRACE_FORCE" value="3" />
  <option name="SPACE_BEFORE_ARRAY_INITIALIZER_LBRACE" value="true" />
  <AndroidXmlCodeStyleSettings>
    <option name="USE_CUSTOM_SETTINGS" value="true" />
    <option name="LAYOUT_SETTINGS">
      <value>
        <option name="INSERT_BLANK_LINE_BEFORE_TAG" value="false" />
      </value>
    </option>
  </AndroidXmlCodeStyleSettings>
  <JSCodeStyleSettings>
    <option name="INDENT_CHAINED_CALLS" value="false" />
  </JSCodeStyleSettings>
  <Python>
    <option name="USE_CONTINUATION_INDENT_FOR_ARGUMENTS" value="true" />
  </Python>
  <TypeScriptCodeStyleSettings>
    <option name="INDENT_CHAINED_CALLS" value="false" />
  </TypeScriptCodeStyleSettings>
  <XML>
    <option name="XML_ALIGN_ATTRIBUTES" value="false" />
    <option name="XML_LEGACY_SETTINGS_IMPORTED" value="true" />
  </XML>
  <codeStyleSettings language="CSS">
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="ECMA Script Level 4">
    <option name="KEEP_BLANK_LINES_IN_CODE" value="1" />
    <option name="ALIGN_MULTILINE_PARAMETERS" value="false" />
    <option name="ALIGN_MULTILINE_FOR" value="false" />
    <option name="CALL_PARAMETERS_WRAP" value="1" />
    <option name="METHOD_PARAMETERS_WRAP" value="1" />
    <option name="EXTENDS_LIST_WRAP" value="1" />
    <option name="BINARY_OPERATION_WRAP" value="1" />
    <option name="BINARY_OPERATION_SIGN_ON_NEXT_LINE" value="true" />
    <option name="TERNARY_OPERATION_WRAP" value="1" />
    <option name="TERNARY_OPERATION_SIGNS_ON_NEXT_LINE" value="true" />
    <option name="FOR_STATEMENT_WRAP" value="1" />
    <option name="ARRAY_INITIALIZER_WRAP" value="1" />
    <option name="IF_BRACE_FORCE" value="3" />
    <option name="DOWHILE_BRACE_FORCE" value="3" />
    <option name="WHILE_BRACE_FORCE" value="3" />
    <option name="FOR_BRACE_FORCE" value="3" />
    <option name="PARENT_SETTINGS_INSTALLED" value="true" />
  </codeStyleSettings>
  <codeStyleSettings language="HTML">
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="JAVA">
    <option name="KEEP_CONTROL_STATEMENT_IN_ONE_LINE" value="false" />
    <option name="KEEP_BLANK_LINES_IN_CODE" value="1" />
    <option name="BLANK_LINES_AFTER_CLASS_HEADER" value="1" />
    <option name="ALIGN_MULTILINE_PARAMETERS" value="false" />
    <option name="ALIGN_MULTILINE_RESOURCES" value="false" />
    <option name="ALIGN_MULTILINE_FOR" value="false" />
    <option name="CALL_PARAMETERS_WRAP" value="1" />
    <option name="METHOD_PARAMETERS_WRAP" value="1" />
    <option name="EXTENDS_LIST_WRAP" value="1" />
    <option name="THROWS_KEYWORD_WRAP" value="1" />
    <option name="METHOD_CALL_CHAIN_WRAP" value="1" />
    <option name="BINARY_OPERATION_WRAP" value="1" />
    <option name="BINARY_OPERATION_SIGN_ON_NEXT_LINE" value="true" />
    <option name="TERNARY_OPERATION_WRAP" value="1" />
    <option name="TERNARY_OPERATION_SIGNS_ON_NEXT_LINE" value="true" />
    <option name="FOR_STATEMENT_WRAP" value="1" />
    <option name="ARRAY_INITIALIZER_WRAP" value="1" />
    <option name="WRAP_COMMENTS" value="true" />
    <option name="IF_BRACE_FORCE" value="3" />
    <option name="DOWHILE_BRACE_FORCE" value="3" />
    <option name="WHILE_BRACE_FORCE" value="3" />
    <option name="FOR_BRACE_FORCE" value="3" />
    <option name="PARENT_SETTINGS_INSTALLED" value="true" />
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="JSON">
    <indentOptions>
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="JavaScript">
    <option name="RIGHT_MARGIN" value="80" />
    <option name="KEEP_BLANK_LINES_IN_CODE" value="1" />
    <option name="ALIGN_MULTILINE_PARAMETERS" value="false" />
    <option name="ALIGN_MULTILINE_FOR" value="false" />
    <option name="CALL_PARAMETERS_WRAP" value="1" />
    <option name="METHOD_PARAMETERS_WRAP" value="1" />
    <option name="BINARY_OPERATION_WRAP" value="1" />
    <option name="BINARY_OPERATION_SIGN_ON_NEXT_LINE" value="true" />
    <option name="TERNARY_OPERATION_WRAP" value="1" />
    <option name="TERNARY_OPERATION_SIGNS_ON_NEXT_LINE" value="true" />
    <option name="FOR_STATEMENT_WRAP" value="1" />
    <option name="ARRAY_INITIALIZER_WRAP" value="1" />
    <option name="IF_BRACE_FORCE" value="3" />
    <option name="DOWHILE_BRACE_FORCE" value="3" />
    <option name="WHILE_BRACE_FORCE" value="3" />
    <option name="FOR_BRACE_FORCE" value="3" />
    <option name="PARENT_SETTINGS_INSTALLED" value="true" />
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="PROTO">
    <option name="RIGHT_MARGIN" value="80" />
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="2" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="protobuf">
    <option name="RIGHT_MARGIN" value="80" />
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="2" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="Python">
    <option name="KEEP_BLANK_LINES_IN_CODE" value="1" />
    <option name="RIGHT_MARGIN" value="80" />
    <option name="ALIGN_MULTILINE_PARAMETERS" value="false" />
    <option name="PARENT_SETTINGS_INSTALLED" value="true" />
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="SASS">
    <indentOptions>
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="SCSS">
    <indentOptions>
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="TypeScript">
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
  </codeStyleSettings>
  <codeStyleSettings language="XML">
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="2" />
      <option name="TAB_SIZE" value="2" />
    </indentOptions>
    <arrangement>
      <rules>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>xmlns:android</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>^$</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>xmlns:.*</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>^$</XML_NAMESPACE>
              </AND>
            </match>
            <order>BY_NAME</order>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:id</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>style</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>^$</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>^$</XML_NAMESPACE>
              </AND>
            </match>
            <order>BY_NAME</order>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:.*Style</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
            <order>BY_NAME</order>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_width</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_height</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_weight</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_margin</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_marginTop</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_marginBottom</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_marginStart</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_marginEnd</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_marginLeft</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_marginRight</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:layout_.*</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
            <order>BY_NAME</order>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:padding</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:paddingTop</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:paddingBottom</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:paddingStart</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:paddingEnd</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:paddingLeft</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*:paddingRight</NAME>
                <XML_ATTRIBUTE />
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*</NAME>
                <XML_NAMESPACE>http://schemas.android.com/apk/res/android</XML_NAMESPACE>
              </AND>
            </match>
            <order>BY_NAME</order>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*</NAME>
                <XML_NAMESPACE>http://schemas.android.com/apk/res-auto</XML_NAMESPACE>
              </AND>
            </match>
            <order>BY_NAME</order>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*</NAME>
                <XML_NAMESPACE>http://schemas.android.com/tools</XML_NAMESPACE>
              </AND>
            </match>
            <order>BY_NAME</order>
          </rule>
        </section>
        <section>
          <rule>
            <match>
              <AND>
                <NAME>.*</NAME>
                <XML_NAMESPACE>.*</XML_NAMESPACE>
              </AND>
            </match>
            <order>BY_NAME</order>
          </rule>
        </section>
      </rules>
    </arrangement>
  </codeStyleSettings>
  <Objective-C>
    <option name="INDENT_NAMESPACE_MEMBERS" value="0" />
    <option name="INDENT_C_STRUCT_MEMBERS" value="2" />
    <option name="INDENT_CLASS_MEMBERS" value="2" />
    <option name="INDENT_VISIBILITY_KEYWORDS" value="1" />
    <option name="INDENT_INSIDE_CODE_BLOCK" value="2" />
    <option name="KEEP_STRUCTURES_IN_ONE_LINE" value="true" />
    <option name="FUNCTION_PARAMETERS_WRAP" value="5" />
    <option name="FUNCTION_CALL_ARGUMENTS_WRAP" value="5" />
    <option name="TEMPLATE_CALL_ARGUMENTS_WRAP" value="5" />
    <option name="TEMPLATE_CALL_ARGUMENTS_ALIGN_MULTILINE" value="true" />
    <option name="ALIGN_INIT_LIST_IN_COLUMNS" value="false" />
    <option name="SPACE_BEFORE_SUPERCLASS_COLON" value="false" />
  </Objective-C>
  <Objective-C-extensions>
    <option name="GENERATE_INSTANCE_VARIABLES_FOR_PROPERTIES" value="ASK" />
    <option name="RELEASE_STYLE" value="IVAR" />
    <option name="TYPE_QUALIFIERS_PLACEMENT" value="BEFORE" />
    <file>
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Import" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Macro" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Typedef" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Enum" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Constant" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Global" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Struct" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="FunctionPredecl" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Function" />
    </file>
    <class>
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Property" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="Synthesize" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="InitMethod" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="StaticMethod" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="InstanceMethod" />
      <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind" value="DeallocMethod" />
    </class>
    <extensions>
      <pair source="cc" header="h" />
      <pair source="c" header="h" />
    </extensions>
  </Objective-C-extensions>
  <codeStyleSettings language="ObjectiveC">
    <option name="RIGHT_MARGIN" value="80" />
    <option name="KEEP_BLANK_LINES_BEFORE_RBRACE" value="1" />
    <option name="BLANK_LINES_BEFORE_IMPORTS" value="0" />
    <option name="BLANK_LINES_AFTER_IMPORTS" value="0" />
    <option name="BLANK_LINES_AROUND_CLASS" value="0" />
    <option name="BLANK_LINES_AROUND_METHOD" value="0" />
    <option name="BLANK_LINES_AROUND_METHOD_IN_INTERFACE" value="0" />
    <option name="ALIGN_MULTILINE_BINARY_OPERATION" value="false" />
    <option name="BINARY_OPERATION_SIGN_ON_NEXT_LINE" value="true" />
    <option name="FOR_STATEMENT_WRAP" value="1" />
    <option name="ASSIGNMENT_WRAP" value="1" />
    <indentOptions>
      <option name="INDENT_SIZE" value="2" />
      <option name="CONTINUATION_INDENT_SIZE" value="4" />
    </indentOptions>
  </codeStyleSettings>
</code_scheme>
```

To import this, save the above file somewhere, and in the IDE go to settings -> Code Style -> Java -> Click on gear icon -> Import scheme -> Intellij idea code style XML, and select the file.

Next we'll need to set some environment variables for the different config setups.

I created an 'Application' config, which I use to run the app from within IntelliJ, and a 'JAR Application`. Both use the environment variables:

```
DATABASE_HOST=localhost
DATABASE_PORT=3306
DATABASE_USER=mark
DATABASE_PASSWORD=topsecret
DATABASE_NAME=sample
GOOGLE_OAUTH2_CLIENT_ID=shhh.apps.googleusercontent.com
GOOGLE_OAUTH2_CLIENT_SECRET=secret
```

The Path to JAR in the Jar application for me is `C:\Users\mark\IdeaProjects\SampleProject\target\service.jar`

Once that's done, everything should work pretty much out of the box. You can run the various maven commands

If you run the application from within IntelliJ, you'll be able to hit the API endpoints as well as the Swagger UI (http://localhost:8080/swagger-ui.html). Trying to hit the site root (/) won't work, as the front-end code doesn't live inside the java codebase.

When running the `package` maven command, there's an extra goodie. It will automatically produce a production build of the React app and place it in the correct location to be packaged up in the fat JAR.

So if you want do a complete test run, you can run the maven package command, then run the jar from the IDE, and the entire app including front-end should be available on http://localhost:8080.

## VS Code

I've installed NodeJS on Windows 10, and it seems to work well with this setup. I've also used WSL (Windows Subsystem for Linux), however WSL is very very sluggish and I don't really recommend it for front-end development. A similar setup with MacOS/brew should also be trivial to setup.

The front-end workflow works out of the box with VS Code in this setup. You can launch the NPM scripts from within the IDE and hit the localhost endpoint (http://localhost:3000) to see your app live-reload while you're coding. Assuming you're running the back-end, the entire app should work.

# Database

I've dumped the database table structure and some test data in the `src\test\fixtures\data.sql` file. 

# Static code analysis

As mentioned in the intro, I've codified some tooling in this setup to enforce styling as well as analyze the code for anti-patterns, programming errors, copy paste detection, unit test code coverage, and dependency vulnerabilities.

I've only covered the back-end portion so far, I'll see if I can modify the pipeline to do the same withe front-end build.

## Unit tests

```
mvn test
```

## Unit test code coverage

```
mvn jacoco:check
```

You can then open the report in your browser in the `target\site\jacoco\index.html`, should look something like this:

![null](/images/coverage-1.jpg)

You can click through the links to view the covered/uncovered part of each file:

![null](/images/coverage-2.jpg)

## OWASP dependancy vulnerability check

```
mvn dependency-check:aggregate
```

## Coding style

```
mvn checkstyle:check
```

## Project mess detection & copy paste detection

```
mvn pmd:check
```

## Spotbugs

```
mvn spotbugs:check
```

# DevOps

This application follows all the best practices of the 12-factor app. The output of the build process (`mvn package`) produces a self-contained jar file (`target/service.jar`) that can easily be thrown into a docker container with a JRE.

All configuration of the application is done via the environment variables.

Alerting can consume the Spring actuator endpoints to retrieve metrics from the runtime and should provide a good level of visibility into the health of the application.

# TODO

* I'd eventually like to get Spring Boot admin working (client/server)
* Some unit tests. As a general rule I don't like writing lots of unit tests that just end up testing against mocks when there is no business logic in the code. As there are only 2 IF statements in the back-end, I didn't feel like it was worth it. I might just add some arbitrary complexity and up the coverage ratio to prove the point.
