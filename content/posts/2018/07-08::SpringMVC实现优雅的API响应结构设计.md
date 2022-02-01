+++
title = "SpringMVC实现优雅的API响应结构设计"
date = "2018-07-08 08:19:00"
url = "archives/292"
tags = ["Spring"]
categories = ["后端"]
+++

一个规范、易懂和优雅，以及结构清晰且易于理解的API响应结构，完全可以省去许多无意义的沟通和文档。

### 预览 ###

操作成功：

```json
{"succeed": true,"timestamp": 1525582485337}
```

操作成功：返回数据

```json
{
    "succeed": true,
    "result": {
        "users": [
            {"id": 1, "name": "name1"},
            {"id": 2, "name": "name2"}
        ]
    },
    "timestamp": 1525582485337
}
```

操作失败：

```json
{
    "succeed": false,
    "error": {
        "error_code": 5002,
        "error_reason": "illegal_argument_error",
        "error_description": "The String argument[1] must have length; it must not be null or empty"
    },
    "timestamp": 1525582485337
}
```

### 实现 ###

#### 定义 ResponseData ####

```java
@Data
@Accessors(chain = true)
public class ResponseData<R extends Object> implements Serializable {

  @JSONField(ordinal = 0)
  public boolean succeed() {
    return null == this.error;
  }

  @JSONField(ordinal = 1)
  private R result;
  @JSONField(ordinal = 2)
  private Error error;

  @JSONField(ordinal = 3)
  public long getTimestamp() {
    return System.currentTimeMillis();
  }

  public static <R> ResponseData<R> withSucceed(Boolean succeed) {
    return ResponseData.withSucceed(succeed, null);
  }

  public static <R> ResponseData<R> withSucceed(Boolean succeed, String ifFailureDescription) {
    return ResponseData.withSucceed(succeed, null, ifFailureDescription);
  }

  public static <R> ResponseData<R> withSucceed(Boolean succeed, ResponseError ifFailureError,
      String ifFailureDescription) {
    // succeed ? 操作成功 : 操作失败，请重试。
    return succeed ? ResponseData.ok()
        : ResponseData.failure(
            null != ifFailureError ? ifFailureError : ResponseError.BUSINESS_ERROR,
            null != ifFailureDescription ? ifFailureDescription : "操作失败，请重试。");
  }

  public static <R> ResponseData<R> ok() {
    return ResponseData.ok(null);
  }

  public static <R> ResponseData<R> ok(R result) {
    return new ResponseData<R>().setResult(result);
  }

  /**
   * 未知错误 (error_code = -1).
   *
   * @return
   * <pre>
   *     {
   *         succeed: false,
   *         timestamp: 1525555307441,
   *         error: {
   *             error_code: -1,
   *             error_reason: "unknown_error",
   *             error_description: "未知错误"
   *         }
   *     }
   * </pre>
   */
  public static <R> ResponseData<R> failure() {
    return ResponseData.failure(ResponseError.UNKNOWN_ERROR);
  }

  public static <R> ResponseData<R> failure(String message) {
    return ResponseData.failure(ResponseError.BUSINESS_ERROR, message);
  }

  /**
   * 系统级别的异常 (error_code = 1000).
   *
   * @return
   * <pre>
   *     {
   *         succeed: false,
   *         timestamp: 1525555307441,
   *         error: {
   *             error_code: 1000,
   *             error_reason: "server_error",
   *             error_description: "服务器内部异常[java.lang.NullPointerException]"
   *         }
   *     }
   * </pre>
   */
  public static <R> ResponseData<R> failure(@NonNull Throwable throwable) {
    ResponseError error = ResponseError.SERVER_ERROR;
    String throwMessage = throwable.getMessage();
    String description = String.format("%s[%s]",
        null != throwMessage ? throwMessage : error.getDescription(),
        throwable.getClass().getTypeName());
    return ResponseData.failure(error.getCode(), error.getReason(), description);
  }


  /**
   * 使用系统定义的错误消息.
   *
   * @see ResponseData#failure(ResponseError, String)
   */
  public static <R> ResponseData<R> failure(ResponseError error) {
    return ResponseData.failure(error, null);
  }

  /**
   * 使用系统定义的错误消息.
   *
   * @param error 错误枚举
   * @param newDescription 覆盖默认的消息提示
   * @return ResponseData
   */
  public static <R> ResponseData<R> failure(@NonNull ResponseError error, String newDescription) {
    return ResponseData.failure(error.getCode(), error.getReason(),
        null != newDescription ? newDescription : error.getDescription());
  }

  /**
   * 自定义错误消息.
   *
   * @return
   * <pre>
   *     {
   *         succeed: false,
   *         timestamp: 1525555307441,
   *         error: {
   *             error_code: errorCode,
   *             error_reason: errorReason,
   *             error_description: errorDescription
   *         }
   *     }
   * </pre>
   */
  public static <R> ResponseData<R> failure(int errorCode, @NonNull String errorReason,
      @NonNull String errorDescription) {
    return new ResponseData().setError(new Error(errorCode, errorReason, errorDescription));
  }

  @Override
  public String toString() {
    return JSON.toJSONString(this, true);
  }

  @Data
  @AllArgsConstructor
  @Accessors(chain = true)
  private static class Error {

    @JSONField(name = "error_code", ordinal = 0)
    private int errorCode;
    @JSONField(name = "error_reason", ordinal = 1)
    private String errorReason;
    @JSONField(name = "error_description", ordinal = 2)
    private String errorDescription;
  }
}
```

