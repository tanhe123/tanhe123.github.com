---
title: 动手编写 Mockito
date: 2016-01-10 22:58
---

在 WebIDE 团队内部是要求写单元测试的！

这是我第一次接触Mockito。

初次使用 Mockito，能够感受到它的神奇，尤其是这样的语法：

```
when(cal.add(0, 1)).thenReturn(1);
```

Mockito 会把它理解成，当 cal 调用 add 方法且参数为 0 和 1 时，则返回 1。

我们知道，java 中的程序调用是以栈的形式实现的，对于 when 方法，add 方法的调用对它是不可见的。when 能接收到的，只有 add 的返回值。

# 那么 Mockito 是如何实现的呢？

我们知道，Mock 使用了代理模式。我们操作的 cal，实际上是继承了 Calculate 的代理类的实例，我们把它称为 proxy 对象。因此对于 cal 的所有操作，都会先经过 proxy 对象。

Mocktio 就是通过这种方式，拦截到了 cal 的所有操作。只要在调用 cal 的方法时，将该方法存放起来，然后在调用 thenReturn 为其设置返回值就可以了。

# 光说不练假把式

既然知道了原理，那我们也来尝试着实现一个简单的 Mockito。顺便加深下对代理模式的理解。

该 Mockito 需要支持的特性有 mock、stub、spy。

## 第一步： 实现 mock

既然要实现 mock，我们要知道，一个对象被 mock 后，有什么特征。

在单元测试中，我们要测试的模块可能依赖一些不易构造或比较复杂的对象，因此，mock 要支持通过接口、具体类构造 mock 对象。被 mock 的对象只是“假装”调用了该方法，然后“应付”的返回“空值”就可以了。

我们通过 cglib 来实现，实现代码很简单。

```
public static <T> T mock(Class<T> clazz) {
	Enhancer enhancer = new Enhancer();
	enhancer.setSuperclass(clazz);
	enhancer.setCallback(new MockInterceptor());
	return (T) enhancer.create();
}

private static class MockInterceptor implements MethodInterceptor {
	@Override
	public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
		return null;
	}
}
```

测试用例我们可以这么写

```
Calculate cal = mock(Calculate.class);
Assert.assertEquals(0, cal.add(0, 1));
```

## 第二步，实现 stub

接下来我们来实现 stub。stub 一个方法，其实就是指定该方法在具体参数下的返回值。而且该返回值无论经过多少次调用都是不变的，除非再次 stub 该方法，用新的返回值将原来的替换掉。

首先，我们定义一个类，用来表示对一个函数的调用。

```
public class Invocation {
	private final Object mock;

	private final Method method;

	private final Object[] arguments;

	private final MethodProxy proxy;

	// 省略其它不重要代码...
}
```

接下来，在 MockInterceptor 类中，需要做两个操作。

1. 为了设置方法的返回值，需要存放对方法的引用（lastInvocation）
2. 调用方法时，检查是否已经设置了该方法的返回值（results）。如果设置了，则返回该值。

实现代码如下：

```
private static class MockInterceptor implements MethodInterceptor {

	@Override
	public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {

		Invocation invocation = new Invocation(proxy, method, args, proxy);

		lastInvocation = invocation;

		if (results.containsKey(invocation)) {
			return results.get(invocation);
		}

		return null;
	}
}

public static <T> When<T> when(T o) {
	return new When<T>();
}

public static class When<T> {
	public void thenReturn(T retObj) {
		results.put(lastInvocation, retObj);
	}
}
```

可以通过下面测试下效果：

```
Calculate cal = mock(Calculate.class);
when(cal.add(0, 1)).thenReturn(1);
Assert.assertEquals(1, cal.add(0, 1));
```

## 第三步，实现 spy

如果说 mock 的处理是返回“空值”，那么 spy 的处理就是通过代理对象调用真实方法了。

我们使用 cglib 提供的方法，使得 spy 生成的代理类，默认调用真实方法。

```
public Object callRealMethod() {
	try {
		return this.proxy.invokeSuper(obj, arguments);
	} catch (Throwable throwable) {
		throwable.printStackTrace();
	}

	return null;
}
```
就这么简单，spy就实现了，可以使用下面的代码测试

```
Calculate cal = spy(Calculate.class);
Assert.assertEquals(3, cal.add(1, 2));
when(cal.add(1, 2)).thenReturn(0);
Assert.assertEquals(0, cal.add(1, 2));
```

**备注：限于篇幅，只贴出关键代码，可以查看源代码了解更多细节**

# 源代码

附上示例代码: https://coding.net/u/tanhe123/p/MockitoSamples
