# Maven uses notes

## 创建工程
mvn archetype:generate


## 编译失败

错误：

```shell
[ERROR] 不再支持源选项 5。请使用 7 或更高版本。
[ERROR] 不再支持目标选项 5。请使用 7 或更高版本。
```

解决方法：在工程的pom.xml的properties节点中指定jdk版本

```xml
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
  </properties>
```

## 生成可运行的jar包

解决方法，在工程的pom.xml的加入 插件：

```xml
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.2.4</version>
        <configuration>
          <!-- put your configurations here -->
          <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                  <mainClass>com.hdz.App</mainClass>
                </transformer>
              </transformers>
        </configuration>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
```

执行：

```shell
mvn clean compile
mvn install
```

后生成的jar包即可执行（Linux）：

```shell
lenovo@lenovo-ThinkStation-K:~/mavenTest/jdttest/jdttest$ java -jar target/jdttest-1.0-SNAPSHOT.jar 
Hello World!
```





