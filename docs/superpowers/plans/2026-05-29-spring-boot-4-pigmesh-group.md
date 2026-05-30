<!--
  Copyright 1999-2026 Alibaba Group Holding Ltd.

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

# Spring Boot 4 PigMesh Group Migration 实现计划

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 将当前 Nacos 3.2.2 代码库迁移到 Spring Boot 4.0.6，并把 Maven groupId 改为 `io.github.pig-mesh.nacos`。

**架构：** 采用定向迁移，不直接 cherry-pick `pig-mesh/nacos` 的大提交。先修 Maven 坐标和 Boot 4 依赖，再修 Spring Boot 4 包名、Jakarta 注解和测试 API，最后用 Maven 编译反馈补齐剩余兼容点。

**技术栈：** Maven multi-module、Java 17 server modules、Java 8 client/API/plugin compatibility、Spring Boot 4.0.6、Jakarta annotations、Spotless。

---

## 文件结构

修改范围按职责分组：

- Maven 坐标和依赖管理：
  - `pom.xml`
  - 所有模块 `pom.xml`
- Spring Boot 4 源码包名：
  - `ai-registry-adaptor/src/main/java/com/alibaba/nacos/airegistry/NacosAiRegistry.java`
  - `ai-registry-adaptor/src/main/java/com/alibaba/nacos/airegistry/config/HttpPathConfiguration.java`
  - `console/src/main/java/com/alibaba/nacos/console/NacosConsole.java`
  - `console/src/main/java/com/alibaba/nacos/console/config/ConsoleWebConfig.java`
  - `core/src/main/java/com/alibaba/nacos/core/code/SpringApplicationRunListener.java`
  - `core/src/main/java/com/alibaba/nacos/core/web/NacosWebServerListener.java`
  - `server/src/main/java/com/alibaba/nacos/NacosServerBasicApplication.java`
  - `server/src/main/java/com/alibaba/nacos/NacosServerWebApplication.java`
  - `plugin-default-impl/nacos-default-auth-plugin/src/main/java/com/alibaba/nacos/plugin/auth/impl/SafeBcryptPasswordEncoder.java`
- Jakarta 注解：
  - 所有仍包含 `javax.annotation` 的非生成 Java 文件
  - 排除 `api/src/main/java/com/alibaba/nacos/api/grpc/auto/**`
- Boot 4 测试 API：
  - `core/src/test/java/com/alibaba/nacos/core/web/NacosWebServerListenerTest.java`
  - `test/config-test/src/test/java/com/alibaba/nacos/test/base/HttpClient4Test.java`
  - `test/core-test/src/test/java/com/alibaba/nacos/test/base/HttpClient4Test.java`
  - `test/naming-test/src/test/java/com/alibaba/nacos/test/base/HttpClient4Test.java`
  - `test/config-test/src/test/java/com/alibaba/nacos/test/config/ConfigBetaConfigITCase.java`
  - `test/naming-test/src/test/java/com/alibaba/nacos/test/naming/MultiTenantInstanceAPINamingITCase.java`
  - `test/core-test/src/test/java/com/alibaba/nacos/test/core/auth/LdapAuthCoreITCase.java`
  - `test/core-test/src/test/java/com/alibaba/nacos/test/core/code/ControllerMethodsCacheCoreITCase.java`
- Native reflection metadata:
  - `console/src/main/resources/META-INF/native-image/com.alibaba.nacos/nacos-console/reflect-config.json`

## 任务 1：建立迁移基线

**文件：**
- 读取：`docs/superpowers/specs/2026-05-29-spring-boot-4-pigmesh-group-design.md`
- 读取：`pom.xml`
- 验证：`git status --short`

- [ ] **步骤 1：确认工作区状态**

运行：

```bash
git status --short --branch
```

预期：

```text
## develop...origin/develop
?? docs/superpowers/specs/2026-05-29-spring-boot-4-pigmesh-group-design.md
?? docs/superpowers/plans/2026-05-29-spring-boot-4-pigmesh-group.md
```

