---
layout:     post
title:      "Animal Sniffer: JVM 上的 API 检查器"
subtitle:   "For Compatibility and Stability"
date:       2018-09-27 13:48:00
author:     "Hexi"
header-img: "img/bg/2018-09-27-animal-sniffer-bg.jpg"
tags:
    - JVM
    - compatibility
    - API
---

### 起因

最近给 `retrofit` 写一个子项目 [`retrofit-processors`](https://github.com/CTKnight/retrofit/tree/master/retrofit-processors)时遇到了一个问题：我在父项目根目录下执行 `mvn compile` 非常正常，然后 `mvn test` 炸了，报的错是

```
Undefined reference: boolean javax.lang.model.element.ExecutableElement.isDefault()
```

`Undefined reference`？？？，那我怎么编译过的？而且 `boolean javax.lang.model.element.ExecutableElement.isDefault()` 难道不是 `Java8` 的 API 咩？

折腾了半天才发现这是因为父项目 `pom.xml` 里面的插件：

```xml
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>animal-sniffer-maven-plugin</artifactId>
  <version>${animal.sniffer.version}</version>
  <executions>
    <execution>
      <phase>test</phase>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <signature>
      <groupId>org.codehaus.mojo.signature</groupId>
      <artifactId>java16</artifactId>
      <version>1.1</version>
    </signature>
  </configuration>
</plugin>
```

这玩意儿是什么意思呢？

大概意思就是在 `maven lifecycle` 的 `test` 阶段检查我所使用的部分 API 是否与 `signature` 中所签名的 `API` 是否兼容，`org.codehaus.mojo.signature.java16:1.1` 显然就是指 `JRE-1.6` 的 `API signature` 了。

而我在子项目中使用 `boolean javax.lang.model.element.ExecutableElement.isDefault()` 这个 `Java8` 才有的 `API` 导致 `test fail`。

### Animal Sniffer

就这样在机缘巧合之下我看了下 Animal Sniffer 的[文档](http://www.mojohaus.org/animal-sniffer/animal-sniffer-maven-plugin/)：它是 maven 上的一个插件，虽然整个项目有点凉（ GitHub 上只有三十个星星），但看起来挺实用的。

Animal Sniffer 一共提供了两个功能，[`build`](http://www.mojohaus.org/animal-sniffer/animal-sniffer-maven-plugin/examples/generating-java-signatures.html) 和 [`check`](http://www.mojohaus.org/animal-sniffer/animal-sniffer-maven-plugin/examples/checking-signatures.html)；`build` 就是根据一个项目生成它的 `API signature`；`check` 很显然是根据配置的 `signature` 对项目进行检查。

#### Build

Animal Sniffer 既可以 `build JRE` 的 `API signature` 也可以 `build` 其它的 `APIs`，详情见文档，本文不作介绍。

- [`JRE API`](http://www.mojohaus.org/animal-sniffer/animal-sniffer-maven-plugin/examples/generating-java-signatures.html)
- [`Other APIs`](http://www.mojohaus.org/animal-sniffer/animal-sniffer-maven-plugin/examples/generating-other-api-signatures.html)


#### Check

在 `pom.xml` 中添加配置

```xml
<project>
  ...
  <build>
    ...
    <plugins>
      ...
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>animal-sniffer-maven-plugin</artifactId>
        <version>1.16</version>
        ...
        <configuration>
          ...
          <signature>
            <groupId>___group id of signature___</groupId>
            <artifactId>___artifact id of signature___</artifactId>
            <version>___version of signature___</version>
          </signature>
          ...
        </configuration>
        ...
      </plugin>
      ...
    </plugins>
    ...
  </build>
  ...
</project>
```

然后你可以选择

- 直接执行命令 `mvn animal-sniffer:check`
- 在 maven 生命周期中的某个阶段进行检查，未通过检查会导致构建失败。配置如下

```xml
<project>
  ...
  <build>
    ...
    <plugins>
      ...
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>animal-sniffer-maven-plugin</artifactId>
        <version>1.16</version>
        <executions>
          ...
          <execution>
            <id>___id of execution___</id>
            ...
            <phase>test</phase>
            ...
            <goals>
              <goal>check</goal>
            </goals>
            ...
          </execution>
          ...
        </executions>
      </plugin>
      ...
    </plugins>
    ...
  </build>
  ...
</project>

```

当然，有些情况下我们应当忽略对一些类或者方法的检查，多见于项目代码中的一些逻辑针对不同版本 API 有不同的实现。如 `retrofit` 项目本身实现了一个 `PlatForm` 工厂类

```java
class Platform {
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }

  private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }

  @Nullable Executor defaultCallbackExecutor() {
    return null;
  }

  CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    if (callbackExecutor != null) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    return DefaultCallAdapterFactory.INSTANCE;
  }

  boolean isDefaultMethod(Method method) {
    return false;
  }

  @Nullable Object invokeDefaultMethod(Method method, Class<?> declaringClass, Object object,
      @Nullable Object... args) throws Throwable {
    throw new UnsupportedOperationException();
  }

  @IgnoreJRERequirement // Only classloaded and used on Java 8.
  static class Java8 extends Platform {
    @Override boolean isDefaultMethod(Method method) {
      return method.isDefault();
    }

    @Override Object invokeDefaultMethod(Method method, Class<?> declaringClass, Object object,
        @Nullable Object... args) throws Throwable {
      // Because the service interface might not be public, we need to use a MethodHandle lookup
      // that ignores the visibility of the declaringClass.
      Constructor<Lookup> constructor = Lookup.class.getDeclaredConstructor(Class.class, int.class);
      constructor.setAccessible(true);
      return constructor.newInstance(declaringClass, -1 /* trusted */)
          .unreflectSpecial(method, declaringClass)
          .bindTo(object)
          .invokeWithArguments(args);
    }
  }

  static class Android extends Platform {
    @IgnoreJRERequirement // Guarded by API check.
    @Override boolean isDefaultMethod(Method method) {
      if (Build.VERSION.SDK_INT < 24) {
        return false;
      }
      return method.isDefault();
    }

    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
      if (callbackExecutor == null) throw new AssertionError();
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
}

```

里面就针对不同的 `JRE` 版本分别实现了 `boolean isDefaultMethod(Method method)`，其中当 `JRE` 版本为 `1.8` 时不可避免地要用到只有 `JRE-1.8` 才有的 `boolean java.lang.reflect.Method.isDefault()` 。

这时可以看到上面有一些方法用了 `@IgnoreJRERequirement`，这个 `annotation` 来自 `org.codehaus.mojo.animal-sniffer-annotations`，可以让类或者方法逃过 API 签名检查。

当然也可以在 `pom.xml` 里配置忽略：

```xml
<project>
  ...
  <build>
    ...
    <plugins>
      ...
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>animal-sniffer-maven-plugin</artifactId>
        <version>1.16</version>
        ...
        <configuration>
          ...
          <ignores>
            ...
            <ignore>java.lang.reflect.Method</ignore>
            ...
          </ignores>
          ...
        </configuration>
        ...
      </plugin>
      ...
    </plugins>
    ...
  </build>
  ...
</project>
```

另外，Animal Sniffer 默认是不检查依赖项目的，如果要确保你的 API 版本跟 `JRE` 版本兼容，可能需要检查依赖项，即在 `pom.xml` 加入

```xml
<project>
  ...
  <build>
    ...
    <plugins>
      ...
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>animal-sniffer-maven-plugin</artifactId>
        <version>1.16</version>
        ...
        <configuration>
          ...
          <ignoreDependencies>false</ignoreDependencies>
          ...
        </configuration>
        ...
      </plugin>
      ...
    </plugins>
    ...
  </build>
  ...
</project>
```

### 后记：我的解决方案

说完了 Animal Sniffer，我们回到最开始的问题，由于我要写的是 `annotation processors`，所以并不需要兼容 `JRE-1.8` 之前版本的 `API`，所以最后在子项目的 `pom.xml` 里面加上了

```xml
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>animal-sniffer-maven-plugin</artifactId>
  <version>${animal.sniffer.version}</version>
  <executions>
    <execution>
      <phase>test</phase>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <signature>
      <groupId>org.codehaus.mojo.signature</groupId>
      <artifactId>java18</artifactId>
      <version>1.0</version>
    </signature>
  </configuration>
</plugin>
```

使用 `JRE-1.8` 的 `API signature` 覆盖了父项目的的 `JRE-1.6`，测试通过

![TEST SUCCESS](/img/in-post/2018-09-27/test-success.png){:class="img-responsive"}


