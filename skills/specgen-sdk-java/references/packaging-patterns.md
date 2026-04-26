# Packaging Patterns — Multi-Release Fat JAR with Maven

This reference describes the canonical Maven configuration the generated spec must
emit. The goal: a single fat JAR that is also a Multi-Release JAR (MR-JAR), runs on
JDK 8 → 21 from one artifact, and bundles OkHttp + transitives under a relocated
package so it can be dropped on any classpath without conflicting with the consumer's
own dependencies.

The two non-trivial parts are:

1. Compiling the JDK 11 overlay sources to a separate directory and packaging them
   under `META-INF/versions/11/` of the JAR.
2. Preserving `Multi-Release: true` in the manifest after `maven-shade-plugin` runs —
   Shade rewrites the JAR and silently drops manifest entries unless explicitly told
   to keep them.

---

## Source Directory Layout

```
src/
├── main/
│   ├── java/                ← JDK 8 baseline. ALL public API lives here.
│   ├── java11/              ← JDK 11 overlay. Same package paths only. NO new public types.
│   └── resources/
└── test/
    └── java/
```

> **Rule:** Every class under `src/main/java11` MUST also exist in `src/main/java`
> with an identical public signature. Overlays are performance/feature alternatives,
> NOT new APIs.

---

## Complete `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.bestinet.urp</groupId>
    <artifactId>{{ARTIFACT_ID}}</artifactId>
    <version>{{VERSION}}</version>
    <packaging>jar</packaging>

    <name>{{APPLICATION_NAME}}</name>
    <description>{{APP_DESCRIPTION}}</description>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.release>8</maven.compiler.release>
        <okhttp.version>4.12.0</okhttp.version>
        <junit.version>5.10.2</junit.version>
        <assertj.version>3.26.3</assertj.version>
        <shade.relocation.base>{{BASE_PACKAGE}}.shaded</shade.relocation.base>
    </properties>

    <dependencies>
        <!-- Sole runtime third-party dependency -->
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>${okhttp.version}</version>
        </dependency>

        <!-- SLF4J facade — provided, NEVER bundled. Optional: include only if Logging = SLF4J facade -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.13</version>
            <scope>provided</scope>
        </dependency>

        <!-- Test scope -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>${assertj.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>mockwebserver</artifactId>
            <version>${okhttp.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <finalName>${project.artifactId}-${project.version}</finalName>

        <plugins>

            <!-- 1. Compile JDK 8 baseline AND JDK 11 overlay separately -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <executions>
                    <!-- Default: compile src/main/java with --release 8 -->
                    <execution>
                        <id>default-compile</id>
                        <configuration>
                            <release>8</release>
                        </configuration>
                    </execution>
                    <!-- Overlay: compile src/main/java11 with --release 11 -->
                    <execution>
                        <id>compile-java11</id>
                        <phase>compile</phase>
                        <goals><goal>compile</goal></goals>
                        <configuration>
                            <release>11</release>
                            <compileSourceRoots>
                                <root>${project.basedir}/src/main/java11</root>
                            </compileSourceRoots>
                            <outputDirectory>${project.build.directory}/classes-java11</outputDirectory>
                            <multiReleaseOutput>true</multiReleaseOutput>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <!-- 2. Build the (thin) Multi-Release JAR -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.4.2</version>
                <configuration>
                    <archive>
                        <manifestEntries>
                            <Multi-Release>true</Multi-Release>
                            <!-- If a Diagnostic main is included, set Main-Class here -->
                            <!-- <Main-Class>{{BASE_PACKAGE}}.cli.Diagnose</Main-Class> -->
                        </manifestEntries>
                    </archive>
                    <!-- Copy JDK 11 overlay classes into META-INF/versions/11/ -->
                    <includes>
                        <include>**</include>
                    </includes>
                </configuration>
                <executions>
                    <execution>
                        <id>default-jar</id>
                        <phase>package</phase>
                        <goals><goal>jar</goal></goals>
                    </execution>
                </executions>
            </plugin>

            <!-- 3. Copy java11 overlay classes into META-INF/versions/11 before packaging -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-resources-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>copy-java11-overlay</id>
                        <phase>prepare-package</phase>
                        <goals><goal>copy-resources</goal></goals>
                        <configuration>
                            <outputDirectory>${project.build.outputDirectory}/META-INF/versions/11</outputDirectory>
                            <resources>
                                <resource>
                                    <directory>${project.build.directory}/classes-java11</directory>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <!-- 4. Shade plugin — produce the fat JAR, relocate transitives, keep MR manifest -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.6.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals><goal>shade</goal></goals>
                        <configuration>
                            <createDependencyReducedPom>true</createDependencyReducedPom>
                            <shadedArtifactAttached>false</shadedArtifactAttached>
                            <minimizeJar>false</minimizeJar>

                            <relocations>
                                <relocation>
                                    <pattern>okhttp3</pattern>
                                    <shadedPattern>${shade.relocation.base}.okhttp3</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>okio</pattern>
                                    <shadedPattern>${shade.relocation.base}.okio</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>kotlin</pattern>
                                    <shadedPattern>${shade.relocation.base}.kotlin</shadedPattern>
                                </relocation>
                            </relocations>

                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                        <exclude>META-INF/MANIFEST.MF</exclude>
                                        <exclude>module-info.class</exclude>
                                    </excludes>
                                </filter>
                            </filters>

                            <transformers>
                                <!-- Preserve Multi-Release: true and other manifest entries -->
                                <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <manifestEntries>
                                        <Multi-Release>true</Multi-Release>
                                        <!-- <Main-Class>{{BASE_PACKAGE}}.cli.Diagnose</Main-Class> -->
                                        <Implementation-Title>{{APPLICATION_NAME}}</Implementation-Title>
                                        <Implementation-Version>${project.version}</Implementation-Version>
                                    </manifestEntries>
                                </transformer>
                                <!-- Merge service files (only if any SPI is exposed) -->
                                <transformer
                                    implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <!-- 5. Sources & Javadoc JARs -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>3.3.1</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals><goal>jar-no-fork</goal></goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <version>3.10.0</version>
                <configuration>
                    <source>8</source>
                    <doclint>none</doclint>
                </configuration>
                <executions>
                    <execution>
                        <id>attach-javadocs</id>
                        <goals><goal>jar</goal></goals>
                    </execution>
                </executions>
            </plugin>

            <!-- 6. Surefire — JUnit 5 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.0</version>
            </plugin>

        </plugins>
    </build>
</project>
```

---

## Why This Works

### Multi-Release JAR layout

The JVM reads `Multi-Release: true` from `META-INF/MANIFEST.MF`. When running on JDK
N (N >= 9), the classloader checks `META-INF/versions/N/`, then `META-INF/versions/N-1/`,
… down to the JAR root. The first match wins. Classes that exist only at the root are
the JDK 8 baseline.

By compiling `src/main/java11` to `target/classes-java11` and copying that tree under
`target/classes/META-INF/versions/11/`, the resulting JAR has the right shape.

### Shade plugin & MR-JAR

Shade rewrites the JAR — and historically loses MR semantics unless told to keep them.
The configuration above:

- Excludes the source artifacts' `META-INF/MANIFEST.MF` (so the consumer's manifest
  comes from the project, not the dependencies)
- Uses `ManifestResourceTransformer` with `<Multi-Release>true</Multi-Release>`
- Relocates OkHttp/Okio/Kotlin under `{{BASE_PACKAGE}}.shaded.*` to prevent classpath
  collisions (e.g., the consumer might use a different OkHttp version)
- `module-info.class` is excluded because we're a classpath JAR, not a JPMS module
  (relocation breaks `module-info.class` in any case)

### Why `<scope>provided</scope>` for SLF4J

The SLF4J facade is the de-facto standard for logging in Java libraries. Including it
as `provided` means:

- It's available at compile time
- It's NOT shaded into the fat JAR
- The consumer's classpath supplies the binding

If the consumer has no SLF4J binding at all, SLF4J's no-op binding kicks in silently —
no errors, no output. This is the documented SLF4J behaviour and is the right choice
for an SDK library.

---

## Verification After `mvn package`

```
mvn clean package

# 1. Manifest contains Multi-Release: true
unzip -p target/{{ARTIFACT_ID}}-{{VERSION}}.jar META-INF/MANIFEST.MF | grep -i multi-release

# 2. JDK 11 overlay classes are present
jar tf target/{{ARTIFACT_ID}}-{{VERSION}}.jar | grep "^META-INF/versions/11/" | head

# 3. OkHttp is relocated under {{BASE_PACKAGE}}/shaded/...
jar tf target/{{ARTIFACT_ID}}-{{VERSION}}.jar | grep "^{{BASE_PACKAGE_SLASHED}}/shaded/okhttp3/" | head

# 4. NO unrelocated transitives at JAR root
jar tf target/{{ARTIFACT_ID}}-{{VERSION}}.jar | grep -E "^okhttp3/|^okio/|^kotlin/" || echo "OK: none"

# 5. (Optional) Run on each JDK if Diagnose is included
JAVA_HOME=/path/to/jdk-8  java -jar target/{{ARTIFACT_ID}}-{{VERSION}}.jar
JAVA_HOME=/path/to/jdk-11 java -jar target/{{ARTIFACT_ID}}-{{VERSION}}.jar
JAVA_HOME=/path/to/jdk-17 java -jar target/{{ARTIFACT_ID}}-{{VERSION}}.jar
JAVA_HOME=/path/to/jdk-21 java -jar target/{{ARTIFACT_ID}}-{{VERSION}}.jar
```

All five checks must pass before the artifact is considered release-ready.

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| `<release>` left at default | Bytecode targets the JDK that ran Maven | Always set `maven.compiler.release` AND per-execution `<release>` |
| Shade drops manifest | `Multi-Release` missing → JDK 11 overlay never used | Use `ManifestResourceTransformer` with the explicit entry |
| OkHttp not relocated | Consumer's OkHttp version conflicts | Add the `<relocation>` entries above |
| `module-info.class` left in fat JAR | Module path consumers fail to load | Exclude `module-info.class` in the Shade `<filters>` |
| JDK 11 overlay introduces a new public type | API breaks on JDK 8 (NoClassDefFoundError) | Overlays MUST mirror baseline signatures |
| `merged-services` not transformed | SPI resolution breaks | Add `ServicesResourceTransformer` (only if SPI exposed) |
