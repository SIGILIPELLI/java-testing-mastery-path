# 06 · Java Setup for Testers

From here on, every module runs real code. This one gets your machine ready:
a JDK, an IDE, and — most importantly for a tester — **Maven**, the build
tool that downloads JUnit, Selenium and TestNG for you and runs your test
suite from a single command.

## 1. Install the JDK

You need a **JDK** (Java Development Kit — compiler + runtime), not just a
JRE. Java 17 or 21 (both Long-Term Support releases) is the right choice for
new test projects.

- **macOS:** `brew install openjdk@21`, or download from
  [adoptium.net](https://adoptium.net/).
- **Windows:** download the Temurin 21 MSI installer from
  [adoptium.net](https://adoptium.net/) and let it set `JAVA_HOME`.
- **Linux:** `sudo apt install openjdk-21-jdk` (Debian/Ubuntu).

Verify:

```bash
java -version
javac -version
echo $JAVA_HOME     # Windows: echo %JAVA_HOME%
```

```
openjdk version "21.0.3" 2024-04-16 LTS
OpenJDK Runtime Environment Temurin-21.0.3+9 (build 21.0.3+9-LTS)
OpenJDK 64-Bit Server VM Temurin-21.0.3+9 (build 21.0.3+9-LTS, mixed mode)
javac 21.0.3
/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home
```

!!! warning "If `JAVA_HOME` is empty"
    Maven and many test tools read `JAVA_HOME`, not your `PATH`. On
    macOS/Linux add to `~/.zshrc` or `~/.bashrc`:
    `export JAVA_HOME=$(/usr/libexec/java_home -v 21)` (macOS) or
    `export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64` (Linux). On
    Windows set it in *System Properties → Environment Variables*. Restart
    your terminal afterwards.

## 2. Install an IDE

**IntelliJ IDEA Community Edition** (free) is the standard for Java test
automation — it has the best JUnit/TestNG integration. Eclipse and VS Code
with the Extension Pack for Java also work.

The IntelliJ features you will use constantly as a tester:

| Feature | Shortcut (macOS / Windows) |
|---|---|
| Run the test under the cursor | ⌃⇧R / Ctrl+Shift+F10 |
| Re-run the last thing | ⌃R / Shift+F10 |
| Go to declaration | ⌘B / Ctrl+B |
| Find usages | ⌥F7 / Alt+F7 |
| Reformat code | ⌥⌘L / Ctrl+Alt+L |
| Search everywhere | Shift Shift |
| Toggle breakpoint | ⌘F8 / Ctrl+F8 |

## 3. Why testers need Maven

Selenium alone pulls in a dozen JAR files, each with its own dependencies.
Downloading them by hand and putting them on the classpath is unmanageable.
Maven solves three problems at once:

1. **Dependency management** — you declare "I want Selenium 4.23.0" and it
   fetches that plus everything Selenium needs.
2. **Standard project layout** — every Maven test project looks the same, so
   anyone can navigate yours.
3. **Lifecycle commands** — `mvn test` compiles and runs your whole suite,
   identically on your laptop and on a CI server (Level 3, Module 03).

Install it:

- **macOS:** `brew install maven`
- **Windows:** download the binary zip from
  [maven.apache.org](https://maven.apache.org/download.cgi), unzip, add the
  `bin` folder to `PATH`.
- **Linux:** `sudo apt install maven`

```bash
mvn -version
```

```
Apache Maven 3.9.8
Maven home: /opt/homebrew/Cellar/maven/3.9.8/libexec
Java version: 21.0.3, vendor: Eclipse Adoptium
Default locale: en_IN, platform encoding: UTF-8
OS name: "mac os x", version: "14.5", arch: "aarch64"
```

## 4. The standard Maven project layout

```
java-testing-practice/
├── pom.xml                          ← the project definition
└── src
    ├── main
    │   ├── java/                    ← application code (often empty in a QA project)
    │   └── resources/
    └── test
        ├── java/                    ← ALL your test classes live here
        │   └── com/example/tests/
        │       └── CalculatorTest.java
        └── resources/               ← test data: config.properties, testng.xml, CSV files
```

The single most important rule: **test classes go in `src/test/java`.** Maven
only runs tests found there, and code in `src/test` is excluded from the
shipped artifact. Put a test in `src/main/java` and `mvn test` will silently
ignore it.

## 5. Create the project

```bash
mkdir -p java-testing-practice/src/test/java/com/example/tests
mkdir -p java-testing-practice/src/main/java
mkdir -p java-testing-practice/src/test/resources
cd java-testing-practice
```

Now the `pom.xml` — the file that defines the whole project. This one carries
everything Modules 07–10 need:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>java-testing-practice</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- JUnit 5 -- Module 07 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.2</version>
            <scope>test</scope>
        </dependency>

        <!-- Selenium WebDriver -- Module 08 -->
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>4.23.0</version>
            <scope>test</scope>
        </dependency>

        <!-- TestNG -- Module 09 -->
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>7.10.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Surefire runs the test suite during `mvn test` -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
        </plugins>
    </build>
</project>
```

### Reading a dependency

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>      <!-- who publishes it -->
    <artifactId>junit-jupiter</artifactId>    <!-- which library -->
    <version>5.10.2</version>                 <!-- which release -->
    <scope>test</scope>                       <!-- only on the test classpath -->
</dependency>
```

Those three coordinates — groupId, artifactId, version — uniquely identify
any library on Maven Central. `<scope>test</scope>` means the JAR is
available to `src/test/java` but never bundled into the application; every
testing library should use it.

Pull everything down:

```bash
mvn dependency:resolve
```

```
[INFO] --- dependency:3.6.1:resolve (default-cli) @ java-testing-practice ---
[INFO] The following files have been resolved:
[INFO]    org.junit.jupiter:junit-jupiter:jar:5.10.2:test
[INFO]    org.seleniumhq.selenium:selenium-java:jar:4.23.0:test
[INFO]    org.testng:testng:jar:7.10.2:test
[INFO] BUILD SUCCESS
```

## 6. Java you actually need as a tester

You do not need to be a Java developer to automate tests, but these six
things appear in every test you will ever write.

```java
// 1. A class -- every test lives inside one
public class Demo {

    // 2. A method -- one test = one method
    public static void main(String[] args) {

        // 3. Variables and types
        String username = "qa.user@example.com";
        int retryCount = 3;
        double price = 249.99;
        boolean isLoggedIn = false;

        // 4. Conditionals -- how assertions decide pass/fail underneath
        if (retryCount > 0) {
            System.out.println("Retrying " + retryCount + " times");
        } else {
            System.out.println("No retries left");
        }

        // 5. Loops -- iterate over test data
        String[] browsers = {"chrome", "firefox", "edge"};
        for (String browser : browsers) {
            System.out.println("Would run on: " + browser);
        }

        // 6. Method calls on objects -- the core of Selenium
        System.out.println(username.toUpperCase());
        System.out.println(username.contains("@"));
        System.out.printf("Price: %.2f%n", price);
    }
}
```

```
Retrying 3 times
Would run on: chrome
Would run on: firefox
Would run on: edge
QA.USER@EXAMPLE.COM
true
Price: 249.99
```

### A class worth testing

Create `src/main/java/com/example/Calculator.java` — Module 07 will test it:

```java
package com.example;

public class Calculator {

    public int add(int a, int b) {
        return a + b;
    }

    public int subtract(int a, int b) {
        return a - b;
    }

    public double divide(int a, int b) {
        if (b == 0) {
            throw new IllegalArgumentException("Cannot divide by zero");
        }
        return (double) a / b;
    }

    /** Returns the discount percentage for an order amount (see Module 05). */
    public int discountFor(double orderAmount) {
        if (orderAmount < 0) {
            throw new IllegalArgumentException("Order amount cannot be negative");
        }
        if (orderAmount >= 10000) return 15;
        if (orderAmount >= 5000)  return 10;
        if (orderAmount >= 1000)  return 5;
        return 0;
    }
}
```

Notice `discountFor` implements exactly the tiered rule you partitioned in
Module 05. In Module 07 you will turn those partitions and boundaries into
executable JUnit tests — that is the bridge between manual technique and
automation.

## 7. The Maven commands you will use

| Command | What it does |
|---|---|
| `mvn clean` | Deletes `target/` — old compiled classes and reports |
| `mvn compile` | Compiles `src/main/java` |
| `mvn test-compile` | Compiles `src/test/java` |
| `mvn test` | Compiles everything and **runs the test suite** |
| `mvn clean test` | The one you will type most — a guaranteed-fresh run |
| `mvn test -Dtest=CalculatorTest` | Run one test class |
| `mvn test -Dtest=CalculatorTest#addsTwoPositiveNumbers` | Run one test method |
| `mvn dependency:tree` | Show all dependencies and where they came from |
| `mvn -v` | Version check |

Reports land in `target/surefire-reports/` as `.txt` and `.xml` — the XML is
what CI servers and reporting tools (Level 2, Module 08) consume.

## 8. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `mvn: command not found` | Maven's `bin` is not on `PATH` |
| `Source option 21 is no longer supported` | Your JDK is older than the `maven.compiler.source` in `pom.xml` — lower it to 17 or install JDK 21 |
| `No tests were executed!` | Test class is outside `src/test/java`, or its name does not match `*Test`/`Test*` |
| `Could not resolve dependencies` | Typo in a coordinate, or no network — check [central.sonatype.com](https://central.sonatype.com/) for the exact version |
| Build hangs on first run | Maven is downloading the internet into `~/.m2/repository`. It is a one-time cost |
| IDE shows red imports but `mvn test` works | Reimport the Maven project in the IDE (IntelliJ: the Maven panel's refresh icon) |

## Cheat sheet

| Item | Value |
|---|---|
| JDK version | 17 or 21 (LTS) |
| Verify Java | `java -version` |
| Build tool | Maven — `mvn -version` |
| Project definition | `pom.xml` |
| Test code location | `src/test/java` |
| Test data location | `src/test/resources` |
| Run all tests | `mvn clean test` |
| Run one class | `mvn test -Dtest=ClassName` |
| Dependency scope for test libs | `<scope>test</scope>` |
| Reports | `target/surefire-reports/` |

## Exercise

1. Install a JDK (17 or 21) and Maven. Paste the output of `java -version`
   and `mvn -version` into a file called `setup-verification.txt` as proof.
2. Create the `java-testing-practice` project exactly as laid out in section
   4, with the `pom.xml` from section 5. Run `mvn dependency:resolve` and
   confirm all three libraries download successfully.
3. Add the `Calculator` class from section 6 under
   `src/main/java/com/example/`, then run `mvn compile` and confirm you see
   `BUILD SUCCESS`.
4. Run `mvn dependency:tree` and answer: how many *transitive* dependencies
   does `selenium-java` pull in that you did not declare yourself? Name three
   of them.
5. Deliberately break something: change the Selenium version to `4.99.0` and
   run `mvn dependency:resolve`. Read the error message carefully, then fix
   it. Being able to read a Maven failure is a genuine job skill.
