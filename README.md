# 每日成长评估系统 - Java 开发指南



本指南将带领你使用 **Java (Spring Boot)** 和 **Vue.js** 从零开始构建一个“每日成长评估系统”。我们将项目拆解成五个核心步骤，确保你能够清晰地理解每个环节的作用和实现方式。



## 步骤 1: 环境准备与项目搭建



### **目的**



为项目创建一个稳定、可运行的基础框架，并配置好必要的工具。



### **操作**



1.  **安装必备工具**：

* **JDK (Java Development Kit) 17+**：Java 程序的运行环境。从官网下载并安装。

 * **IntelliJ IDEA Community Edition**：一款功能强大的 Java 集成开发环境（IDE）。

 * **MySQL 数据库**：项目的后端数据库。

* **Maven**：Java 项目构建工具，通常已集成在 IntelliJ IDEA 中。

* **Postman/cURL**：用于测试后端 API 接口的工具。



2.  **创建 Spring Boot 项目**：

* 打开 **IntelliJ IDEA**，选择 **`New Project`**。

* 在左侧导航栏选择 **`Spring Initializr`**。

* **项目元数据**：

    * `Name`：`daily-assessment-system`

    * `Language`：`Java`

    * `JDK`：选择你安装的 JDK 17+。

    * `Packaging`：`Jar`

* **添加依赖**：

    * 搜索并勾选 `Spring Web`。

    * 搜索并勾选 `Spring Data JPA`。

    * 搜索并勾选 `MySQL Driver`。

    * 搜索并勾选 `Lombok`。

    * 搜索并勾选 `Spring WebFlux` (用于调用外部 API)。

* 点击 **`Create`**，等待项目自动生成。



### **预期效果**

你将得到一个完整的、可直接运行的 Spring Boot 项目骨架，项目结构清晰，所有必要的库文件都已配置好。



-----



## 步骤 2: 数据库配置与数据模型



### **目的**

让你的程序能够连接到数据库，并定义数据在数据库中的存储形式。



### **操作**

1.  **创建数据库**：

* 打开 MySQL 客户端，执行以下 SQL 命令：

  ```sql

  CREATE DATABASE `daily_assessment_system`;

  ```

2.  **配置数据库连接**：

* 打开 `src/main/resources/application.properties` 文件。

* 添加以下配置，将 `your_password` 替换为你的 MySQL 密码：

```properties
# 数据库连接URL
spring.datasource.url=jdbc:mysql://localhost:3306/daily_assessment_system?useSSL=false&serverTimezone=UTC
# 用户名
spring.datasource.username=root
# 密码
spring.datasource.password=your_password
# 启动时自动创建或更新表结构
spring.jpa.hibernate.ddl-auto=update
# 在控制台显示SQL语句
spring.jpa.show-sql=true
```



3.  **创建数据实体类（Entity）**：



* 在 `src/main/java/com/daily/assessment/system` 目录下，新建一个 `entity` 包。

* 在 `entity` 包下，创建 `Record.java` 文件，用于映射数据库中的 `records` 表。

* 代码如下：
```java
package com.daily.assessment.system.entity;

import lombok.Data;
import javax.persistence.*;
import java.util.Date;

@Entity
@Table(name = "records")
@Data
public class Record {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Long userId;
    @Temporal(TemporalType.DATE)
    private Date recordDate;
    @Lob
    private String content;
    private Integer overallScore;
    private Integer knowledgeScore;
    private Integer actionScore;
    private Integer mindsetScore;
    private Integer healthScore;
    private Integer socialScore;
    private String summary;
    @Temporal(TemporalType.TIMESTAMP)
    private Date createdAt;
}
```

### **预期效果**

当你启动 Spring Boot 应用时，`daily_assessment_system` 数据库中会自动创建一张名为 `records` 的表，其字段与 `Record.java` 中的属性完全对应。



-----



## 步骤 3: 数据访问与业务逻辑



### **目的**

创建与数据库交互的“工具箱”（Repository）和处理核心业务逻辑的“大厨”（Service）。



### **操作**

1.  **创建数据访问层（Repository）**：

* 在 `src/main/java/com/daily/assessment/system` 目录下，新建一个 `repository` 包。

* 创建 `RecordRepository.java` 接口，它继承自 `JpaRepository`。

