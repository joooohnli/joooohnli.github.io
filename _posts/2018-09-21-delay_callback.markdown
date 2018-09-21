---
layout: post
title:  "分布式定时回调服务"
date:   2018-09-21 14:42:49 +0800
categories: timer callback
---

 
![A distributed, reliable timer / timed callback service, based on dubbo&redis.](https://favicon.yandex.net/favicon/github.com)[View on GitHub](https://github.com/joooohnli/delay-callback)

# 一、背景
业务需要延时执行一段逻辑，比如注册会员成功3小时自动开通发言权限、下单之7天内未支付自动关闭订单。
该场景特点是：1定时时间为动态设置；2由业务自身处理任务，因此需要这样一个定时回调服务。

# 二、概览
![overview](/images/delay-callback.png)

### 功能：
- 1.应用注册任务(设置执行时间)，服务器到时回调应用；
- 2.任务回调前，应用可取消任务；
- 3.服务器保证回调成功(注册任务时设置重试策略)；
- 4.服务器保证HA，部署多台即可；
- 5.任务注册支持幂等；
- 6.回调幂等需要应用自己去保证；
- 7.应用方编写代码简单，类似本地java回调；

# 三、服务器
### 存储
使用redis。其中：
- zset保存由任务主数据（uid、执行时间），定时任务扫描该zset获取当前时间附近的数据即可
- hash保存任务详情等
- setnx实现分布式锁

### 定时任务
使用spring boot scheduler，多实例时会选主，也即只有一台执行定时任务。
定时任务有：
- 扫描当前可执行任务的定时任务；
- 扫描过去可执行任务的定时任务，用于异常情况如服务器全挂、更新任务数据失败；
- 监控任务，监控任务总量，重试后仍失败的任务量等；

### 分发器
定时任务扫描出的任务列表分发出去交由执行器执行，这里的分发功能使用dubbo接口的负载均衡功能。

### 执行器
执行回调任务，更新任务数据。

# 四、客户端
### 帮助类

# 五、使用方法
## 1.启动服务器
```bash
cd delay-callback-server
# 编辑application.properties，保证server和应用在同一个zk注册
mvn spring-boot:run
```
## 2.应用引入客户端
```xml
<dependency>
    <groupId>com.johnli</groupId>
    <artifactId>delay-callback-client</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```
## 3.应用编写代码
关注两个方法即可
```java
    /**
	 * 注册
     * @param callbackParam 参数
     * @param delayCallback 实现回调业务
     * @return 唯一ID
     */
    public static RegisterResult register(CallbackParam callbackParam, DelayCallback delayCallback)
	
	/**
     * 取消注册
     *
     * @param uid 唯一ID
     * @return
     */
    public static UnRegisterResult unRegister(String uid)
```
### 3.1.基本用法

register方法的第2个参数是一个```DelayCallback```对象，所以大致有两种写法。
##### 3.1.1.使用匿名内部类
```java
    public void test() {
        // 参数列表
        List<String> args = new ArrayList<>();
        // 延时时间
        int delaySeconds = 10;
        CallbackParam callbackParam = new CallbackParam(args, delaySeconds);

        RegisterResult register = DelayCallbackHelper.register(callbackParam, new DelayCallback() {
            @Override
            public String alias() {
                // 定义回调别名，建议用一个常量类管理
                return Constants.ALIAS_1;
            }

            @Override
            public boolean onCallback(CallbackRequest request) {
                // 注册时保存的参数
                CallbackParam requestCallbackParam = request.getCallbackParam();
                // 可根据该uid做幂等
                String uid = request.getUid();
                // 处理业务...

                // 返回true表示回调完成；返回false或抛异常，将进行重试(如有可用重试次数)
                return true;
            }
        });

        // 可判断是否注册成功
        if (register.isSuccess()) {
            // 注册成功，会返回uid
            // 可在回调执行之前，取消注册
            DelayCallbackHelper.unRegister(register.getUid());
        }
    }
```
#####3.1.2.使用spring bean
```java
    @Autowired
    private DelayCallbackService delayCallbackService;

    public void test2(){
        // 用法同上例，区别是第个传参传入bean
        DelayCallbackHelper.register(new CallbackParam(new ArrayList<>(), 3), (DelayCallback) delayCallbackService);
    }
```
其中，```delayCallbackService```是一个实现了```DelayCallback```的bean
```java
@Service
public class DelayCallbackServiceImpl implements DelayCallbackService, DelayCallback {
    @Override
    public String alias() {
        return "callback02";
    }

    @Override
    public boolean onCallback(CallbackRequest request) {
        System.out.println("on callback");
        return true;
    }
}
```
### 3.2.高级用法
#### 3.2.1.幂等
注册回调时，可设置幂等键CallbackParam.idempotentKey:

- a. 若设置，则同一alias下，相同idempotentKey的多次注册，只会注册一次回调，即返回首次注册的uid；
- b. 反之，每次注册均生成新的回调，返回不同的uid

注意：这里的幂等只保证注册时的幂等。回调时的幂等需要应用自身保证（根据uid）


#### 3.2.2.自定义重试策略
默认: 新注册的回调将重试10次，首次间隔10秒，后续按10*2^i 递增

如需自定义，设置CallbackParam.retryStrategy，其属性如下：
```
    /**
     * 重试次数
     * <p> <0.非法参数
     * <p> =0.不重试
     * <p> >0.固定次数
     */
    private int times = TIMES_DEFAULT;
    /**
     * 重试类型
     * <p>基数为interval
     * <p>1.固定时间间隔
     * <p>2.时间间隔倍数增长[interval, 2*interval, 3*interval, ..., n*interval]
     * <p>3.时间间隔指数增长[interval, 2^1*interval, 2^2*interval, ... , 2^n*interval]
     */
    private int type = TYPE_DEFAULT;
    /**
     * 重试间隔
     * <p>单位：秒
     */
    private int interval = INTERVAL_DEFAULT;
    /**
     * 最大重试间隔
     * <p>单位：秒
     */
    private int maxInterval = MAX_INTERVAL_DEFAULT;
```

# 六、注意事项
### 1.接入要求
回调通过dubbo接口，需要保证应用已开启dubbo端口。
### 2.使用限制
使用方式与本地java回调基本一致，除了一个限制：以匿名内部类方式实现的回调，方法体中禁止包含外部方法的局部变量。   错误示例：   
```
    public void wrongUsage() {
        // 局部变量
        String hello = "world";
        RegisterResult register = DelayCallbackHelper.register(new CallbackParam(new ArrayList<>(), 10), new DelayCallback() {
            @Override
            public String alias() {
                return Constants.ALIAS_2;
            }

            @Override
            public boolean onCallback(CallbackRequest request) {
                // 错：使用了 外部方法 的 局部变量
                System.out.println(hello);

                return true;
            }
        });

    }
```
  若错误使用，应用启动时会抛异常且中断：
  
```
2018-08-30 11:54:26.335 ERROR 18485 --- [           main] o.s.boot.SpringApplication               : Application startup failed

com.john.callback.CallbackException: Callback init error,reason:Anonymous inner class implementation of DelayCallback can not include local variables of external method,class:class provider.test.callback.DelayCallbackTest$2

```


