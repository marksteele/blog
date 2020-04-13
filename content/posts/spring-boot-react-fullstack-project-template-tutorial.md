---
layout: blog
title: Spring Boot/React - FullStack Project Template/Tutorial
date: '2020-04-12T15:58:27-04:00'
cover: /images/didntsayword-mark-1-.gif
categories:
  - tutorial
  - Java
  - Spring Boot
---
Here's a Spring Boot project template/tutorial that I've put together to try to get some best practices codified on how to quickly throw together a service.

We'll cover: setting up REST endpoints, scheduled tasks, thread-pools to parallelize work, React, MySQL, 12-factor app best practices, OAuth2 (google), monitoring, unit testing, code coverage, project mess detection, spotbugs, DevOps and more!

This blog post is going to be a bit of a beast, so before I dig too deep let's set the stage on what we're trying to accomplish. First, we'd like to build a modern front-end and back-end system. For the front-end, we'll use React. For the back-end, we'll use Java+Spring Boot.

Our fictitious project is called Visitors. We have been tasked to build a form where users can enter an IP address, and we are supposed to save it to a database. We'd also like to see the entries in the database as well.

The stakeholders in this type of project typically are: The product owner, front-end engineer, back-end engineer, QA, DevOps & Security. Hopefully by the time we're done, everyone's happy.

In this example, we want the front-end to be packaged and deployed inside the back-end application.

On the back-end side, we'd like some APIs:

* List all IPs for which we don't have country information for.
* Lookup the location of an IP address in real-time.
* Trigger a background task that scans the database immediately and attempts to geo-locate all unknown IP locations.
* An API endpoint to add new IPs
* An API endpoint to view all the entries in the table.

In addition to the APIs, we'd also want the back-end to trigger resolving all unknown IP locations on a daily basis.

Furthermore, as the numbers of entries in the database might get large, we'd want the geo-lookups to be done in parallel, several simultaneously.

We're always security conscious, so we'll want the whole shebang to be protected via OAuth2, and authenticate via our google domain.

Our CI/CD pipelines are build around Maven and we want to create a fat jar that can run everything stand-alone. 

Our DevOps wants everything configured via environment variables.

To help with quality, consistency, and security we'll run the following checks against the code every time we build it: 

* Unit tests, 
* Unit test code coverage (>75%)
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
  datasource.url: "jdbc:mysql://${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}?useSSL=false"
  datasource.username: "${DATABASE_USER}"
  datasource.password: "${DATABASE_PASSWORD}"
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

## Security boilerplate

The following is some boilerplate code to setup OAuth2 to work with Google.

TO BE CONTINUED...
