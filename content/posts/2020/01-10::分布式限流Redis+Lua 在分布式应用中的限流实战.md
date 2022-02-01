+++
title = "分布式限流::Redis+Lua 在分布式应用中的限流实战"
date = "2020-01-10 17:01:00"
url = "archives/673"
tags = ["Java","限流"]
categories = ["后端"]
+++

### 概述 ###

作为分布式应用的三大法宝之一（缓存、降级、限流），限流系统尤其是对外开放系统中，显得尤其重要。限流的目的是通过对并发访问进行限速，一旦达到一定的速率就可以拒绝服务，从而避免业务高峰期因暴增的流量直接将服务器打死。

### 常见的限流算法 ###

 *  信号量
 *  计数器
 *  漏桶算法
 *  令牌桶算法

在这里，咱们主要研究两种比较常用的（ **计数器** 、 **令牌桶算法** ）

#### 计数器 ####

计数器的方案比较简单。比如限制1秒钟内请求数最多为10个，每当进来一个请求，则计数器+1。当计数器达到上限时，则触发限流。时间每经过1秒，则重置计数器。

```java
public class RateLimiter {
    private long updateTimeStamp;
    private int intervalMilliSeconds;
    private int maxPermits;
    private long storedPermits;

    public RateLimiter(int maxPermits) {
        this(maxPermits, 1);
    }

    public RateLimiter(int maxPermits, int intervalSeconds) {
        this.maxPermits = maxPermits;
        this.intervalMilliSeconds = intervalSeconds * 1000;
    }

    public synchronized Boolean acquire(int permits) {
        while (true) {
            long now = System.currentTimeMillis();
            if (now < updateTimeStamp + intervalMilliSeconds) {
                if (storedPermits + permits <= maxPermits) {
                    storedPermits += permits;
                    updateTimeStamp = now;
                    return true;
                }
                return false;
            } else {
                storedPermits = 0;
                updateTimeStamp = now;
            }
        }
    }
}
```

计数器存在一个很大的问题，在第1秒的后半时间内，涌入了大量流量，然后到第2秒的前半时间，又涌入了大量流量。由于从第1秒到第2秒，请求计数是清零的，所以在这瞬间的qps有可能超过系统的承载

#### 令牌桶算法 ####

令牌桶算法的原理是，系统以固定的速率往令牌桶中放入令牌；当请求进来时，则从桶中取走令牌；当桶中令牌为空时，触发限流。  
![131728\_d2223c6f\_955503][131728_d2223c6f_955503]

Guava中的RateLimiter提供了令牌桶算法的实现，我们可以直接使用：

```java
@Component
public class RateLimitInterceptor extends AbstractInterceptor {
    // 构建一个令牌桶（qps=100)
    private static final RateLimiter rateLimiter = RateLimiter.create(100.0d);

    @Override
    protected boolean preHandle(HttpServletRequest request) {
        if (!rateLimiter.tryAcquire()) {
            // 此处进行服务熔断处理，如返回错误码：业务繁忙，请稍后重试。
            return false; 
        }
        return true; // 放行
    }
}
```

##### Google Guava RateLimiter 原理分析 #####

 *  关于RateLimiter：

1.  每个acquire()方法如果必要的话会阻塞直到一个permit可用，然后消费它。获得permit以后不需要释放。
2.  RateLimiter在并发环境下使用是安全的：它将限制所有线程调用的总速率。注意，它不保证公平调用。  
    3。 RateLimiter在并发环境下使用是安全的：它将限制所有线程调用的总速率。注意，它不保证公平调用。Rate limiter（直译为：速度限制器）经常被用来限制一些物理或者逻辑资源的访问速率。这和java.util.concurrent.Semaphore正好形成对照。
3.  一个RateLimiter主要定义了发放permits的速率。如果没有额外的配置，permits将以固定的速度分配，单位是每秒多少permits。默认情况下，Permits将会被稳定的平缓的发放。
4.  可以配置一个RateLimiter有一个预热期，在此期间permits的发放速度每秒稳步增长直到到达稳定的速率

![132623\_145aa3e0\_955503][132623_145aa3e0_955503]

 *  SmoothBursty以稳定的速度生成permit
 *  SmoothWarmingUp是渐进式的生成，最终达到最大值趋于稳定  
    ![132757\_47f42b67\_955503][132757_47f42b67_955503]

###### 源码片段解读 ######

