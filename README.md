# Agent项目学习万字总结

## 一、3月9日（第一天：代码pom文件入手）

### 1.spring-boot-configuration-processor依赖

**作用：**生成 `spring-configuration-metadata.json`
		IDEA 自动提示配置；
		`@ConfigurationProperties` 自动补全；

### 2.spring-boot-devtools依赖

**作用：**
		开发工具，提供：
		热部署、自动重启、LiveReload
		适合开发阶段。

### 3.向量数据库milvus-sdk-java依赖

**作用：**连接向量数据库
		用于**RAG**检索增强生成

流程：

```
文本 -> 向量化 -> 存入Milvus
用户问题 -> 向量 -> 相似搜索 -> 返回知识
```

典型代码：

```java
MilvusClient client = new MilvusServiceClient();
```

### 4.Gson依赖

为什么项目中**同时有 Jackson 和 Gson**？
		答：两者分工

- **Jackson**：Spring Boot 默认，用于 REST API、配置文件等
- **Gson**：专门用于处理 **DashScope SDK** 的 JSON

因为 DashScope SDK 内部使用 Gson，所以需要引入。

在你的 AI Agent 项目中，Gson 主要用于：

#### 1). **处理 AI 模型的请求/响应**

```java
// DashScope API 的请求体
{
  "model": "qwen-max",
  "input": {
    "messages": [
      {"role": "user", "content": "你好"}
    ]
  },
  "parameters": {
    "temperature": 0.8
  }
}

// Gson 将 Java 对象转成上面的 JSON
ChatRequest request = new ChatRequest();
request.setModel("qwen-max");
// ... 设置其他参数

String json = gson.toJson(request);  // 转成JSON发送给AI
```

#### 2). **解析 AI 的响应**

```java
// AI 返回的 JSON
{
  "output": {
    "choices": [
      {
        "message": {
          "content": "你好！有什么可以帮你的？"
        }
      }
    ]
  }
}

// Gson 解析成 Java 对象
String responseJson = callAI();
ChatResponse response = gson.fromJson(responseJson, ChatResponse.class);
String answer = response.getOutput().getChoices().get(0).getMessage().getContent();
```

#### 3). **处理工具调用的参数**

```java
// AI 要调用工具时，参数是 JSON 格式
{
  "function": "searchKnowledge",
  "arguments": {
    "query": "Spring Boot 教程",
    "limit": 5
  }
}

// Gson 解析工具参数
JsonObject args = gson.fromJson(argumentsJson, JsonObject.class);
String query = args.get("query").getAsString();
int limit = args.get("limit").getAsInt();
```

### 5.dashscope-sdk-java依赖

官方SDK，用于调用大模型，向量化，图片生成

项目中已经有了 `spring-ai-alibaba-starter-dashscope`，为什么还要单独引入这个？

**二者关系：**

- **spring-ai-alibaba-starter-dashscope**：Spring AI 的封装，更高层

- **dashscope-sdk-java**：阿里云官方底层 SDK，更底层

  ### 使用场景对比

  | 场景           | 使用哪个               | 原因                         |
  | :------------- | :--------------------- | :--------------------------- |
  | 简单的对话     | Spring AI Starter      | 更简单，符合Spring习惯       |
  | **文本向量化** | **dashscope-sdk-java** | Spring AI 可能不支持某些特性 |
  | 流式输出       | Spring AI Starter      | 支持 Reactive                |
  | **多模态识别** | **dashscope-sdk-java** | 最新功能支持                 |
  | **批量处理**   | **dashscope-sdk-java** | 更灵活的API                  |

### 6.Lombok

减少样板代码。

### 7.Json Schema

jsonschema-generator
	   jsonschema-module-jackson

**作用：**这两个依赖用于**从 Java 类自动生成 JSON Schema**，在 AI Agent 项目中主要用于**定义工具（Function Calling）的参数结构**。

Json Schema: **JSON Schema** 是描述 JSON 数据结构的"元数据"，就像 Java 类描述对象一样。

### 例如，这个 Java 类：

```java
public class SearchRequest {
    private String query;        // 搜索关键词
    private int limit;           // 返回数量
    private List<String> tags;   // 标签过滤
}
```



### 生成的 JSON Schema：

```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "搜索关键词"
    },
    "limit": {
      "type": "integer",
      "description": "返回数量",
      "minimum": 1,
      "maximum": 20
    },
    "tags": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "description": "标签过滤"
    }
  },
  "required": ["query"]
}
```

转换成Json格式后，即可将可调用的工具（上述Json）提供给AI，AI决定调用哪个工具。

**两个依赖分工：**
		**jsonschema-generator**

- 提供核心的 Schema 生成功能
- 扫描 Java 类的字段、方法
- 生成基本的 JSON Schema 结构

 **jsonschema-module-jackson（增强）**

- 让生成器理解 Jackson 的注解（@JsonProperty, @JsonIgnore 等）
- 支持 @JsonFormat 等格式化注解
- 与 Spring Boot 默认的 JSON 处理保持一致

### 8.spring-ai-bom

```xml
<!-- 在 properties 中定义一次版本 -->
<properties>
    <spring-ai.version>1.1.0</spring-ai.version>
</properties>

<!-- 在 dependencyManagement 中引入 BOM -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 实际使用时不需要指定版本 -->
<dependencies>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai</artifactId>
        <!-- 版本由 BOM 统一管理，这里不用写 -->
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-ollama</artifactId>
        <!-- 版本由 BOM 统一管理，这里不用写 -->
    </dependency>
</dependencies>
```

### 9.spring-ai-alibaba-extensions-bom

是阿里扩展的Spring AI  BOM，提供阿里特有的增强功能。

该BOM包含：

```xml
<!-- 这个 BOM 管理的典型依赖 -->
<dependencyManagement>
    <dependencies>
        <!-- 阿里云对象存储 OSS -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-oss</artifactId>
            <version>1.1.0.0-RC2</version>
        </dependency>
        
        <!-- 阿里云表格存储 TableStore -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-tablestore</artifactId>
            <version>1.1.0.0-RC2</version>
        </dependency>
        
        <!-- 阿里云日志服务 SLS -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-sls</artifactId>
            <version>1.1.0.0-RC2</version>
        </dependency>
        
        <!-- 阿里云消息队列 RocketMQ -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-rocketmq</artifactId>
            <version>1.1.0.0-RC2</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 10. spring-ai-alibaba-bom

这是**阿里云官方适配 BOM**，专门用于将 **Spring AI 接入阿里云通义系列模型**。

BOM包含的核心内容：

```xml
<!-- 这个 BOM 管理的核心依赖 -->
<dependencyManagement>
    <dependencies>
        <!-- DashScope 聊天模型 -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-dashscope</artifactId>
            <version>1.1.0.0-RC2</version>
        </dependency>
        
        <!-- DashScope 嵌入模型（向量化） -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-dashscope-embedding</artifactId>
            <version>1.1.0.0-RC2</version>
        </dependency>
        
        <!-- DashScope 图像模型 -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-dashscope-image</artifactId>
            <version>1.1.0.0-RC2</version>
        </dependency>
        
        <!-- DashScope 音频模型 -->
        <dependency>
            <groupId>com.alibaba.cloud.ai</groupId>
            <artifactId>spring-ai-alibaba-dashscope-audio</artifactId>
            <version>1.1.0.0-RC2</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

#### 10.1 **spring-ai-alibaba-starter-dashscope**

这是**阿里云 DashScope 的 Spring Boot Starter**，使能在 Spring Boot 中**一键集成通义千问模型**。

#### 10.2 **spring-ai-alibaba-agent-framework**

这是**阿里提供的 Agent 框架**，用于构建智能代理应用，让你的 AI 能**使用工具、规划任务、记忆对话**。

两者结合，才能构建完整的 AI Agent：

- 用通义千问**思考理解**
- 用 Agent 框架**行动执行**

## 知识库Agent

## 二、3月12日 （第二天：indexSingleFile() 读取文件功能）

### 2.1  Paths.get() 方法

简单来说就是，因为Java是跨平台的，所以在括号内输入实参（也就是具体路径）后，首先会判断是什么操作系统，然后返回该操作系统的文件系统，再将实参（路径）传给返回的文件系统对象，最后进行解析，**将字符串路径转换为当前操作系统可理解的 Path 对象**。

Paths.get(filePath)内部做的：

