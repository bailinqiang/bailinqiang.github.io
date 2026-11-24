---
title: java编译器优化指令执行顺序
date: 2019-04-05
tags: []
---

随着现在jvm的改进，为了加速在指令上都有一些优化，话不多说 直接看例子
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
14
15
16
17
18
19
20
21
22
23
24

static boolean status = false;
static int number = 0;

public static void main(String[] args) &#123;
    new Thread()&#123;
            @Override
            public void run() &#123;
                number = 100;
                status = true;
            &#125;
        &#125;.start();

    new Thread() &#123;
        @Override
        public void run() &#123;
            if (status) &#123;
                System.out.println("status is true number is "+ number);
            &#125; else &#123;
                System.out.println("number is " + number);
            &#125;
        &#125;
    &#125;.start();
&#125;

这段代码，我们咋一看 由于线程start时机不受我们程序控制。那么可能限制性 第一个线程，可能先执行第二个线程，或者两个一起执行。
先不管这个，反正根据竞态条件，第一感觉有两种print:
1、status is true number is 100;
2、number is false;
3、numbr is 100;
那么问题来了，实际执行过程中会存在第四种情况 4、status is true  number is false;
这是为什么呢，其实是jvm判断第一个线程执行的时候，根据语义 指令number = 100 和指令status = true  不存在before and happend关系，就可以进行指令的重排序，
那么 就会出现 可能线程1 先执行status = true,在执行number = 100,但是在这两个指令过程中间，线程2执行了，所以会出现第四中情况。
结果请看代码 (第四种属于比较不常见现象，所以多试几次才会出现，当然代码写的low一些，不过道理是这么个道理)
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
14
15
16
17
18
19
20
21
22
23
24
25
26
27
public static void main(String[] args) throws InterruptedException &#123;
    for (int i = 0; i < 2000; i++) &#123;
        // 每次 main函数初始化，，当然最好的做法是用原子 操作。final
        status = false;
        number = 0;
        new Thread()&#123;
            @Override
            public void run() &#123;
                number = 100;
                status = true;
            &#125;
        &#125;.start();

        new Thread() &#123;
            @Override
            public void run() &#123;
                if (status) &#123;
                    System.out.println("status is true number is "+ number);
                &#125; else &#123;
                    System.out.println("number is " + number);
                &#125;
            &#125;
        &#125;.start();
        // 休息2ms  保证下次循环不会修改本次的结果
        Thread.sleep(2);
    &#125;
&#125;

结果我就贴图吧