* 代码如下：

```java
package com.daily.assessment.system.repository;

import com.daily.assessment.system.entity.Record;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.Date;
import java.util.List;

@Repository
public interface RecordRepository extends JpaRepository<Record, Long> {
    List<Record> findByUserIdAndRecordDateBetween(Long userId, Date startDate, Date endDate);
    List<Record> findByUserId(Long userId);
}

```



2.  **创建 Gemini API 服务**：

* 在 `src/main/java/com/daily/assessment/system` 目录下，新建一个 `service` 包。

* 创建 `GeminiService.java`，用于封装调用 Gemini API 的逻辑。

* 你需要使用 `WebClient` 来发送 HTTP 请求。在 `pom.xml` 中确保有 `spring-boot-starter-webflux` 依赖。

* **核心逻辑**：构造一个包含你的 Prompt 和用户文本的 JSON 请求体，发送给 Gemini API，并解析返回的 JSON 以获取评分和总结。

3.  **创建业务逻辑服务**：
* 在 `service` 包下，创建 `RecordService.java`。
* 使用 `@Autowired` 注解注入 `RecordRepository` 和 `GeminiService`。
* 编写以下核心方法：
    * `submitRecord(Long userId, String content)`：调用 `GeminiService` 获取评分，封装成 `Record` 对象，再调用 `recordRepository.save()` 存入数据库。
    * `getHistory(Long userId, Date startDate, Date endDate)`：调用 `recordRepository.findByUserIdAndRecordDateBetween()` 查询历史记录。
    * `calculateRewardStatus(Long userId)`：查询所有记录，计算总分和奖励状态。

### **预期效果**

你现在拥有了处理所有后端核心逻辑的模块。从接收前端数据，到调用 AI 评估，再到保存和查询数据，所有业务流程都已封装完毕。

-----

## 步骤 4: API 接口与前端对接



### **目的**

创建前端可以调用的 RESTful API 接口，实现前后端数据通信。

### **操作**

1.  **创建 API 控制器**：

* 在 `src/main/java/com/daily/assessment/system` 目录下，新建一个 `controller` 包。
* 创建 `RecordController.java`，使用 `@RestController` 注解定义 API。
* 代码如下：

```java

package com.daily.assessment.system.controller;

import com.daily.assessment.system.dto.RecordDto;
import com.daily.assessment.system.entity.Record;
import com.daily.assessment.system.service.RecordService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/records")
public class RecordController {
    
    @Autowired
    private RecordService recordService;

    @PostMapping("/submit")
    public Record submitRecord(@RequestBody RecordDto recordDto) {
        return recordService.submitRecord(recordDto.getUserId(), recordDto.getContent());
    }

    @GetMapping("/history")
    public List<Record> getHistory(@RequestParam Long userId, @RequestParam String startDate, @RequestParam String endDate) {
        // 这里需要将字符串日期转换为 Date 对象
        // ...
        return recordService.getHistory(userId, startDate, endDate);
    }
}
```

2.  **前端页面开发**：

* 使用 **Vue.js 3** 和 **Element Plus** 搭建前端项目。
* 使用 **ECharts** 实现数据可视化图表。
* 在 Vue 组件中，编写 `axios` 或 `fetch` 请求，将数据发送到 `http://localhost:8080` 上的后端接口。



### **预期效果**

前端能够通过 API 接口与后端进行数据交互，用户提交的记录能够被后端处理并保存，历史记录和评分也能被前端获取并展示在图表上。

-----

## 步骤 5: 运行、测试与部署

### **目的**

验证项目功能，并准备上线。

### **操作**

1.  **启动后端**：

* 在 IntelliJ IDEA 中，点击项目主类旁的绿色“运行”按钮。

* 项目将在 `http://localhost:8080` 上启动。

2.  **启动前端**：

* 在前端项目目录下，运行 `npm run dev`。

3.  **功能测试**：

* 在浏览器中打开前端页面，提交记录，检查数据库中是否新增了数据。

* 通过前端界面查看历史记录和图表，验证数据是否正确显示。

* 使用 Postman/cURL 直接调用后端接口，确保每个接口都能正常工作。

### **预期效果**

项目前后端联调成功，所有功能均可正常使用。你现在拥有了一个完整的“每日成长评估系统”Web 应用。