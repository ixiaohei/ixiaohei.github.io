---
author: ixiaohei
pubDatetime: 2023-09-05T21:13:52.737Z
title: spring cloud feign重试源码分析
postSlug: spring-cloud-feign-retry
featured: true
ogImage: https://user-images.githubusercontent.com/53733092/215771435-25408246-2309-4f8b-a781-1f3d93bdf0ec.png
tags:
  - spring cloud
  - feign
  - java
description: spring cloud feign重试机制源码分析
---

在spring cloud中feign是没有重试机制的，但是其复用Ribbon重试逻辑

### RetryTemplate
1. 重试逻辑一般使用重试模版`org.springframework.retry.support.RetryTemplate`实现
2. 其核心逻辑在`org.springframework.retry.support.RetryTemplate#doExecute`方法中

### 重试逻辑

```java
//循环判断是否可重试和流程无中断
while (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {

	try {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Retry: count=" + context.getRetryCount());
		}
		// Reset the last exception, so if we are successful
		// the close interceptors will not think we failed...
		lastException = null;
		//执行业务逻辑
		return retryCallback.doWithRetry(context);
	}
	catch (Throwable e) {

		lastException = e;

		try {
			//异常处理，此处会增加重试次数
			registerThrowable(retryPolicy, state, context, e);
		}
		catch (Exception ex) {
			throw new TerminatedRetryException("Could not register throwable",
					ex);
		}
		finally {
			doOnErrorInterceptors(retryCallback, context, e);
		}

		if (canRetry(retryPolicy, context) && !context.isExhaustedOnly()) {
			try {
				backOffPolicy.backOff(backOffContext);
			}
			catch (BackOffInterruptedException ex) {
				lastException = e;
				// back off was prevented by another thread - fail the retry
				if (this.logger.isDebugEnabled()) {
					this.logger
							.debug("Abort retry because interrupted: count="
									+ context.getRetryCount());
				}
				throw ex;
			}
		}

		if (this.logger.isDebugEnabled()) {
			this.logger.debug(
					"Checking for rethrow: count=" + context.getRetryCount());
		}

		if (shouldRethrow(retryPolicy, context, state)) {
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Rethrow in retry for policy: count="
						+ context.getRetryCount());
			}
			throw RetryTemplate.<E>wrapIfNecessary(e);
		}

	}

	/*
	 * A stateful attempt that can retry may rethrow the exception before now,
	 * but if we get this far in a stateful retry there's a reason for it,
	 * like a circuit breaker or a rollback classifier.
	 */
	if (state != null && context.hasAttribute(GLOBAL_STATE)) {
		break;
	}
}
```

其中`canRetry`会委托给`org.springframework.retry.RetryPolicy`判断
```java
protected boolean canRetry(RetryPolicy retryPolicy, RetryContext context) {
	return retryPolicy.canRetry(context);
}
```

### FeignRetryPolicy
而在Feign中是由`org.springframework.cloud.netflix.feign.ribbon.FeignRetryPolicy`实现了`org.springframework.retry.RetryPolicy`
```java
@Override
public boolean canRetry(RetryContext context) {
	if(context.getRetryCount() == 0) {
		return true;
	}
	return super.canRetry(context);
}
```

其第一次直接返回可重试，第二次其才委托给父类`org.springframework.cloud.client.loadbalancer.InterceptorRetryPolicy`

### InterceptorRetryPolicy
```java
@Override
public boolean canRetry(RetryContext context) {
    LoadBalancedRetryContext lbContext = (LoadBalancedRetryContext)context;
    if(lbContext.getRetryCount() == 0  && lbContext.getServiceInstance() == null) {
        //We haven't even tried to make the request yet so return true so we do
        lbContext.setServiceInstance(serviceInstanceChooser.choose(serviceName));
        return true;
    }
    return policy.canRetryNextServer(lbContext);
}
```
此处会委托给`org.springframework.cloud.netflix.ribbon.RibbonLoadBalancedRetryPolicy`