```java
public abstract class RateLimiter {

    /**
     * 用给定的吞吐量（“permits per second”）创建一个RateLimiter。
     * 通常是QPS
     */
    public static RateLimiter create(double permitsPerSecond) {
        return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
    }
    
    static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
        RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
        rateLimiter.setRate(permitsPerSecond);
        return rateLimiter;
    }
    
    /**
     * 用给定的吞吐量(QPS)和一个预热期创建一个RateLimiter
     */
    public static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit) {
        checkArgument(warmupPeriod >= 0, "warmupPeriod must not be negative: %s", warmupPeriod);
        return create(permitsPerSecond, warmupPeriod, unit, 3.0, SleepingStopwatch.createFromSystemTimer());
    }

    static RateLimiter create(
            double permitsPerSecond,
            long warmupPeriod,
            TimeUnit unit,
            double coldFactor,
            SleepingStopwatch stopwatch) {
        RateLimiter rateLimiter = new SmoothWarmingUp(stopwatch, warmupPeriod, unit, coldFactor);
        rateLimiter.setRate(permitsPerSecond);
        return rateLimiter;
    }

    private final SleepingStopwatch stopwatch;

    // 锁
    private volatile Object mutexDoNotUseDirectly;

    private Object mutex() {
        Object mutex = mutexDoNotUseDirectly;
        if (mutex == null) {
            synchronized (this) {
                mutex = mutexDoNotUseDirectly;
                if (mutex == null) {
                    mutexDoNotUseDirectly = mutex = new Object();
                }
            }
        }
        return mutex;
    }
    
    /**
     * 从RateLimiter中获取一个permit，阻塞直到请求可以获得为止
     * @return 休眠的时间，单位是秒，如果没有被限制则是0.0
     */
    public double acquire() {
        return acquire(1);
    }
  
    /**
     * 从RateLimiter中获取指定数量的permits，阻塞直到请求可以获得为止
     */
    public double acquire(int permits) {
        long microsToWait = reserve(permits);
        stopwatch.sleepMicrosUninterruptibly(microsToWait);
        return 1.0 * microsToWait / SECONDS.toMicros(1L);
    }
    
    /**
     * 预定给定数量的permits以备将来使用
     * 直到这些预定数量的permits可以被消费则返回逝去的微秒数
     */
    final long reserve(int permits) {
        checkPermits(permits);
        synchronized (mutex()) {
            return reserveAndGetWaitLength(permits, stopwatch.readMicros());
        }
    }
    
    private static void checkPermits(int permits) {
        checkArgument(permits > 0, "Requested permits (%s) must be positive", permits);
    }
    
    final long reserveAndGetWaitLength(int permits, long nowMicros) {
        long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
        return max(momentAvailable - nowMicros, 0);
    }
}


abstract class SmoothRateLimiter extends RateLimiter {
    
    /** The currently stored permits. */
    double storedPermits;

    /** The maximum number of stored permits. */
    double maxPermits;

    /**
     * The interval between two unit requests, at our stable rate. E.g., a stable rate of 5 permits
     * per second has a stable interval of 200ms.
     */
    double stableIntervalMicros;
    
    /**
     * The time when the next request (no matter its size) will be granted. After granting a request,
     * this is pushed further in the future. Large requests push this further than small requests.
     */
    private long nextFreeTicketMicros = 0L; // could be either in the past or future
    
    final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
        resync(nowMicros);
        long returnValue = nextFreeTicketMicros;
        double storedPermitsToSpend = min(requiredPermits, this.storedPermits);    // 本次可以获取到的permit数量
        double freshPermits = requiredPermits - storedPermitsToSpend;    // 差值，如果存储的permit大于本次需要的permit数量则此处是0，否则是一个正数
        long waitMicros =
            storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
                + (long) (freshPermits * stableIntervalMicros);    // 计算需要等待的时间（微秒）

        this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
        this.storedPermits -= storedPermitsToSpend;    // 减去本次消费的permit数
        return returnValue;
    }
    
    void resync(long nowMicros) {
        // if nextFreeTicket is in the past, resync to now
        if (nowMicros > nextFreeTicketMicros) {    // 表示当前可以获得permit
            double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();    // 计算这段时间可以生成多少个permit
            storedPermits = min(maxPermits, storedPermits + newPermits);    // 如果超过maxPermit，则取maxPermit，否则取存储的permit+新生成的permit
            nextFreeTicketMicros = nowMicros;    // 设置下一次可以获得permit的时间点为当前时间
        }
    }
}
```

Guava RateLimiter实现的令牌桶算法，不仅可以应对正常流量的限速，而且可以处理突发暴增的请求，实现平滑限流。

