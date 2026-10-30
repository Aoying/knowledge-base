---
知识点: Java 异常体系
层级: 基础
适用版本: JDK 8+ / Spring Boot 2.7
更新日期: 2026-10-31
关联:
  - Java
  - Spring-事务
  - 接口设计与文档
标签:
  - Java
  - 原理
  - 实践
---

# Java 异常体系

## 1. 核心概念

Java 的异常体系只有一棵树：`Throwable` 是根，下面分 `Error` 和 `Exception` 两大支。关键不是背 API，而是分清三件事：哪些是程序不该管的（Error）、哪些必须显式表态的（受检异常）、哪些是编码错误或业务失败信号的（非受检异常）。

```text
Throwable
├── Error                  # JVM/系统级，通常不捕获：OutOfMemoryError、StackOverflowError 等
└── Exception
    ├── RuntimeException   # 非受检（unchecked）：NullPointerException、IllegalArgumentException 等
    └── 其他 Exception      # 受检（checked），编译器强制处理：IOException、SQLException 等
```

- **Error**：虚拟机或系统层面的致命问题（内存溢出、栈溢出、类加载失败）。业务代码不捕获、不恢复，捕获了也处理不了。
- **受检异常（checked）**：继承 `Exception` 但不继承 `RuntimeException`，编译器强制二选一：`try-catch` 处理，或方法签名上 `throws` 声明。设计意图是「调用方有能力且有义务恢复」的失败，如文件不存在、网络超时。
- **非受检异常（unchecked）**：`RuntimeException` 及其子类，不强制处理。多由编程错误（空指针、参数非法、状态不对）或业务规则不通过触发。

实务结论：现代框架整体倒向非受检——JdbcTemplate 把 `SQLException` 全转成 `DataAccessException`（RuntimeException），因为绝大多数受检异常调用方能做的只有「记日志 + 往上抛」，强制声明只会污染签名。自己设计业务异常同样默认继承 `RuntimeException`，除非确实存在「调用方必须当场恢复」的场景。

## 2. 常见异常谱系速查

| 异常 | 典型触发 | 归类判断 |
|---|---|---|
| `NullPointerException` | 对 null 调方法/取字段 | 编码问题，修代码不是加 catch |
| `IllegalArgumentException` | 入参越界、格式不合 | 调用方给错了，快速失败 |
| `IllegalStateException` | 对象状态不满足调用前提 | 状态机校验不通过 |
| `IOException` / `SQLException` | 文件/网络/SQL 失败 | 受检，框架层一般转非受检 |

原则：JDK 已定义好的语义不要再造同义异常；空指针、参数非法这类直接抛标准异常，自定义异常只用于表达**业务语义**（见第 4 节）。

## 3. try-with-resources