### RibbonLoadBalancedRetryPolicy
#### 核心方法`canRetryNextServer()`
其中简单判断当前`nextServerCount`是否大于和等于`getMaxRetriesOnNextServer()`和请求是否可以重试
```java
@Override
public boolean canRetryNextServer(LoadBalancedRetryContext context) {
	//this will be called after a failure occurs and we increment the counter
	//so we check that the count is less than or equals to too make sure
	//we try the next server the right number of times
	return nextServerCount <= lbContext.getRetryHandler().getMaxRetriesOnNextServer() && canRetry(context);
}
```
其中`getMaxRetriesOnNextServer()`由`ribbon.MaxAutoRetriesNextServer`配置，默认为1

#### 方法`org.springframework.cloud.netflix.ribbon.RibbonLoadBalancedRetryPolicy#canRetry()`
```java
public boolean canRetry(LoadBalancedRetryContext context) {
	HttpMethod method = context.getRequest().getMethod();
	return HttpMethod.GET == method || lbContext.isOkToRetryOnAllOperations();
}
```
此方法仅仅简单判断请求是否`GET`方式和是否配置`ribbon.OkToRetryOnAllOperations`,默认为`false`

### 异常重试
通过以上代码我们发现：如果异常情况下，仿佛while循环中的条件一直为true，无法结束。其实有个重要逻辑在`registerThrowable()`方法中实现

#### registerThrowable()
异常会进入此方法
```java
protected void registerThrowable(RetryPolicy retryPolicy, RetryState state,
		RetryContext context, Throwable e) {
	retryPolicy.registerThrowable(context, e);
	registerContext(context, state);
}
```
其委托给`InterceptorRetryPolicy`处理
```java
LoadBalancedRetryContext lbContext = (LoadBalancedRetryContext) context;
//this is important as it registers the last exception in the context and also increases the retry count
lbContext.registerThrowable(throwable);
//let the policy know about the exception as well
policy.registerThrowable(lbContext, throwable);
```
其中2行代码都很重要:
#### 第一行
第一行代码会执行RetryContextSupport中的registerThrowable()
```java
public void registerThrowable(Throwable throwable) {
	this.lastException = throwable;
	if (throwable != null)
		count++;
}
```
其中简单的增加`count`；是`FeignRetryPolicy#canRetry()`第一次判断逻辑
```java
if(context.getRetryCount() == 0) {
	return true;
}
```

#### 第二行
第二行会执行RibbonLoadBalancedRetryPolicy中的registerThrowable()
```java
@Override
public void registerThrowable(LoadBalancedRetryContext context, Throwable throwable) {
	//if this is a circuit tripping exception then notify the load balancer
	if (lbContext.getRetryHandler().isCircuitTrippingException(throwable)) {
		updateServerInstanceStats(context);
	}
	
	//Check if we need to ask the load balancer for a new server.
	//Do this before we increment the counters because the first call to this method
	//is not a retry it is just an initial failure.
	if(!canRetrySameServer(context)  && canRetryNextServer(context)) {
		context.setServiceInstance(loadBalanceChooser.choose(serviceId));
	}
	//This method is called regardless of whether we are retrying or making the first request.
	//Since we do not count the initial request in the retry count we don't reset the counter
	//until we actually equal the same server count limit.  This will allow us to make the initial
	//request plus the right number of retries.
	if(sameServerCount >= lbContext.getRetryHandler().getMaxRetriesOnSameServer() && canRetry(context)) {
		//reset same server since we are moving to a new server
		sameServerCount = 0;
		nextServerCount++;
		if(!canRetryNextServer(context)) {
			context.setExhaustedOnly();
		}
	} else {
		sameServerCount++;
	}
}
```
最终会变成处理`nextServerCount`属性；第二次`FeignRetryPolicy#canRetry()`使用此逻辑