通过代码，我们可以看到它可以预消费：nextFreeTicketMicros表示下一次请求获得permits的最早时间。每次授权一个请求以后，这个值会向后推移，即向未来推移。因此，大的请求会比小的请求推得更。这里的大小指的是获取permit的数量。这个应该很好理解，因为上一次请求获取的permit数越多，那么下一次再获取授权时更待的时候会更长，反之，如果上一次获取的少，那么时间向后推移的就少，下一次获得许可的时间更短。

### Redis+Lua 实现分布式限流 ###

Guava虽好，但始终只是个工具包，难以应对现在的分布式应用（其只能在单节点上生效，因为一系列算法逻辑都是在内存中完成，部署多个节点后，就不好使了）

为了解决这个问题，我们可以模仿 Guava 的实现，将相关的数据存储到 Redis 中，然后读写操作都使用 Lua 脚本来完成（Redis是单线程的，直接在代码中一读一写，难以保证原子性）

#### 相关需求 ####

现有需求如下：某个 SaaS 平台，针对每个租户提供了开放接口，但是根据版本的不同，有不同的限制策略，其默认策略如下：

 *  每秒请求: 100次/s，即QPS=100。
 *  每天请求总量：100000次/天，每天清零。

由此可见，该系统限流需要同时进行两个维度限制，首先限制请求总量，然后再限制 QPS。

#### Lua 脚本的编写 ####

##### 维度一：基于计数器算法的单元时间内总请求数量限制 > counter\_limiter.lua #####

首先是每天的请求总量，这个很好办，在 Redis 中弄一个 Key，每天 0 点清零，每次请求后自增+1 即可。

```lua
-- 维度一：基于计数器算法的单元时间内总请求数量限制 > counter_limiter.lua
-- 基于计数器的限流器
local key = KEYS[1]
-- 时间窗口内最大并发数
local max_permits = tonumber(ARGV[1])
-- 窗口的间隔时间
local interval_seconds = tonumber(ARGV[2])
-- 获取的并发数
local permits = tonumber(ARGV[3])
local current_permits = tonumber(redis.call('get', key) or 0)
-- 超过最大并发数
if (current_permits + permits > max_permits) then
    return 0
end
-- 增加并发计数
redis.call('incrby', key, permits)
-- 新的时间窗，重设过期时间
if (current_permits == 0) then
    redis.call('expire', key, interval_seconds)
end
return 1
```

##### 维度二：基于令牌桶算法的秒级 QPS 限制 > smooth\_limiter.lua #####

```lua
--[[ 基于令牌桶的限流器 > 调用 acquire 方法： eval 'lua脚本' 3 '自定义的key' '最大存储的令牌数' '每秒钟产生的令牌数' '请求的令牌数' - 返回获取请求成功后，线程需要等待的微秒数 > 调用 tryAcquire 方法： eval 'lua脚本' 3 '自定义的key' '最大存储的令牌数' '每秒钟产生的令牌数' '请求的令牌数' '最大等待的微秒数' - 返回需要等待的微秒数，将该返回值与最大等待微秒数做比较，如果redis返回的值较大，则说明失败；反之则是成功，并根据返回值让线程等待。 ]]
local key = KEYS[1]
-- 最大存储的令牌数
local max_permits = tonumber(KEYS[2])
-- 每秒钟产生的令牌数
local permits_per_second = tonumber(KEYS[3])
-- 请求的令牌数
local required_permits = tonumber(ARGV[1])
-- 最大等待的微秒数
local timeout_micros = tonumber(ARGV[2])
-- 下次请求可以获取令牌的起始时间
local next_free_ticket_micros = tonumber(redis.call('hget', key, 'next_free_ticket_micros') or 0)
-- 当前时间
local time = redis.call('time')
local now_micros = tonumber(time[1]) * 1000000 + tonumber(time[2])
-- 查询获取令牌是否超时
if (timeout_micros ~= nil) then
    local micros_to_wait = next_free_ticket_micros - now_micros
    -- 不能获取到令牌，直接返回
    if (micros_to_wait > timeout_micros) then
        return micros_to_wait
    end
end
-- 当前存储的令牌数
local stored_permits = tonumber(redis.call('hget', key, 'stored_permits') or 0)
-- 添加令牌的时间间隔
local stable_interval_micros = 1000000 / permits_per_second
-- 补充令牌
if (now_micros > next_free_ticket_micros) then
    local new_permits = (now_micros - next_free_ticket_micros) / stable_interval_micros
    stored_permits = math.min(max_permits, stored_permits + new_permits)
    next_free_ticket_micros = now_micros
end
-- 消耗令牌
local stored_permits_to_spend = math.min(required_permits, stored_permits)
local fresh_permits = required_permits - stored_permits_to_spend
local wait_micros = fresh_permits * stable_interval_micros
redis.replicate_commands()
redis.call('hset', key, 'stored_permits', stored_permits - stored_permits_to_spend)
redis.call('hset', key, 'next_free_ticket_micros', next_free_ticket_micros + wait_micros)
-- 1小时后令牌桶过期
redis.call('expire', key, 60 * 60)
-- 返回需要等待的时间长度
return next_free_ticket_micros - now_micros
```

