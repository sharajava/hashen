下面把 EventPlugin 从“发现 → 注册 → 调度 → 隔离” 的全流程拆开讲，并给出关键代码片段。所有代码可直接放进前面那套示例项目中运行。


---

1. 发现（Discovery）

1.1 本地 Bean（最简单）

@Configuration
class PluginAutoConfiguration {

    @Bean
    public PluginRegistry pluginRegistry(List<EventPlugin<?>> springBeans) {
        return new PluginRegistry(springBeans);  // Spring 已扫描到全部 @Component EventPlugin
    }
}

约束

任何实现 EventPlugin<?> 且带 @Supports 注解的 Spring Bean 都会被注入到构造器 springBeans 里。

不需要写额外 XML 或 SPI 文件——Spring 的类路径扫描搞定。


1.2 动态插件 JAR（可选）

若你想让用户热插拔 JAR，可在 BOOT-INF/plugins/ 目录下放插件包，框架启动时手动加载：

class ExternalPluginLoader {

    List<EventPlugin<?>> load(File jar) throws Exception {
        URLClassLoader cl = new URLClassLoader(new URL[]{jar.toURI().toURL()}, getClass().getClassLoader());
        List<String> classes = Files.readAllLines(
            Paths.get("jar:" + jar.toURI() + "!/META-INF/plugins.idx")); // 每行一个 FQCN
        List<EventPlugin<?>> plugins = new ArrayList<>();
        for (String fqcn : classes) {
            Class<?> c = cl.loadClass(fqcn);
            EventPlugin<?> p = (EventPlugin<?>) c.getDeclaredConstructor().newInstance();
            Injector.autowire(p);     // 利用 Spring BeanFactory 注入依赖
            plugins.add(p);
        }
        return plugins;
    }
}

> 插件包格式

my-plugin.jar
 └─ META-INF/plugins.idx   # 每行：com.foo.OrderPreparePlugin




---

2. 注册（Registry）

class PluginRegistry {

    private final Map<Key, List<PluginDescriptor>> table = new ConcurrentHashMap<>();

    PluginRegistry(Collection<EventPlugin<?>> plugins) {
        for (EventPlugin<?> p : plugins) add(p);
    }

    void add(EventPlugin<?> impl) {
        Supports meta = impl.getClass().getAnnotation(Supports.class);
        Key k = new Key(meta.entity(), meta.event(), meta.phase());
        table.computeIfAbsent(k, __ -> new ArrayList<>())
             .add(new PluginDescriptor(meta.order(), impl));
        table.get(k).sort(Comparator.comparingInt(PluginDescriptor::order));
    }

    List<EventPlugin<?>> find(String entity, EventType evt, Phase ph) {
        return table.getOrDefault(new Key(entity, evt, ph), List.of())
                    .stream().map(PluginDescriptor::impl).toList();
    }

    record Key(String entity, EventType evt, Phase ph) {}
    record PluginDescriptor(int order, EventPlugin<?> impl) {}
}

特点

三元 Key：entity + event + phase 决定唯一插件列表。

order 排序：越小越先执行；ON 阶段若短路即停止后续插件。

Registry 本身线程安全，可在插件热加载时增删。



---

3. 调度（Dispatcher 调用）

class EventDispatcher {

    private final PluginRegistry registry;
    private final ContextProxyFactory proxyFactory;
    private final Map<String, GenericProvider<?>> providers;    // e.g. OrderCreateProvider

    public <I,O> O dispatch(EventContextImpl<I,O> ctx) {
        // ---------- BEFORE ----------
        invokeAll(ctx, Phase.BEFORE, false);
        if (ctx.isStopped()) return ctx.getResult();

        // ---------- ON --------------
        if (!invokeAll(ctx, Phase.ON, true)) {
            // 未短路且未写 result → 调用内置 provider
            GenericProvider<I,O> gp = (GenericProvider<I,O>) providers.get(key(ctx));
            if (gp != null) gp.handle(ctx);
        }

        // ---------- AFTER -----------
        invokeAll(ctx, Phase.AFTER, false);

        return ctx.getResult();
    }

