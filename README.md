
**大纲**


**1\.InnoDB引擎架构**


**2\.Buffer Pool**


**3\.Page管理机制之Page页分类**


**4\.Page管理机制之Page页管理**


**5\.Change Buffer**


**6\.Log Buffer**


 


**1\.InnoDB引擎架构**


**(1\)InnoDB引擎架构图**


**(2\)InnoDB内存结构**


 


**(1\)InnoDB引擎架构图**


下面是InnoDB引擎架构图，主要分为内存结构和磁盘结构两大部分。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/69656885bcb847b1be1d745dbab93240~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=i5kioxMjshVbnGbGurx66GSp14E%3D)
**(2\)InnoDB内存结构**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bc0c30ce75e34ab69e47f2b01fe0f157~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=yRVunNJZgdtJHADDLbuWO1%2FOHmA%3D)
 


**2\.Buffer Pool**


**(1\)Buffer Pool基本概念**


**(2\)如何判断数据页是否缓存在Buffer Pool**


 


**(1\)Buffer Pool基本概念**


Buffer Pool是缓冲池的意思。Buffer Pool的作用是缓存表数据与索引数据，减少磁盘IO，提升效率。


 


Buffer Pool由缓存数据页(Page)和对缓存数据页进行描述的控制块组成。控制块存储着缓存页的表空间、数据页号、在Buffer Pool中的地址等。


 


Buffer Pool默认大小是128M，以Page页为单位，Page页默认大小16K。而控制块的大小约为数据页的5%，大概是800字节。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/82bb606d543c408cafe83b3aaa7b34c9~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=TmMfqui87GTeEzAenWGU%2B%2FC1RXI%3D)
注意：Buffer Pool大小为128M指的就是缓存页的总大小。控制块则一般占5%，所以每次会多申请6M的内存空间用于存放控制块。


 


**(2\)如何判断数据页是否缓存在Buffer Pool**


MySQl中有一个哈希表数据结构：key是表空间号 \+ 数据页号，然后value就是缓存页对应的控制块。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/7971291c0c13494b8fd7a09692ea1078~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=x7m68qd4L9jChnmbsZRcDVZl%2FoY%3D)
当需要访问某个页的数据时：会先从哈希表中根据表空间号 \+ 页号看看是否存在对应的缓存页。如果有，则直接使用。如果没有，则从Free链表中选出一个空闲的缓存页，然后把磁盘中对应的页加载到该缓存页的位置。


 


**3\.Page管理机制之Page页分类**


Buffer Pool的底层采用链表数据结构管理Page。在InnoDB访问表记录和索引时会在Buffer Pool中的Page页缓存，以后使用同样的表记录和索引时，就可以减少磁盘IO操作，提升效率。


 


Page根据状态可以分为三种类型：


一.Free Page：空闲Page，未被使用。


二.Clean Page：被使用Page，没被修改。


三.Dirty Page：被使用Page，已被修改。


脏页中的数据和磁盘的数据产生了不一致。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/072b0b5aa3dc474c82be628f60d26034~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=9RNjKLxHzCBAMcf7FuTP8Sl1kfY%3D)
 


**4\.Page管理机制之Page页管理**


**(1\)Free List空闲缓冲区**


**(2\)Flush List需刷盘的缓冲区**


**(3\)LRU List正在使用的缓冲区**


 


针对上面的三种Page类型，InnoDB会通过三种链表结构来维护和管理。


 


**(1\)Free List表示空闲缓冲区(管理Free Page)**


**一.Free链表的初始化**


Buffer Pool初始化时会先向操作系统申请连续的内存空间，然后把它划分成若干个控制块\&缓存页，接着把所有空闲的缓存页对应的控制块作为节点放到一个链表中，这个链表就是Free链表。


 


**二.Free链表的基节点**


Free链表中只有一个基节点是不记录缓存页信息的(单独申请空间)，基节点存放了Free链表的头节点地址、尾节点地址和节点个数。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/148bbf955dc045ebafa88d040c5077aa~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=5xTfVZRIbT9pIr6XKsvaXPtYpeA%3D)
**三.从磁盘加载数据页的流程**


步骤1：首先从Free链表中取出一个空闲的控制块(对应缓存页)。


 


步骤2：然后把该缓存页对应的控制块信息填上，如缓存页所在的表空间、数据页号之类的信息。


 


步骤3：接着把该缓存页对应的Free链表节点(即控制块)从链表中移除，表示该缓存页已经被使用了。


 


**(2\)Flush List表示需要刷新到磁盘的缓冲区(管理Dirty Page)**


Flush List管理的Dirty Page会按修改时间排序。InnoDB引擎为了提高处理效率，在每次修改缓存页后，并非立刻把修改刷新到磁盘上，而是在未来某个时间点进行刷新操作。


 