#### 封装自己的分布式 RateLimiter ####

![134552\_06b7dfb8\_955503][134552_06b7dfb8_955503]
首先，将两个lua脚本放在项目中，然后通过工具类去加载之，后续执行的时候需要派上用场

```java
import com.google.common.collect.Maps;
import org.springframework.core.io.ClassPathResource;

import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.util.Map;

public class LuaScriptUtils {
   private final static Map<String, String> LUA_SCRIPT_CACHED = Maps.newConcurrentMap();

   public static String getScript(String path) {
       if (LUA_SCRIPT_CACHED.containsKey(path)) {
           return LUA_SCRIPT_CACHED.get(path);
       }

       final ClassPathResource classPathResource = new ClassPathResource(path);
       try (
           InputStream inputStream = classPathResource.getInputStream();
           BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream))) {
           final StringBuilder stringBuilder = new StringBuilder();
           String line;
           while ((line = bufferedReader.readLine()) != null) {
               stringBuilder.append(line).append(System.lineSeparator());
           }
           LUA_SCRIPT_CACHED.put(path, stringBuilder.toString());
           return stringBuilder.toString();
       } catch (Throwable e) {
           e.printStackTrace();
       }
       return "";
   }
}
```


```java
public abstract class BudoRateLimiter {
   private final SleepingStopwatch stopwatch;
   protected Counter counter = null;

   private volatile Object mutexDoNotUseDirectly;

   private Object mutex() {
       Object mutex = mutexDoNotUseDirectly;
       if (mutex == null) {
           synchronized (this) {
               mutex = mutexDoNotUseDirectly;
               if (mutex == null) {
                   mutexDoNotUseDirectly = mutex = new Object();
               }
           }
       }
       return mutex;
   }

   BudoRateLimiter(SleepingStopwatch stopwatch) {
       this.stopwatch = checkNotNull(stopwatch);
   }

   public final void setRate(double permitsPerSecond) {
       checkArgument(permitsPerSecond > 0.0 && !Double.isNaN(permitsPerSecond), "rate must be positive");
       synchronized (mutex()) {
           doSetRate(permitsPerSecond);
       }
   }

   public final void setCounter(Counter counter) {
       this.counter = counter;
   }

   protected abstract void doSetRate(double permitsPerSecond);

   public final double getRate() {
       synchronized (mutex()) {
           return doGetRate();
       }
   }

   abstract double doGetRate();

   public double acquire() {
       return acquire(1);
   }

   public double acquire(int permits) {
       checkPermits(permits);
       long microToWait = waitMicros(permits);
       stopwatch.sleepMicrosUninterruptibly(microToWait);
       return 1.0 * microToWait / SECONDS.toMicros(1L);
   }

   public TryAcquireResponse tryAcquire(long timeout, TimeUnit unit) {
       return tryAcquire(1, timeout, unit);
   }

   public TryAcquireResponse tryAcquire(int permits) {
       return tryAcquire(permits, 0, MICROSECONDS);
   }

   public TryAcquireResponse tryAcquire() {
       return tryAcquire(1, 0, MICROSECONDS);
   }

   public TryAcquireResponse tryAcquire(int permits, long timeout, TimeUnit unit) {
       if (!this.tryAcquireWithCounter(permits)) {
           return TryAcquireResponse.COUNTER_LIMITED; // 计数器不通过
       }

       long timeoutMicros = max(unit.toMicros(timeout), 0);
       checkPermits(permits);
       long microsToWait = this.queryWaitMicros(permits, timeoutMicros);
       if (microsToWait > timeoutMicros) {
           return TryAcquireResponse.FREQUENCY_LIMITED; // 令牌桶无法取得令牌
       }
       stopwatch.sleepMicrosUninterruptibly(microsToWait);
       return TryAcquireResponse.SUCCESSFUL;
   }

   final long waitMicros(int permits) {
       long waitMicros = queryWaitMicros(permits, null);
       return max(waitMicros, 0);
   }

   abstract boolean tryAcquireWithCounter(int permits);

   abstract long queryWaitMicros(int permits, Long timeoutMicros);

   @Override
   public String toString() {
       String counterString = "";
       if (this.counter != null) {
           counterString = String.format(Locale.ROOT, ",counter=%d/%ds", //
               this.counter.getMaxPermits(), this.counter.getIntervalSeconds());
       }
       return String.format(Locale.ROOT, "RateLimiter[smooth=%3.1fqps%s]", getRate(), counterString);
   }

   public abstract static class SleepingStopwatch {
       protected  SleepingStopwatch() {}

       protected abstract long readMicros();

       protected abstract void sleepMicrosUninterruptibly(long micros);

       public static SleepingStopwatch createFromSystemTimer() {
           return new SleepingStopwatch() {
               final Stopwatch stopwatch = Stopwatch.createStarted();

               @Override
               protected long readMicros() {
                   return stopwatch.elapsed(MICROSECONDS);
               }

               @Override
               protected void sleepMicrosUninterruptibly(long micros) {
                   if (micros > 0) {
                       Uninterruptibles.sleepUninterruptibly(micros, MICROSECONDS);
                   }
               }
           };
       }
   }

   public static class Counter {
       private long maxPermits;
       private long intervalSeconds;

       public Counter(long maxPermits, long intervalSeconds) {
           this.maxPermits = maxPermits;
           this.intervalSeconds = intervalSeconds;
       }

       public long getMaxPermits() {
           return maxPermits;
       }

       public long getIntervalSeconds() {
           return intervalSeconds;
       }
   }

   public enum TryAcquireResponse {
       SUCCESSFUL,
       COUNTER_LIMITED,
       FREQUENCY_LIMITED
   }

   private static void checkPermits(int permits) {
       checkArgument(permits > 0, "Requested permits (%s) must be positive", permits);
   }

   protected abstract JedisPool getJedisPool();
}
```


