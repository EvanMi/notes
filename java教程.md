# jvm

## 类加载机制深度解析

```java
class B {
	static {
		System.out.println("B初始化");
	}
}

class Main {
	public static void main (string[] args) {
		B b = null;
	}
}
```

上面这段代码中是不会执行初始化方法的。

在没有打破双亲委派机制的前提下，整个jvm中的class是唯一的；打破双亲委派机制后，在classLoader中class是唯一的。