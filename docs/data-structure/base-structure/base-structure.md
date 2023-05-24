**HashMap（数据+链表）**，Node结构（hash,key,value,nextNode） value组成entity对象存放在bukcket数组中，若hash发生碰撞以链表形式链接在相同index后，链表长度超过8，转成红黑树. 初始size:16扩容：2n

**HashTable:（数据+链表**） 跟hashmap类似，主要区别：线程安全，初始size:11,扩容2n+1

**ConcurrentHashMap**:采用数据分段锁，提高效率。entity对象value使用volatile修饰保证可见性，读取时不需要加锁。Segment继承ReentrantLock 并且包含一段数据实现分段，单个Segment内部扩容

**ArrayMap**：数组 单独数组存放hash，单独数组存放key,value（二者存放在同一个数组），key有序存放，采用二分查找

**TreeMap**：树，Node结构（key,value,left,right,parent) 默认按key升序，可以自定义比较器

**LinkedHashMap**：hashmap基础上增加双向链表 重写hashmap put remove等空回调，同时对双向链表保证有序性。重写keyset interator保证遍历时的有序,accessOrder设置为true的时候，在进行get和put时，将操作元素移动到链表尾部

**HashSet**：采用hashmap同样数据结构 将value取hash，value放置node中key位置，node中的value放空的object。因为value放在key的位置所以value不能重复