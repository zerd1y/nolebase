# Redis
## 安装配置

>[!tip] 注意
> 基于 Windows + WSL

### 第一步：安装 wsl
### 第二步：安装 Redis  

1. 打开 wsl
2. [Install Redis on Windows](https://redis.io/docs/latest/operate/oss_and_stack/install/archive/install-redis/install-redis-on-windows/)
### 第三步：配置

用 vim 编辑器进入配置文件

```Shell
sudo vim /etc/redis/redis.conf
```

>[!tip]
>修改一下配置：
>1. 允许访问的地址，默认是127.0.0.1，会导致只能在本地访问。修改为0.0.0.0则可以在任意IP访问，生产环境不要设置为0.0.0.0
>```shell
>bind 0.0.0.0
>```
>2. 守护进程，修改为yes后即可后台运行
>```shell
>daemonize yes 
>```
>3. 密码，设置后访问Redis必须输入密码
>```shell
>requirepass 101024
>```
>4. 日志文件，默认为空，不记录日志，可以指定日志文件名
>```shell
>logfile "redis.log"
>```

设置开机自启（只有在打开 WSL 时启动Redis）

>[!tip]
>1. 进入配置文件
>```Shell
>nano ~/.bashrc
>```
>2. 添加 Redis 启动命令
>```Shell
># Start Redis server if not already running
>if ! pgrep -x redis-server > /dev/null; then
>sudo service redis-server start
>fi
>```
>3. 保存并退出
>`Ctrl+O` -> `Enter` -> `Ctrl+X`
>
>4. 生效命令
>```Shell
>source ~/.bashrc

检查 Redis 是否运行：

```Shell
sudo service redis-server status
```

## Redis 客户端

### CLI

进入 Windows WSL2：

```shell
redis-cli -a 101024
```

### GUI

下载 Another Redis Desktop Manager

获取 WSL 动态 IP（**每次启动 WSL 都会更新 IP**）

```Shell
ip addr show eth0 | grep -oP 'inet\s+\K[\d.]+'
```

进入客户端连接

## 序列化

#TODO
## 连接池

#TODO

## 缓存
![](assets/image-20220523214414123.png)
### 添加缓存
![](assets/1653322097736.png)
Service 层代码：
```java
@Override  
public Result queryById(Long id) {  
    String key = CACHE_SHOP_KEY + id;  
    String shopJson = stringRedisTemplate.opsForValue().get(key);  
    if (StrUtil.isNotBlank(shopJson)) {  
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);  
        return Result.ok(shop);  
    }  
    Shop shop = getById(id);  
    if (shop == null) {  
        return Result.fail("商铺不存在");  
    }  
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop));  
    return Result.ok(shop);  
}
```

### 缓存更新 

![](assets/1653322506393.png)
>高一致性需求：采用先**操作数据库，再删除缓存**的方法
![](assets/1653323595206.png)

Service 层代码：
```java
@Override  
@Transactional  
public Result update(Shop shop) {  
    Long id = shop.getId();  
    if (id == null) {  
        return Result.fail("店铺id不能为空");  
    }  
    updateById(shop);  
    stringRedisTemplate.delete(CACHE_SHOP_KEY + id);  
    return Result.ok();  
}
```

### 缓存穿透

**缓存穿透**指的是当用户或攻击者请求一个既不在缓存中也不在数据库中的数据时，每次请求都会穿透缓存，直接打到数据库上。如果这种请求量很大，数据库就会不堪重负。

1. 缓存空对象
2. 布隆过滤
![](assets/1653326156516.png)
![](assets/1653327124561.png)

```java
@Override  
public Result queryById(Long id) {  
    String key = CACHE_SHOP_KEY + id;  
    String shopJson = stringRedisTemplate.opsForValue().get(key);  
    if (StrUtil.isNotBlank(shopJson)) {  
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);  
        return Result.ok(shop);  
    }  
    if (shopJson != null) {  
        return Result.fail("店铺信息不存在");  
    }  
    Shop shop = getById(id);  
    if (shop == null) {  
        stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES  
        );  
        return Result.fail("店铺不信息存在");  
    }  
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);  
    return Result.ok(shop);  
}
```
- `StrUtil.isNotBlank(shopJson)`：检查字符串是否**非空（not null）**、**非空白（not empty）且不只包含空格符**（存在）
- `if (shopJson != null)`：`shopJson` 要么是空字符串 `""`，要么只包含空格（不存在但已缓存）
- `if (shop == null)`：数据库查询结果也为 `null`，说明这个店铺确实不存在（不存在且未缓存）

### 缓存雪崩

**解决方案：**
- 给不同的Key的TTL添加随机值
- 利用Redis集群提高服务的可用性
- 给缓存业务添加降级限流策略
- 给业务添加多级缓存
![](assets/1653327884526.png)

>[!tip] 具体实现
>利用 SpringCloud 中的各种安全方案解决

### 缓存击穿

![](assets/1653328022622.png)

>解决方案一：互斥锁
![](assets/1653328288627.png)

>解决方案二：逻辑过期
![](assets/1653328663897.png)

>解决方案对比
![](assets/1653357522914.png)

一致性和可用性之间要做出抉择

>双重检查锁定（双检）

**先检查，后（加锁）再检查，最后（如果仍然需要）才重建缓存**

1. 第一次检查
	- **目的**：**减少锁的竞争，提高性能**
	- 解释：在大多数情况下，缓存数据是有效的
2. 获取锁：
	- **目的**：**确保只有一个线程能进入关键区，避免重复操作和数据不一致**
	- 解释： 第一次检查发现数据已逻辑过期时，才尝试获取分布式锁
3. 第二次检查（加锁后）
	- **目的**：**防止重复重建，解决“双重检查问题”**
	- 解释：A 异步重建后释放锁，B 可能会获得锁，如果不进行第二次检查，就会无条件开始又一次缓存重建，造成资源浪费
4. 重建缓存（如果仍然需要）
	- **目的**：**更新过期数据**
	- 解释：经过两次检查，并确认数据确实需要更新时，才触发真正的缓存重建操作


>互斥锁

![](assets/1653357860001.png)
```java
@Override  
public Result queryById(Long id) {  
    Shop shop = queryWithMutex(id);  
    if (shop == null) {  
        return Result.fail("店铺不存在");  
    }  
    return Result.ok(shop);  
}  
  