``` java
public static Path get(String first, String... more) {
        return Path.of(first, more);//windows返回的是 WindowsPath 对象，linux返回的是。。。对象。
    }

public static Path of(String first, String... more) {
        return FileSystems.getDefault().getPath(first, more);
    }

public static FileSystem getDefault() {
        if (VM.isModuleSystemInited()) {
            return DefaultFileSystemHolder.defaultFileSystem;
        } else {
            // always use the platform's default file system during startup
            return DefaultFileSystemProvider.theFileSystem();
        }
    }

// lazy initialization of default file system
    private static class DefaultFileSystemHolder {
        static final FileSystem defaultFileSystem = defaultFileSystem();

        // returns default file system
        private static FileSystem defaultFileSystem() {
            // load default provider
            @SuppressWarnings("removal")
            FileSystemProvider provider = AccessController
                .doPrivileged(new PrivilegedAction<>() {
                    public FileSystemProvider run() {
                        return getDefaultProvider();
                    }
                });

            // return file system
            return provider.getFileSystem(URI.create("file:///"));
        }
    }
```

然后一系列工作做完后，将路径规范化[ **normalize()**方法 ] ，处理路径中的 `.` 和 `..`，**解析掉路径中的冗余部分**，得到最简洁的路径形式。

``` java
Path path = Path.of("/home/user/docs/../file.txt");
// 规范化前: /home/user/docs/../file.txt
// 规范化后: /home/user/file.txt
// 解释: docs/.. 相互抵消，回到 user 目录

Path path = Path.of("/home/user/./docs/file.txt");
// 规范化前: /home/user/./docs/file.txt
// 规范化后: /home/user/docs/file.txt
// 解释: 把 "./" 直接去掉
```

### 2.2 path.toFile()方法

将 **NIO 的 `Path` 对象** 转换为 **传统的 `File` 对象** 的方法。

``` java
public void indexSingleFile(String filePath) {
    Path path = Paths.get(filePath).normalize();
    
    // 为什么要转成 File？
    File file = path.toFile();
    
    // 因为 File 类有一些 Path 没有的便捷方法
    if (!file.exists() || !file.isFile()) {
        // exists() 和 isFile() 是 File 类的方法
        // Path 也有这些方法（通过 Files 类），但 File 更直接
        throw new IllegalArgumentException("文件不存在");
    }
    
    // 其他 File 特有的方法：
    file.length();              // 文件大小
    file.lastModified();        // 最后修改时间
    file.canRead();             // 是否可读
    file.canWrite();            // 是否可写
    file.isHidden();            // 是否隐藏
    file.setReadOnly();         // 设置只读
    file.deleteOnExit();        // 程序退出时删除
}
```

### 2.3  deleteExistingData(filePath);

该方法是要删除该文件的旧数据（如果存在）。

- `0`：加载成功
- `65535`：集合已经加载过（也是成功状态）
- 其他状态码：加载失败，抛出异常

#### 2.3.1 replace(File.separator, "/")方法

这是 **String 类的 `replace()` 方法**，用于**将路径中的系统分隔符统一替换为正斜杠**。

```Java
File.separator   // 获取系统路径分隔符
    
// Windows 系统
System.out.println(File.separator);  // 输出: "\"

// Linux/Mac 系统  
System.out.println(File.separator);  // 输出: "/"
```

 **`replace(旧字符, 新字符)`**

因为Milvus 中统一用正斜杠存储，避免表达式解析错误，所以需要统一替换成正斜杠。

#### 2.3.2 构建删除表达式

```Java
String expr = String.format("metadata[\"_source\"] == \"%s\"", normalizedPath);

// 假设 normalizedPath = "C:/Users/docs/file.txt"
expr = "metadata[\"_source\"] == \"C:/Users/docs/file.txt\""

// 这个表达式告诉 Milvus：
// 删除所有 metadata 字段中，_source 属性等于指定路径的记录
```

为什么用 **metadata["_source"]** 这种格式？
		答：因为 Milvus 的数据结构是这样设计的：

```json
// Milvus 中的一条记录
{
    "id": 1001,                          // 主键（自动生成）
    "content": "Spring Boot 简介...",     // 文本内容
    "vector": [0.123, -0.456, ...],       // 向量（用于搜索）
    "metadata": {                          // 元数据字段（JSON格式）
        "_source": "C:/Users/docs/file.md",  // 文件来源
        "chunk_index": 0,                     // 分片序号
        "file_size": 1024,                     // 文件大小
        "index_time": "2024-01-15"             // 索引时间
    }
}
```

在 Milvus 的查询语法中，访问 JSON 字段要用 [ ] ;
		Java 字符串中的 \\" 表示一个真正的双引号字符。

```Java
String part1 = "metadata[\"_source\"] == \"";  
// 实际内容: metadata["_source"] == "
```

#### 2.3.3 确保 collection 已加载(删除操作需要集合已加载)

Milvus 中的集合有两种状态：未加载，**数据在磁盘上，不能查询**；已加载，**数据在内存中，可以查询和删除。**
	   **删除操作前必须保证集合已加载到内存中**。

**源代码：**

``` java
// 确保 collection 已加载（删除操作需要集合已加载）
R<RpcStatus> loadResponse = milvusClient.loadCollection(
                LoadCollectionParam.newBuilder()
                    .withCollectionName(MilvusConstants.MILVUS_COLLECTION_NAME)
                    .build()
            );
```

- `milvusClient.loadCollection()`: 调用 Milvus 的加载接口

``` java
public R<RpcStatus> loadCollection(LoadCollectionParam requestParam) {
        return this.retry(() -> {
            return super.loadCollection(requestParam);
        });
    }

```

- `LoadCollectionParam.newBuilder()`: 创建参数构建器，构建loadCollection函数的实参

- `.withCollectionName(...)`: 指定要加载的集合名称

``` java 
LoadCollectionParam.newBuilder()
                    .withCollectionName(MilvusConstants.MILVUS_COLLECTION_NAME) //在 Milvus 数据库中查找名为 "biz" 的集合，并将其加载到内存中。这里的biz是用常量表示的，让代码更规范、更易维护。
                    .build()
```

- `.build()`: 构建参数对象
- `R`<RpcStatus>: 统一的响应包装类，包含状态码和消息

#### 2.3.4 执行删除操作（DeleteParam）

**所以必须先加载集合到内存，才能根据条件找到并删除数据**。

```Java
// Milvus 中的数据存储：
磁盘（永久存储）         内存（工作区）
───────────────────────────────
biz集合的所有数据  →  加载后 →  可操作的数据
[数据文件]               [内存副本]

// 未加载时：数据在磁盘文件中，无法直接查询或删除
// 加载后：数据在内存中，可以快速查找和修改
```

**删除操作执行流程：**

```
// 删除操作需要三步：
// 1. 查找：根据 expr 条件找到要删除的记录
// 2. 标记：标记这些记录为已删除
// 3. 同步：将删除标记写回磁盘

// 这三步都必须在内存中完成！
```

``` java
R<MutationResult> response = milvusClient.delete(deleteParam);
//对应实现
public R<MutationResult> delete(DeleteParam requestParam) {
        return this.retry(() -> {
            return super.delete(requestParam); //milvus服务端返回删除响应结果
        });
    }
//为什么要用retry（重试）？
//1.网络偶尔不稳定，第一次可能失败
//2.Milvus 服务暂时不可用
//3.资源竞争
```

### 2.4 chunkDocument(String content, String filePath) 文档分片方法

#### 2.4.1 splitByHeadings(String content)

**按照Markdown标题分割文件**

```Java
// 匹配 Markdown 标题的正则表达式
Pattern headingPattern = Pattern.compile("^(#{1,6})\\s+(.+)$", Pattern.MULTILINE);

// 正则解释：
// ^           - 行开头
// (#{1,6})    - 1到6个#号（标题级别）
// \\s+        - 至少一个空格
// (.+)        - 标题内容（任意字符）
// $           - 行结尾
// MULTILINE   - 让 ^ 和 $ 匹配每一行的开头和结尾

// 能匹配的格式：
// # 标题1
// ## 标题2  
// ### 标题3
// ###### 标题6
```

主要思路就是，先开辟一个list数组sections，来存放每个section对象,然后利用正则表达式创建匹配规则，随后让整个文本content和这个规则开始匹配，匹配到标题然后就进行切分，分成标题和正文还有当前标题开始的位置。

#### 2.4.2 chunkSection(Section section, int startChunkIndex) 

**对单个章节进行分片**

1）splitByParagraphs(String content)；章节内容较长，需按段落分割文本
		`split()` 是 Java 中 String 类的方法，用于**将字符串按指定的正则表达式分割成字符串数组**。
		`trim()`是用来去除首尾空格。

2）开辟一个空分片，便于存储不超过分片的最大容量的段落（可能不止一段）

3）模拟:

```java
/*第一段*/
currentChunk = "Spring Boot 是一个基于 Java 的框架，用于简化 Spring 应用开发。\n\n"//不进入if循环
// 当前长度 ≈ 52
    
/*第二段*/   
// 检查：52 + 45 + 2(换行) = 99 < 100
// 不超过最大尺寸，直接添加
currentChunk = "Spring Boot 是一个基于 Java 的框架，用于简化 Spring 应用开发。\n\n它提供了自动配置、起步依赖等特性，让开发者可以快速启动项目。\n\n"  //不进入if循环
// 当前长度 ≈ 99
    
/*第三段*/
// 检查：99 + 40 + 2 = 141 > 100
// 超过最大尺寸，需要创建新分片

// 保存分片1
chunk1 = "Spring Boot 是一个基于 Java 的框架，用于简化 Spring 应用开发。\n\n它提供了自动配置、起步依赖等特性，让开发者可以快速启动项目。"
chunks.add(chunk1)

```

4）String getOverlapText(String text);

**获取重叠文本**

`Math.*min*(chunkConfig.getOverlap(), text.length()); ` 当文本比配置的重叠还短时，取整个文本;
	   `substring(int beginIndex)` 从指定位置开始提取到字符串末尾; 

```java
// 尝试在句子边界截断（查找最后一个句号、问号、感叹号）
        int lastSentenceEnd = Math.max(
            overlap.lastIndexOf('。'),
            Math.max(overlap.lastIndexOf('？'), overlap.lastIndexOf('！'))
        );
```

情况为：段落长度大于最大分片长度时，需要用到上述截断。

```java
// 获取重叠部分（假设取最后20字符）
overlap = "所以需要配置这些东西。快速启动项目"

// 创建新分片2，从重叠开始
currentChunk = "快速启动项目"

// 添加段落3
currentChunk = "快速启动项目它还包含了嵌入式服务器，无需部署 WAR 文件即可运行。\n\n"
```

### 2.5  List\<Float> generateEmbedding(String content)

**生成向量嵌入**

#### 2.5.1 TextEmbeddingResult类

这个类就是 SDK 用来**封装和解析**这个 JSON 响应的 Java 对象。

当执行到`TextEmbeddingResult result = textEmbedding.call(param);`这里时，SDK 才会**真正地向阿里云服务器发起 HTTPS 请求**。这个请求的 Header 中会携带 API Key进行网络鉴权。这行代码返回的是一个**包含了向量化结果以及其他元数据的对象**。

这个对象中的核心内容：

1. output **(最重要的部分)**

   ```java
   public final class TextEmbeddingOutput {
       private List<TextEmbeddingResultItem> embeddings;
   }
   
   public class TextEmbeddingResultItem {
       @SerializedName("text_index")
       private Integer textIndex;
       private List<Double> embedding;
   }
   
   public List<TextEmbeddingResultItem> getEmbeddings() {
           return this.embeddings;
       }
   
   ```

2. usage (元数据)
   这部分记录了本次调用的计费信息。

   - `total_tokens`：本次请求消耗的 token 总数。虽然 embedding 模型的计算方式和对话模型不同，但也会有一个计费单元，通常也会放在这里。

3. request_id (元数据)
   这是阿里云服务端为这次请求生成的唯一标识符（UUID）。当请求出现问题，需要联系阿里云技术支持时，提供这个 ID 可以让他们快速定位问题。

#### 2.5.2 List\<Float> getFloats(TextEmbeddingResult result) 方法

```java
public List<Float> getFloats(TextEmbeddingResult result) {
    // 1. 从结果中获取所有 embeddings
    List<EmbeddingItem> embeddings = result.getOutput().getEmbeddings();
    
    // 2. 获取第一个文本的向量数据
    EmbeddingItem firstItem = embeddings.get(0);  // 第一个文本，因为我们只传入了一个分片，所以只需要get(0)
    List<Double> embeddingDoubles = firstItem.getEmbedding();  // 假设有512个Double
    
    // 3. 创建 Float 列表，预分配 512 个位置
    List<Float> floatEmbedding = new ArrayList<>(embeddingDoubles.size());  // =512
    
    // 4. 逐个转换并添加
    for (Double value : embeddingDoubles) {  // 循环512次
        floatEmbedding.add(value.floatValue()); //Double转Float型
    }
    
    return floatEmbedding;  // 返回512个Float
}
```

**为什么不用Double直接存？**

```Java
// ❌ 如果直接用 Double 存
Milvus 集合 {
    vector: DataType.DoubleVector  // 大部分 Milvus 版本不支持！
    // 或者需要特殊配置
}

// ✅ 用 Float 存
Milvus 集合 {
    vector: DataType.FloatVector  // 标准支持
    dimension: 512                 // 512 维
}

// Float 已经足够：
// - 向量搜索不需要超高精度
// - 节省一半存储空间（512维 × 4字节 vs 8字节）
// - 计算更快
```

### 2.6  Map<String, Object> buildMetadata(String filePath, DocumentChunk chunk, int totalChunks) 方法

**构建元数据（包含文件信息)**

```java
        metadata.put("_source", normalizedPath);//存储完整路径
        metadata.put("_extension", extension);//存储扩展名（.md、.txt）
        metadata.put("_file_name", fileNameStr);//存储文件名
        
        // 分片信息
        metadata.put("chunkIndex", chunk.getChunkIndex());//存储当前分片序号
        metadata.put("totalChunks", totalChunks);//存储分片总数
        
        // 标题信息
        if (chunk.getTitle() != null && !chunk.getTitle().isEmpty()) {
            metadata.put("title", chunk.getTitle());//当前存储分片标题
        }
```

### 2.7 insertToMilvus(String content, List\<Float> vector,  Map<String, Object> metadata, int chunkIndex)

**插入向量到Milvus**

假如要插入一条数据：

```java
// 输入的参数
String id = "doc1_chunk0";
String content = "Spring Boot是一个框架...";
List<Float> vector = [0.123, -0.456, 0.789, ...];  // 512个浮点数
Map<String, Object> metadata = {
    "_source": "C:/Users/docs/spring-boot.md",
    "_extension": ".md",
    "_file_name": "spring-boot.md",
    "chunkIndex": 0,
    "totalChunks": 5,
    "title": "Spring Boot简介"
}
```

构建字段数据（代码拆解）：

```java
// 创建空的字段列表
List<InsertParam.Field> fields = new ArrayList<>();

// 1. 添加ID字段
fields.add(new InsertParam.Field("id", Collections.singletonList(id)));
// 此时 fields = [
//   Field{name="id", values=["doc1_chunk0"]}
// ]

// 2. 添加content字段
fields.add(new InsertParam.Field("content", Collections.singletonList(content)));
// fields = [
//   Field{name="id", values=["doc1_chunk0"]},
//   Field{name="content", values=["Spring Boot是一个框架..."]}
// ]

// 3. 添加vector字段
fields.add(new InsertParam.Field("vector", Collections.singletonList(vector)));
// fields = [
//   Field{name="id", values=["doc1_chunk0"]},
//   Field{name="content", values=["Spring Boot是一个框架..."]},
//   Field{name="vector", values=[[0.123, -0.456, ...]]}
// ]

// 4. 转换metadata为JsonObject并添加
Gson gson = new Gson();
JsonObject metadataJson = gson.toJsonTree(metadata).getAsJsonObject();
fields.add(new InsertParam.Field("metadata", Collections.singletonList(metadataJson)));
// fields = [
//   Field{name="id", values=["doc1_chunk0"]},
//   Field{name="content", values=["Spring Boot是一个框架..."]},
//   Field{name="vector", values=[[0.123, -0.456, ...]]},
//   Field{name="metadata", values=[{_source="C:/...", title="..."}]}
// ]
```

构建插入参数：

```java
InsertParam insertParam = InsertParam.newBuilder()
        .withCollectionName("biz")  // 指定集合名
        .withFields(fields)          // 设置字段数据
        .build();

// insertParam 对象的结构：
// InsertParam {
//     collectionName: "biz",
//     fields: [
//         Field("id", ["doc1_chunk0"]),
//         Field("content", ["Spring Boot是一个框架..."]),
//         Field("vector", [[0.123, -0.456, ...]]),
//         Field("metadata", [{"_source":"...", "title":"..."}])
//     ]
// }
```

执行插入：

```java
R<MutationResult> insertResponse = milvusClient.insert(insertParam);  //在这里发生insertParam对象 转Protobuf（高效） 二进制（SDK内部发生的，我们看不到）
```

**发送给Milvus的实际请求（Json格式）：**