    /** @return true if chain was stopped */
    private boolean invokeAll(EventContextImpl<?,?> raw, Phase ph, boolean allowShortCircuit) {
        for (EventPlugin<?> plugin : registry.find(raw.getEntity(), raw.getEventType(), ph)) {
            Object typed = proxyFactory.asTyped(raw, plugin.expectedContextType());
            try {
                plugin.handle(typed);
            } catch (Throwable e) {
                raw.setThrowable(e);
                if (ph == Phase.ON) throw e;   // 冒泡回事务层
            }
            if (allowShortCircuit && raw.isStopped()) return true;
        }
        return false;
    }

    private String key(EventContextImpl<?,?> ctx) {
        return ctx.getEntity() + "#" + ctx.getEventType();
    }
}


---

4. 内置 Provider 如何被 Dispatcher 找到？

@Component
class ProviderRegistry implements ApplicationContextAware {

    Map<String, GenericProvider<?>> map = new ConcurrentHashMap<>();

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        ctx.getBeansOfType(GenericProvider.class).values()
           .forEach(p -> map.put(key(p.contextType()), p));
    }
    private String key(Class<? extends EventContext<?,?>> c) {
        // OrderCreateContext → "Order#CREATE"
        String entity = c.getSimpleName().replace("Context","")
                                         .replaceAll("(Create|Read|Update|Delete).*", "");
        EventType evt = /* parse after entity part */ ;
        return entity + "#" + evt;
    }
}

Dispatcher 注入 ProviderRegistry.map 即可在缺省情况下找到并调用。


---

5. 开发者插件到底怎么被调度？（时间线）

1. 启动阶段

Spring 扫描项目 + plugin JAR，实例化所有 EventPlugin、注入依赖。

PluginRegistry 读取 @Supports，按 entity/event/phase 填表。



2. 请求进入

ServiceAspect 构造 EventContextImpl<OrderCreateIn, OrderDTO>，phase=BEFORE。

EventDispatcher 查 Registry → 拿到 BEFORE 插件列表 [OrderPreparePlugin]。

逐个通过 ContextProxyFactory 生成 强类型代理 并调用 handle().



3. 插件执行

插件读/写 ctx.getData() 或 ctx.setResult()，也可 ctx.stop() 短路链。

若抛异常 → Dispatcher 捕获写入 ctx.setThrowable()，最终冒泡给事务与 Controller。



4. 链条继续

若 BEFORE 没 stop → 进入 ON 插件 / Provider → AF TER 插件 → 返回结果。





---

6. 插件隔离与健壮性（可选增强）

需求	实现建议

超时保护	invokeAll() 内使用 CompletableFuture.supplyAsync(...).get(timeout)
熔断	为每个插件持有独立 CircuitBreaker（resilience4j），连续失败后跳过调用
ClassLoader 隔离	动态 JAR 用子 URLClassLoader，且插件调用完成后不向外暴露类实例
插件级事务	BEFORE/ON 插件在同一 DB 事务内；AFTER 插件可声明 @Transactional(propagation=REQUIRES_NEW)



---

7. 一张小表回顾 “谁调用谁”

步骤	调用者	被调用者

1. 启动	Spring 容器	所有 EventPlugin → PluginRegistry.add()
2. 请求	ServiceAspect	EventDispatcher.dispatch()
3. BEFORE	EventDispatcher	[plugins] 列表
4. ON	EventDispatcher	[plugins] → GenericProvider (if needed)
5. AFTER	EventDispatcher	[plugins]
6. 返回	ServiceAspect	Controller / Client



---

▸ 结语

EventPlugin 被框架调用 = “启动期归档 + 请求期查表 + 运行时代理 + 顺序执行” 四步：

1. 归档：PluginRegistry 解析 @Supports 并按三元组存放。


2. 查表：Dispatcher 收到上下文后，用 (entity, event, phase) 找到插件列表。


3. 代理：将通用 EventContextImpl 封装成插件声明的强类型接口。


4. 执行：按 order 顺序同步/异步执行；可短路、可抛异常、可 timeout。



这样既 保留开发者零配置即可插桩的简单性，又 支持 SaaS 场景下的热插拔、租户隔离与故障熔断。

