---
tags:
  - "#kotlin"
  - "#函数式编程"
---

> runCatching是Kotlin标准库中常用于异常处理的函数式工具，相比于传统try-catch方法具有**可读性强**、**支持函数式编程的链式调用**、**防止嵌套地狱**等优点。

## 常用链式操作

### `.getOrNull()`:成功时返回值，失败时返回null

Android中Room数据库读取，失败不崩溃:

```kotlin
fun getUserOrNull(userId: Int): User? {
    return runCatching {
        userDao.getUser(userId)
    }.getOrNull()
}

```

### `.getOrElse{}`:失败时将该方法的结果作为返回值

Android中网络请求+本地缓存降级业务逻辑:

```kotlin
fun loadUserProfile(userId: String): UserProfile {
    return runCatching {
        remoteDataSource.fetchUserProfile(userId) // 尝试从网络获取
    }.getOrElse {
        // 如果失败（如超时、无网络），就从本地缓存获取
        localCache.getUserProfile(userId)
    }
}

```

### `.getOrThrow()`:失败重新抛出异常

Spring Boot中获取用户，读取失败则直接抛出异常：

```kotlin
val user = runCatching {
    userService.getById(id)
}.getOrThrow()  // 如果获取不到，就直接抛出异常

```

### `.getOrDefault()`:失败则返回默认值

读取配置文件，读取失败则使用默认值:

```kotlin
fun loadConfig(): AppConfig {
    return runCatching {
        val json = File("config.json").readText()
        parseConfig(json)
    }.getOrDefault(AppConfig.default())
}

```

### `.onSuccess {}`:成功时执行某操作，返回自身用于链式调用

当查到用户后进行输出，不影响调用的结果:

```kotlin
val result = runCatching {
    userService.getUserById(123)
}.onSuccess {
    println("查到了用户：${it.name}")
}

val user = result.getOrNull() // 继续拿来用

```

### `.onFailure {}`:失败时执行某操作，返回自身用于链式调用

Spring Boot中发送验证码，失败则记录到日志:

```kotlin
fun sendSmsCode(phone: String): Boolean {
    return runCatching {
        smsClient.send(phone, "验证码是1234")
    }.onFailure {
        logger.warn("发送短信失败: ${it.message}")
    }.isSuccess
}

```

### `.fold()`:根据执行结果执行不同分支

Spring Boot中统一包装API响应:

```kotlin
@GetMapping("/{id}")
fun getUserById(@PathVariable id: Long): ResponseEntity<ApiResult<UserResponse>> {
    return runCatching {
        userService.getById(id)
    }.fold(
        onSuccess = {
            val result = ApiResult.ok(it.toResponse())
            ResponseEntity.ok(result)
        },
        onFailure = {
            val err = ApiResult.notFound("用户不存在", null)
            ResponseEntity.status(HttpStatus.NOT_FOUND).body(err)
        }
    )
}

```