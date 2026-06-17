---
share: true
dir: content/doc/redis-lua-serialization/
description: About the serialization problem caused by Redis when using Lua scripts
---

# Redis Lua 脚本“原子性失效”与参数序列化问题深度分析


## 1. 问题现象描述


在执行一段用于“报警去重”的 Lua 脚本时，出现了逻辑不一致的现象：

- **现象**：`red:alarm`（锁定标识）执行了 `SETNX` 成功存入，但 `red:state` 中的 `alarmed` 状态位却没有被更新为 `1`。
    
- **后果**：导致业务逻辑判断出现偏差，看似锁定了报警，但状态记录不完整。
    

---

## 2. 核心原因排查

### A. Redis 原子性的误区

**Redis 不支持事务回滚。** Lua 脚本的原子性指的是“执行期间不会被其他命令插入”。如果脚本在执行到一半时（例如第三行）抛出异常，**已经执行成功的命令（第一、二行）不会撤销**。

### B. 致命的参数序列化（"L" 后缀与引号）

通过 `MONITOR` 命令抓取发现，Java 端传递参数时，由于 `RedisTemplate` 配置了特定的序列化器（如 Jackson），导致参数在传输时发生了变形：

1. **Long 类型后缀**：`86400` 被序列化成了 `"86400L"`。
    
2. **多重引号**：字符串被序列化成了 `"\"86400\""`。
    

### C. Lua 脚本中断逻辑链

在脚本执行过程中：

1. `SETNX KEYS[2] '1'` 执行成功。
    
2. `redis.call('EXPIRE', KEYS[2], tonumber(ARGV[1]))` 执行。
    
3. **关键点**：由于 `ARGV[1]` 是 `"86400L"`，Lua 的 `tonumber()` 转换结果为 `nil`。
    
4. **报错**：`EXPIRE` 命令接收到 `nil` 参数，抛出 `ERR Lua redis lib command arguments must be strings or integers` 异常。
    
5. **中断**：脚本在 `EXPIRE` 处崩溃，后面的 `HSET` 语句被跳过。
    

---

## 3. 解决方案

### 第一步：Java 端规范传参

避免使用容易产生歧义的 `Long` 或手动 `String.valueOf()`，推荐使用 `Integer` 或纯数字。

Java

```
// 推荐传 Integer，避开 Long 序列化可能带 L 的坑
int expireSeconds = (int) TimeUnit.DAYS.toSeconds(1); 

redisTemplate.execute(
    new DefaultRedisScript<>(luaScript, Long.class),
    Arrays.asList(stateKey, alarmKey),
    expireSeconds
);
```

### 第二步：Lua 脚本健壮性加固

在脚本内部增加参数清洗逻辑，并调整执行顺序，确保状态先更新，过期时间后设置。

Lua

```
-- 1. 先清洗参数：确保 ARGV 是纯数字（防止引号或后缀干扰）
local raw_ttl = tostring(ARGV[1]):match('%d+')
local ttl = tonumber(raw_ttl)

if redis.call('HGET', KEYS[1], 'alarmed') ~= '1' then
    -- 2. 尝试占坑
    local setnxRes = redis.call('SETNX', KEYS[2], '1')
    
    if setnxRes == 1 then
        -- 3. 先更新状态位（保证业务完整）
        redis.call('HSET', KEYS[1], 'alarmed', '1')
        -- 4. 后设置过期时间
        if ttl then
            redis.call('EXPIRE', KEYS[2], ttl)
        end
        return 1
    end
end
return 0
```

### 第三步：清理存量脏数据

由于之前的“半成功”执行，Redis 中已经存在了一些**没有过期时间**的 `red:alarm` Key。这些 Key 会导致 `SETNX` 永远返回 0。

- **必须手动执行**：`redis-cli DEL [对应的AlarmKey]`，让逻辑重新触发。
    

---

## 4. 调试经验总结

- **善用 MONITOR**：`redis-cli MONITOR` 是排查 Lua 脚本最有效的手段，能直观看到 Lua 内部到底发出了什么命令。
    
- **参数敏感**：Redis Lua 对 `nil` 和 `string/int` 的类型要求极严，传参前一定要确认序列化后的真实模样。
    
- **防御性编程**：在 Lua 脚本中使用 `tonumber()` 和 `nil` 判断，可以防止单次报错导致整个业务流程崩盘。