```java
import com.google.common.collect.Lists;
import redis.clients.jedis.Jedis;

import java.util.List;

import static java.util.concurrent.TimeUnit.SECONDS;

public abstract class BudoSmoothRateLimiter extends BudoRateLimiter {
   protected String key;
   protected String smoothLimiterScript;
   protected String counterLimiterScript;
   protected double storedPermits = 0;
   protected double permitsPerSecond = 1;
   protected double maxPermits = 0;
   protected double stableIntervalMicros = 0;

   protected BudoSmoothRateLimiter(SleepingStopwatch stopwatch) {
       super(stopwatch);
   }

   @Override
   public void doSetRate(double permitsPerSecond) {
       double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
       this.stableIntervalMicros = stableIntervalMicros;
       this.doSetRate(permitsPerSecond, stableIntervalMicros);
       this.queryWaitMicros(0, null); // 初始化令牌桶
   }

   abstract void doSetRate(double permitsPerSecond, double stableIntervalMicros);

   @Override
   double doGetRate() {
       return SECONDS.toMicros(1L) / stableIntervalMicros;
   }

   @Override
   boolean tryAcquireWithCounter(int permits) {
       if (null != this.counter) {
           // @see src/main/resources/counter_limiter.lua
           final List<String> params = Lists.newArrayList(//
               String.format("%s:COUNTER", key), //
               String.valueOf(this.counter.getMaxPermits()), //
               String.valueOf(this.counter.getIntervalSeconds()), //
               String.valueOf(permits));

           try (Jedis jedis = this.getJedisPool().getResource()) {
               return (Long) jedis.eval(this.counterLimiterScript, 1, params.toArray(new String[0])) != 0;
           } catch (Throwable e) {
               e.printStackTrace();
           }
       }
       return true;
   }

   @Override
   long queryWaitMicros(int permits, Long timeoutMicros) {
       // @see src/main/resources/smooth_limiter.lua

       // 调用 acquire 方法
       final List<String> params = Lists.newArrayList(key, //
           String.valueOf(maxPermits), //
           String.valueOf(permitsPerSecond), //
           String.valueOf(permits));

       // 调用 tryAcquire 方法
       if (null != timeoutMicros) {
           params.add(String.valueOf(timeoutMicros));
       }

       try (Jedis jedis = this.getJedisPool().getResource()) {
           return (Long) jedis.eval(this.smoothLimiterScript, 3, params.toArray(new String[0]));
       } catch (Throwable e) {
           e.printStackTrace();
       }
       return 0;
   }
}
```