如果还有用户已有改动，记录路径；不要还原这些改动。

- [ ] **步骤 2：确认参考提交仍可读**

运行：

```bash
git -C /tmp/pigmesh-nacos-ref show -s --format='%h %s' a3025de7c 9f3fa6272 e997fd149 5b20ad569
```

预期输出包含：

```text
a3025de7c chore(deps): spring-boot upgrade from 3.4.10 to 4.0.5
9f3fa6272 chore(deps): micrometer upgrade from 1.12.8 to 1.13.0
e997fd149 refactor: migrate from javax to jakarta annotations in multiple files
5b20ad569 refactor(pom): update groupId and version for Nacos dependencies
```

如果 `/tmp/pigmesh-nacos-ref` 不存在，重新克隆：

```bash
git clone --filter=blob:none --single-branch --branch develop https://github.com/pig-mesh/nacos.git /tmp/pigmesh-nacos-ref
```

## 任务 2：迁移 Maven groupId 和内部坐标

**文件：**
- 修改：`pom.xml`
- 修改：所有模块 `pom.xml`
- 验证：`mvn -pl core -am -DskipTests compile`

- [ ] **步骤 1：修改根 POM 坐标**

在 `pom.xml` 中保持版本体系，只修改 group：

```xml
<groupId>io.github.pig-mesh.nacos</groupId>
<artifactId>nacos-all</artifactId>
<version>${revision}</version>
```

保留：

```xml
<revision>3.2.2</revision>
```

保留 SCM tag 的 revision 写法：

```xml
<tag>nacos-all-${revision}</tag>
```

- [ ] **步骤 2：修改所有模块 parent groupId**

对所有模块 `pom.xml` 的 parent 坐标做相同变更：

```xml
<parent>
    <groupId>io.github.pig-mesh.nacos</groupId>
    <artifactId>nacos-all</artifactId>
    <version>${revision}</version>
</parent>
```

覆盖这些路径：

```text
address/pom.xml
ai-registry-adaptor/pom.xml
ai/pom.xml
api/pom.xml
auth/pom.xml
bootstrap/pom.xml
client-basic/pom.xml
client/pom.xml
cmdb/pom.xml
common/pom.xml
config/pom.xml
consistency/pom.xml
console/pom.xml
copilot/pom.xml
core/pom.xml
distribution/pom.xml
example/pom.xml
istio/pom.xml
k8s-sync/pom.xml
lock/pom.xml
logger-adapter-impl/log4j2-adapter/pom.xml
logger-adapter-impl/logback-adapter-12/pom.xml
logger-adapter-impl/pom.xml
maintainer-client/pom.xml
naming/pom.xml
persistence/pom.xml
plugin-default-impl/nacos-default-ai-importer-plugin/pom.xml
plugin-default-impl/nacos-default-ai-pipeline-plugin/pom.xml
plugin-default-impl/nacos-default-ai-trace-plugin/pom.xml
plugin-default-impl/nacos-default-auth-plugin/pom.xml
plugin-default-impl/nacos-default-control-plugin/pom.xml
plugin-default-impl/nacos-default-datasource-plugin/nacos-datasource-plugin-base/pom.xml
plugin-default-impl/nacos-default-datasource-plugin/nacos-datasource-plugin-derby/pom.xml
plugin-default-impl/nacos-default-datasource-plugin/nacos-datasource-plugin-mysql/pom.xml
plugin-default-impl/nacos-default-datasource-plugin/nacos-datasource-plugin-oracle/pom.xml
plugin-default-impl/nacos-default-datasource-plugin/nacos-datasource-plugin-postgresql/pom.xml
plugin-default-impl/nacos-default-datasource-plugin/pom.xml
plugin-default-impl/nacos-default-plugin-all/pom.xml
plugin-default-impl/nacos-ldap-auth-plugin/pom.xml
plugin-default-impl/nacos-oidc-auth-plugin/pom.xml
plugin-default-impl/pom.xml
plugin/ai/pom.xml
plugin/auth/pom.xml
plugin/config/pom.xml
plugin/control/pom.xml
plugin/datasource/pom.xml
plugin/encryption/pom.xml
plugin/environment/pom.xml
plugin/pom.xml
plugin/trace/pom.xml
plugin/visibility/pom.xml
prometheus/pom.xml
server/pom.xml
sys/pom.xml
test/config-test/pom.xml
test/core-test/pom.xml
test/naming-test/pom.xml
test/openapi-test/pom.xml
test/pom.xml
```

