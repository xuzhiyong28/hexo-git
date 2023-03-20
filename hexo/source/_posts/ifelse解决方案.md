## 策略模式+工厂模式

```java
// 1. 策略接口 继承InitializingBean(spring中初始化会执行afterPropertiesSet方法)
public interface Handler extends InitializingBean {
    public void AAA(String name);
    public void BBB(String name);
}
// 2. 工厂类
public class Factory {
    private static Map<String, Handler> stringHandlerMap = new HashMap<>();

    public static Handler getInvokeStrategy(String name) {
        return stringHandlerMap.get(name);
    }
	// 注册进工厂
    public static void register(String name, Handler handler) {
        if(StringUtils.isEmpty(name) || null == handler) {
            return;
        }
        stringHandlerMap.put(name, handler);
    }
}

// 2.具体策略
public class OneHandler implements Handler {
     @Override
    public void AAA(String name) {
        
    }

    @Override
    public void BBB(String name) {
    }
    
    @Override
    public void afterPropertiesSet() throws Exception {
        Factory.register("One", this);
    }
}

public class TwoHandler implements Handler {
     @Override
    public void AAA(String name) {
    }

    @Override
    public void BBB(String name) {
    }
    
    @Override
    public void afterPropertiesSet() throws Exception {
        Factory.register("Two", this);
    }
}

@Test
public void test(){
   Handler handler = Factory.getInvokeStrategy("one");
   handler.AAA();
}
```

上面这种方式很大程度上消除了IF-ELSE，但是还有有一些缺点，假设

- OneHandler在业务上只需要用到AAA方法
- TwoHandler在业务只需要用到BBB方法
- ThreeHandler需要新增一个CCC方法

按照上面的假设，由于三个Handler都实现Handler接口，当其中一个实现需要加方法时其余两个Handler也需要跟着实现新方法，不管需不需要新方法。

**继续改进 : 引入模板方法**

## 策略模式+工厂模式+模板方法

```java
// 1.采用模板方法，新增Handler抽象类
public abstract class AbstractHandler implements InitializingBean {
    public void AAA(String name) {
        throw new UnsupportedOperationException();
    }

    public void BBB(String name) {
        throw new UnsupportedOperationException();
    }
    
    public void CCC(String name) {
        throw new UnsupportedOperationException();
    }
}

// 2.工厂方法
public class Factory {
    private static Map<String, AbstractHandler> stringHandlerMap = new HashMap<>();

    public static AbstractHandler getInvokeStrategy(String name) {
        return stringHandlerMap.get(name);
    }

    public static void register(String name, AbstractHandler abstractHandler) {
        if(StringUtils.isEmpty(name) || null == abstractHandler) {
            return;
        }
        stringHandlerMap.put(name, abstractHandler);
    }
}

// 具体实现
public class OneHandler extends AbstractHandler {

    @Override
    public void AAA(String name) {
        System.out.println("One-AAA");
    }

    @Override
    public void BBB(String name) {
        System.out.println("One-BBB");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        Factory.register("One", this);
    }
}

public class TwoHandler extends AbstractHandler {

    @Override
    public void AAA(String name) {
        System.out.println("Two-AAA");
    }

    @Override
    public void BBB(String name) {
        System.out.println("Two-BBB");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        Factory.register("Two", this);
    }
}

public class ThreeHandler extends AbstractHandler {

    @Override
    public void CCC(String name) {
        System.out.println("Two-CCC");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        Factory.register("Three", this);
    }
}
```

引入了模板方法后，解决了上述缺点，不需要每一个实现类都实现新方法，对于不需要实现的方法只要不重写即可。