```java
public abstract class BudoSmoothBursty extends BudoSmoothRateLimiter {
   final double maxBurstSeconds;

   BudoSmoothBursty(String key, SleepingStopwatch stopwatch, double maxBurstSeconds) {
       super(stopwatch);
       this.key = key;
       this.maxBurstSeconds = maxBurstSeconds;
       this.smoothLimiterScript = LuaScriptUtils.getScript("smooth_limiter.lua");
       this.counterLimiterScript = LuaScriptUtils.getScript("counter_limiter.lua");
   }

   @Override
   void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
       double oldMaxPermits = this.maxPermits;
       this.permitsPerSecond = permitsPerSecond;
       maxPermits = maxBurstSeconds * permitsPerSecond;
       if (oldMaxPermits == Double.POSITIVE_INFINITY) {
           storedPermits = maxPermits;
       } else {
           storedPermits = (oldMaxPermits == 0.0) ? 0.0 : storedPermits * maxPermits / oldMaxPermits;
       }
   }
}
```


```java
import redis.clients.jedis.JedisPool;

public class BudoSmoothBurstyImpl extends BudoSmoothBursty {
   private JedisPool jedisPool;

   public BudoSmoothBurstyImpl(JedisPool jedisPool, String key, SleepingStopwatch stopwatch, double maxBurstSeconds) {
       super(key, stopwatch, maxBurstSeconds);
       this.jedisPool = jedisPool;
   }

   @Override
   protected JedisPool getJedisPool() {
       return this.jedisPool;
   }
}
```


```java
import redis.clients.jedis.JedisPool;

public class BudoRateLimiterFactory {
   private JedisPool jedisPool;

   public BudoRateLimiterFactory setJedisPool(JedisPool jedisPool) {
       this.jedisPool = jedisPool;
       return this;
   }

   public BudoRateLimiter build(String key, double permitsPerSecond) {
       return build(key, permitsPerSecond, null);
   }

   /**
    * 构建令牌桶
    *
    * @param key 令牌桶唯一标识
    * @param permitsPerSecond QPS
    * @param counter 计数器
    * @return org.budo.limiter.BudoRateLimiter
    */
   public BudoRateLimiter build(String key, double permitsPerSecond, BudoRateLimiter.Counter counter) {
       Assert.notNull(jedisPool, "jedisPool is null");

       BudoRateLimiter rateLimiter = new BudoSmoothBurstyImpl(//
           jedisPool, key, BudoRateLimiter.SleepingStopwatch.createFromSystemTimer(), 1.0);
       rateLimiter.setRate(permitsPerSecond);
       rateLimiter.setCounter(counter);
       return rateLimiter;
   }
}
```

#### 如何使用呢？ ####

1）将限流器工厂`BudoRateLimiterFactory`配置成 Bean；

```xml
<bean 
    class="xxx.limiter.BudoRateLimiterFactory"
    p:jedisPool-ref="jedisPool"
 />
 ```

2）在需要使用的地方，注入限流器工厂：

```java
@Resource
BudoRateLimiterFactory budoRateLimiterFactory;
```

3）使用限流器工厂构建令牌桶:

```java
// QPS=10 (单个维度，限制每秒的请求数量）
final BudoRateLimiter budoRateLimiter1 = budoRateLimiterFactory
    .build("redis-limiter:test01", 10.0d);

// 每60秒只能请求80次且QPS=10 (两个维度：1.限制单位时间内的总请求次数，2.限制每秒的请求数量)
final BudoRateLimiter budoRateLimiter2 = budoRateLimiterFactory
    .build("redis-limiter:test02", 10.0d, new BudoRateLimiter.Counter(80, 60));
```

4) 拿到令牌桶后，做进一步的使用，获取令牌：

```java
if (budoRateLimiter.tryAcquire()) {
   // 可以调用
} else {
   // 服务熔断处理
}
```

5）业务系统进行更进一步的封装，Interceptor、Annotation等...

#### 模拟小测试 ####