```json
{
  "collection_name": "biz",
  "fields_data": [
    {
      "field_name": "id",
      "type": "VarChar",
      "values": ["doc1_chunk0"]
    },
    {
      "field_name": "content",
      "type": "VarChar",
      "values": ["Spring Boot是一个框架..."]
    },
    {
      "field_name": "vector",
      "type": "FloatVector",
      "values": [[0.123, -0.456, 0.789, ...]]
    },
    {
      "field_name": "metadata",
      "type": "JSON",
      "values": [
        {
          "_source": "C:/Users/docs/spring-boot.md",
          "_extension": ".md",
          "_file_name": "spring-boot.md",
          "chunkIndex": 0,
          "totalChunks": 5,
          "title": "Spring Boot简介"
        }
      ]
    }
  ]
}
```

## 三、3月17日 （第三天：searchSimilarDocuments()搜素相似文档）

### 3.1 SearchParam类

```java
SearchParam searchParam = SearchParam.newBuilder()
        .withCollectionName(MilvusConstants.MILVUS_COLLECTION_NAME)  // 指定集合
        .withVectorFieldName("vector")  // 指定向量字段
        .withVectors(Collections.singletonList(queryVector))  // 查询向量
        .withTopK(topK)  // 返回最相似的K条
        .withMetricType(io.milvus.param.MetricType.L2)  // 距离计算方式
        .withOutFields(List.of("id", "content", "metadata"))  // 需要返回的字段
        .withParams("{\"nprobe\":10}")  // 搜索参数
        .build();
```

### 3.2 执行搜素

```java
 R<SearchResults> searchResponse = milvusClient.search(searchParam); //将查找的含结果的对象赋值给searchResponse，这个响应里包含data，status。。。
```



发送给 Milvus 的请求大致是：

```json
{
  "collection_name": "biz",
  "vectors": [[0.123, -0.456, 0.789, ...]],
  "topk": 5,
  "metric_type": "L2",
  "output_fields": ["id", "content", "metadata"]
}
```

### 3.3 解析搜索结果

``` java
SearchResultsWrapper wrapper = new SearchResultsWrapper(searchResponse.getData().getResults());//把results传给新new的这个对象
```

### 3.4 getRowRecords(int indexOfTarget)

```java
public List<QueryResultsWrapper.RowRecord> getRowRecords(int indexOfTarget)//因为传入只有一组向量（一个queryVector），所以这里为0 
{
        List<QueryResultsWrapper.RowRecord> records = new ArrayList();
        List<IDScore> idScore = this.getIDScore(indexOfTarget);//获取该组搜索结果的ID和分数列表，这里的分数指的搜索结果与查询文本之间的相似程度，
        long topK = Math.min(this.results.getTopK(), (long)idScore.size());//确定实际返回的数量，你请求返回的数量（比如5条），实际找到的数量（可能只有3条），取较小值，确保不会越界。返回几个元素是由topk的值决定的（topk=3，返回三个）

        for(int i = 0; (long)i < topK; ++i) {
            IDScore score = (IDScore)idScore.get(i);
            QueryResultsWrapper.RowRecord record = new QueryResultsWrapper.RowRecord();//每条结果创建一个新的 RowRecord 对象
            if (score.getStrID().isEmpty()) {
                record.put(this.primaryKey, score.getLongID());
            } else {
                record.put(this.primaryKey, score.getStrID());
            }
//Milvus的 ID 可以是两种类型：
            // 1. 长整型（自动生成）
IDScore {
    longID: 10001,           // 有值
    strID: "",               // 为空
    score: 0.85
}

// 2. 字符串（你自己指定的）
IDScore {
    longID: 0,                // 无效
    strID: "doc1_chunk2",     // 有值
    score: 0.85
}
            record.put("score", score.getScore());
            this.buildRowRecord(record, (long)indexOfTarget * topK + (long)i);//这个公式用来定位字段数据的全局索引位置
            records.add(record);
        }

        return records;
    }
```

## 四、3月19日（第四天：文件上传接口整合FileUploadController）

`consumes = "multipart/form-data"`: 是HTTP请求的一种内容类型（Content-Type），接收 multipart/form-data 格式（文件上传专用）。

返回 `ResponseEntity`：可以返回各种 HTTP 响应。

### 4.1 文件格式相关内容

<img src="C:\Users\wsy\AppData\Roaming\Typora\typora-user-images\image-20260319131103568.png" alt="image-20260319131103568" style="zoom: 80%;" />

```java
String fileExtension = getFileExtension(originalFilename);
        if (!isAllowedExtension(fileExtension)) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                    .body("不支持的文件格式，仅支持: " + fileUploadConfig.getAllowedExtensions());
        }

private String getFileExtension(String filename) {
        int lastIndexOf = filename.lastIndexOf(".");
        if (lastIndexOf == -1) {
            return "";
        }
        return filename.substring(lastIndexOf + 1).toLowerCase();
    }

//允许上传的文件格式
private boolean isAllowedExtension(String extension) {
        String allowedExtensions = fileUploadConfig.getAllowedExtensions();
        if (allowedExtensions == null || allowedExtensions.isEmpty()) {
            return false;
        }
        List<String> allowedList = Arrays.asList(allowedExtensions.split(","));
        return allowedList.contains(extension.toLowerCase());
    }

//文件上传配置类
@Getter
@Configuration 
@ConfigurationProperties(prefix = "file.upload") //yml配置文件中的前缀
public class FileUploadConfig {

    private String path;
    private String allowedExtensions;//这个字段名要和yml文件中对应相同

    public void setPath(String path) {
        this.path = path;
    }

    public void setAllowedExtensions(String allowedExtensions) {
        this.allowedExtensions = allowedExtensions;
    }
}
```

**工作原理**：

- Spring Boot 启动时，扫描所有 `@ConfigurationProperties` 注解
- 找到 `prefix = "file.upload"`
- 去 `application.yml` 中找所有以 `file.upload` 开头的配置
- 自动将配置值赋给同名的字段
- `@ConfigurationProperties` 注解告诉 Spring：“帮我管理这个类”

### 4.2 上传文件的这一步

`String uploadPath = fileUploadConfig.getPath();`**从配置对象中获取上传文件的存储路径**。

`uploadDir.resolve(originalFilename);`拼接语法

uploads目录是保存在本地的目录

```java
//先在本地看，文件存不存在
            if (Files.exists(filePath)) {
                logger.info("文件已存在，将覆盖: {}", filePath);
                Files.delete(filePath);//删除旧同名文件
            }
            
            Files.copy(file.getInputStream(), filePath);//copy,保存新文件
```

## 对话Agent

## 五、3月20日（第五天：ChatController）

```java
//创建日志记录器，用于输出调试和错误信息
private static final Logger logger = LoggerFactory.getLogger(ChatController.class);

 @Autowired
    private AiOpsService aiOpsService; //自动注入AI运维服务
    
    @Autowired
    private ChatService chatService;

    @Autowired
    private ToolCallbackProvider tools; //自动注入工具回调提供者

    private final ExecutorService executor = Executors.newCachedThreadPool();  //创建缓存线程池，用于异步任务处理

    // 存储会话信息
    private final Map<String, SessionInfo> sessions = new ConcurrentHashMap<>();  //线程安全的Map，存储会话信息，键为会话ID，值为会话对象
    
    // 最大历史消息窗口大小（成对计算：用户消息+AI回复=1对）
    private static final int MAX_WINDOW_SIZE = 6;  //最大历史消息窗口大小（6对消息，即12条消息）
```

### 5.1 ResponseEntity<ApiResponse\<ChatResponse>> chat(@RequestBody ChatRequest request) 普通对话接口 

#### 5.1.1 内部静态类**`ChatRequest`**

``` java 
    @Setter
    @Getter
    public static class ChatRequest {
        @com.fasterxml.jackson.annotation.JsonProperty(value = "Id")//当JSON中出现"Id"字段时，会映射到这个Java属性
        @com.fasterxml.jackson.annotation.JsonAlias({"id", "ID"})//支持多种命名方式：当JSON中出现"id"或"ID"时，也可以映射到同一个Java属性
        private String Id;
        
        @com.fasterxml.jackson.annotation.JsonProperty(value = "Question")
        @com.fasterxml.jackson.annotation.JsonAlias({"question", "QUESTION"})//同理
        private String Question;

    }  
```

#### 5.1.2 泛型类**`ApiResponse<T>`**

- <T>：泛型声明，`T`可以是任何类型（String、List、Object等）
- **`ApiResponse<T>`**：泛型类，使用`T`作为数据类型占位符

```java
public static <T> ApiResponse<T> success(T data) {//
            ApiResponse<T> response = new ApiResponse<>();
            response.setCode(200);
            response.setMessage("success");
            response.setData(data);
            return response;
        }
```

- **`<T>`**：声明这个方法有自己的泛型参数`T`
- **`ApiResponse<T>`**：返回类型是`ApiResponse`，泛型类型为`T`
- **`(T data)`**：接收一个泛型参数`data`，类型为`T`