- [ ] **步骤 3：convert reactor dependencies to project group**

For Nacos artifacts built by this reactor, change dependency groupIds from
literal `com.alibaba.nacos` to `${project.groupId}`. Use this form:

```xml
<dependency>
    <groupId>${project.groupId}</groupId>
    <artifactId>nacos-core</artifactId>
</dependency>
```

Do this for internal artifacts such as:

```text
nacos-address
nacos-ai
nacos-ai-registry-adaptor
nacos-api
nacos-auth
nacos-bootstrap
nacos-client
nacos-client-basic
nacos-cmdb
nacos-common
nacos-config
nacos-consistency
nacos-console
nacos-copilot
nacos-core
nacos-istio
nacos-lock
nacos-maintainer-client
nacos-naming
nacos-persistence
nacos-plugin
nacos-plugin-default-impl
nacos-prometheus
nacos-server
nacos-sys
nacos-control-plugin
nacos-config-plugin
nacos-datasource-plugin
nacos-datasource-plugin-base
nacos-datasource-plugin-derby
nacos-datasource-plugin-mysql
nacos-datasource-plugin-oracle
nacos-datasource-plugin-postgresql
nacos-encryption-plugin
nacos-trace-plugin
nacos-visibility-plugin
nacos-auth-plugin
nacos-ai-plugin
nacos-default-ai-importer-plugin
nacos-default-ai-pipeline-plugin
nacos-default-ai-trace-plugin
nacos-default-auth-plugin
nacos-default-control-plugin
nacos-default-plugin-all
nacos-ldap-auth-plugin
nacos-oidc-auth-plugin
```

Keep literal `com.alibaba.nacos` only if the artifact is intentionally consumed
from an external old coordinate and is not produced by the current reactor.

- [ ] **步骤 4：检查旧 groupId 残留**

运行：

```bash
rg -n "<groupId>com\\.alibaba\\.nacos</groupId>" -g 'pom.xml'
```

预期：只剩明确外部兼容坐标。对每个残留行判断是否应改为 `${project.groupId}`。

- [ ] **步骤 5：验证 Maven reactor 能解析新坐标**

运行：

```bash
mvn -pl core -am -DskipTests compile
```

预期：

```text
BUILD SUCCESS
```

如果失败信息包含 `Could not find artifact com.alibaba.nacos:*:3.2.2`，回到步骤 3，把对应 dependency groupId 改为 `${project.groupId}`。

## 任务 3：升级 Spring Boot 依赖管理和模块依赖

**文件：**
- 修改：`pom.xml`
- 修改：`ai-registry-adaptor/pom.xml`
- 修改：`config/pom.xml`
- 修改：`console/pom.xml`
- 修改：`core/pom.xml`
- 修改：`prometheus/pom.xml`
- 修改：`server/pom.xml`
- 修改：`test/pom.xml`
- 验证：`mvn -pl core,config,console,server,prometheus -am -DskipTests compile`

- [ ] **步骤 1：设置 Spring Boot 4.0.6**

在根 `pom.xml` 中改为：

```xml
<spring-boot-dependencies.version>4.0.6</spring-boot-dependencies.version>
```

不要改成 `4.1.0-RC1`。

