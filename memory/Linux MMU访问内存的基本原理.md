# Linux MMU访问内存的基本原理

## 假设页表只有一级
对于一个有MMU的CPU而言，MMU开启后，CPU是这样寻址的：CPU任何时候，一切时候，发出的地址都是虚拟地址，这个虚拟地址发给MMU后,MMU通过页表来在页表里面查出来这个虚拟地址对应的物理地址是什么，从而去访问外面的内存条。MMU里面的页表地址寄存器，记录了页表本身的存放位置。
![一级页表MMU结构图](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytI9k2b7fIYiasftN1IXtSDMTPiblibVkM6SibaF8eqVJ2aDRbnC4wiaFBUcTfxRyQbSD3LdzTQFKChiavA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

现在我们假设每一页的大小是4KB，而且假设页表只有一级，这个页表长成下面这个样子，页表的每一行是32个bit。

![](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytI9k2b7fIYiasftN1IXtSDMbjraPgXDXs23OC7tSCJKlrq6Bexv2dX2WrYJ8BZnY1pMxWNetRYKicw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当CPU访问虚拟地址0的时候，MMU会去查上面页表的第0行，发现第0行没有命中，于是无论以何种形式（R读，W写，X执行）访问，MMU都会给CPU发出page fault，CPU自动跳到fault的代码去处理fault。

当CPU访问虚拟地址4KB的时候，MMU会去查上面页表的第1行(4KB/4KB=1)，发现第1行命中，如果这个时候

* 用户是执行读或者执行，则MMU去访问内存条的6MB这个地址，因为页表里面记录该页的权限是RX；

* 用户是去写4KB，由于页表里面第1行记录的权限是RX，没有记录你有写的权限，MMU会给CPU发出page fault，CPU自动跳到fault的代码去处理fault。

当CPU访问虚拟地址8KB+16的时候，MMU会去查上面页表的第2行(8KB/4KB=2)，发现第2行命中了物理地址8M，这个时候，MMU会访问内存条的8MB+16这个物理地址。当然，权限检查也是需要的。

…

当CPU访问虚拟地址3GB的时候，MMU会去查上面页表的第3GB/4KB行，表中记录命中了，查到虚拟地址3GB对应的物理地址是0，于是MMU去访问内存条上的地址0。但是，这个访问分成2种情况：

* CPU在执行用户态程序的时候，去访问3GB，由于页表里面记录的U+K权限只有K，所以U是没权限的，MMU会给CPU发出page fault，CPU自动跳到fault的代码去处理fault；

* CPU在执行内核态程序的时候，去访问3GB，由于页表里面记录的U+K权限只有K，所以K是有权限的，MMU不会给CPU发出page fault，程序正常执行。

由此可以得知，如果页表只有1级，每4KB的虚拟地址空间就需要页表里面的一行（32bit），那么CPU要覆盖到整个4GB的内存，就需要这个页表的大小是：

> 4GB/4KB *4 = 4MB

注意页表是无缝全覆盖！！！你页表不覆盖全，CPU访问虚拟地址的时候，MMU都不知道查哪里了....

*所以，这个页表的大小是4MB，覆盖了整个0-4GB的虚拟地址空间，任何一个虚拟地址，都可以用地址的高20位（由于一页是4KB，低12位就是叶内偏移了），作为页表这个表的行号去读对应的页表项。
这个查水表的过程，由MMU硬件自动完成。*

现在我们假设在Linux里面有2个进程，一个是QQ，一个是Firefox，他们的页表分别如下：

![](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytI9k2b7fIYiasftN1IXtSDM7VYMV9qjHwht70rVoo3PnKdPlNLDuuWL8HibicB7TgpPIiae7SG0ITZrw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytI9k2b7fIYiasftN1IXtSDMooic3OLBLDrZSvKSwXfpUBFfBWAQwOibtHbhuFtwdibWQzvBqx1emqLrw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当CPU在执行QQ的时候，Linux会把QQ的页表的物理地址255MB，填入MMU的页表地址寄存器，于是这个时候，**QQ的页表生效**。根据页表内容，CPU如果访问4KB这个虚拟地址的话，MMU访问内存条的6MB物理地址；CPU如果访问8KB这个虚拟地址的话，MMU访问内存条的8MB物理地址；CPU如果访问3GB这个虚拟地址的话，MMU访问内存条的0MB物理地址；

当CPU在执行Firefox的时候，Linux会把Firefox的页表的物理地址280MB，填入MMU的页表地址寄存器，于是这个时候，**Firefox的页表生效**，QQ的页表淡出江湖。根据页表内容，CPU如果访问4KB这个虚拟地址的话，MMU访问内存条的100MB物理地址；CPU如果访问8KB这个虚拟地址的话，MMU访问内存条的200MB物理地址；CPU如果访问3GB这个虚拟地址的话，MMU访问内存条的0MB物理地址。

上面我们发现一个共同点，QQ和Firefox去访问3GB虚拟地址的时候，最终MMU访问的都是0MB这个物理地址，具体原因非常简单，**QQ和Firefox，这2张页表里面，3GB/4KB这一行，里面填的是完全一样的东东。**

## 多级页表：真实的存在

上面我们发现，如果采用一级页表的话，每个进程都需要1个4MB的页表，这个空间浪费还是很大，于是我们可以采用二级或者三级页表。举例如下，假设我们用地址的高10位作为一级页表的索引，中间10位作为2级页表的索引。CPU访问虚拟地址16，这个地址如果分解为10/10/12位的话，就是这个样子：

![](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytI9k2b7fIYiasftN1IXtSDMj7enomR3ia8OB7PC80jnExwjWRzqmaODtV2bzxicibAtdfK6u9xjJ94FQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

那么MMU会用0这个下标去访问一级页表（一级页表的地址填入MMU的页表地址寄存器）的第0行，第0行的内容写的是2MB(此处不再是最终的物理地址，而是二级页表的物理地址),证明二级页表的地址在2MB，于是MMU自动去以中间的10位作为下标，去查询位置在2MB的二级页表,在2级页表里面，最终查到第0页(地址范围0x00000000~0x00000FFF)这个虚拟地址的物理地址是1GB，于是MMU去访问内存条的1GB+16这个物理地址。

![](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytI9k2b7fIYiasftN1IXtSDMqOdicAGMe7kK2gDvwvxDLNUaBiczq9ugqw8ACBJM8kicmlU9bXAzAYz1Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

据以上分析，1级页表占据的内存是2的10次方，再乘以4，即4KB。而每个二级页表，也是2的10次方，再乘以4，即4KB。分级机制的主要好处是，二级页表不是一定存在了，比如一级页表的第2行不命中，也即如下地址都无效的话：

![](https://mmbiz.qpic.cn/mmbiz_png/Ass1lsY6bytI9k2b7fIYiasftN1IXtSDMZB06zQqeVDaHPxAroibp3fZ2h3E2pscVlJCtOE3y6cZQLCiasicooTf5Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

*那么这一行对应的二级页表，就整个都不需要了，于是就省掉了这段区间4KB二级页表的内存占用。* 页表当然还有是三级甚至更多。

至于有多级页表的时候，其实MMU也只需要知道一级页表的基地址即可。每次切换进程的时候，把一级页表的地址重新填入MMU，把新的进程的页表激活即可。