这里的`T` **只替换 `data` 字段的数据类型**，其他字段（`code`、`message`）的数据类型是固定的，不会改变。

如，当 T = User 时：

```java
ApiResponse<User> response = new ApiResponse<>();
response.setCode(200);        // code 还是 int
response.setMessage("成功");   // message 还是 String
response.setData(new User()); // data 是 User 对象

// 结构：
{
    "code": 200,           // ← 永远是 int
    "message": "成功",      // ← 永远是 String
    "data": {              // ← 现在是 User 对象
        "name": "张三",
        "age": 25
    }
}
```

#### 5.1.3 SessionInfo getOrCreateSession(String sessionId) 获取或创建会话

**根据会话ID获取或创建新的会话对象。**

首先了解`ConcurrentHashMap`，`ConcurrentHashMap` 是 Java 并发包（`java.util.concurrent`）中的**线程安全的哈希表**实现，专门为高并发场景设计。

**与 HashMap 的对比**：

| 特性          | HashMap             | Hashtable        | ConcurrentHashMap    |
| :------------ | :------------------ | :--------------- | :------------------- |
| **线程安全**  | ❌ 不安全            | ✅ 安全（全表锁） | ✅ 安全（分段锁/CAS） |
| **性能**      | 单线程最高          | 并发性能差       | 高并发性能好         |
| **允许 null** | ✅ key/value可为null | ❌ 都不允许null   | ❌ 都不允许null       |
| **迭代器**    | 快速失败            | 快速失败         | 弱一致性             |

在代码开始我们声明了：`private final Map<String, SessionInfo> sessions = new ConcurrentHashMap<>();`

```java
// 线程安全的 put-if-absent
SessionInfo session = sessions.computeIfAbsent(sessionId, SessionInfo::new);
// 多个线程同时调用，保证只有一个线程创建SessionInfo对象
```

**如果不保证唯一性会发生什么？**

```java
// 场景：用户 "user-123" 同时发送两个请求
// 请求1：查询天气
// 请求2：查询新闻

// 没有并发控制的情况：
Thread 1: sessions.get("user-123") → null
Thread 2: sessions.get("user-123") → null

// 两个线程都认为会话不存在，各自创建
Thread 1: SessionInfo session1 = new SessionInfo(); // 会话对象A
Thread 2: SessionInfo session2 = new SessionInfo(); // 会话对象B

// 最终Map中只会保留一个（取决于最后put的）
sessions.put("user-123", session1); // 线程1放入
sessions.put("user-123", session2); // 线程2覆盖，session1丢失

// 结果：用户的两个请求被不同的会话对象处理
// 会话对象A：记录"查询天气"，但被丢弃
// 会话对象B：记录"查询新闻"
// 用户丢失了对话历史！❌
```

**ConcurrentHashMap 如何解决?**

`sessions.computeIfAbsent()`解决:

```java
// 使用 computeIfAbsent 保证原子性
private final Map<String, SessionInfo> sessions = new ConcurrentHashMap<>();

private SessionInfo getOrCreateSession(String sessionId) {
    return sessions.computeIfAbsent(sessionId, id -> {
        System.out.println("创建新会话: " + id);
        return new SessionInfo();
    });
}

// 并发执行结果：
T1: computeIfAbsent("user-123") → 发现不存在 → 创建 SessionInfo@001
T2: computeIfAbsent("user-123") → 发现已存在 → 返回 SessionInfo@001
T3: computeIfAbsent("user-123") → 发现已存在 → 返回 SessionInfo@001

// Map最终状态
sessions = {
    "user-123": SessionInfo@001  // 只有一个会话对象
}

// 所有请求共享同一个会话对象 ✅
```

#### 5.1.4 List<Map<String, String>> getHistory() 获取历史消息

**这里为什么选用List<Map<String, String>>作为返回值保存历史记录？**

**答：List 保证顺序 ， Map 不保证顺序。**

```java
// ✅ 使用 List - 保证顺序
List<Map<String, String>> history = new ArrayList<>();
history.add(Map.of("user", "我叫张三"));
history.add(Map.of("ai", "你好，张三"));
history.add(Map.of("user", "我今年25岁"));
history.add(Map.of("ai", "25岁，很年轻的年龄"));
history.add(Map.of("user", "我叫什么名字？"));
history.add(Map.of("ai", "你叫张三"));

// 遍历时，永远按照添加顺序
for (Map<String, String> msg : history) {
    // 输出顺序：用户消息 → AI回复 → 用户消息 → AI回复 ...
}

// ❌ 使用 HashMap - 不保证顺序
Map<String, String> history = new HashMap<>();
history.put("user1", "我叫张三");
history.put("ai1", "你好，张三");
history.put("user2", "我今年25岁");
history.put("ai2", "25岁，很年轻的年龄");

// 遍历时，顺序可能是：
// ai2 → user1 → ai1 → user2
// 完全打乱了对话逻辑！AI会理解错上下文
```

**这里为什么上锁？**

```java
public List<Map<String, String>> getHistory() {
            lock.lock();
            try {
                return new ArrayList<>(messageHistory);//将已有的历史记录messageHistory放在新开辟的ArrayList<>中并返回
            } finally {
                lock.unlock();
            }
        }
```

虽然每个用户有每个用户自己的SessionInfo对象，访问时互不干扰，但是会避免不了有一种情况：**同一个用户可能同时发送多个请求**。还有种情况是，在AI回答完问题后会新增用户历史消息记录，这时候如果有查询进来，就会冲突，所以需要在这里也上锁。

```java
// 危险场景：同一个用户同时发送2个请求
// 用户A快速连续点击：
// 请求1: 发送问题 "帮我查天气"
// 请求2: 发送问题 "帮我查新闻"（在请求1还没处理完时）

// 服务器处理：
Thread-1（请求1）: sessionA.addMessage("查天气", "晴天")
Thread-2（请求2）: sessionA.addMessage("查新闻", "重要新闻")
                ↓
        同时访问同一个 SessionInfo@001 对象！
        如果没有锁保护，会出问题！
```

#### 5.1.4 DashScopeApi createDashScopeApi() 创建DashScope API实例

创建并返回一个**DashScope API客户端实例**，用于与阿里云DashScope平台进行通信。

#### 5.1.5 DashScopeChatModel createStandardChatModel(DashScopeApi dashScopeApi) 创建标准对话ChatModel

基于API客户端，创建一个**配置了标准参数的聊天模型**。

```java
/**
     * 创建标准对话 ChatModel（默认参数）
     */
    public DashScopeChatModel createStandardChatModel(DashScopeApi dashScopeApi) {
        return createChatModel(dashScopeApi, 0.7, 2000, 0.9);
    }
```



**参数详解:**

| 参数                 | 值   | 含义           | 作用                             |
| :------------------- | :--- | :------------- | :------------------------------- |
| 温度 (temperature)   | 0.7  | 控制随机性     | 值越高回答越有创造性，越低越保守 |
| 最大长度 (maxTokens) | 2000 | 最大输出字符数 | 限制AI回答的长度                 |
| Top P                | 0.9  | 核采样参数     | 控制词汇选择的多样性             |

```java
/**
     * 创建 ChatModel
     * @param temperature 控制随机性 (0.0-1.0)
     * @param maxToken 最大输出长度
     * @param topP 核采样参数
     */
    public DashScopeChatModel createChatModel(DashScopeApi dashScopeApi, double temperature, int maxToken, double topP) {
        return DashScopeChatModel.builder()
                .dashScopeApi(dashScopeApi)
                .defaultOptions(DashScopeChatOptions.builder()
                        .withModel(DashScopeChatModel.DEFAULT_MODEL_NAME)
                        .withTemperature(temperature)
                        .withMaxToken(maxToken)
                        .withTopP(topP)
                        .build())
                .build();
    }
```

**在整体架构中的位置:**

```markdown
ChatController (控制器)
    ↓
ChatService (服务层)
    ↓
createDashScopeApi()     → 创建API客户端（连接凭证）
    ↓
createStandardChatModel() → 创建AI模型（配置参数）
    ↓
createReactAgent()       → 创建智能代理（支持工具调用）
    ↓
executeChat()            → 执行对话
```

#### 5.1.6 void logAvailableTools()  记录可用工具列表

作用：调试和监控，验证工具注册，了解Agent能力。

```java
/**
     * 记录可用工具列表：mcp服务提供的工具
     */
    public void logAvailableTools() {
        ToolCallback[] toolCallbacks = tools.getToolCallbacks();
        logger.info("可用工具列表:");
        for (ToolCallback toolCallback : toolCallbacks) {
            logger.info(">>> {}", toolCallback.getToolDefinition().name());
        }
    }
```

