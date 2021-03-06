## 业务场景

应用中需要实现一个功能： 需要将数据上传到远程存储服务，同时在返回处理成功情况下做其他操作。这个功能不复杂，分为两个步骤：第一步调用远程的Rest服务逻辑包装给处理方法返回处理结果；第二步拿到第一步结果或者捕捉异常，如果出现错误或异常实现重试上传逻辑，否则继续逻辑操作。

## 解决方案演化

这个问题的技术点在于能够触发重试，以及重试情况下逻辑有效执行。

### 解决方案一：try-catch-redo简单重试模式

包装正常上传逻辑基础上，通过判断返回结果或监听异常决策是否重试，同时为了解决立即重试的无效执行(假设异常是有外部执行不稳定导致的)，休眠一定延迟时间重新执行功能逻辑。

```java
public void commonRetry(Map<String, Object> dataMap) throws InterruptedException {
		Map<String, Object> paramMap = Maps.newHashMap();
		paramMap.put("tableName", "creativeTable");
		paramMap.put("ds", "20160220");
		paramMap.put("dataMap", dataMap);
		boolean result = false;
		try {
			result = uploadToOdps(paramMap);
			if (!result) {
				Thread.sleep(1000);
				uploadToOdps(paramMap);  //一次重试
			}
		} catch (Exception e) {
			Thread.sleep(1000);
			uploadToOdps(paramMap);//一次重试
		}
	}
```

### 解决方案二：try-catch-redo-retry strategy策略重试模式

上述方案还是有可能重试无效，解决这个问题尝试增加重试次数retrycount以及重试间隔周期interval，达到增加重试有效的可能性。

```java
public void commonRetry(Map<String, Object> dataMap) throws InterruptedException {
		Map<String, Object> paramMap = Maps.newHashMap();
		paramMap.put("tableName", "creativeTable");
		paramMap.put("ds", "20160220");
		paramMap.put("dataMap", dataMap);
		boolean result = false;
		try {
			result = uploadToOdps(paramMap);
			if (!result) {
				reuploadToOdps(paramMap,1000L,10);//延迟多次重试
			}
		} catch (Exception e) {
			reuploadToOdps(paramMap,1000L,10);//延迟多次重试
		}
	}
```

 

方案一和方案二存在一个问题：正常逻辑和重试逻辑强耦合，重试逻辑非常依赖正常逻辑的执行结果，对正常逻辑预期结果被动重试触发，对于重试根源往往由于逻辑复杂被淹没，可能导致后续运维对于重试逻辑要解决什么问题产生不一致理解。重试正确性难保证而且不利于运维，原因是重试设计依赖正常逻辑异常或重试根源的臆测。

## 优雅重试方案尝试

那有没有可以参考的方案实现正常逻辑和重试逻辑解耦，同时能够让重试逻辑有一个标准化的解决思路？答案是有：那就是基于代理设计模式的重试工具，我们尝试使用相应工具来重构上述场景。

### 尝试方案一：应用命令设计模式解耦正常和重试逻辑

命令设计模式具体定义不展开阐述，主要该方案看中命令模式能够通过执行对象完成接口操作逻辑，同时内部封装处理重试逻辑，不暴露实现细节，对于调用者来看就是执行了正常逻辑，达到解耦的目标，具体看下功能实现。（类图结构）

![img](../assets/image-20191008172731604.png)