实现了 `AutoCloseable` 的资源（连接、流、Statement 等）一律用 try-with-resources，不要手写 `finally` 关流——手写版本一旦 `close()` 本身抛异常，就会把 try 块里的原始异常盖掉，排查时看到的是假现场。try-with-resources 按声明逆序自动关闭：

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    // 业务代码
} catch (SQLException e) {
    throw new DataAccessException("查询失败", e);
}
```

- 关闭时抛出的异常不会丢，会挂在主异常的 `getSuppressed()` 里（压制异常），堆栈中显示为 `Suppressed:` 段。

## 4. 处理规范

**不吞异常**。最坏的 catch 是空块，其次是只 `e.printStackTrace()`（绕开日志框架、生产环境找不到）和 catch 后返回 null/默认值假装成功——失败被掩盖，下游拿脏数据继续跑，最后在无关的地方爆掉。不知道怎么处理就往上抛，不要就地「消化」。

**保留 cause 链**。转换异常时必须把原异常作为 cause 传进去（`throw new BizException(code, msg, e)`），只取 `e.getMessage()` 拼新异常等于亲手剪断堆栈，排障时根源那一层永远看不到。日志同理：用 `log.error("...", e)` 输出完整堆栈，不要只打 message。

**不要用异常控制正常流程**。异常有构建堆栈的成本，更重要的是语义：可预期的分支（查不到数据、状态不允许）用状态校验和返回值表达，不要靠抛/捕获异常来走。审批流里「单据已被处理」用状态机条件更新处理（见 [REST 接口设计](工程实践/接口设计与文档.md) 的幂等设计），不是抛异常让上层 catch 着玩。

**业务异常设计**。一个够用的业务异常只需要三样：业务码（与统一信封的业务码同源）、给人看的消息、cause：

```java
public class BizException extends RuntimeException {
    private final String code;
    public BizException(String code, String message) { super(message); this.code = code; }
    public BizException(String code, String message, Throwable cause) { super(message, cause); this.code = code; }
    public String getCode() { return code; }
}
```

- 业务码分段与 [REST 接口设计](工程实践/接口设计与文档.md) 的约定保持一致，不要每个接口自创一套；不要为每种业务失败建一个异常类，一个 `BizException` + 业务码足够。
- 给用户看的消息和给日志看的信息分开：message 写可展示文案，别把 SQL、内部类名塞进 message 透传到前端。

## 5. 全局异常处理：@RestControllerAdvice

接口层的异常出口只留一个：`@RestControllerAdvice` 统一收口成信封 `{code, msg, data, traceId}`（结构与分工见 [REST 接口设计](工程实践/接口设计与文档.md)）。Controller 不写 try-catch 拼错误响应，Service 只管抛。

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)   // 业务可预期失败：消息可透传，状态码按约定（如 422）
    public ResponseEntity<ApiResult<Void>> handleBiz(BizException e) {
        return ResponseEntity.status(422)
            .body(ApiResult.fail(e.getCode(), e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)   // 参数校验：返回字段级错误
    public ResponseEntity<ApiResult<Void>> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining("; "));
        return ResponseEntity.badRequest().body(ApiResult.fail("40001", msg));
    }

    @ExceptionHandler(Exception.class)      // 兜底：完整堆栈只进日志，前端只给兜底文案
    public ResponseEntity<ApiResult<Void>> handleUnknown(Exception e) {
        log.error("未预期异常", e);
        return ResponseEntity.status(500)
            .body(ApiResult.fail("50000", "系统异常，请稍后重试"));
    }
}
```

要点：业务异常消息可透传，系统异常只给兜底文案；参数校验异常（与 Bean Validation 配套）单独接；兜底 `Exception` 处理器兜住一切未预期异常，杜绝默认错误页和异常栈直接回给前端。

## 6. 疑难杂症

- **事务该回滚却提交了**：Spring 声明式事务默认只对 `RuntimeException` 和 `Error` 回滚。受检异常没配 `rollbackFor`、或异常被 catch 吞掉，事务会正常提交，出现「报错了但数据写进去了」。解法：`@Transactional(rollbackFor = Exception.class)`，且不要在事务方法里吞异常。机制见 [Spring 事务](开发语言/Java/Spring/Spring-事务.md)。
- **日志里只有一行异常信息没有堆栈**：写了 `log.error("失败: {}", e.getMessage())`。改成把异常对象作最后一个参数：`log.error("失败", e)`。
- **包装了三层还是看不到根源**：中间层转换异常时没传 cause。全局搜自定义异常的构造点逐个检查。
- **finally 里 return**：会吞掉 try/catch 里的异常和返回值，编译器只警告不报错，规范上直接禁止。

## 小技巧与冷知识