| 代码部分                                          | 含义                             |
| :------------------------------------------------ | :------------------------------- |
| `for (ToolCallback toolCallback : toolCallbacks)` | 遍历每个工具回调                 |
| `toolCallback.getToolDefinition()`                | 获取工具的元数据定义             |
| `.name()`                                         | 获取工具的名称                   |
| `logger.info(">>> {}", ...)`                      | 打印工具名称，`>>>` 前缀突出显示 |

**ToolCallback:**

```java
// ToolCallback 封装了工具的定义和调用逻辑
public interface ToolCallback {
    ToolDefinition getToolDefinition();  // 获取工具定义（名称、描述、参数）
    Object call(Object input);            // 执行工具的逻辑
}

// 例如：天气查询工具
class WeatherToolCallback implements ToolCallback {
    @Override
    public ToolDefinition getToolDefinition() {
        return ToolDefinition.builder()
            .name("getWeather")
            .description("查询指定城市的天气")
            .build();
    }
    
    @Override
    public Object call(Object input) {
        // 实际查询天气的代码
        return "晴天，25度";
    }
}
```

**工具的实用场景：**

```java
// 用户问题："北京今天天气怎么样？"
// AI Agent的处理流程：

// 1. Agent看到问题
// 2. 检查可用工具列表（从logAvailableTools知道有getWeather）
// 3. 决定调用getWeather工具
// 4. 工具返回结果："北京今天晴天，25度"
// 5. Agent根据工具结果生成回答："北京今天天气晴朗，温度25度。"
```

#### 5.1.7 String buildSystemPrompt(List<Map<String, String>> history) 构建系统提示词

本质就是将基础系统提示词（包括可调用的工具）后面追加上该用户的对话历史记录，让AI大模型能够拥有上下文记忆功能，最后返回一个字符串。

#### 5.1.8 ReactAgent createReactAgent(DashScopeChatModel chatModel, String systemPrompt)  创建ReactAgent

```java
/**
     * 创建 ReactAgent
     * @param chatModel 聊天模型
     * @param systemPrompt 系统提示词
     * @return 配置好的 ReactAgent
     */
    public ReactAgent createReactAgent(DashScopeChatModel chatModel, String systemPrompt) {
        return ReactAgent.builder()
                .name("intelligent_assistant")
                .model(chatModel)
                .systemPrompt(systemPrompt)
                .methodTools(buildMethodToolsArray())
                .tools(getToolCallbacks())
                .build();
    }
```

**最后methodTools和tools的区别：**

```java
// 开发/测试环境
// cls.mock-enabled = true
buildMethodToolsArray() = [
    DateTimeTools,      // 本地工具
    InternalDocsTools,  // 本地工具
    QueryMetricsTools,  // 本地工具
    QueryLogsTools      // 本地Mock工具（模拟日志查询）
]

getToolCallbacks() = [
    // MCP工具（可能为空或模拟）
]

// 生产环境
// cls.mock-enabled = false
buildMethodToolsArray() = [
    DateTimeTools,      // 本地工具
    InternalDocsTools,  // 本地工具
    QueryMetricsTools   // 本地工具
    // QueryLogsTools 不包含（避免冲突）
]

getToolCallbacks() = [
    MCPQueryLogsTool,   // 真正的日志查询工具
    MCPMonitorTool,     // 真正的监控工具
    MCPDeployTool       // 真正的部署工具
]
```

Agent最终可用的工具 = 本地工具 ∪ MCP工具。

**补充**：**`Object[]` 是Java中的对象数组，可以存储任何类型的对象。**

```java
// Object[] 可以存储任何Java对象
Object[] array = new Object[3];
array[0] = "字符串";           // String
array[1] = 123;                // Integer（自动装箱）
array[2] = new DateTimeTools(); // 自定义对象

//动态构建方法工具数组
public Object[] buildMethodToolsArray() {
        if (queryLogsTools != null) {
            // Mock 模式：包含 QueryLogsTools
            return new Object[]{dateTimeTools, internalDocsTools, queryMetricsTools, queryLogsTools};
        } else {
            // 真实模式：不包含 QueryLogsTools（由 MCP 提供日志查询功能）
            return new Object[]{dateTimeTools, internalDocsTools, queryMetricsTools};
        }
    }
```

#### 5.1.9 String executeChat(ReactAgent agent, String question) 执行ReactAgent对话(非流式)

```java
/**
     * 执行 ReactAgent 对话（非流式）
     * @param agent ReactAgent 实例
     * @param question 用户问题
     * @return AI 回复
     */
    public String executeChat(ReactAgent agent, String question) throws GraphRunnerException {
        logger.info("执行 ReactAgent.call() - 自动处理工具调用");
        var response = agent.call(question);
        String answer = response.getText();
        logger.info("ReactAgent 对话完成，答案长度: {}", answer.length());
        return answer;
    }
```

- **`var`**：Java 10+的类型推断，编译器自动推断为`ChatResponse`类型
- **`agent.call(question)`**：调用ReactAgent，传入用户问题
- **自动处理**：Agent内部会：
  1. 分析问题是否需要工具
  2. 如果需要，自动调用相应工具
  3. 获取工具结果
  4. 生成最终回答

**ReactAgent.call() 内部可能的工作流程：**

```java
// 伪代码：agent.call() 内部实现
public ChatResponse call(String question) {
    // 1. 将问题发送给AI模型
    String thought = model.call(buildPrompt(question));
    
    // 2. 如果AI决定调用工具
    if (thought.contains("需要调用工具")) {
        // 3. 解析工具名称和参数
        String toolName = extractToolName(thought);
        Map<String, Object> params = extractParams(thought);
        
        // 4. 查找并执行工具
        Tool tool = findTool(toolName);
        Object toolResult = tool.execute(params);
        
        // 5. 将工具结果返回给AI
        String finalAnswer = model.call(buildFinalPrompt(question, toolResult));
        
        return new ChatResponse(finalAnswer);
    }
    
    // 6. 不需要工具，直接返回
    return new ChatResponse(thought);
}
```

### 5.2 SseEmitter chatStream(@RequestBody ChatRequest request)  ReactAgent 对话接口(SSE 流式模式)

- **`produces = "text/event-stream;charset=UTF-8"`**：指定返回内容类型为SSE流式数据
- **`SseEmitter`**：Spring提供的SSE（Server-Sent Events）发射器，用于推送流式数据

` new SseEmitter(300000L);` 创建SSE发射器。超时时间为五分钟，意思是，从创建这个emitter开始计时。如果五分钟内还没有调用emitter.complete()，spring就会强制关闭连接，前端就收不到后续的数据了。

#### 5.2.1 emitter.send()

```java
emitter.send(SseEmitter.event().name("message").data(SseMessage.error("问题内容不能为空"), MediaType.APPLICATION_JSON));
                
```

**通过SSE连接向前端发送一条错误消息**.

```java
emitter.send(
    SseEmitter.event()                    // 1. 创建SSE事件构建器
        .name("message")                  // 2. 设置事件名称
        .data(SseMessage.error("问题内容不能为空"), MediaType.APPLICATION_JSON), // 3. 设置事件数据部分
    MediaType.APPLICATION_JSON            // 4. 指定数据格式为json
);
```

以上代码实际生成的数据格式：

```http
event: message
id: 123456
data: {"type":"error","content":"问题内容不能为空"}
retry: 10000
```

#### 5.2.2 emitter.complete()

是**正常结束SSE连接**的方法，表示数据推送完成，连接可以安全关闭。

#### 5.2.3 emitter.completeWithError(e)

异常关闭，客户端收到错误

#### 5.2.4 executor.execute()  —— 异步通信关键

因为这个方法是流式输出，所以我们需要在代码中设置线程池，在执行线程代码时，先return emitter ，将响应头`Content-Type: text/event-stream`告知前端，让前端知道我们这次发送的是流式数据，同时前端进入"流式接收模式"，不再有“等待完整响应”的超时。

**实际流程：**

```java
T0: 创建emitter（5分钟超时）
T0: 返回emitter，前端连接建立
T0-T5: 后台处理，逐步推送数据
T5: 5分钟到，Spring销毁emitter，关闭HTTP连接
T5: 前端检测到连接关闭，触发onerror或onclose
T5: 前端认为"连接结束了"
    
// 所以需要：
// 1. 立即返回emitter（避免等待超时）
// 2. 持续推送数据（避免空闲断开）
// 3. 合理设置后端超时（避免资源浪费）
```



