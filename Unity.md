# Unity

## 对象池

### 对象池简介

核心思想是使用过的对象缓存下来，在下一次使用相同的对象时直接从缓存中取，避免频繁实例化对象。

### 对象数量分配

- 分配触发：负责在池中对象不足时分配新资源。
- 空池触发：任何时候，只要池空了，就分配对象。
- 水位线：空池触发的缺点是，某次对象请求会因为执行对象分配而中断。为了避免这种情况，可以使用水位线触发。当从池中请求新对象时，检查池中可用对象的数量。如果可用对象小于某个阈值，就触发分配过程。
- Lease/Return 速度：大多数时候，水位线触发已经足够，但有时候可能会需要更高的精度。在这种情况下，可以使用 lease 和 return 速度。例如，如果池中有 100 个对象，每秒有 20 个对象被取走，但只有 10 个对象返回，那么 9 秒后池就空了。开发者可以使用这种信息，提前做好对象分配计划。

### 内存管理

- 引用泄露：对象在系统中某个地方注册了，但没有返回到池中。
- 过早回收：消费者已经决定将对象返还对象池，但仍然持有它的引用，并试图执行写或读操作。
- 隐式回收：当使用引用计数时可能会出现这种情况。
- 大小错误：这种情况在使用字节缓冲区和数组时非常常见，对象应该有不同的大小，而且是以定制的方式构造，但返回对象池后却作为通用对象重用。
- 重复下单：这是引用泄露的一个变种，存在多路复用时特别容易发生，一个对象被分配到多个地方，但其中一个地方释放了该对象。
- 就地修改：对象不可变是最好的，但如果不具备那样做的条件，就可能在读取对象内容时遇到内容被修改的问题。
- 缩小对象池：当池中有大量的未使用对象时，要缩小对象池。
- 对象重新初始化：确保每次从池中取得的对象不含有上次使用时留下的脏字段。