- 【小技巧】排查“日志里只有一行异常、没有堆栈”时，先看日志调用是不是把异常拼进了占位符：`log.error("失败: {}", e.getMessage())` 只输出字符串；SLF4J 里把异常对象作为最后一个参数（`log.error("失败", e)`）才会打印完整堆栈。这个坑容易被忽略，因为代码能编译、日志也有输出，只是根源信息被悄悄丢掉了。
- 【小技巧】多 catch 块要按“先子类后父类”排列，否则子类分支永远走不到且编译报错；JDK 7 起可以用多重 catch（`catch (IOException | SQLException e)`）把同一种处理逻辑的多个异常合并，省掉重复代码。注意多重 catch 里的异常类型不能有继承关系，否则编译器会报“替代异常已捕获”。
- 【冷知识】`finally` 块里写 `return` 会吞掉 try/catch 里的返回值甚至正在传播的异常：方法会按 finally 的 return 返回，异常就此消失。编译器只给警告不报错，所以只能靠规范和 Code Review 拦，线上查这种问题时往往要先怀疑 finally。
- 【小技巧】try-with-resources 的资源按声明的逆序关闭，先声明的后关闭；如果关闭时也抛异常，它不会盖掉主异常，而是挂在主异常的 `getSuppressed()` 上，堆栈里显示为 `Suppressed:` 段。排查“关流失败”时别只看主异常，suppressed 段里往往藏着真正的第二个故障点。
- 【冷知识】`Throwable` 有四个构造器，其中 `Throwable(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace)` 可以关掉堆栈填充和压制异常。JDK 内部少数对性能敏感的异常就用它来省掉填堆栈的开销；业务代码一般不需要，但理解它能解释为什么有的异常堆栈是空的。
- 【小技巧】自定义异常至少提供带 cause 的构造器，并在转换异常时把原异常传进去（`new BizException(code, msg, e)`），只取 `e.getMessage()` 拼新异常等于剪断 cause 链。排障时根源那一层永远看不到的案子，十有八九是某个中间层转换异常时丢了 cause。
- 【冷知识】受检异常和非受检异常的分界只看继承关系：继承 `RuntimeException` 的是非受检，其余 `Exception` 子类是受检；`Error` 及其子类同样是非受检，但那是给 JVM 用的。`throws` 声明只能声明受检异常的检查义务——方法签名上写 `throws RuntimeException` 不起任何强制作用，编译器不会要求调用方处理。
- 【小技巧】不要在循环里用 try-catch 做“逐条容错”时顺手把循环变量也吞掉：catch 里只记日志不记录当前处理的是哪条数据，出问题后根本定位不到是哪条记录触发的。正确做法是在日志里带上当前元素的关键标识（如主键、单号），再决定继续还是中断。
- 【冷知识】`StackOverflowError` 是 `Error` 不是 `Exception`，而且它发生时栈往往已经满了，catch 住它再想做复杂恢复（比如记日志、调方法）本身就可能再次触发栈溢出。正确姿势是修递归的终止条件或改迭代，而不是试图捕获它兜底。
- 【小技巧】异常的 `getMessage()` 可能返回 null：无参构造或只传 cause 的异常没有 message，日志模板里直接拼接 `e.getMessage()` 会打出 null。需要稳妥展示时用异常类名 + message 组合，或者直接把异常对象交给日志框架打印完整堆栈。

## 7. 面试与自测

- Error 和 Exception 的区别？哪些异常你会主动捕获，哪些不会？
- 受检异常的设计意图是什么？Spring 为什么把 SQLException 转成非受检？
- try-with-resources 比手写 finally 好在哪？什么是压制异常？
- Spring 事务为什么对受检异常不回滚？怎么改？

### 参考答案

1. 两者都是 `Throwable` 的分支：Error 是 JVM/系统级致命问题，如内存溢出、栈溢出、类加载失败，业务代码不捕获、不恢复，捕获了也处理不了；Exception 是程序可处理的异常。会主动捕获的是调用方有能力恢复的受检异常和业务异常（一般在统一异常处理层收口）；不会主动捕获的是 Error，以及空指针、参数非法这类编码错误——后者应修代码，而不是加 catch 掩盖。
2. 受检异常的设计意图是「调用方有能力且有义务恢复」的失败，如文件不存在、网络超时，因此编译器强制二选一：try-catch 处理，或在签名上 throws 声明。Spring/JdbcTemplate 把 `SQLException` 转成非受检的 `DataAccessException`，是因为绝大多数受检异常调用方能做的只有记日志 + 往上抛，强制声明只会污染签名；自己设计业务异常同样默认继承 `RuntimeException`，除非确实存在调用方必须当场恢复的场景。
3. try-with-resources 要求资源实现 `AutoCloseable`，按声明逆序自动关闭，不用手写 finally 关流；手写版本一旦 `close()` 本身抛异常，就会盖掉 try 块里的原始异常，排查时看到的是假现场。关闭时抛出的异常不会丢失，会挂在主异常的 `getSuppressed()` 里，堆栈中显示为 `Suppressed:` 段，即压制异常。
4. Spring 声明式事务默认只对 `RuntimeException` 和 `Error` 回滚，受检异常不在默认回滚范围内；若受检异常没配 `rollbackFor`，或异常在事务方法里被 catch 吞掉，事务就会正常提交，出现「报错了但数据写进去了」。改法是 `@Transactional(rollbackFor = Exception.class)`，且不要在事务方法里吞异常。

## 8. 关联知识

- [REST 接口设计](工程实践/接口设计与文档.md)：统一信封、业务码分段、状态码纪律——异常落到接口上的口径。
- [Spring 事务](开发语言/Java/Spring/Spring-事务.md)：事务回滚规则与异常类型的关系。