public Shop queryWithMutex(Long id) {  
    String key = CACHE_SHOP_KEY + id;  
    // 1. 从Redis查询商铺缓存
    String shopJson = stringRedisTemplate.opsForValue().get(key); 
    // 2. 判断是否存在 
    if (StrUtil.isNotBlank(shopJson)) {  
	    // 3. 存在，直接返回
        return JSONUtil.toBean(shopJson, Shop.class);  
    }  
    // 判断命中的是否是空值
    if (shopJson != null) {  
	    // 返回一个错误信息
        return null;  
    }  
    // 4. 实现缓存重建
    String lockKey = "lock:shop:" + id;  
    Shop shop = null;  
    try {  
	    // 4.1 获取互斥锁
        boolean isLock = tryLock(lockKey);  
        // 4.2 判断是否获取成功
        if (!isLock) {  
	        // 4.3 失败，则休眠并重试
            Thread.sleep(50);  
            return queryWithMutex(id);  
        }  
        // 4.4 成功，进行双重检查
        // 再次从Redis查询商铺缓存
        shopJson = stringRedisTemplate.opsForValue().get(key);  
        // 再次判断是否存在
        if (StrUtil.isNotBlank(shopJson)) {  
	        // 存在，释放锁并返回
            return JSONUtil.toBean(shopJson, Shop.class);  
        }  
        // 再次判断命中的是否是空值（防止在这段时间内有其他线程写入了空值）
        if (shopJson != null) {  
	        // 返回一个错误信息
            return null;  
        }  
        // 4.5 不存在，根据id查询数据库
        shop = getById(id);  
        // 4.6 数据库中不存在，将空值写入Redis
        if (shop == null) {  
            stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES  
            );  
            return null;  
        }  
        // 4.7 数据库中存在，写入Redis
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);  
    } catch (InterruptedException e) {  
        throw new RuntimeException(e);  
    } finally {  
	    // 4.8 释放锁
        unlock(lockKey);  
    }  
    return shop;  
}  
  
private boolean tryLock(String key) {  
    Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);  
    return BooleanUtil.isTrue(flag);  
}  
  
private void unlock(String key) {  
    stringRedisTemplate.delete(key);  
}
```

>逻辑过期

![](assets/1653360308731.png)

```java
@Override  
public Result queryById(Long id) {  
    // 1. 互斥锁解决缓存击穿  
    //Shop shop = queryWithMutex(id);  
  
    // 2. 逻辑过期解决缓存击穿  
    Shop shop = queryWithLogicalExpire(id);  
    if (shop == null) {  
        return Result.fail("店铺不存在");  
    }  
    return Result.ok(shop);  
}
// 创建了一个固定大小为 10 的线程池:用于异步执行缓存重建任务
private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