- [ ] **步骤 2：保留或调整本地较新的依赖版本**

根 `pom.xml` 中保留当前 3.2.2 已经更高的版本，不按 PigMesh 参考降级。至少确认这些属性仍是当前值或更高：

```xml
<logback.version>1.5.32</logback.version>
<postgresql.version>42.7.11</postgresql.version>
<mcp.version>0.18.2</mcp.version>
<micrometer.version>1.15.11</micrometer.version>
```

如果 Boot 4.0.6 dependency management 和显式 `micrometer.version` 冲突，优先用编译和测试结果决定是否移除显式 override；不要直接套用 PigMesh 的 `1.13.0`。

- [ ] **步骤 3：替换 AOP starter**

在 `config/pom.xml` 和 `core/pom.xml` 中替换：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aspectj</artifactId>
</dependency>
```

删除对应旧依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

- [ ] **步骤 4：补充 LDAP Boot 模块**

在 `ai-registry-adaptor/pom.xml`、`console/pom.xml`、`server/pom.xml` 中添加：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-ldap</artifactId>
</dependency>
```

位置放在现有 Spring Boot / LDAP 相关依赖附近。

- [ ] **步骤 5：补充 web server 模块**

在 `core/pom.xml` 中添加：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-web-server</artifactId>
</dependency>
```

位置放在 `spring-boot-starter` 或 web 相关依赖附近。

- [ ] **步骤 6：补充 Web MVC test 模块**

在 `config/pom.xml`、`console/pom.xml`、`core/pom.xml`、`prometheus/pom.xml` 中添加测试依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-webmvc-test</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **步骤 7：补充 TestRestTemplate 模块**

在 `test/pom.xml` 中添加：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-resttestclient</artifactId>
    <scope>test</scope>
</dependency>
```

- [ ] **步骤 8：验证 Boot 依赖解析**

运行：

```bash
mvn -pl core,config,console,server,prometheus -am -DskipTests compile
```

预期：

```text
BUILD SUCCESS
```

如果失败信息提示缺少 `org.springframework.boot.*` 类，记录类名并在任务 4 或任务 6 中处理。

## 任务 4：更新 Spring Boot 4 moved packages

**文件：**
- 修改：`ai-registry-adaptor/src/main/java/com/alibaba/nacos/airegistry/NacosAiRegistry.java`
- 修改：`ai-registry-adaptor/src/main/java/com/alibaba/nacos/airegistry/config/HttpPathConfiguration.java`
- 修改：`console/src/main/java/com/alibaba/nacos/console/NacosConsole.java`
- 修改：`console/src/main/java/com/alibaba/nacos/console/config/ConsoleWebConfig.java`
- 修改：`core/src/main/java/com/alibaba/nacos/core/code/SpringApplicationRunListener.java`
- 修改：`core/src/main/java/com/alibaba/nacos/core/web/NacosWebServerListener.java`
- 修改：`server/src/main/java/com/alibaba/nacos/NacosServerBasicApplication.java`
- 修改：`server/src/main/java/com/alibaba/nacos/NacosServerWebApplication.java`
- 修改：`core/src/test/java/com/alibaba/nacos/core/web/NacosWebServerListenerTest.java`
- 验证：`rg -n "org\\.springframework\\.boot\\.(autoconfigure\\.ldap|autoconfigure\\.jackson|web\\.embedded\\.tomcat|web\\.context|ConfigurableBootstrapContext)" -g '*.java'`

- [ ] **步骤 1：更新 LDAP auto-configuration import**

在这些文件中：

```text
ai-registry-adaptor/src/main/java/com/alibaba/nacos/airegistry/NacosAiRegistry.java
console/src/main/java/com/alibaba/nacos/console/NacosConsole.java
server/src/main/java/com/alibaba/nacos/NacosServerBasicApplication.java
server/src/main/java/com/alibaba/nacos/NacosServerWebApplication.java
```

替换 import：