在 Redis `MONITOR` 中看到的输出，是经过序列化后最终传输给 Redis 的 **协议字符串**。以下是不同情况下的输出对比：

---

### 1. 情况一：不加 `tonumber()` 时

当你在脚本里直接写 `redis.call('EXPIRE', KEYS[2], ARGV[1])` 时：

|**Java 传入类型**|**MONITOR 看到的输出**|**Redis 命令执行结果**|**原因分析**|
|---|---|---|---|
|**String**|`"\"86400\""`|**报错**|JSON 序列化器给字符串加了转义双引号，Redis 不认这个格式。|
|**Long**|`"86400L"`|**报错**|某些序列化器（如 Jackson）为了标记类型，在传输时带了 `L` 字母。|
|**Integer**|`"86400"`|**成功**|`Integer` 序列化最干净，通常不带任何干扰字符。|

**此时的风险：**

如果不加 `tonumber`，脚本会直接把带有双引号或 `L` 的“脏字符串”扔给 `EXPIRE` 命令。Redis 内部会抛出 `value is not an integer` 错误，导致脚本在 `SETNX` 之后中断，造成你之前遇到的“半成功”状态。

---

### 2. 情况二：加上 `tonumber()` 时

当你在脚本里写 `redis.call('EXPIRE', KEYS[2], tonumber(ARGV[1]))` 时：

|**Java 传入类型**|**MONITOR 看到的输出**|**Lua 内部转换结果**|**Redis 执行结果**|
|---|---|---|---|
|**String**|`"\"86400\""`|`nil`|**报错**|
|**Long**|`"86400L"`|`nil`|**报错**|
|**Integer**|`"86400"`|`86400` (数字)|**成功**|

**此时的差异：**

加了 `tonumber` 后，报错的主体变了。报错信息会变成 `ERR Lua redis lib command arguments must be strings or integers`。这是因为 Lua 把转换失败的 `nil` 传给了 Redis。




你现在遇到的不是 Lua 的问题，而是 **Spring `RedisTemplate` 执行 Lua 时对 ARGV 使用了默认 value serializer**，导致参数被 JSON/FastJson 序列化了。

所以：

- 传 `"86400"`，实际进 Lua 可能是 `"\"86400\""`，带 JSON 双引号。
- 传 `Long`，实际可能是 `"86400L"`。
- 传 `Integer` 正好没踩坑，但这只是碰巧，不是理想方案。

更推荐的做法是：**执行 Lua 脚本时显式指定参数序列化器为 `StringRedisSerializer`**。

示例：

```
DefaultRedisScript<Long> script = new DefaultRedisScript<>(luaScript, Long.class);

Long result = redisTemplate.execute(
        script,
        new StringRedisSerializer(),
        new GenericToStringSerializer<>(Long.class),
        Arrays.asList(stateKey, alarmKey),
        String.valueOf(TimeUnit.DAYS.toSeconds(1))
);
```

不过不同 Spring Data Redis 版本的重载方法略有差异，更常见写法是用这个重载：

```
Long result = redisTemplate.execute(
        script,
        RedisSerializer.string(),
        RedisSerializer.java(),
        Arrays.asList(stateKey, alarmKey),
        String.valueOf(TimeUnit.DAYS.toSeconds(1))
);
```

核心思想是：**ARGV 必须按普通字符串传给 Redis，不要走 JSON 序列化器。**

Lua 里也可以不 `tonumber`：

```
redis.call('EXPIRE', KEYS[2], ARGV[1])
```

`EXPIRE` 本身可以接收字符串形式的整数参数，所以 TTL 这种参数没必要先转数字。

如果数字很大，超过 `int`，也应该传字符串：

```
String ttlSeconds = String.valueOf(veryLargeLong);
```

Lua 中：

```
redis.call('EXPIRE', alarmKey, ARGV[1])
```

这样不会有 `int` 溢出问题，也不会有 `Long` 被序列化成 `86400L` 的问题。

需要注意：如果你在 Lua 里要对大整数做数学运算或大小比较，再用 `tonumber` 就要小心。Redis Lua 的 number 有精度限制，特别大的整数可能丢精度。TTL 这种只是传给 Redis 命令的参数，最好保持字符串，不转数字。

所以结论是：

**不要靠传 `int` 规避序列化问题，应该显式指定 Lua 参数序列化方式，统一把 ARGV 当字符串传入；TTL 在 Lua 中直接传给 `EXPIRE`/`SET NX EX`，无需 `tonumber`。**