IRetry约定了上传和重试接口，其实现类OdpsRetry封装ODPS上传逻辑，同时封装[重试机制](https://link.juejin.im?target=http%3A%2F%2Fwww.liuhaihua.cn%2Farchives%2Ftag%2F%e9%87%8d%e8%af%95%e6%9c%ba%e5%88%b6)和重试策略。与此同时使用recover方法在结束执行做恢复操作。

而我们的调用者LogicClient无需关注重试，通过重试者Retryer实现约定接口功能，同时 Retryer需要对重试逻辑做出响应和处理， Retryer具体重试处理又交给真正的IRtry接口的实现类OdpsRetry完成。通过采用命令模式，优雅实现正常逻辑和重试逻辑分离，同时通过构建重试者角色，实现正常逻辑和重试逻辑的分离，让重试有更好的扩展性。

### 尝试方案二：spring-retry 规范正常和重试逻辑

spring-retry是一个开源工具包，目前可用的版本为1.1.2.RELEASE，该工具把重试操作模板定制化，可以设置重试策略和回退策略。同时重试执行实例保证线程安全，具体场景操作实例如下：

```java
public void upload(final Map<String, Object> map) throws Exception {
		// 构建重试模板实例
		RetryTemplate retryTemplate = new RetryTemplate();
		// 设置重试策略，主要设置重试次数
		SimpleRetryPolicy policy = new SimpleRetryPolicy(3, Collections.<Class<? extends Throwable>, Boolean> singletonMap(Exception.class, true));
		// 设置重试回退操作策略，主要设置重试间隔时间
		FixedBackOffPolicy fixedBackOffPolicy = new FixedBackOffPolicy();
		fixedBackOffPolicy.setBackOffPeriod(100);
		retryTemplate.setRetryPolicy(policy);
		retryTemplate.setBackOffPolicy(fixedBackOffPolicy);
		// 通过RetryCallback 重试回调实例包装正常逻辑逻辑，第一次执行和重试执行执行的都是这段逻辑
		final RetryCallback<Object, Exception> retryCallback = new RetryCallback<Object, Exception>() {
			//RetryContext 重试操作上下文约定，统一spring-try包装 
			public Object doWithRetry(RetryContext context) throws Exception {
				System.out.println("do some thing");
				Exception e = uploadToOdps(map);
				System.out.println(context.getRetryCount());
				throw e;//这个点特别注意，重试的根源通过Exception返回
			}
		};
		// 通过RecoveryCallback 重试流程正常结束或者达到重试上限后的退出恢复操作实例
		final RecoveryCallback<Object> recoveryCallback = new RecoveryCallback<Object>() {
			public Object recover(RetryContext context) throws Exception {
				System.out.println("do recory operation");
				return null;
			}
		};
		try {
			// 由retryTemplate 执行execute方法开始逻辑执行
			retryTemplate.execute(retryCallback, recoveryCallback);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
```

 

简单剖析下案例代码，RetryTemplate 承担了重试执行者的角色，它可以设置SimpleRetryPolicy(重试策略，设置重试上限，重试的根源实体)，FixedBackOffPolicy（固定的回退策略，设置执行重试回退的时间间隔）。      RetryTemplate通过execute提交执行操作，需要准备RetryCallback 和RecoveryCallback 两个类实例，前者对应的就是重试回调逻辑实例，包装正常的功能操作，RecoveryCallback实现的是整个执行操作结束的恢复操作实例。

RetryTemplate的execute 是线程安全的，实现逻辑使用ThreadLocal保存每个执行实例的RetryContext执行上下文。

Spring-retry工具虽能优雅实现重试，但是存在两个不友好设计：一个是 重试实体限定为Throwable子类，说明重试针对的是可捕捉的功能异常为设计前提的，但是我们希望依赖某个数据对象实体作为重试实体，但Sping-retry框架必须强制转换为Throwable子类。另一个就是重试根源的断言对象使用的是doWithRetry的Exception 异常实例，不符合正常内部断言的返回设计。

Spring Retry提倡以注解的方式对方法进行重试，重试逻辑是同步执行的，重试的“失败”针对的是Throwable，如果你要以返回值的某个状态来判定是否需要重试，可能只能通过自己判断返回值然后显式抛出异常了。

## Spring 对于Retry的抽象

“抽象”是每个程序员必备的素质。对于资质平平的我来说，没有比模仿与理解优秀源码更好地进步途径了吧。为此，我将其核心逻辑重写了一遍…下面就看看Spring Retry对于“重试”的抽象。

### “重试”逻辑

```
while(someCondition()) {
    try{
        doSth();
        break;    
    } catch(Throwable th) {
        modifyCondition();
        wait();
    }
}
if(stillFail) {
    doSthWhenStillFail();
}复制代码
```

同步重试代码基本可以表示为上述，但是Spring Retry对其进行了非常优雅地抽象，虽然主要逻辑不变，但是看起来却是舒服多了。主要的接口抽象如下图所示：

![img](../assets/image-20191008173632939.png)

- RetryCallback: 封装你需要重试的业务逻辑（上文中的doSth）
- RecoverCallback：封装在多次重试都失败后你需要执行的业务逻辑(上文中的doSthWhenStillFail)
- RetryContext: 重试语境下的上下文，可用于在多次Retry或者Retry 和Recover之间传递参数或状态（在多次doSth或者doSth与doSthWhenStillFail之间传递参数）
- RetryOperations : 定义了“重试”的基本框架（模板），要求传入RetryCallback，可选传入RecoveryCallback；
- RetryListener：典型的“监听者”，在重试的不同阶段通知“监听者”（例如doSth，wait等阶段时通知）
- RetryPolicy : 重试的策略或条件，可以简单的进行多次重试，可以是指定超时时间进行重试（上文中的someCondition）
- BackOffPolicy: 重试的回退策略，在业务逻辑执行发生异常时。如果需要重试，我们可能需要等一段时间(可能服务器过于繁忙，如果一直不间隔重试可能拖垮服务器)，当然这段时间可以是0，也可以是固定的，可以是随机的（参见tcp的拥塞控制算法中的回退策略）。回退策略在上文中体现为wait()；
- RetryTemplate :RetryOperations的具体实现，组合了RetryListener[]，BackOffPolicy，RetryPolicy。

### 尝试方案三：guava-retryer 分离正常和重试逻辑

Guava retryer工具与spring-retry类似，都是通过定义重试者角色来包装正常逻辑重试，但是Guava retryer有更优的策略定义，在支持重试次数和重试频度控制基础上，能够兼容支持多个异常或者自定义实体对象的重试源定义，让重试功能有更多的灵活性。Guava Retryer也是线程安全的，入口调用逻辑采用的是Java.util.concurrent.Callable的call方法，示例代码如下：

```
public void uploadOdps(final Map<String, Object> map) {
		// RetryerBuilder 构建重试实例 retryer,可以设置重试源且可以支持多个重试源，可以配置重试次数或重试超时时间，以及可以配置等待时间间隔
		Retryer<Boolean> retryer = RetryerBuilder.<Boolean> newBuilder()
				.retryIfException().//设置异常重试源
				retryIfResult(new Predicate<Boolean>() {//设置自定义段元重试源，
			@Override
			public boolean apply(Boolean state) {//特别注意：这个apply返回true说明需要重试，与操作逻辑的语义要区分
				return true;
			}
		})
		.withStopStrategy(StopStrategies.stopAfterAttempt(5))//设置重试5次，同样可以设置重试超时时间
		.withWaitStrategy(WaitStrategies.fixedWait(100L, TimeUnit.MILLISECONDS)).build();//设置每次重试间隔

		try {
			//重试入口采用call方法，用的是java.util.concurrent.Callable<V>的call方法,所以执行是线程安全的
			boolean result = retryer.call(new Callable<Boolean>() { 
				@Override
				public Boolean call() throws Exception {
					try {
						//特别注意：返回false说明无需重试，返回true说明需要继续重试
						return uploadToOdps(map);
					} catch (Exception e) {
						throw new Exception(e);
					}
				}
			});

		} catch (ExecutionException e) {
		} catch (RetryException ex) {
		}
	}
复制代码
```

 

示例代码原理分析：

RetryerBuilder是一个factory创建者，可以定制设置重试源且可以支持多个重试源，可以配置重试次数或重试超时时间，以及可以配置等待时间间隔，创建重试者Retryer实例。

RetryerBuilder的重试源支持Exception异常对象 和自定义断言对象，通过retryIfException 和retryIfResult设置，同时支持多个且能兼容。

RetryerBuilder的等待时间和重试限制配置采用不同的策略类实现，同时对于等待时间特征可以支持无间隔和固定间隔方式。

Retryer 是重试者实例，通过call方法执行操作逻辑，同时封装重试源操作。

## 优雅重试共性和原理

1. 正常和重试优雅解耦，重试断言条件实例或逻辑异常实例是两者沟通的媒介。
2. 约定重试间隔，差异性重试策略，设置重试超时时间，进一步保证重试有效性以及重试流程稳定性。
3. 都使用了命令设计模式，通过委托重试对象完成相应的逻辑操作，同时内部封装实现重试逻辑。
4. Spring-tryer和guava-tryer工具都是线程安全的重试，能够支持并发业务场景的重试逻辑正确性。

## 优雅重试适用场景

1. 功能逻辑中存在不稳定依赖场景，需要使用重试获取预期结果或者尝试重新执行逻辑不立即结束。比如远程接口访问，数据加载访问，数据上传校验等等。
2. 对于异常场景存在需要重试场景，同时希望把正常逻辑和重试逻辑解耦。
3. 对于需要基于数据媒介交互，希望通过重试轮询检测执行逻辑场景也可以考虑重试方案。