```java
import org.springframework.boot.ldap.autoconfigure.LdapAutoConfiguration;
```

删除旧 import：

```java
import org.springframework.boot.autoconfigure.ldap.LdapAutoConfiguration;
```

- [ ] **步骤 2：更新 Tomcat connector customizer import**

在 `ai-registry-adaptor/src/main/java/com/alibaba/nacos/airegistry/config/HttpPathConfiguration.java` 中替换为：

```java
import org.springframework.boot.tomcat.TomcatConnectorCustomizer;
```

删除旧 import：

```java
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
```

- [ ] **步骤 3：更新 Jackson customizer bean**

在 `console/src/main/java/com/alibaba/nacos/console/config/ConsoleWebConfig.java` 中改 import：

```java
import org.springframework.boot.jackson.autoconfigure.JsonMapperBuilderCustomizer;
```

并改 bean：

```java
@Bean
public JsonMapperBuilderCustomizer jacksonJsonMapperCustomization() {
    return jsonMapperBuilder -> jsonMapperBuilder.defaultTimeZone(TimeZone.getDefault());
}
```

确保导入：

```java
import java.util.TimeZone;
```

删除旧 import：

```java
import org.springframework.boot.autoconfigure.jackson.Jackson2ObjectMapperBuilderCustomizer;
import java.time.ZoneId;
```

- [ ] **步骤 4：更新 bootstrap context import**

在 `core/src/main/java/com/alibaba/nacos/core/code/SpringApplicationRunListener.java` 中替换为：

```java
import org.springframework.boot.bootstrap.ConfigurableBootstrapContext;
```

删除旧 import：

```java
import org.springframework.boot.ConfigurableBootstrapContext;
```

- [ ] **步骤 5：更新 web server listener imports**

在 `core/src/main/java/com/alibaba/nacos/core/web/NacosWebServerListener.java` 中替换为：

```java
import org.springframework.boot.web.server.context.WebServerInitializedEvent;
```

在 `core/src/test/java/com/alibaba/nacos/core/web/NacosWebServerListenerTest.java` 中替换为：

```java
import org.springframework.boot.web.server.WebServer;
import org.springframework.boot.web.server.context.WebServerApplicationContext;
import org.springframework.boot.web.server.context.WebServerInitializedEvent;
```

并把字段改为：

```java
@Mock
private WebServer webServer;
```

- [ ] **步骤 6：扫描旧 Boot 包残留**

运行：

```bash
rg -n "org\\.springframework\\.boot\\.(autoconfigure\\.ldap|autoconfigure\\.jackson|web\\.embedded\\.tomcat|web\\.context|ConfigurableBootstrapContext)" -g '*.java'
```

预期：无输出。

## 任务 5：迁移 javax.annotation 到 jakarta.annotation

**文件：**
- 修改：包含 `javax.annotation` 的非生成 Java 文件
- 不修改：`api/src/main/java/com/alibaba/nacos/api/grpc/auto/**`
- 验证：`rg -n "javax\\.annotation" -g '*.java'`

- [ ] **步骤 1：迁移普通源码 import**