```java
public class BudoRateLimiterTest {

    public static void main(String[] args) throws InterruptedException {
        test01();
        test02();
    }

    private static void test02() throws InterruptedException {
        // 构建令牌桶，每分钟只能请求80次，且 qps=10
        final BudoRateLimiter budoRateLimiter = getBudoRateLimiterFactory()//
            .build("redis-limiter:test02", 10.0d, //
                new BudoRateLimiter.Counter(80, 60));
        System.out.println("创建令牌桶：" + budoRateLimiter);

        // 并发 100 消费令牌
        doAnything(budoRateLimiter);

        // 休眠 5s 后，继续并发 100 消费令牌
        System.out.println(budoRateLimiter + "###Thread.sleep(5000)...");
        Thread.sleep(5000);

        doAnything(budoRateLimiter);
    }

    private static void test01() throws InterruptedException {
        // 构建令牌桶，qps=10
        final BudoRateLimiter budoRateLimiter = getBudoRateLimiterFactory()//
            .build("redis-limiter:test01", 10.0d);
        System.out.println("创建令牌桶：" + budoRateLimiter);

        // 并发 100 消费令牌
        doAnything(budoRateLimiter);

        // 休眠 5s 后，继续并发 100 消费令牌
        System.out.println(budoRateLimiter + "###Thread.sleep(5000)...");
        Thread.sleep(5000);

        doAnything(budoRateLimiter);
    }

    static void doAnything(final BudoRateLimiter budoRateLimiter) throws InterruptedException {
        System.out.println(budoRateLimiter + "开始消费数据..");

        final CountDownLatch countDownLatch = new CountDownLatch(100);
        final AtomicInteger successCount = new AtomicInteger(0);

        // 100并发消费令牌
        for (int i = 0; i < 100; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    final String time = new SimpleDateFormat("hh:MM:ss").format(new Date());
                    final String name = Thread.currentThread().getName();

                    if (budoRateLimiter.tryAcquire() == BudoRateLimiter.TryAcquireResponse.SUCCESSFUL) {
                        successCount.getAndAdd(1);
                        System.out.println(budoRateLimiter + "<" + time + ">" + name + " is running..");
                    } else {
                        // System.out.println("[" + time + "]" + name + " fuse processing.."); // 熔断
                    }
                    countDownLatch.countDown();
                }
            }).start();
        }
        countDownLatch.await();
        System.out.println(budoRateLimiter + ", successCount: " + successCount.get());
    }

    private static JedisPool getJedisPool() {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        return new JedisPool(poolConfig, "192.168.137.104", 6379, 20000, "123456");
    }

    private static BudoRateLimiterFactory getBudoRateLimiterFactory() {
        BudoRateLimiterFactory budoRateLimiterFactory = new BudoRateLimiterFactory();
        budoRateLimiterFactory.setJedisPool(getJedisPool());
        return budoRateLimiterFactory;
    }
}
```

##### 执行结果 #####

```bash
创建令牌桶：RateLimiter[smooth=10.0qps]
RateLimiter[smooth=10.0qps]开始消费数据..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-67 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-99 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-80 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-76 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-100 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-70 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-68 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-69 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-84 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-58 is running..
RateLimiter[smooth=10.0qps]<01:01:59>Thread-89 is running..
RateLimiter[smooth=10.0qps], successCount: 11
RateLimiter[smooth=10.0qps]###Thread.sleep(5000)...
RateLimiter[smooth=10.0qps]开始消费数据..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-103 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-105 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-106 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-104 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-107 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-108 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-109 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-110 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-112 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-113 is running..
RateLimiter[smooth=10.0qps]<01:01:04>Thread-114 is running..
RateLimiter[smooth=10.0qps], successCount: 11
创建令牌桶：RateLimiter[smooth=10.0qps,counter=80/60s]
RateLimiter[smooth=10.0qps,counter=80/60s]开始消费数据..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-210 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-250 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-252 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-204 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-244 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-273 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-220 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-253 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-274 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-222 is running..
RateLimiter[smooth=10.0qps,counter=80/60s]<01:01:04>Thread-226 is running..
RateLimiter[smooth=10.0qps,counter=80/60s], successCount: 11
RateLimiter[smooth=10.0qps,counter=80/60s]###Thread.sleep(5000)...
RateLimiter[smooth=10.0qps,counter=80/60s]开始消费数据..
RateLimiter[smooth=10.0qps,counter=80/60s], successCount: 0

Process finished with exit code 0
```

#### 业务系统集成 ####

配置BudoRateLimiterFactory, 需要注入JedisPool

```xml
<bean class="xxx.BudoRateLimiterFactory" p:jedisPool-ref="jedisPool"/>
```

Logic 封装，主要是从数据库查询限流策略配置和构建 RateLimiter（有一定时间的缓存）