```java
// ✅ 使用异步 - 真正流式
@PostMapping("/stream")
public SseEmitter stream() {
    SseEmitter emitter = new SseEmitter();
    
    // 立即返回，Spring立即发送响应头
    executor.execute(() -> {
        // 此时响应头已经发送
        // 每次send都会立即推送到客户端
        for (int i = 0; i < 10; i++) {
            Thread.sleep(1000);
            emitter.send("数据");  // 立即发送到客户端
        }
        emitter.complete();
    });
    
    return emitter;  // 立即返回
}

// 实际发生了什么：
// 1. 创建emitter，立即返回
// 2. Spring立即发送响应头到客户端
// 3. 后台线程开始循环
// 4. 每次emitter.send()都立即推送到客户端
// 5. 前端每秒收到一个数据块 → 真正的流式效果！
```

#### 5.2.5 核心： 流式处理

使用 *agent.stream()* 进行流式对话
`Flux<NodeOutput> stream = agent.stream(request.getQuestion());`

- **`Flux`**：Project Reactor的响应式流，可以推送多个数据项
- **`NodeOutput`**：流中的每个数据单元，包含不同类型的信息（AI思考、工具调用、文本片段等）
- **`agent.stream()`**：启动**ReAct模式**的流式处理

**简化的内部实现逻辑：**

```java
// ReactAgent内部的大致实现（简化版）
public Flux<NodeOutput> stream(String question) {
    return Flux.create(sink -> {
        // 1. 创建ReAct循环
        ReActLoop loop = new ReActLoop(model, tools, systemPrompt);
        
        // 2. 开始处理
        loop.process(question, new ReActCallback() {
            @Override
            public void onThought(String thought) {
                // AI思考时推送
                sink.next(createNodeOutput(OutputType.AGENT_THOUGHT, thought));
            }
            
            @Override
            public void onToolCall(String toolName, Map<String, Object> params) {
                // 调用工具时推送
                sink.next(createNodeOutput(OutputType.AGENT_TOOL_START, toolName));
                
                // 执行工具
                Object result = executeTool(toolName, params);
                
                // 工具完成时推送
                sink.next(createNodeOutput(OutputType.AGENT_TOOL_FINISHED, result));
            }
            
            @Override
            public void onStreamingToken(String token) {
                // 🔥 关键：AI生成每个token时立即推送
                sink.next(createNodeOutput(OutputType.AGENT_MODEL_STREAMING, token));
            }
            
            @Override
            public void onComplete(String finalAnswer) {
                // 完成时推送
                sink.next(createNodeOutput(OutputType.AGENT_MODEL_FINISHED, finalAnswer));
                sink.complete();
            }
            
            @Override
            public void onError(Throwable error) {
                sink.error(error);
            }
        });
    });
}
```

#### 5.2.6 stream.subscribe()

这是整个流式对话的**核心处理逻辑**，负责接收ReactAgent产生的每个数据块并处理。

```java
stream.subscribe(
    onNext,      // 处理每个数据块
    onError,     // 处理错误
    onComplete   // 处理完成
);
```

##### 第一个回调：onNext(处理数据)

```java
output -> {
    try {
        // 检查是否为 StreamingOutput 类型
        if (output instanceof StreamingOutput streamingOutput) {
            OutputType type = streamingOutput.getOutputType();
            
            // 根据不同类型处理...
        }
    } catch (IOException e) {
        logger.error("发送流式消息失败", e);
        throw new RuntimeException(e);
    }
}
```

(1)

```java
if (output instanceof StreamingOutput streamingOutput) {
```



- Java 16+ 的模式匹配语法
- 检查`output`是否是`StreamingOutput`类型
- 如果是，同时将其转换为`streamingOutput`变量

- 等价于：

```java
if (output instanceof StreamingOutput) {
    StreamingOutput streamingOutput = (StreamingOutput) output;
}
```

(2)

```java
OutputType type = streamingOutput.getOutputType();
```

- 获取当前数据块的类型
- 可能的值：
  - `AGENT_MODEL_STREAMING`：AI正在生成文本（最重要）
  - `AGENT_MODEL_FINISHED`：AI生成完成
  - `AGENT_TOOL_FINISHED`：工具调用完成
  - `AGENT_HOOK_FINISHED`：钩子执行完成

(3)

```java
if (type == OutputType.AGENT_MODEL_STREAMING) {
    // 流式增量内容，逐步显示
    String chunk = streamingOutput.message().getText();
    if (chunk != null && !chunk.isEmpty()) {
        fullAnswerBuilder.append(chunk);
        
        // 实时发送到前端
        emitter.send(SseEmitter.event()
                .name("message")
                .data(SseMessage.content(chunk), MediaType.APPLICATION_JSON));
        
        logger.info("发送流式内容: {}", chunk);
    }
}
```

以上这段代码是隐式循环(响应式循环)，会不断执行，直到没有数据。

**详细拆解：**

```java
// 1. 获取文本片段（chunk）
String chunk = streamingOutput.message().getText();
// 例如："你"、"好"、"啊" 等

// 2. 累积完整答案
fullAnswerBuilder.append(chunk);
// 例如："" → "你" → "你好" → "你好啊"

// 3. 发送给前端
emitter.send(SseEmitter.event()
        .name("message")                    // SSE事件名称
        .data(SseMessage.content(chunk),    // 包装成消息对象
              MediaType.APPLICATION_JSON)); // JSON格式

// 4. 记录日志
logger.info("发送流式内容: {}", chunk);
// 控制台输出：发送流式内容: 你
```

**四种类型及其触发时机**

```java
public enum OutputType {
    AGENT_MODEL_STREAMING,  // AI正在生成内容（逐字输出）
    AGENT_MODEL_FINISHED,   // AI生成完成
    AGENT_TOOL_FINISHED,    // 工具调用完成
    AGENT_HOOK_FINISHED     // 钩子执行完成
}
```

**类型切换的完整流程：**

*场景示例：用户问：北京天气怎么样？*

```java
// ReactAgent 内部执行流程
public Flux<NodeOutput> stream(String question) {
    return Flux.create(sink -> {
        
        // ========== 阶段1: AI思考，决定调用工具 ==========
        // 这里可能也会产生流式输出，表示AI的思考过程
        // 但不一定，取决于具体实现
        
        // ========== 阶段2: 调用工具 ==========
        // 工具执行完成后，推送 AGENT_TOOL_FINISHED
        tool.execute("getWeather", "北京", new ToolCallback() {
            @Override
            public void onComplete(Object result) {
                // 🔥 类型变为 AGENT_TOOL_FINISHED
                ToolFinishedOutput output = new ToolFinishedOutput(
                    OutputType.AGENT_TOOL_FINISHED,  // ← 这里设置
                    "getWeather",
                    result
                );
                sink.next(output);
            }
        });
        
        // ========== 阶段3: 根据工具结果生成答案 ==========
        // AI开始生成最终答案（流式）
        model.stream("北京天气晴天", new StreamCallback() {
            @Override
            public void onToken(String token) {
                // 🔥 类型变为 AGENT_MODEL_STREAMING
                StreamingOutput output = new StreamingOutput(
                    OutputType.AGENT_MODEL_STREAMING,  // ← 这里设置
                    new Message(token)
                );
                sink.next(output);
            }
            
            @Override
            public void onComplete() {
                // 🔥 类型变为 AGENT_MODEL_FINISHED
                StreamingOutput output = new StreamingOutput(
                    OutputType.AGENT_MODEL_FINISHED,  // ← 这里设置
                    null
                );
                sink.next(output);
                sink.complete();
            }
        });
    });
}
```

##### 第二个回调：onError(错误处理)

```java
error -> {
    // 1. 记录错误日志
logger.error("ReactAgent 流式对话失败", error);
// 输出：ReactAgent 流式对话失败: API调用超时

// 2. 尝试发送错误消息给前端
try {
    emitter.send(SseEmitter.event()
            .name("message")
            .data(SseMessage.error(error.getMessage()), 
                  MediaType.APPLICATION_JSON));
    // 前端收到：{"type":"error","content":"API调用超时"}
} catch (IOException ex) {
    // 如果发送失败（连接已断开），记录日志
    logger.error("发送错误消息失败", ex);
}

// 3. 异常关闭连接
emitter.completeWithError(error);
}
```

##### 第三个回调：onComplete(完成处理)