public Shop queryWithLogicalExpire(Long id) {  
    String key = CACHE_SHOP_KEY + id;  
    // 1. 从 Redis 查询商铺缓存数据
    String shopJson = stringRedisTemplate.opsForValue().get(key); 
    // 2. 判断 Redis 中是否存在这个键的数据 
    if (StrUtil.isBlank(shopJson)) {  
        return null;  
    }  
    // 3. 如果 Redis 中有数据，则反序列化为 RedisData 对象
    RedisData redisData = JSONUtil.toBean(shopJson, RedisData.class);  
    // 4. 获取 RedisData 中的实际店铺数据和逻辑过期时间
    JSONObject data = (JSONObject) redisData.getData();  
    Shop shop = JSONUtil.toBean(data, Shop.class);  
    LocalDateTime expireTime = redisData.getExpireTime();  
    // 5. 判断当前缓存数据是否逻辑过期
    if (expireTime.isAfter(LocalDateTime.now())) {  
	    // 5.1 未过期：如果逻辑过期时间在当前时间之后，说明数据仍然有效
        return shop;  
    }  
    // 6. 已过期：如果数据已逻辑过期，需要尝试重建缓存
    String lockKey = LOCK_SHOP_KEY + id;  
    // 7. 尝试获取互斥锁（防止多个线程同时重建同一个缓存）
    boolean isLock = tryLock(lockKey);  
    // 8. 判断是否成功获取锁
    if (isLock) {  
	    // 8.1 成功获取锁：执行“双重检查”
        shopJson = stringRedisTemplate.opsForValue().get(key);  
        if (StrUtil.isNotBlank(shopJson)) {  
            redisData = JSONUtil.toBean(shopJson, RedisData.class);  
            if (redisData.getExpireTime().isAfter(LocalDateTime.now())) {  
                unlock(lockKey);  
                return JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);  
            }  
        }  
        // 8.2 如果双重检查后，缓存仍然是逻辑过期（或为空），则提交异步任务进行缓存重建
        CACHE_REBUILD_EXECUTOR.submit(() -> {  
            try {  
                saveShop2Redis(id, 20L);  
            } catch (Exception e) {  
                throw new RuntimeException(e);  
            } finally {  
                unlock(lockKey);  
            }  
        });  
    }  
    // 9. 返回旧数据：无论是否获取到锁，或者是否触发了异步重建，都会立即返回当前缓存中的 shop 对象。
    // 这保证了高可用性，用户不会因为缓存重建而等待。
    return shop;  
}

public void saveShop2Redis(Long id, Long expireSeconds) {  
    Shop shop = getById(id);  
    RedisData redisData = new RedisData();  
    redisData.setData(shop);  
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(expireSeconds));  
    stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, JSONUtil.toJsonStr(redisData));  
}

private boolean tryLock(String key) {  
    Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);  
    return BooleanUtil.isTrue(flag);  
}  
  
private void unlock(String key) {  
    stringRedisTemplate.delete(key);  
}
```

### 缓存穿透 vs 缓存击穿

>缓存穿透

**定义**
缓存穿透指的是查询一个**不存在的数据**，导致请求绕过缓存，直接访问数据库。由于这个数据永远不会被缓存，所以每次请求都会穿透缓存，直接到达数据库。

**发生原因**
通常是恶意攻击者通过大量查询不存在的数据，或者业务逻辑中存在一些罕见的查询条件导致。

**解决方案**
- 缓存空对象
- 布隆过滤

>缓存击穿

**定义**
缓存击穿指的是一个**热点数据**在缓存中**失效**的瞬间，大量的并发请求同时涌入，这些请求都无法从缓存中获取数据，从而全部打到数据库上。

**发生原因**
通常是由于某个**高并发访问的热点数据**的缓存**过期**了，而此时正好有大量请求访问这个数据。

**解决方案**
- 互斥锁
- 逻辑过期

>[!tip] 代码实现
>在设计和实现缓存策略时，通常会将**缓存穿透**和**缓存击穿**的解决方案**整合到一份代码中**，这样能更全面地提升系统的健壮性


## Lua 脚本

参考教程：[菜鸟教程](https://www.runoob.com/lua/lua-tutorial.html)
![](assets/1653393304844%201.png)