```java
@Service
public class ApiRateLimiterLogic {

    @Resource
    BudoRateLimiterFactory budoRateLimiterFactory;

    @EhCacheConfig(timeToLiveSeconds = 10 * 60, timeToIdleSeconds = 60, maxElementsInMemory = 600)
    @Cacheable(value = "RATE_LIMITER_CACHED", key = "#providerId+''")
    public BudoRateLimiter rateLimiter(Integer providerId, double qps, int counter) {
        final Calendar calendar = Calendar.getInstance();
        calendar.add(Calendar.DAY_OF_YEAR, 1);
        calendar.set(Calendar.HOUR_OF_DAY, 0);
        calendar.set(Calendar.SECOND, 0);
        calendar.set(Calendar.MINUTE, 0);
        calendar.set(Calendar.MILLISECOND, 0);
        final long secondsNextEarlyMorning = (calendar.getTimeInMillis() - System.currentTimeMillis()) / 1000;
        
        // 构建限流器（qps=100, counter=100000（计数器过期时间为明日凌晨 0 点）
        return budoRateLimiterFactory.build(//
            String.format("OPEN-API-RATE-LIMITER:%d", providerId), qps, //
            new BudoRateLimiter.Counter(counter, secondsNextEarlyMorning)//
        );
    }

    @EhCacheConfig(timeToLiveSeconds = 60 * 60 * 24, timeToIdleSeconds = 60 * 60, maxElementsInMemory = 3000)
    @Cacheable(value = "RATE_LIMITER_CONFIG_MAP_CACHED", key = "#providerId+''")
    public Map<String, Integer> getRateLimiterConfigMap(Integer providerId) {
        return xxx; // 从数据库加载providerId对应的限流策略（如：qps=100,counter=100000)
    }
}
```

Filter 封装，过滤特定的请求

```java
@Component
public class ApiRateLimiterFilter extends AbstractFilter {
    private static final Logger log = Slf4j.getLogger();

    private static ApiRateLimiterLogic apiRateLimiterLogic;

    @Override
    public void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)//
        throws IOException, ServletException {
        final Integer providerId = request.getParameter("provider_id")
        try {
            final Map<String, Integer> limiterConfigMap = this.getApiRateLimiterLogic().getRateLimiterConfigMap(providerId);
            final double qps = (double) MapUtil.getInteger(limiterConfigMap, "qps", 100);
            final int counter = MapUtil.getInteger(limiterConfigMap, "counter", 100000);
            final BudoRateLimiter budoRateLimiter = this.getApiRateLimiterLogic().rateLimiter(providerId, qps, counter);

            final BudoRateLimiter.TryAcquireResponse acquire = budoRateLimiter.tryAcquire();
            if (acquire != BudoRateLimiter.TryAcquireResponse.SUCCESSFUL) {
                log.warn("#0109 RequestLimited({}), ##requestUrl={}, ##providerId={}, ##paramsMap={}", //
                    budoRateLimiter, request.getRequestURL(), providerId, Mvcs.getRequestParameterMap());

                final String errorDescription = acquire == BudoRateLimiter.TryAcquireResponse.FREQUENCY_LIMITED//
                    ? String.format("接口调用频率(%d次/s)已超限制，请稍后再试", (int) qps) : String.format("今日接口调用量(%d次/day)已达上限，请明日再试", counter);
                Mvcs.printJson(ApiError.REQUEST_LIMITED.newObjectWithErrorDescription(errorDescription).toJson());
                return;
            }
        } catch (Throwable e) {
            log.error("#0109 ApiRateLimiterFilter error, ##providerId=" + providerId, e);
        }
        super.doFilter(request, response, chain);
    }

    private ApiRateLimiterLogic getApiRateLimiterLogic() {
        if (null == apiRateLimiterLogic) {
            apiRateLimiterLogic = SpringUtil.getBean(ApiRateLimiterLogic.class);
        }
        return apiRateLimiterLogic;
    }
}
```

```xml
<filter>
       <filter-name>ApiRateLimiterFilter</filter-name>
       <filter-class>xxx.filter.ApiRateLimiterFilter</filter-class>
</filter>
<filter-mapping>
       <filter-name>ApiRateLimiterFilter</filter-name>
       <url-pattern>/api/v1/*</url-pattern>
</filter-mapping>
```


[131728_d2223c6f_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200110/65fda3f7-170f-400c-918d-5844032e8a51.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[132623_145aa3e0_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200110/69fb3705-a54d-4839-9a0f-9494b3ff42eb.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[132757_47f42b67_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200110/d834b4c7-f8e5-4c71-ad01-d123b1d5abc5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[134552_06b7dfb8_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200110/e16e3d92-7b64-4e99-9ff9-421438d59ac1.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg