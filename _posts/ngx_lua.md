---
layout: post
title: 与nginx绝配的编程语言lua
---

- **前言**

先来看一个简单的性能测试。都只是hello world,ngx_lua每秒请求10842多次（在一台几年前配的台式机上），而nginx用php-fpm，每秒只有55次。nginx静态文件（把文件放在/dev/shm内存文件中)每秒9246多次，golang用net/http，每秒8179次，nodejs每秒5090次，tinyhttpd静态文件（也放在内存中），每秒4394次。

同样是高级语言，php跟lua差距为啥这么大。

- **lua为啥这么快**
1. ***嵌入式***

lua是嵌入到nginx里面的，lua不是独立的进程，而是直接在nginx进程内部跑。而php是需要通过nginx连接php-fpm服务器，中间多了一层网络调用。这样的好处是nginx即业务，nginx本身速度很快，并发高，业务代码直接在nginx里面执行了，那么速度自然不会慢。

2. ***异步***

nginx为啥能这么快，其中一个因素是用了异步方式，避免了不必要的线程切换开销。同步方式需要创建很多线程，线程同步(访问临界资源)，线程切换开销很大。nginx只需要一个线程（实际会开几个worker，因为cpu是多核的，一个核一个），用来处理所有请求。lua嵌入nginx里面，也要支持异步方式，否则同步调用就把线程给阻塞了，而nginx一个cpu内核就1个线程，阻塞之后就不能处理其他请求了。这样会导致性能急剧下降。而lua支持异步编程，这是能跟nginx无缝结合的基础。比如mysql操作，php通过类库调用mysql api，这个操作是同步的，操作完成之后才会返回php代码继续执行。而lua不直接调用api，而是让nginx来调用api。openresty通过libdrizzle来调用mysql，这个是支持异步调用mysql的库，把mysql操作封装成nginx的一个location，通过子请求，ngx.location.capture来完成。这样在等到mysql操作完成的过程中，nginx无需等待，可以去处理其他请求。读取文件，memcache，redis都一样，都是通过nginx子请求来实现。这样lua代码中基本不会发生阻塞操作。性能自然是杠杠的。

3. ***一次加载，终身受用***

lua跟php一样，也是解释执行的，与php不同的是，lua对象只需要加载一次，编译一次，以后不需要每次编译。
     
    --Test.lua
    local i=0;
    function test()
    i=i+1;
    ngx.say(i)
    end
    
    --index.lua
    local test=require("Test");
    test.test();
    ngx.exit(200);
   
运行以上代码，每次执行都会不同，这说明Test.lua只加载了一次，当然这个是对于一个nginx进程来说的，一般ningx有多个worker，那么重复执行，会发生如下情况
root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3

22

root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3

23

root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3

24

root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3

1

root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3

25

root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3

26

root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3

1

root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3

2

root@ha23:~/tmp# curl 192.168.2.204:8090/lapp/test3


这就是多个worker引起的，每个work内存不共享的，是独立的。因此每个lua文件在每个worker都会加载一次。

php同样也可以测试下，会发现每次结果都是1，每个请求都会重新加载编译一边。

lua只加载一次的特性还有其他好处，比如初始化，某个业务可能需要初始化一堆数据，这个过程可能很耗时，但是不碍事，他只加载一边（每个worker）。比如加载一个字典，或者做中文分词的时候，会把字典加载进内存中，lua只需要封装到一个lua文件里面就好了。lua是支持面向对象的，就当中一个对象加载。加载一次，以后每次使用就很快了。

很久之前就用php写过中文分词的应用，太太慢了，其中一个因素是字典每次都要加载，每次加载几M的文件，性能可想而知。

4. ***luajit***

luajit可以编译成字节码，加快执行速度。曾经做过简单的测试，luajit大概是c的3倍。当然php也有一些优化手段。luajit，已经让lua足够快了，大部分业务足够。

5. ***c库***

lua可以跟c语言很好的交互，如果lua不够快，可以写用c语言写个库，通过lua调用即可。lua甚至可以直接调用c库。lua内部库很少，但是可以无缝使用c库。

    local test=require("Test");
  
  Test这个可以是Test.lua，也可以是一个c语言写的类。对用户来说是透明的。比如项目初期可以用lua快速完成初级迭代。需要的时候完全可以用c重写基础类，以便于提高性能。
  
  php的类据我目前的了解，没法用c语言写。可以用c重写函数，但是不支持类级别的重写。这样势必要修改代码。
  