凡是被修改过的缓存页对应的控制块都会作为节点加入到Flush链表，Flush链表的结构与Free链表的结构相似。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/1000f473b2c54fe5aefc8f986580f3a1~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=KuooRcygjawVfuElUNAIm1q539o%3D)
脏页既存在于Flush链表中，也存在于LRU链表中，但它们互不影响。LRU链表负责管理Page的可用性和释放，Flush链表负责管理脏页的刷盘操作。


 


**(3\)LRU List表示正在使用的缓冲区(管理Clean Page和Dirty Page)**


**一.普通LRU算法**


**二.普通LRU链表的优缺点**


**三.改进型LRU算法**


**四.冷数据区的数据页何时会被转到到热数据区**


 


缓冲区以midpoint为基点：链表的前面部分称为热数据列表，存放经常访问的数据，占63%。链表的后面部分称为冷数据列表，存放使用较少数据，占37%。


 


**一.普通LRU算法**


LRU \= Least Recently Used(最近最少使用)，就是末尾淘汰法。新数据从链表头部加入，释放空间时从末尾淘汰。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/adf23020b6654fbf97e6a212c1ce1f9b~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=zJ%2BNrajNBtARSrSX6FrunwQMrK4%3D)
步骤1：当要访问某个不在Buffer Pool中的数据页时，就把该数据页加载到Buffer Pool，并且把其缓存页对应的控制块作为节点添加到LRU链表的头部。


 


步骤2：当要访问某个在Buffer Pool中的数据页时，就把其缓存页对应的控制块移动到LRU链表头部。


 


步骤3：当需要释放空间时，从最末尾淘汰。


 


**二.普通LRU链表的优缺点**


**优点：(热数据最快被获取)**


所有最近使用的数据都在链表表头，最近未使用的数据都在链表表尾，可以保证热数据能最快被获取到。


 


**缺点：(全表扫描 \+ 预读机制)**


如果发生全表扫描，则可能将真正的热数据淘汰。由于MySQL中存在预读机制，很多预读的页会被放到LRU链表的表头。如果这些预读的页没用到，那么就可能会导致真正的热数据在尾部被淘汰。全表扫描的发生场景是：没有建立合适的索引或查询时使用select \*等。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/3ba5f2c065ac4b98a2171de3083f11e3~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=grVUDgkEthAo6kyCb7MewHkxUAY%3D)
**三.改进型LRU算法**


改进的LRU链表分为热数据和冷数据两个部分。往LRU链表加入元素时并不从表头插入，而是从中间midpoint位置插入，也就是从磁盘中新读出的数据会放在冷数据区的头部。


 


如果数据很快被访问，那么Page就会向热数据列表头部移动；如果数据没有被访问，那么Page会逐步向冷数据列表尾部移动，等待淘汰。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4492d1d39e3342aa8051818af8819987~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=lMfwFv1e2a6RxfWh398gEnTHPtA%3D)
**四.冷数据区的数据页何时会被转到到热数据区**


在对某个处于冷数据列表的缓存页进行第一次访问时，就会在它对应的控制块中记录下这个访问时间。


 


如果后续的访问时间与第一次访问的时间在某个时间间隔内，那么该缓存页就不会从冷数据列表移动到热数据列表的头部，否则就将该缓存页从冷数据列表移动到热数据列表的头部，从而避免全表扫描带来的访问频率很低但占用大量缓存页的问题。


 


这个间隔时间由innodb\_old\_blocks\_time控制，默认是1s。这也就意味着，对于从磁盘加载到LRU链表冷数据列表的缓存页来说，如果第一次和最后一次访问的时间间隔小于1s，则不会加入热数据列表。


 


**5\.Change Buffer**


**(1\)Change Buffer基本概念**


**(2\)Change Buffer的数据更新流程**


**(3\)为什么写缓冲区仅适用于二级索引页**


**(4\)什么情况下会进行merge**


**(5\)Change Buffer的使用场景(写多读少)**


 


**(1\)Change Buffer基本概念**


Change Buffer是写缓冲区，用于优化对二级索引(辅助索引)页的更新。


 


对于DML操作：如果请求的是辅助索引(非唯一键索引)且没有在缓冲池中，那么不会立刻将数据页加载到Buffer Pool中，而是会先在Change Buffer中记录数据的变更，等未来数据被读取时再将数据合并恢复放到Buffer Pool，以减少磁盘IO。


 


Change Buffer默认占Buffer Pool空间25%，最大占50%。可根据读写业务量进行调整，参数是innodb\_change\_buffer\_max\_size。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d0b5b4ec4235424886088b28439c6288~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=mKTnUJkin28Jr7byQ0cYyFLyHH8%3D)
**(2\)Change Buffer的数据更新流程**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bfcf1eef62df4fe2ac07ebe9a7df2074~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=YmMkKrb5kmGipBh9Vty5RNfisBg%3D)
**情况1：**