#### 定义 BusinessException ####

```java
/**
 * 业务异常, 抛出后最终由 SpringMVC 拦截器统一处理为通用异常信息格式 JSON 并返回;
 */
@Data
public class CloudlyResponseBusinessException extends CloudlyRuntimeException {

  private final static ResponseError UNKNOWN_ERROR = ResponseError.UNKNOWN_ERROR;

  private int code = UNKNOWN_ERROR.getCode();
  private String reason = UNKNOWN_ERROR.getReason();
  private String description = UNKNOWN_ERROR.getDescription();

  /**
   * 未知错误
   *
   * @see ResponseData#failure()
   */
  public CloudlyResponseBusinessException() {
  }

  /**
   * 指定系统定义的错误, 错误消息使用异常信息中携带的消息
   *
   * @see ResponseData#failure(Throwable)
   */
  public CloudlyResponseBusinessException(ResponseError error, Throwable cause) {
    super(cause);
    this.init(error);
    this.description = String.format("%s[%s]", cause.getMessage(), cause.getClass().getTypeName());
  }


  /**
   * 指定系统定义的错误, 但指定了新的错误消息.
   *
   * @see ResponseData#failure(ResponseError, String)
   */
  public CloudlyResponseBusinessException(ResponseError error, String newDescription) {
    this.init(error);
    this.description = newDescription;
  }

  /**
   * 业务异常，自己指定消息
   */
  public CloudlyResponseBusinessException(String description) {
    this(ResponseError.BUSINESS_ERROR, description);
  }

  /**
   * 使用系统定义的错误
   *
   * @see ResponseData#failure(ResponseError)
   */
  public CloudlyResponseBusinessException(ResponseError error) {
    this.init(error);
  }

  /**
   * server_error, cause.getMessage();
   *
   * @see ResponseData#failure(Throwable)
   */
  public CloudlyResponseBusinessException(Throwable cause) {
    this(null, cause);
  }

  /**
   * 自定义错误消息
   *
   * @see ResponseData#failure(int, String, String)
   */
  public CloudlyResponseBusinessException(int code, String reason, String description) {
    this.init(code, reason, description);
  }

  private void init(ResponseError error) {
    this.init(error.getCode(), error.getReason(), error.getDescription());
  }

  private void init(int code, String reason, String description) {
    this.code = code;
    this.reason = reason;
    this.description = description;
  }
}
```

#### @ExceptionHandler: 异常拦截处理 ####