```java
() -> {
    try {
        // 1. 获取完整答案
String fullAnswer = fullAnswerBuilder.toString();
// fullAnswerBuilder累积了所有chunk
// 例如："你好啊，我是智能助手"

// 2. 记录完成日志
logger.info("ReactAgent 流式对话完成 - SessionId: {}, 答案长度: {}", 
    request.getId(), fullAnswer.length());
// 输出：ReactAgent 流式对话完成 - SessionId: 123, 答案长度: 15

// 3. 保存到会话历史（关键！）
session.addMessage(request.getQuestion(), fullAnswer);
// 将完整对话保存，供多轮对话使用
// 历史：[用户:你好, AI:你好啊，我是智能助手]
        logger.info("已更新会话历史 - SessionId: {}, 当前消息对数: {}", 
            request.getId(), session.getMessagePairCount());
        
        // 4. 发送完成标记给前端
        emitter.send(SseEmitter.event()
                .name("message")
                .data(SseMessage.done(), MediaType.APPLICATION_JSON));
        // 前端收到：{"type":"done"}
        
        // 5. 正常关闭连接
        emitter.complete();
    } catch (IOException e) {
        logger.error("发送完成消息失败", e);
        emitter.completeWithError(e);
    }
}
```

### 5.3  SseEmitter aiOps()  AI智能运维接口（SSE流式模式）

#### 5.3.1 Optional<OverAllState> executeAiOpsAnalysis(DashScopeChatModel chatModel, ToolCallback[] toolCallbacks)  执行AI Ops告警分析流程

**构建Supervisor Agent（监督者）：**

```java
SupervisorAgent supervisorAgent = SupervisorAgent.builder()
        .name("ai_ops_supervisor")           // Agent名称
        .description("负责调度 Planner 与 Executor 的多 Agent 控制器")  // 功能描述
        .model(chatModel)                     // 共享大模型实例
        .systemPrompt(buildSupervisorSystemPrompt())  // 系统提示词
        .subAgents(List.of(plannerAgent, executorAgent))  // 管理的子Agent列表
        .build();
```

```java
private ReactAgent buildPlannerAgent(DashScopeChatModel chatModel, ToolCallback[] toolCallbacks) {
        return ReactAgent.builder()
                .name("planner_agent")
                .description("负责拆解告警、规划与再规划步骤")
                .model(chatModel)
                .systemPrompt(buildPlannerPrompt())
                .methodTools(buildMethodToolsArray())
                .tools(toolCallbacks)
                .outputKey("planner_plan") //是设置 Agent 执行结果的存储键名，用于在多 Agent 协作中传递和共享数据。
                .build();
    }
```



- **Supervisor模式**: 创建一个总控Agent，负责协调和调度
- **职责**: 决定何时调用Planner、何时调用Executor，形成工作流
- **子Agent**: 将Planner和Executor作为可调度的子单元

**为什么用 Optional<OverAllState> overAllStateOptional而不直接用OverAllState?**

答：

不能直接用 `OverAllState` 是因为：

overAllStateOptional对象可能为null，

1. **null 是不安全的** - 容易导致空指针异常
2. **语义不明确** - 无法从类型看出"可能没有结果"
3. **没有语法强制** - 调用方可能忘记检查 null
4. **缺少函数式方法** - 无法链式调用处理

`Optional` 是 Java 8 引入的**容器对象**，专门用来优雅地处理"可能为空"的情况，让代码更安全、更清晰。

#### 5.3.2 Optional<String> extractFinalReport(OverAllState state) 从执行结果中提取最终报告文本

```java
* 从执行结果中提取最终报告文本
     *
     * @param state 执行状态
     * @return 报告文本（如果存在）
     */
    public Optional<String> extractFinalReport(OverAllState state) {
        logger.info("开始提取最终报告...");

        // 提取 Planner 最终输出（包含完整的告警分析报告）
        Optional<AssistantMessage> plannerFinalOutput = state.value("planner_plan")
                .filter(AssistantMessage.class::isInstance)
                .map(AssistantMessage.class::cast);

        if (plannerFinalOutput.isPresent()) {
            String reportText = plannerFinalOutput.get().getText();
            logger.info("成功提取到 Planner 最终报告，长度: {}", reportText.length());
            return Optional.of(reportText);
        } else {
            logger.warn("未能提取到 Planner 最终报告");
            return Optional.empty();
        }
    }
```

**分步解析：**

1. `state.value("planner_plan")`

- 从状态对象中获取键为 `"planner_plan"` 的值
- 返回 `Optional`（因为状态中存储的值可以是任意类型）
- 这个键正是之前 `buildPlannerAgent()` 中设置的 `.outputKey("planner_plan")`

2. `.filter(AssistantMessage.class::isInstance)`

- 过滤：只保留 `AssistantMessage` 类型的对象
- 如果不是 `AssistantMessage` 类型，过滤掉（返回空的Optional）
- 这是一种**类型安全检查**

3. `.map(AssistantMessage.class::cast)`

- 将 `Object` 类型安全地转换为 `AssistantMessage` 类型
- 因为前面已经过滤过了，这里转换是安全的

他已经保留了`AssistantMessage`对象了，说明里面都只剩这个类型的对象了，为什么还要再将object转化为`AssistantMessage`?

答：虽然 `message` 是 `AssistantMessage` 类型，但放入 `Map` 后，**编译器只记住它是 Object**。

**编译器视角 vs 运行时视角**

```java
// ❌ 编译器会报错
Object value = state.value("planner_plan").get();
String text = value.getText();  // 编译错误！
// 编译器：Object 类没有 getText() 方法
```



```java
// ✅ 需要告诉编译器真实类型
Object value = state.value("planner_plan").get();
AssistantMessage msg = (AssistantMessage) value;  // 强制类型转换
String text = msg.getText();  // 编译通过
```

#### 5.3.3 梳理运维数据流转过程

1. **Supervisor 调用流程**

```java
// 调用 Supervisor Agent
supervisorAgent.invoke(taskPrompt)
    ↓
// Supervisor 内部调度子 Agent
// 1. 先调用 Planner Agent
// 2. 根据结果决定是否调用 Executor Agent
// 3. 可能循环调用
```

2. **Planner Agent 执行后保存的数据格式**

**不是直接保存 Map 格式**，而是保存 **`AssistantMessage` 对象**，但这个对象被存储在 **Supervisor 的上下文 Map 中**。

```java
// ReactAgent 内部执行流程（简化）
public class ReactAgent {
    private String outputKey;  // = "planner_plan"
    
    public AssistantMessage execute(AgentContext context) {
        // 1. Agent 执行，生成输出
        AssistantMessage result = AssistantMessage.builder()
            .text("这是 Planner 生成的报告内容...")
            .role("assistant")
            .timestamp(System.currentTimeMillis())
            .build();
        
        // 2. 将结果存储到上下文中，使用 outputKey 作为键
        context.put(outputKey, result);  // ← 存储 AssistantMessage 对象
        
        return result;
    }
}
```

3. **Supervisor 的上下文结构**

```java
// Supervisor 的上下文实际上是一个 Map
public class AgentContext {
    private Map<String, Object> data = new HashMap<>();  // ← 这里是 Map
    
    public void put(String key, Object value) {
        data.put(key, value);
    }
    
    public Optional<Object> value(String key) {
        return Optional.ofNullable(data.get(key));
    }
}
```

4. **完整的数据存储过程**

```java
// 步骤1: Supervisor 创建上下文
AgentContext context = new AgentContext();  // 内部有 Map<String, Object>

// 步骤2: Supervisor 调用 Planner Agent
plannerAgent.execute(context);
    ↓
    // Planner Agent 内部执行
    AssistantMessage plannerOutput = AssistantMessage.builder()
        .text("告警分析报告...")
        .build();
    ↓
    // 存储到上下文的 Map 中
    context.put("planner_plan", plannerOutput);  // key="planner_plan", value=AssistantMessage对象

// 步骤3: Supervisor 调用 Executor Agent
executorAgent.execute(context);
    ↓
    // Executor Agent 可以从上下文读取 Planner 的输出
    Optional<Object> plannerData = context.value("planner_plan");  // 取出的是 AssistantMessage 对象
    ↓
    // Executor 执行后也存储自己的结果
    context.put("executor_result", executorOutput);  // 也是 AssistantMessage 对象

// 步骤4: 最终整个上下文返回给调用方
return context;  // 这个 context 就是 OverAllState
```

5. **数据存储结构图**

```java
OverAllState (实际上是 AgentContext)
│
└── data: Map<String, Object>
    │
    ├── "planner_plan" → AssistantMessage {
    │                       text: "告警分析报告...",
    │                       role: "assistant",
    │                       timestamp: 1234567890
    │                   }
    │
    ├── "executor_result" → AssistantMessage {
    │                        text: "已查询CPU指标...",
    │                        role: "assistant"
    │                      }
    │
    └── 其他可能的键值对...
```

**Planner 保存的数据不是 Map 格式。**

具体来说：

1. **Planner 保存的是 `AssistantMessage` 对象**（不是 Map）
2. **但这个对象被存储在 Supervisor 的上下文的 Map 中**
3. **Map 的键是 `"planner_plan"`，值是 `AssistantMessage` 对象**