对于唯一索引，需要将数据页读入内存。然后判断没有冲突才插入更新值，语句执行结束。


 


**情况2：**


对于普通索引，更新一条记录时，步骤如下：


步骤一：如果该记录在Buffer Pool中存在，那么就直接在Buffer Pool中修改，进行一次内存操作。


步骤二：如果该记录在Buffer Pool中不存在(没有命中)，那么在不影响数据一致性的前提下，InnoDB会将该记录的更新操作缓存在Change Buffer中，先不用去磁盘查询数据，从而避免一次磁盘IO。


步骤三：当下次查询该记录时，InnoDB才会将数据页读入内存，然后执行Change Buffer中与该记录有关的操作。


 


**(3\)为什么写缓冲区仅适用于二级索引页**


如果新增或修改发生在唯一索引中，那么InnoDB必须要做唯一性校验。此时就必须查询磁盘，进行一次IO操作。也就是会直接将记录查询到Buffer Pool中，然后在缓冲池修改，不需要在Change Buffer操作了。


 


如果新增或修改发生在非索引中，那么InnoDB还是要做唯一性校验。此时也必须查询磁盘，进行一次IO操作。


 


**(4\)什么情况下会进行merge**


将Change Buffer中数据的变更应用到原数据页的过程称为merge。Change Buffer上的缓存数据是可以持久化的，以下情况会进行持久化：


一.访问这个数据页会触发merge


二.系统有后台线程会定期merge


三.在数据库正常关闭的过程中也会执行merge


 


**(5\)Change Buffer的使用场景**


Change Buffer的主要目的是将记录的变更操作缓存下来，所以在merge发生前应当尽可能多的缓存变更信息，这样Change Buffer的优势发挥得就越明显。


 


应用场景是写多读少的业务。此时页面在写完后马上被访问的概率较小，Change Buffer使用效果最好。这种业务模型常见的就是账单类、日志类的系统。


 


**6\.Log Buffer**


**(1\)Log Buffer的作用**


**(2\)Log Buffer的刷盘策略**


**(3\)Adaptive Hash Index**


 


**(1\)Log Buffer的作用**


Log Buffer指的是日志缓冲区。


 


Log Buffer是用来保存要写入磁盘log文件(Redo/Undo)的log数据。Log Buffer可以优化每次更新操作后要写文件而产生的磁盘IO问题，因为每次更新操作都是需要写log到redo log和undo log磁盘文件的。


 


Log Buffer日志缓冲区的内容会定期刷新到磁盘log文件中，Log Buffer日志缓冲区满时会自动将其刷新到磁盘。当遇到BLOB或多行更新的大事务时，增加日志缓冲区可节省磁盘IO。


 


Log Buffer主要是用于记录InnoDB引擎日志，InnoDB在DML操作时会产生redo和undo日志。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/3b9af300a53242a68df908aeb261a5d8~tplv-obj.image?lk3s=ef143cfe&traceid=20241128215008658804E4DA13AB5044A5&x-expires=2147483647&x-signature=gaygw%2Fu8smccRDai6ir7orv24G4%3D)
Log Buffer空间满了，会自动写入磁盘，默认16M。可以通过将innodb\_log\_buffer\_size参数调大，减少磁盘IO频率。


 


**(2\)Log Buffer的刷盘策略**


innodb\_flush\_log\_at\_trx\_commit参数控制日志刷新行为，默认为1。


 


**一.innodb\_flush\_log\_at\_trx\_commit \= 0**


每隔1秒写日志文件Log Buffer和刷盘操作，最多丢失1秒数据。写日志文件Log Buffer \-\> OS cache \-\> 刷盘OS Cache \-\> 磁盘文件。


 


**二.innodb\_flush\_log\_at\_trx\_commit \= 1**


事务提交，立刻写日志文件和刷盘，数据不丢失，但是会频繁IO操作。


 


**三.innodb\_flush\_log\_at\_trx\_commit \= 2**


事务提交，立刻写日志文件Log Buffer，每隔1秒钟进行刷盘操作。


 


**(3\)Adaptive Hash Index**


自适应哈希索引，用于优化对Buffer Pool数据的查询，InnoDB存储引擎会监控对表索引的查找。


 


自适应哈希索引指的是：如果观察到建立哈希索引可以带来速度的提升，则建立哈希索引。InnoDB存储引擎会自动根据访问的频率和模式来为某些页建立哈希索引。


 


 本博客参考[楚门加速器](https://chuanggeye.com)。转载请注明出处！