对所有非生成 Java 文件执行等价替换：

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import jakarta.annotation.Resource;
```

删除对应旧 import：

```java
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import javax.annotation.Resource;
```

当前已知要覆盖的目录包括：

```text
ai/src/main/java
cmdb/src/main/java
config/src/main/java
console/src/main/java
copilot/src/main/java
core/src/main/java
istio/src/main/java
naming/src/main/java
persistence/src/main/java
plugin-default-impl/nacos-default-auth-plugin/src/main/java
test/core-test/src/test/java
```

- [ ] **步骤 2：保留 generated grpc 代码**

如果扫描结果只剩这些文件，不修改：

```text
api/src/main/java/com/alibaba/nacos/api/grpc/auto/BiRequestStreamGrpc.java
api/src/main/java/com/alibaba/nacos/api/grpc/auto/RequestGrpc.java
```

这些是生成代码路径，保持原样，避免人工编辑生成物。

- [ ] **步骤 3：验证 javax.annotation 残留**

运行：

```bash
rg -n "javax\\.annotation" -g '*.java'
```

预期输出最多只包含：

```text
api/src/main/java/com/alibaba/nacos/api/grpc/auto/BiRequestStreamGrpc.java
api/src/main/java/com/alibaba/nacos/api/grpc/auto/RequestGrpc.java
```

## 任务 6：更新 Spring Boot 4 测试 API 和 Tomcat 测试适配

**文件：**
- 修改：`test/config-test/src/test/java/com/alibaba/nacos/test/base/HttpClient4Test.java`
- 修改：`test/core-test/src/test/java/com/alibaba/nacos/test/base/HttpClient4Test.java`
- 修改：`test/naming-test/src/test/java/com/alibaba/nacos/test/base/HttpClient4Test.java`
- 修改：`test/config-test/src/test/java/com/alibaba/nacos/test/config/ConfigBetaConfigITCase.java`
- 修改：`test/naming-test/src/test/java/com/alibaba/nacos/test/naming/MultiTenantInstanceAPINamingITCase.java`
- 修改：`test/core-test/src/test/java/com/alibaba/nacos/test/core/auth/LdapAuthCoreITCase.java`
- 修改：`test/core-test/src/test/java/com/alibaba/nacos/test/core/code/ControllerMethodsCacheCoreITCase.java`
- 验证：`mvn -pl test -am -DskipTests test-compile`

- [ ] **步骤 1：更新 TestRestTemplate imports**

在三个 `HttpClient4Test.java` 和两个直接注入 `TestRestTemplate` 的 IT 文件中使用：

```java
import org.springframework.boot.resttestclient.TestRestTemplate;
import org.springframework.boot.resttestclient.autoconfigure.AutoConfigureTestRestTemplate;
import org.springframework.boot.test.context.SpringBootTest;
```

删除旧 import：

```java
import org.springframework.boot.test.web.client.TestRestTemplate;
```

- [ ] **步骤 2：给通用 HTTP test base 加测试配置注解**

在三个 `HttpClient4Test.java` 类声明前添加：

```java
@SpringBootTest
@AutoConfigureTestRestTemplate
public class HttpClient4Test {
```

保留原有字段：

```java
@Autowired
protected TestRestTemplate restTemplate;
```

- [ ] **步骤 3：给直接使用 TestRestTemplate 的 IT 类加配置注解**

在 `ConfigBetaConfigITCase` 的类注解区域加入：

```java
@AutoConfigureTestRestTemplate
@SuppressWarnings("checkstyle:AbbreviationAsWordInName")
@ExtendWith(SpringExtension.class)
@SpringBootTest(classes = Nacos.class, properties = {
        "server.servlet.context-path=/nacos"}, webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class ConfigBetaConfigITCase {
```

在 `MultiTenantInstanceAPINamingITCase` 的类注解区域加入：

```java
@AutoConfigureTestRestTemplate
@SpringBootTest(classes = Nacos.class, properties = {
        "server.servlet.context-path=/nacos"}, webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
class MultiTenantInstanceAPINamingITCase {
```

- [ ] **步骤 4：替换 UriComponentsBuilder.fromHttpUrl**

在上述 HTTP test helper 和 IT 文件中替换为：

```java
UriComponentsBuilder builder = UriComponentsBuilder.fromUriString(this.base.toString() + path)
        .queryParams(params);
```

在 `ConfigBetaConfigITCase` 中使用 `this.url` 的两个 request 方法替换为：

```java
UriComponentsBuilder builder = UriComponentsBuilder.fromUriString(this.url.toString() + path).queryParams(params);
```

- [ ] **步骤 5：更新 MockitoBean**

在 `test/core-test/src/test/java/com/alibaba/nacos/test/core/auth/LdapAuthCoreITCase.java` 中改 import：

```java
import org.springframework.test.context.bean.override.mockito.MockitoBean;
```

并改字段注解：

```java
@MockitoBean
private LdapTemplate ldapTemplate;
```

删除旧 import 和注解：

```java
import org.springframework.boot.test.mock.mockito.MockBean;
@MockBean
```

- [ ] **步骤 6：更新 Tomcat Request 构造方式**

在 `test/core-test/src/test/java/com/alibaba/nacos/test/core/code/ControllerMethodsCacheCoreITCase.java` 中把 request 构造改为：

```java
private Request buildRequest(String method, String path, Map<String, String> parameters) {
    Connector connector = new Connector();
    connector.setParseBodyMethods("GET,POST,PUT,DELETE,PATCH");
    org.apache.coyote.Request coyoteRequest = new org.apache.coyote.Request();
    Request request = new Request(connector, coyoteRequest);
    MessageBytes messageBytes = coyoteRequest.requestURI();
    messageBytes.setString(path);
    coyoteRequest.setMethod(method);
    if (parameters != null) {
        for (Map.Entry<String, String> entry : parameters.entrySet()) {
            coyoteRequest.getParameters().addParameter(entry.getKey(), entry.getValue());
        }
    }
    return request;
}
```

- [ ] **步骤 7：验证测试编译**

运行：

```bash
mvn -pl test -am -DskipTests test-compile
```

预期：

```text
BUILD SUCCESS
```

## 任务 7：更新 Spring Security BCrypt override

**文件：**
- 修改：`plugin-default-impl/nacos-default-auth-plugin/src/main/java/com/alibaba/nacos/plugin/auth/impl/SafeBcryptPasswordEncoder.java`
- 验证：`mvn -pl plugin-default-impl/nacos-default-auth-plugin -am -DskipTests compile`

- [ ] **步骤 1：更新 override 方法签名**

将文件改为使用 Spring Security 7 的 non-null override：

```java
import org.jspecify.annotations.NonNull;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
```

方法改为：

```java
@Override
public boolean matchesNonNull(@NonNull String rawPassword, @NonNull String encodedPassword) {
    // Reject excessively long passwords immediately
    if (rawPassword.length() > AuthConstants.MAX_PASSWORD_LENGTH) {
        return false;
    }
    return super.matchesNonNull(rawPassword, encodedPassword);
}
```

删除旧方法：

```java
@Override
public boolean matches(CharSequence rawPassword, String encodedPassword) {
    // Reject excessively long passwords immediately
    if (rawPassword != null && rawPassword.length() > AuthConstants.MAX_PASSWORD_LENGTH) {
        return false;
    }
    return super.matches(rawPassword, encodedPassword);
}
```

- [ ] **步骤 2：验证 auth plugin 编译**

运行：

```bash
mvn -pl plugin-default-impl/nacos-default-auth-plugin -am -DskipTests compile
```

预期：

```text
BUILD SUCCESS
```

## 任务 8：更新 native reflection metadata

**文件：**
- 修改：`console/src/main/resources/META-INF/native-image/com.alibaba.nacos/nacos-console/reflect-config.json`
- 验证：`rg -n "org\\.springframework\\.boot\\.(autoconfigure\\.ldap|autoconfigure\\.jackson|web\\.context)" console/src/main/resources/META-INF/native-image`

- [ ] **步骤 1：替换 moved class names**

在 reflect config 中同步 Boot 4 包名：

```json
"name": "org.springframework.boot.ldap.autoconfigure.LdapAutoConfiguration"
```

如存在旧 Jackson customizer 或 web context 包名，也替换为源码中的 Boot 4 包名。

- [ ] **步骤 2：验证 native metadata 没有旧包**

运行：

```bash
rg -n "org\\.springframework\\.boot\\.(autoconfigure\\.ldap|autoconfigure\\.jackson|web\\.context)" console/src/main/resources/META-INF/native-image
```

预期：无输出。

## 任务 9：Spotless 和分层编译验证

**文件：**
- 验证：Maven affected modules

- [ ] **步骤 1：运行 Spotless apply**

运行：

```bash
mvn spotless:apply
```

预期：

```text
BUILD SUCCESS
```

- [ ] **步骤 2：运行 Spotless check**

运行：

```bash
mvn spotless:check
```

预期：

```text
BUILD SUCCESS
```

- [ ] **步骤 3：运行核心编译验证**

运行：

```bash
mvn -pl core,config,console,server,prometheus,plugin-default-impl/nacos-default-auth-plugin -am -DskipTests compile
```

预期：

```text
BUILD SUCCESS
```

- [ ] **步骤 4：运行测试编译验证**

运行：

```bash
mvn -pl test -am -DskipTests test-compile
```

预期：

```text
BUILD SUCCESS
```

- [ ] **步骤 5：运行最终静态检查命令**

运行：

```bash
mvn -B clean compile apache-rat:check checkstyle:check spotbugs:check spotless:check -DskipTests
```

预期：

```text
BUILD SUCCESS
```

如果耗时或环境依赖导致命令无法完成，记录最后一个失败模块、错误摘要和已经通过的较小验证命令。

## 任务 10：最终差异审查

**文件：**
- 验证：`git diff --stat`
- 验证：`git diff -- pom.xml`
- 验证：`rg` scans

- [ ] **步骤 1：确认没有版本回退**

运行：

```bash
rg -n "<revision>3\\.2\\.2</revision>|<spring-boot-dependencies.version>4\\.0\\.6</spring-boot-dependencies.version>|<groupId>io\\.github\\.pig-mesh\\.nacos</groupId>" pom.xml
```

预期包含三行：revision、Spring Boot version、root groupId。

- [ ] **步骤 2：确认没有引入 PigMesh 无关功能**

运行：

```bash
git diff --stat
```

预期差异集中在：

```text
pom.xml
*/pom.xml
Spring Boot moved package imports
Jakarta annotation imports
Boot 4 test API compatibility
native reflection metadata
docs/superpowers/specs/2026-05-29-spring-boot-4-pigmesh-group-design.md
docs/superpowers/plans/2026-05-29-spring-boot-4-pigmesh-group.md
```

如果出现 console promotional navigation、AI registry disable switch 或其他功能文件，检查是否误引入并移除本任务无关改动。

- [ ] **步骤 3：确认旧 Boot 包名残留**

运行：

```bash
rg -n "org\\.springframework\\.boot\\.(autoconfigure\\.ldap|autoconfigure\\.jackson|web\\.embedded\\.tomcat|web\\.context|test\\.mock\\.mockito|test\\.web\\.client|ConfigurableBootstrapContext)" -g '*.java'
```

预期：无输出。

- [ ] **步骤 4：确认 javax.annotation 残留只有生成代码**

运行：

```bash
rg -n "javax\\.annotation" -g '*.java'
```

预期输出最多只包含：

```text
api/src/main/java/com/alibaba/nacos/api/grpc/auto/BiRequestStreamGrpc.java
api/src/main/java/com/alibaba/nacos/api/grpc/auto/RequestGrpc.java
```

- [ ] **步骤 5：总结验证证据**

在最终交付中列出：

```text
- Spring Boot target: 4.0.6
- Nacos revision: 3.2.2
- Maven groupId: io.github.pig-mesh.nacos
- Passed validation commands: mvn spotless:check; mvn -pl core,config,console,server,prometheus,plugin-default-impl/nacos-default-auth-plugin -am -DskipTests compile; mvn -pl test -am -DskipTests test-compile
- Not run / failed validation commands: mvn -B clean compile apache-rat:check checkstyle:check spotbugs:check spotless:check -DskipTests, with the exact failure reason if it does not pass
```
