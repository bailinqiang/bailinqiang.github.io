---
title: classCastExceptionInGeneric
date: 2019-03-18
tags: []
---

### 例子

简单代码如下1
2
3
4
5
6
7
8
9
public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException &#123;
    List<String> aa = getNames();
    System.out.println(aa);
    for (String s : aa) &#123;
        System.out.println(s);
    &#125;
&#125;
// 先不管getNames的 具体实现，可以当成是rpc调用返回结果
// 或者 List<String> 作为rpc调用的参数。无需关心对方是如何设置这个值的

报错信息

1
2
3
[1, 2]   // 这个是第一行打印  可以正确的展示出来，并行显示的类型不是String
Exception in thread "main" java.lang.ClassCastException: java.lang.Integer cannot be cast to java.lang.String
	at com.moon.switchExplore.ListGenericForBlog.main(ListGenericForBlog.java:20)

那么问题来了 1、为啥 List<String> 可以存放 Integer 呢  2、List<String> 是如何 add(Integer) 的呢 3、强转发生在什么时候?

### 接下来一个一个来探索

泛型的作用范围， 一句话 泛型只在编译阶段有效。在运行期会被擦除  那么就可以解释 List<String> 存放Integer的值了，因为只在编译阶段有效。只要通过编译即可。或者骗一骗编译器。接下来是第二个问题 List<String> 怎么add Integer
骗过泛型

1
2
3
4
5
6
7
8
9
10
11
12
13
List<String> aa = new ArrayList<>();
aa.add(1);  // 这么写 编辑器是会报错的，并行编译阶段会报错 javac时  会报参数不匹配
// 那怎么绕过检查呢  其实就是反射调用  例子

Method method = aa.getClass().getDeclaredMethod("add", Object.class);
method.setAccessible(true);
method.invoke(aa, 1);
method.invoke(aa, 2);
/*
* 这里有一点值得注意下  getDeclaredMethod 的时候 第二个参数要填，必须填Object.class 这是为什么呢 ？ 我们看下 字节码中 List.add方法
* 可以找到下面这句  能够直观的看出来 在运行期  add 方法接受的参数是 Object 所以  反射调用的必须是Object.class
*/
InterfaceMethodref #34.#35        // java/util/List.add:(Ljava/lang/Object;)Z

强转什么时候发生1
33: checkcast     #8                  // class java/lang/String

查看字节码发现 checkcast这个指令 这个指令去执行类型转换，结果Integer抓换String  不符合jvm规范 所以报错了

### 参考

查看字节码  javap -v (javac 编译后的 class 文件)
jvm 关于checkcast指令的规定
