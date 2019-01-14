学习、适用内核，中断和异常是不能绕过的一大知识点。只可惜出道这么多年，一直都没有认真地潜心研究这部分的内容。那这次就从硬件开始。

# 从硬件开始 -- IDT

中断和异常的处理需要硬件的支持，知道了硬件可以说就是了解了一大半。而这部分的硬件在x86平台上就是熟知的IDT了。

[从IDT开始][1]

# 中断和异常的区别

中断和异常这两个词我想大家经常听到，但是这两者之间有什么区别呢？你能想到啥？那就听我慢慢道来吧～

[中断？异常？有什么区别][2]

# 插播--系统调用的实现

学习过程中突然想到以前学习的时候说系统调用是通过**int 0x80**来实现的，那就干脆找找系统调用是怎么处理的呗。

这不看不打紧，一看发现现在的系统调用实现采用了新的方式。所以在这里插播一下～

[系统调用的实现][3]

# 异常处理

书归正传，看过了系统调用，现在继续回来看异常处理。

在这里我们只看看异常向量表是如何初始化，并关联到异常处理函数的。

[异常向量表的设置][4]

# 中断函数

最后我们来看看中断。当然我们同样暂时不看很多具体的细节，只是把中断向量表和中断函数串起来。能够对系统架构有一个大致的理解。

[中断向量和中断函数][5]

[1]: /interrupt_exception/01-idt.md
[2]: /interrupt_exception/02-difference.md
[3]: /interrupt_exception/03-syscall.md
[4]: /interrupt_exception/04-exception_vector_setup.md
[5]: /interrupt_exception/05-interrupt_handler.md