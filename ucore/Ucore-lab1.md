# Ucore-lab1

## 练习1

> 这个练习的重点在于理解makefile的语法。

### 变量赋值

~~~makefile
PROJ	:= challenge
EMPTY	:=
SPACE	:= $(EMPTY) $(EMPTY)
SLASH	:= /

V       := @
~~~

如上是变量赋值，makefile中有多种变量赋值的方法。

~~~
VARIABLE = value
# 在执行时扩展，允许递归扩展,其就是遍历整个Makefile，获得value的最终值来赋给右值。

VARIABLE := value
# 在定义时扩展。

VARIABLE ?= value
# 只有在该变量为空时才设置值。

VARIABLE += value
# 将值追加到变量的尾端。
~~~

### @符号的使用

makefile在执行的过程中会进行输出，但是在命令前加上@就可以关闭回声。

### 条件语句

~~~makefile
ifndef GCCPREFIX
GCCPREFIX := $(shell if i386-elf-objdump -i 2>&1 | grep '^elf32-i386$$' >/dev/null 2>&1; \
	then echo 'i386-elf-'; \
	elif objdump -i 2>&1 | grep 'elf32-i386' >/dev/null 2>&1; \
	then echo ''; \
	else echo "***" 1>&2; \
	echo "*** Error: Couldn't find an i386-elf version of GCC/binutils." 1>&2; \
	echo "*** Is the directory with i386-elf-gcc in your PATH?" 1>&2; \
	echo "*** If your i386-elf toolchain is installed with a command" 1>&2; \
	echo "*** prefix other than 'i386-elf-', set your GCCPREFIX" 1>&2; \
	echo "*** environment variable to that prefix and run 'make' again." 1>&2; \
	echo "*** To turn off this error, run 'gmake GCCPREFIX= ...'." 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif
~~~

这个其实不难理解，跟c语言里面的很像，值得注意的是$符号,makefile中的变量以$开头，为了避免和shell的变量冲突，shell里面的变量要以$$开头。

#### 语义解析

如果没有定义GCCPREFIX变量的话，$里面的语句将得到执行，这个语句我没太懂，就看了binlep师傅的笔记，原来是判断 i386-elf-objdump 这一命令是否存在，存在的话就由echo输出`i386-elf-`命令就会结束，因为是$()里面的命令，所以会被当做value被赋值给GCCPREFIX。

如果命令不存在的话，则对`elf32-i386`命令进行测试。

如果都不存在的话，那么就输出错误信息，提示对错误进行纠正。

### 条件语句2

~~~makefile
QEMU := $(shell if which qemu-system-i386 > /dev/null; \
	then echo 'qemu-system-i386'; exit; \
	elif which i386-elf-qemu > /dev/null; \
	then echo 'i386-elf-qemu'; exit; \
	elif which qemu > /dev/null; \
	then echo 'qemu'; exit; \
	else \
	echo "***" 1>&2; \
	echo "*** Error: Couldn't find a working QEMU executable." 1>&2; \
	echo "*** Is the directory containing the qemu binary in your PATH" 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif
~~~

#### 语义解析

这个语句跟上一个差不多，但是多了个if which，其实也是进行判断的意思，判断是否有这个程序存在。

### 语句3

~~~makefile
# eliminate default suffix rules
.SUFFIXES: .c .S .h

# delete target files if there is an error (or make is interrupted)
.DELETE_ON_ERROR:

# define compiler and flags
ifndef  USELLVM
HOSTCC        := gcc
HOSTCFLAGS    := -g -Wall -O2
CC        := $(GCCPREFIX)gcc
CFLAGS    := -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
CFLAGS    += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
else
HOSTCC        := clang
HOSTCFLAGS    := -g -Wall -O2
CC        := clang
CFLAGS    := -march=i686 -fno-builtin -fno-PIC -Wall -g -m32 -nostdinc $(DEFS)
CFLAGS    += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
endif

CTYPE    := c S

LD      := $(GCCPREFIX)ld
LDFLAGS    := -m $(shell $(LD) -V | grep elf_i386 2>/dev/null | head -n 1)
LDFLAGS    += -nostdlib

OBJCOPY := $(GCCPREFIX)objcopy
OBJDUMP := $(GCCPREFIX)objdump

COPY    := cp
MKDIR   := mkdir -p
MV        := mv
RM        := rm -f
AWK        := awk
SED        := sed
SH        := sh
TR        := tr
TOUCH    := touch -c

OBJDIR    := obj
BINDIR    := bin

ALLOBJS    :=
ALLDEPS    :=
TARGETS    :=
~~~

#### 语义解析

定义了使用的各个工具，同时也是可以看出上面的GCCPREFIX是为了确定GCC版本和一些其他工具的版本，同时删除错误文件和一些特定结尾的文件。

### include语句

~~~makefile
include tools/function.mk
~~~

这表示引用了一个文件，这个文件真的看不懂系列（我太菜了吧，哭死），只能看看其他师傅(**binlep**)的笔记了，加了一些自己的想法。

#### addsuffix函数

~~~makefile
$(addprefix <prefix>, <name1 name2 ...>)
~~~

把 \<prefix> 加到 name 序列中的每一个元素前面，即将 \<prefix> 作为序列中每个元素的前缀

注：addsuffix 用法与 addprefix 相同，只是一个是前缀，一个是后缀，简单来说这就是一个拼接用的函数。

**实例：**

```makefile
result = $(addprefix %., c cpp)
test:
    @echo $(result)
```

**输出：**

```
%.c %.cpp
```

###### if 函数：

```makefile
$(if <condition>, <then-part>,<else-part>)
```

\<condition> 参数是 if 的表达式，如果其返回的为非空字符串，那么这个表达式就相当于返回真

于是 \<then-part> 会被计算，返回计算结果字符串；否则 \<else- part> 会被计算，返回计算结果字符串

注：\<else-part> 可以省略

类似于c语言的 if elesif之类的。

**实例：**

```makefile
suffix := 
result1 := $(if $(suffix), $(addprefix %.,$(suffix)), %)

suffix := c cpp
result2 := $(if $(suffix), $(addprefix %.,$(suffix)), %)

test:
    @echo result1 is $(result1)
    @echo result2 is $(result2)
```

**输出：**

```
result1 is %
result2 is %.c %.cpp
```

###### wildcard 函数：

```makefile
$(wildcard <pattern1 pattern2 ...>)
```

展开 pattern 中的通配符

**实例：**

```makefile
$(wildcard src/*)
```

输出结果为src目录下的文件列表。

#### filter函数

这个比较简单了，从字面上可以看出这是一个过滤器的作用

```makefile
$(filter <pattern1 pattern2 ...>, <text>)
```

以 \<pattern> 模式过滤 \<text> 字符串中的单词，保留符合模式 \<pattern> 的单词，可以有多个模式

**实例：**

```makefile
$(filter %.c %.cpp, $(wildcard src/*))
```

**输出：**

src 目录下所有后缀是 \.c 和 \.cpp 的文件序列。

> 文件内重要函数都懂了，就开始自己分析一下文件内容吧。

### function.mk -- part1

~~~makefile
OBJPREFIX	:= __objs_

.SECONDEXPANSION:
# -------------------- function begin --------------------

# list all files in some directories: (#directories, #types)
listf = $(filter $(if $(2),$(addprefix %.,$(2)),%),\
		  $(wildcard $(addsuffix $(SLASH)*,$(1))))

# get .o obj files: (#files[, packet])
toobj = $(addprefix $(OBJDIR)$(SLASH)$(if $(2),$(2)$(SLASH)),\
		$(addsuffix .o,$(basename $(1))))

# get .d dependency files: (#files[, packet])
todep = $(patsubst %.o,%.d,$(call toobj,$(1),$(2)))

totarget = $(addprefix $(BINDIR)$(SLASH),$(1))

# change $(name) to $(OBJPREFIX)$(name): (#names)
packetname = $(if $(1),$(addprefix $(OBJPREFIX),$(1)),$(OBJPREFIX))
~~~

`listf`里面就是先判断 $(2) 里面是否为空字符串，如果不为空的话，就把其内容追加到%.的后面，如果为空的话，就返回%,`$(wildcard $(addsuffix $(SLASH)*,$(1)))` 用来获得 \$(1) 目录下的所有文件

`filter` 函数的作用就是从 \$(1) 目录下找出符合 ==if 函数返回值内序列== 模式的所有文件

其他的大概也都差不多，不过还有一个函数。

#### patsubst函数

patsubst（ patten substitude, 匹配替换的缩写）函数。它需要3个参数：第一个是一个需要匹配的式样，第二个表示用什么来替换它，第三个是一个需要被处理的由空格分隔的字列。例如，处理那个经过上面定义后的变量， 

​    OBJS = $(patsubst %.c，%.o，$(SOURCES))

这行将处理所有在 SOURCES列个中的字（一列文件名），如果它的 结尾是 '.c' ，就用'.o' 把 '.c' 取代。注意这里的 % 符号将匹配一个或多个字符，而它每次所匹配的字串叫做一个‘柄’(stem) 。在第二个参数里， % 被解读成用第一参数所匹配的那个柄。

#### basename函数

```makefile
$(basename <names...>)
```

从文件名序列 `<names>` 中取出各个文件名的前缀部分

可以理解为截取文件的文件名，从头开始直到最后一个 `.` 的前一个字符

返回文件名序列 `<names>` 的前缀序列，如果文件没有前缀，则返回空字串

### function.mk -- part2

~~~makefile
define cc_template
$$(call todep,$(1),$(4)): $(1) | $$$$(dir $$$$@)
	@$(2) -I$$(dir $(1)) $(3) -MM $$< -MT "$$(patsubst %.d,%.o,$$@) $$@"> $$@
$$(call toobj,$(1),$(4)): $(1) | $$$$(dir $$$$@)
	@echo + cc $$<
	$(V)$(2) -I$$(dir $(1)) $(3) -c $$< -o $$@
ALLOBJS += $$(call toobj,$(1),$(4))
endef

# compile file: (#files, cc[, flags, dir])
define do_cc_compile
$$(foreach f,$(1),$$(eval $$(call cc_template,$$(f),$(2),$(3),$(4))))
endef

# add files to packet: (#files, cc[, flags, packet, dir])
define do_add_files_to_packet
__temp_packet__ := $(call packetname,$(4))
ifeq ($$(origin $$(__temp_packet__)),undefined)
$$(__temp_packet__) :=
endif
__temp_objs__ := $(call toobj,$(1),$(5))
$$(foreach f,$(1),$$(eval $$(call cc_template,$$(f),$(2),$(3),$(5))))
$$(__temp_packet__) += $$(__temp_objs__)
endef

# add objs to packet: (#objs, packet)
define do_add_objs_to_packet
__temp_packet__ := $(call packetname,$(2))
ifeq ($$(origin $$(__temp_packet__)),undefined)
$$(__temp_packet__) :=
endif
$$(__temp_packet__) += $(1)
endef

# add packets and objs to target (target, #packes, #objs[, cc, flags])
define do_create_target
__temp_target__ = $(call totarget,$(1))
__temp_objs__ = $$(foreach p,$(call packetname,$(2)),$$($$(p))) $(3)
TARGETS += $$(__temp_target__)
ifneq ($(4),)
$$(__temp_target__): $$(__temp_objs__) | $$$$(dir $$$$@)
	$(V)$(4) $(5) $$^ -o $$@
else
$$(__temp_target__): $$(__temp_objs__) | $$$$(dir $$$$@)
endif
endef

# finish all
define do_finish_all
ALLDEPS = $$(ALLOBJS:.o=.d)
$$(sort $$(dir $$(ALLOBJS)) $(BINDIR)$(SLASH) $(OBJDIR)$(SLASH)):
	@$(MKDIR) $$@
endef
~~~

至此，我决定先封笔，我tmd要回去看看Shell编程再和你们玩。

### 未完待续。。。

### 练习1.2

一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

#### 答案：

这个就简单了，老师直接说了，扇区最后两个字节为0x55AA,该扇区有512个字节。

## 练习2：使用qemu执行并调试lab1中的软件

看了result里面的makefile ，qemu起，然后gdb连上就行，跟之前调试内核挺像。

## 练习3：分析bootloader进入保护模式的过程









