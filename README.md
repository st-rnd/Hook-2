[![](https://jitpack.io/v/renaudcerrato/Hook.svg)](https://jitpack.io/#renaudcerrato/Hook) [![Android Arsenal](https://img.shields.io/badge/Android%20Arsenal-Hook-green.svg?style=true)](https://android-arsenal.com/details/1/3959)

# Hook

Minimalist, annotation based, hook framework for Android built on top of [AspectJ](https://eclipse.org/aspectj/). 

* [Basic Usage](#basic-usage)
* [Annotations](#annotations)
    * [@Hooked](#hooked)
    * [@Hook](#hook-1)
    * [@Call](#call)
    * [@Before](#before)
    * [@After](#after)
    * [@Returning](#returning)
    * [@Register/@Unregister](#register--unregister)
* [Advanced Usage](#advanced-usage)
* [Installation](#installation)
* [Proguard](#proguard)

# Basic Usage

```java

public class Math {

    @Hooked("my_hook")
    public int add(int a,int b) {
        return a+b;
    }
}

public class IncrementHook {

    @Hook("my_hook")
    int hook(HookedMethod<Integer> method) throws Throwable {
        return method.proceed() + 1; // add one 
    }
}
    
final Math math = new Math();
final IncrementHook hook = new IncrementHook();

Hooker.instance().register(hook);
assertThat(math.add(5, 3), is(9)); // 5 + 3 = 9
Hooker.instance().unregister(hook);
assertThat(math.add(5, 3), is(8)); // 5 + 3 = 8    

```

# Annotations

Hook let you easily hook into methods using simple annotations:

## @Hooked

In order to be able to hook a method, you must annotate it as `@Hooked`.

By annotating arguments using `@Param` annotations, you'll be later able to easily capture them by name (instead of relying on their declaration order) :

```java
	@Hooked("my_hook")
    public int add(@Param("a") int a, @Param("b") int b) {
        return a+b;
    }
```

## @Hook

By using `@Hook` annotations, you're given a chance to alter arguments and/or the return value of the original `@Hooked` method. `@Hook` annotated methods have a mandatory first arguments of type [`HookedMethod`](https://github.com/renaudcerrato/Hook/blob/master/runtime/src/main/java/com/mypopsy/hook/HookedMethod.java). 

You can easily capture arguments by using `@Param` annotations and you can also capture the enclosing object of the `@Hooked` method using the `@Target` annotation:

```java
	@Hook("my_hook")
    int hook(HookedMethod<Integer> method, @Param("a") int a, @Param("b") int b, @Target Math math) throws Throwable {
        return method.proceed(2*a, 2*b); // double operands
    }
```

Multiple `@Hook` method can be defined, and you can specify the execution order using the `priority` parameter:

```java
	@Hook(value="my_hook", priority=1000)
    int highPriorityHook(HookedMethod<Integer> method) throws Throwable {
        return method.proceed();(
    }
    
	@Hook(value="my_hook", priority=1)
    int lowPriorityHook(HookedMethod<Integer> method) throws Throwable {
        return method.proceed();
    }
```

## @Call

If you don't mind altering arguments nor the return value, you can use `@Call` annotations instead. Since `@Call` methods will be invoked first (i.e before any `@Hook` annotated methods), you can be sure that arguments have not been altered.

```java
 	@Call("my_hook")
    void call(int a, int b) {
        ...
    }
```

Both `@Param` and `@Target` capture annotations are supported.

## @Before

`@Before` annotated methods are called right **before** the original method is called - that mean that both `@Call` and `@Hook` method(s) were called already (possibly altering arguments). Of course, `@Before` methods **won't be called** if a single `@Hook` method didn't called `HookedMethod::proceed()`. 

```java
	@Before("my_hook")
    void before(int a, int b) {
        ...
    }
```

Both `@Param` and `@Target` capture annotations are supported.

## @After

`@After` annotated methods are called right **after** the original method is called - but before `@Hook` annotated methods returns. Of course, `@After` methods **won't be called** if a single `@Hook` method didn't called `HookedMethod::proceed()`.

You can capture the return value using the `@Result` annotation:

```java
 	@After("my_hook")
    void after(@Param("a") int a, @Param("b") int b, @Target Object object, @Result int result) {
        ...
    }
```    

Both `@Param` and `@Target` capture annotations are also supported.

## @Returning

`@Returning` annotated methods are called right **before** returning to the original caller. You can capture the return value using the `@Result` annotation:

```java
	@Returning("my_hook")
    void returning(@Param("a") int a, @Param("b") int b, @Result int result) {
        ...
    }
```

Both `@Param` and `@Target` capture annotations are also supported.

## @Register / @Unregister 

`@Register` annotated methods will automatically register the enclosing instance **right before** execution, exactly in the same way as if `Hooker.instance().register(this)` were called on the instance.

`@Unregister` annotated methods will automatically unregister the enclosing instance **right after** execution, exactly in the same way as if `Hooker.instance().unregister(this)` were called on the instance.

```java
public class MainActivity extends Activity {

    @Register
	@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Unregister
    @Override
    protected void onDestroy() {
        super.onDestroy();
    }
    ...
}
```

# Advanced Usage

```java

public class Math {

    @Hooked("my_hook")
    public int add(@Param("a") int a, @Param("b") int b) {
        System.out.println("    calling add("+a+", "+b+")");
        return a+b;
    }
}

public class Hooks {

	@Before("my_hook")
    void before(int a, int b) {
        System.out.println("  @Before(a="+a+", b="+b+")");
    }

    @Call("my_hook")
    void call(int a, int b) {
        System.out.println("@Call(a="+a+", b="+b+")");
    }

    @After("my_hook")
    void after(@Param("a") int a, @Param("b") int b, @Target Math math, @Result int result) {
        System.out.println("  @After(a="+a+", b="+b+", result="+result+")");
    }

    @Returning("my_hook")
    void returning(@Param("a") int a, @Param("b") int b, @Result int result) {
        System.out.println("@Returning(a="+a+", b="+b+", result="+result+")");
    }

    @Hook("my_hook")
    int hook(HookedMethod<Integer> method, @Param("a") int a, @Param("b") int b, @Target Math math) throws Throwable {
        System.out.println("entering @Hook(a="+a+", b="+b+")");
        int ret = method.proceed(2*a, 2*b) + 1; // double operands and increment result
        System.out.println("exiting @Hook (return "+ret+")");
        return ret;
    }
}

final Math math = new Math();
final Hooks hooks = new Hooks();

Hooker.instance().register(hooks);
assertThat(math.add(5, 3), is(17));
Hooker.instance().unregister(hooks);
```

For clarity, here's the output of the snippet above :

```
@Call(a=5, b=3)
entering @Hook(a=5, b=3)
  @Before(a=10, b=6)
    calling add(10, 6)
  @After(a=10, b=6, result=16)
exiting @Hook (return 17)
@Returning(a=10, b=6, result=17)
```


# Installation

## Android Studio

The project binaries are hosted on [JitPack](https://jitpack.io): the Android integration is made easy thanks to the use of a custom gradle plugin.

**Step 1** Import the plugin in your root build.gradle at the `buildscript` closure:

```gradle
buildscript {
    repositories {
        ...
        maven { url "https://jitpack.io" }
    }
    dependencies {
        ...
        classpath 'com.github.renaudcerrato.Hook:plugin:1.0.0'
    }
}
```

**Step 2** Add Jitpack in your root build.gradle at the end of repositories:
```gradle
allprojects {
	repositories {
		...
		maven { url "https://jitpack.io" }
	}
}
```

**Step 3** Apply the plugin in your app build.gradle:

```gradle
apply plugin: 'com.android.application'
apply plugin: 'hook'
...
```

# Proguard

Make sure your proguard rule set includes following lines:

```proguard
-keep class com.mypopsy.hook.** { *; }

-keepclasseswithmembernames class * {
    @com.mypopsy.hook.** <methods>;
}
```
# License

```
Copyright 2016 Cerrato Renaud

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

