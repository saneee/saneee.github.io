---
layout: post
title: ngx_lua 变量内存解析
---

传统意义的变量分为全局变量，局部变量，临时变量，按照这个来区分lua变量并不合适。全局变量不一定全局。先来看个例子。

    --Test.lua某个类文件
    local i=0;--局部变量，仅在Test.lua里面有效
    function test()
    i=i+1;
    ngx.say(i);
    end

如果按照传统的理解，变量i为局部变量，每次执行test()，i都会加1，局部变量其实跟全局变量是一样的，只是作用域不同而已，实际也是保存在堆中的。如果是c语音，那么以上Test()函数就是对i进行加一，并且下次的时候继续加一，执行结果应该是1 2 3 4 5 ...

我们来看看实际执行结果：
root@ha23:~# curl 192.168.2.204:8090/lapp/test3<br>
1<br>
root@ha23:~# curl 192.168.2.204:8090/lapp/test3<br>
2<br>
root@ha23:~# curl 192.168.2.204:8090/lapp/test3<br>
3<br>
root@ha23:~# curl 192.168.2.204:8090/lapp/test3<br>
4<br>
root@ha23:~# curl 192.168.2.204:8090/lapp/test3<br>
1<br>
root@ha23:~# curl 192.168.2.204:8090/lapp/test3<br>
5<br>
在![与nginx绝配的编程语言lua]http://saneee.github.io/ngx_lua/ 一文说过，lua类文件只加载一次，但是一个nginx有多个进程（worker），每个进程是不共享内存的，是独立的。如果有4个worker，那么有4份Test类被加载进内存，就看请求是被哪个worker给处理的。因此，不能用静态变量（全局变量，局部变量）来保存全局的内容，只能保存只读的。 因此，用静态变量来代替memcache也是不行的，会发生冲突。

那么静态变量（一般不用全局变量，只用局部变量）能干嘛？

一是保存只读内容，比如字典。一个2M的字典，每个进程保存一份， 8个进程也就16M，对于现在来说这点内存开销不算啥，性能却能极大的提升。

二是作为缓存。比如经过很复杂的操作，就是为了获取一个对于关系，key,val，这种对应关系又不大变化，缓存不一致关系不大的情况下，可以使用。用来作为一个比memcache更快速的一个缓存。**切记这个缓存要加上时间限制，否则nginx几个月不重启，这个缓存有效期就是几个月**

我之前写了一个这样的类， 很少用到，仅供参考，os.time()最好用ngx.time代替。
      
      module("Seee", package.seeall)
      _VERSION = '1.00'
      local mt = { __index = Seee }
      
      local registry={}
      local timearr={}
      function set(k,v,t)
      	registry[k]=v;
      	if t then timearr[k]=os.time()+t end
      end
      function get(k)
      	local t=timearr[k];
      	if t and t<os.time() then
      		registry[k]=nil;
      		timearr[k]=nil;
      		return nil
      	end
      	return registry[k];
      end
      	
      
      getmetatable(Seee).__newindex = function (table, key, val)
          error('attempt to write to undeclared variable "' .. key .. '": '
                  .. debug.traceback())
      end

用静态变量做缓存有许多限制，我自己项目中用的比较少。

还有一种需要缓存的情况是，这个缓存只对当前请求有效，或者说在这个请求过程中不会发生变化，或者变化了关系也不大。举个例子，members表，这个表一般通过memcache缓存，mysql写。但是一个请求处理过程中，members表可能会多次用到，每次都去请求memcache，不是很有必要， 允许在处理当前用户请求的过程中，当前用户members表数据发现修改了还继续使用之前的缓存，大部分情况下这种冲突不会发生，而且大部分情况下即便冲突影响也不大。那么可以第一次取得memcache之后，保存这个值，下次要用直接返回。

这个变量就是ngx.ctx，nginx在处理请求的时候，每次会生成一个ngx.ctx变量，那么在业务代码上直接使用这个变量，就可以保证这个有效期是请求相关的。 那么members表可以用如下方式获取：

   --Member.lua
   local memc=require("Memcache");
   local sdo=require("Sdo");
   function get(userid)
    local res=ngx.ctx.member;
    if not res then 
      res=memc:getData("member"..userid);
      if not res then
        local sql="select * from members where userid='"..(userid+0).."'";
        local rows=sdo.fetch(sql);
        res=rows and rows[1];
        if  res then 
          memc:setData("member"..userid,res,3600);
        else
          ngx.log(ngx.ALERT,sql); 
        end
      end
      ngx.ctx.member=res;
    end
    
    return res;
  end
  
memcache，不能算严格意义上变量，但是可以当做变量，而且是支持全局唯一的“变量”，所有worker，所有业务服务器都共用一份变量。常见的就是上面说的members表缓存。memcache已经很流行了，就不具体啰嗦了。

主要讲讲memcache如何保持一致性。上面讲的静态变量缓存，ngx.ctx缓存，都是会不一致的。那么有时候更新的时候需要一致性，该如何解决呢。比如members表里面有个config项，可以做成json格式存储，这样要增加config选项就很轻松了，如果每个config对应members表里的一列，那么members表可能会经常更新。一般来说不常用的配置项可以用一个text类型来存储json。那么更新这个config的时候，如果2个请求同时更新就会冲突。比如a请求 config.a='111'`，b请求`config.b='222'`，a，b分别都获取了memcache缓存，都没有a,b配置，如原来的config是这样的`{c='333',d='444'}`,那么a或者b的更新就会丢失。这样的冲突显然是不允许的。我们允许读取到旧的配置，但是我们不允许配置丢失或者冲突。

要解决这个冲突，我们不用互斥量，不用锁，那些性能都太低了，可以在表里增加一个字段,ver，表示版本。那么以上更新代码如下：

    local res=member:get(userid);
    ...
    local ver=res.ver;
    local sql="update members set config="..sdo:escape(sql)..",ver=ver+1 where userid='"..(userid+0).."' and ver='"..ver.."'";
    local ret=sdo:update(sql);
    if ret.errcode==myconst.MATCH_NONE then
    --更新失败，ver发生了变化，重新读取res。
    ...
    end
    