```java
@Slf4j
@ResponseBody
@ControllerAdvice
public class GlobalExceptionMessageHandlerConfig {

  @ExceptionHandler(Exception.class)
  public ResponseEntity<ResponseData> handlerException(Exception ex, HttpServletRequest request) {
    String requestInfo = String
        .format("HttpRequest -> <URI: %s, Method: %s, QueryString: %s, body: %s>, ", //
            request.getRequestURI(),//
            request.getMethod(),//
            request.getQueryString(),//
            Mvcs.getBody(request));
    // 参数错误 (HttpStatus: 400)
    if (ex instanceof IllegalArgumentException) {
      log.warn("{}ILLEGAL_ARGUMENT_ERROR: {}", requestInfo, ExceptionUtil.getMessage(ex));
      return ResponseEntity.badRequest()
          .body(ResponseData.failure(ResponseError.ILLEGAL_ARGUMENT_ERROR, ex.getMessage()));
    }
    // 业务异常 (HttpStatus: 200)
    if (ex instanceof CloudlyResponseBusinessException) {
      CloudlyResponseBusinessException throwable = (CloudlyResponseBusinessException) ex;

      if (!StrUtil.equals(ResponseError.SERVER_ERROR.getReason(), throwable.getReason()) &&
          !StrUtil.equals(ResponseError.UNKNOWN_ERROR.getReason(), throwable.getReason())) {
        log.warn("{}{}", requestInfo, throwable.toString());
        return ResponseEntity.ok(ResponseData.failure(
            throwable.getCode(), throwable.getReason(), throwable.getDescription()));
      }
    }
    // 服务器内部错误 (HttpStatus: 500), 打印堆栈、发送报警日志。
    log.error("{}SERVER_ERROR: {}", requestInfo, ex.getMessage(), ex);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(ResponseData.failure(ex));
  }
}
```

关于 **httpStatus** 的设定，建议如下：

| error                  | error_reason | http_status |
|------------------------|--------------|-------------|
| illegal_argument_error | 参数错误         | 400         |
| unknown_error          | 未知错误         | 500         |
| server_error           | 服务器内部异常      | 500         |
| xxx                    | 其他           | 200         |


### 使用 ###

#### JSONEntity ####

```bash
@GetMapping("/user/{id}")
public ResponseData<?> user(@PathVariable Integer id) {
    User user = find(id);

    if (null == user) {
        return ResponseData.failure(ResponseError.USER_NOT_FOUND);
    }
    return ResponseData.ok(ImmutableMap.of("user": user));
}

-->
{
    "succeed": true,
    "result": {
        "user": {"id": 1, "name": "user1"}
    },
    "timestamp": 1525582485337
}

{
    "succeed": false,
    "error": {
        "error_code": 10086,
        "error_reason": "user_not_found",
        "error_description": "没有找到用户 ##user1"
    },
    "timestamp": 1525582485337
}
```

#### Assert: 参数检查 ####

```bash
Assert.noEmpty(name, "用户名不能为空");

// 或者手动抛出IllegalArgumentException异常
if (StringUtil.isEmpty(name)) {
    throw new IllegalArgumentException("用户名不能为空");
}

-->
{
    "succeed": false,
    "error": {
        "error_code": 5002,
        "error_reason": "illegal_argument_error",
        "error_description": "用户名不能为空"
    },
    "timestamp": 1525582485337
}
```

#### Exception: 抛出业务异常 ####

```bash
// 使用系统定义的错误
throw new CloudlyResponseBusinessException(ResponseError.USER_NOT_FOUND)

// 指定系统定义的错误, 错误消息使用异常信息中携带的消息
throw new CloudlyResponseBusinessException(ResponseError.USER_NOT_FOUND, ex);

// 指定系统定义的错误, 但指定了新的错误消息.
throw new CloudlyResponseBusinessException(ResponseError.USER_NOT_FOUND, "用户 XXX 没有找到");

// 手动抛出服务器异常 (SERVER_ERROR)
throw new CloudlyResponseBusinessException(ex);

// 抛出一个未知的异常 (UNKNOWN_ERROR)
throw new CloudlyResponseBusinessException();

// 自定义错误消息
thow new CloudlyResponseBusinessException(10086, "user_email_exists", "用户邮箱已经存在了");

{
    "succeed": false,
    "error": {
        "error_code": 10086,
        "error_reason": "user_email_exists",
        "error_description": "用户邮箱已经存在了"
    },
    "timestamp": 1525582485337
}
```

### 补充：提供可维护的 ErrorCode 列表 ###

```java
@GetMapping("/error_code")
public ResponseData<?> errors() {
    return ResponseData.ok(ImmutableMap.of(//
        "enums", ResponseError.values(),//
        "markdownText", ResponseError.toMarkdownTable(), //
        "jsonString", ResponseError.toJsonArrayString()));
}
```

最终，该接口会根据系统定义（ResponseError）的异常信息，返回 Markdown 文本或 JSON 字符串，前端解析后生成表格即可，就像下面这样：

| error_code | error_reason             | error_description |
|------------|--------------------------|-------------------|
| -1         | unknown_error            | 未知错误              |
| 5000       | server_error             | 服务器内部异常           |
| 5001       | illegal_argument_error   | 参数错误              |
| 5002       | json_serialization_error | JSON 序列化失败        |
| ...        | ...                      |
