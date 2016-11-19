标题: Rakudo and NQP Internals
子标题: The guts tormented implementers made
作者: Jonathan Worthington

## 关于这个课程

Perl 6 是一种大型语言，包含许多要求正确实现的功能。

这样的软件项目很容易被难控制的复杂性淹没。
Rakudo 和 NQP 项目的早期阶段已经遭受了这样的困难，因为我们学到了 - 艰难的方式 - 关于复杂性，出现并可能在实现过程中不受限制地扩散。

本课程将教您如何使用 Rakudo 和 NQP 内部。 在他们的设计中编码是一个大量学习的过程，关于如何（以及如何不）写一个 Perl 6 实现, 这个过程持续了多年。 因此，本课程还将教你事情的来龙去脉。

## 关于讲师

* 计算机科学背景
* 选择旅行世界,并帮助实现 Perl 6, 而不是做博士
* 有不止一种方法来获得"永久头部损伤" `:-)`
* 不知何故在 Edument AB 被聘用, 作为讲师/顾问
* 从 2008 年开始成为 Rakudo Perl 6 核心开发者
* 6model, MoarVM, NQP 和 Rakudo 各个方面的缔造者

## 课程大纲 - 第一天

* 鹰的视角: 编译器和 NQP/Rakudo 架构
* NQP 语言
* 编译管道
* QAST
* 探索 nqp::ops

## 课程大纲 - 第二天

* 6model
* 有界序列化和模块加载
* 正则表达式和 grammar 引擎
* JVM 后端
* MoarVM 后端

# 鹰的视角

*编译器和 NQP/Rakudo 架构*

## 编译器做什么

编译器真的是"只是"翻译。

编译器把高级语言 (例如 Perl 6) 翻译成低级语言 (例如 JVM 字节码)。

![input-compiler-output](https://github.com/ohmycloud/rakudo-and-nqp-internals-course-zh/blob/master/images/input-compiler-output.svg)

接收直截了当的输入（文本）并产生直截了当的输出（文本或二进制），但**内部的数据结构很丰富**

像字符串那样处理东西，通常是最后的手段

## 运行时做什么

运行像 Perl 6 这样的语言不仅仅是将它转换为低级代码。 此外, 它需要**运行时支持**来提供：

* 内存管理
* I/O, IPC, OS 交互
* 并发
* 动态优化

## 构建我们需要的东西来构建东西

我们已经以现有的编译器构造技术进行了各种尝试来构建 Perl 6。 编译器的早期设计至少有一部分是基于常规假设的。

这样的尝试是信息性的，但从长远来看还不够好。

Perl 6 提出了一些有趣的挑战...

## Perl 6 是使用 Perl 6 解析的

Perl 6 的标准文法(grammar)是用 Perl 6 写的。它依赖于...

* 可传递的 **最长 token 匹配** (我们会在之后看到更多关于它的东西)
* 能够在不同语言之间来回切换 (主语言, 正则表达式语言, 引用(quoting)语言)
* 能够**动态地派生出新语言** (新运算符，自定义引用构造)
* 在自下而上的表达式解析和自顶向下的更大的结构解析之间无缝集成
* 保持**令人惊叹的错误报告**的各种状态

所有这些本质上代表了解析中的新范例。

## 非静态类型或动态类型

Perl 6 是一种 **渐进类型化**的语言。

    my int $distance = distance-between('Lund', 'Kiev');
    my int $time = prompt('Travel time: ').Int;
    say "Average speed: { $distance / $time }";

我们想利用 `$distance` 和 `$time` 原生整数来产生更好的代码，如果我们不知道类型（应该只是输出代码中的原生除法指令）。

## 模糊编译时和运行时

运行时可以做一些编译时:

    EVAL slurp @demos[$n];

编译时可以做一些运行时:

    my $comp-time = BEGIN now;

**注意编译时计算的结果必须持久化直到运行时，这它们之间可能有一个处理(process)边界!**

## NQP 作为语言

Perl 6 文法显然需要用 Perl 6 表示。这反过来将需要集成到编译器的其余部分。用 Perl 6 编写整个编译器是很自然的。

然而，一个完整的 Perl 6 太大，给它写一个好的优化器花费太多时间。
因此， **NQP (Not Quite Perl 6)** 语言诞生了：它是 Perl 6 的一个子集，用于实现编译器。NQP 和 Rakudo 大部分都是用 NQP 写的。

## NQP 作为编译器构造工具链

NQP `src` 目录中不仅仅是 NQP 本身。

* **NQP, how, core**: 这些包含 NQP 编译器, 元对象(它指定了 NQP 的类和 roles 的工作方式)和内置函数。
* **HLL**: 构造高级语言编译器的通用结构，在 Rakudo 和 NQP 之间共享。
* **QAST**: Q 抽象语法树的节点。代表着程序语法的树节点。（即，当它执行时会做什么）。
* **QRegex**: 解析和执行 regexes 和 grammars 时所提到的对象。
* **vm**: 虚拟机抽象层。因为 NQP 和 Rakudo 可以运行在 Parrot, JVM 和 MoarVM 上。

## QAST

QAST 树是 NQP 和 Rakudo 内部最重要的数据结构之一。

An **Abstract Syntax Tree** represents what a program does when executed. It is abstract in the sense of being abstracted away from the particular language that a program was written in.
**抽象语法树**表示程序在执行时执行的操作。它是抽象的意思是从一个程序被写入的特定语言中抽象出来。

![30%](eps/example-qast.eps)

## QAST

不同的 QAST 节点代表着像下面这样的东西:

* 变量
* 运算 (算术, 字符串, 调用, 等等.)
* 字面值
* Blocks

注意像类那样的东西没有 QAST 节点, 因为那些是编译时声明而非运行时执行。

## nqp::op 集合

编译器工具链的另一个重要部分是 `nqp::op` 指令集。你会有两种方式遇到它，并且了解它们之间的差异很重要！

您可以在 **NQP 代码** 中使用它们，在这种情况下，您说您希望在程序中的那个点上执行该操作：

    say(nqp::time_n())

在代表着正在编译的程序的 **QAST树** 中也使用完全相同的指令集：

    QAST::Op.new(
        :op('call'), :name('&say'),
        QAST::Op.new( :op('time_n') )
    )

## Bootstrapping in a nutshell

人们可能会想知道 NQP 几乎完全用 NQP 编写时是如何编译的。

在每个 `vm` 子目录中都有一个 `stage0` 目录。它包含一个编译的 NQP（Parrot上是PIR文件，JVM上是JAR文件等）然后：

![40%](eps/nqp-bootstrapping-stages.eps)

因此，你 `make test` 的 NQP 是可以重新创建自身的 NQP。

通常，我们使用最新版本来更新 `stage0`

## How Rakudo uses NQP

Rakudo 本身不是一个自举编译器，这使得它的开发容易一点。大部分 Rakudo 是用 NQP 编写的。这包括：

* **编译器本身的核心**，它解析 Perl 6 源代码，构建 QAST，管理声明并进行各种优化
* **元对象**，它指定了不同类型（类，roles，枚举，subsets）是如何工作的
* **bootstrap**，它将足够的 Perl 6 核心类型组合在一起，以便能够在 Perl 6 中编写内置的类，角色和例程

Thus, while some of Rakudo is accessible if you know Perl 6, knowing NQP - both as a language and as a compiler toolchain - is the gateway to working with most of the rest of Rakudo.
因此，虽然一些 Rakudo 是可访问的，如果你知道 Perl 6，知道 NQP  - 既作为一种语言又作为一种编译器工具链 - 是和大部分 Rakudo 的其他部分工作的入口。

# NQP 语言

*它不完全是 Perl 6(Not Quite Perl 6), 但是能很好地构建 Perl 6*

## 设计目标

NQP 被设计为……

* 理想的编写编译器相关的东西
* 几乎是 Perl 6 的一个子集
* 比 Perl 6 更容易编译和优化

注意，它避免了

* 赋值
* Flattening and laziness
* 操作符的多重分派（因此没有重载）
* 有很多 built-ins

## 字面量

整数字面量

    0       42      -100

浮点字面量 (NQP 中没有 rat!)

    0.25    1e10    -9.9e-9

字符串字面量

    'non-interpolating'         "and $interpolating"
    q{non-interpolating}        qq{and $interpolating}
    Q{not even backslashes}

## Sub 调用

在 NQP 中这些总是需要圆括号：

    say('Mushroom, mushroom');

像 Perl 6 中一样，这为子例程的名字添加 & 符号并对该例程做词法查询。

然而，没有列表操作调用语法:

    plan 42;    # "Confused" parse error
    foo;        # Does not call foo; always a term

这可能是 NQP 初学者最常见的错误。

## 变量

可以是 `my` (lexical) 或 `our` (package) 作用域的:

    my $pony;
    our $stable;

常用的符号集也是可用的：

    my $ark;                # Starts as NQPMu
    my @animals;            # Starts as []
    my %animal_counts;      # Starts as {}
    my &lasso;              # Starts as 

也支持动态变量

    my @*blocks;

## 绑定

NQP 没有提供 `=` 赋值操作符。只提供了 `:=` 绑定操作符。这使 NQP 免于 Perl 6 容器语义的复杂性。

下面是一个简单的标量示例:

    my $ast := QAST::Op.new( :op('time_n') );

## 绑定与数组

注意绑定拥有**item 赋值优先**, 所以你不可以这样写:

    my @states := 'start', 'running', 'done';    # Wrong!

相反, 这应该被表示为下面的其中之一:

    my @states := ['start', 'running', 'done'];  # Fine
    my @states := ('start', 'running', 'done');  # Same thing
    my @states := <start running done>;          # Cutest

## 原生类型化变量

目前，NQP 并不真正支持对变量的类型约束。唯一的例外是它会注意**原生类型**。

    my int $idx := 0;
    my num $vel := 42.5;
    my str $mug := 'coffee'; 

**注意:** 在 NQP 中，绑定用于原生类型！这在 Perl 6 中是非法的，Perl 6 中原生类型只能被赋值。尽管这是非常武断的，目前 Perl 6 中原生类型的赋值实际上被编译到 `nqp::bind(...)` op 中！

## 控制流

大部分 Perl 6 条件结构和循环结构也存在于 NQP 中。就像在真实的 Perl 6 中一样, 条件的周围不需要圆括号, 并且还能使用 pointy 块。循环结构支持 `next`/`last`/`redo`.

    if $optimize {
        $ast := optimize($ast);
    }
    elsif $trace {
        $ast := insert_tracing($ast);
    }

**可用的:** if, unless, while, until, repeat, for

**还没有的:** loop, given/when, FIRST/NEXT/LAST phasers

## 子例程

与 Perl 6 中子例程的声明很像，但是 NQP 中即使没有参数，参数列表也是强制的。你可以 `return` 或使用最后一个语句作为隐式返回值。

    sub mean(@numbers) {
        my $sum;
        for @numbers { $sum := $sum + $_ }
        return $sum / +@numbers;
    }

Slurpy 参数也是可用的，就像 `|` 用来展开参数列表那样。

**注意：** 参数可以获得类型约束，但是与变量一样，当前只有原生类型。（例外：多重分派;以后会有更多。）

## Named arguments and parameters

支持命名参数:

    sub make_op(:$name) {
        QAST::Op.new( :op($name) )
    }
    
    make_op(name => 'time_n');  # 胖箭头语法
    make_op(:name<time_n>);     # Colon-pair 语法
    make_op(:name('time_n'));   # The same

**注意：** NQP 中没有 `Pair` 对象！Pairs - colonpairs 或 fat-arrow 对儿 - 仅在参数列表上下文中有意义。

## Blocks 和 pointy blocks

尖尖块提供了熟悉的 Perl 6 语法：

    sub op_maker_for($op) {
        return -> *@children, *%adverbs {
            QAST::Op.new( :$op, |@children, |%adverbs )
        }
    }

从这个例子可以看出，它们有闭包语义。

**注意：** 普通块也可用作闭包，但不像 Perl 6 那样使用隐式的 `$_` 参数。

## Built-ins 和 nqp::ops

NQP 具有相对较少的内置函数。但是，它提供了对 NQP 指令集的完全访问。这里有几个常用的指令，知道它们会很有用。

    # On arrays
    nqp::elems, nqp::push, nqp::pop, nqp::shift, nqp::unshift
    
    # On hashes
    nqp::elems, nqp::existskey, nqp::deletekey
    
    # On strings
    nqp::substr, nqp::index, nqp::uc, nqp::lc

我们将在课程中发现更多。

## 异常处理

可以使用 `nqp::die` 指令抛出一个异常:

    nqp::die('Oh gosh, something terrible happened');

`try` 和 `CATCH` 指令也是可用的, 尽管不像完全的 Perl 6，你没有期望去智能匹配 `CATCH` 的内部。一旦你到了那儿，就认为异常被捕获到了（弹出一个显式的 `nqp::rethrow`）。

    try {
        something();
        CATCH { say("Oops") }
    }

## 类，属性和方法

就像在 Perl 6 中一样, 使用 `class`, `has` 和 `method` 关键字来声明。类可以是词法（`my`）作用域的或包（`our`）作用域的（默认）。

    class VariableInfo {
        has @!usages;
        
        method remember_usage($node) {
            nqp::push(@!usages, $node)
        }
        
        method get_usages() {
            @!usages
        }
    }

`self` 关键字也是可用的，方法可以具有像 subs 那样的参数。

## More on attributes

NQP 没有自动的存取器生成，所以你不能这样做：

    has @.usages; # 不支持

支持原生类型的属性，并且将直接有效地存储在对象体中。任何其他类型都被忽略。

    has int $!flags;

与 Perl 6 不同，默认构造函数可用于设置私有属性，因为这是我们所拥有的。

    my $vi := VariableInfo.new(usages => @use_so_far);

## Roles (1)

NQP 支持 roles. 像类那样, roles 可以拥有属性和方法。

    role QAST::CompileTimeValue {
        has $!compile_time_value;
        
        method has_compile_time_value() {
            1
        }
        
        method compile_time_value() {
            $!compile_time_value
        }
        
        method set_compile_time_value($value) {
            $!compile_time_value := $value
        }
    }

## Roles (2)

role 可以使用 `does` trait 组合到类中:

    class QAST::WVal is QAST::Node does QAST::CompileTimeValue {
        # ...
    }

或者，MOP 可用于将 role 混合到单个对象中：

    method set_compile_time_value($value) {
        self.HOW.mixin(self, QAST::CompileTimeValue);
        self.set_compile_time_value($value);
    }

## 多重分派

支持基本多重分派。它是 Perl 6 语义的一个子集，使用更简单（但兼容）的候选排序算法版本。

与完全的 Perl 6 不同，你**必须写一个 `proto` ** sub 或方法; 没有自动生成。

    proto method as_jast($node) {*}
    
    multi method as_jast(QAST::CompUnit $cu) {
        # compile a QAST::CompUnit
    }
    
    multi method as_jast(QAST::Block $block) {
        # compile a QAST::Block
    }

## 练习 1

有机会熟悉基本的 NQP 语法，如果你还没有这样做。

还有机会学习常见的错误看起来是什么样的，所以如果你在实际工作中遇到他们，你可以认出它们。 :-)

## Grammars

虽然在许多领域 NQP 相比完全的 Perl 6 相当有限，但是 grammar 几乎支持相同的水平。这是因为 NQP 语法必须足够好以处理解析 Perl 6 本身。

Grammars 是一种类，并且使用 `grammar` 关键字引入。

    grammar INIFile {
    }


事实上，grammars 太像类了，以至于在 NQP 中，它们是由相同的元对象实现的。区别是它们默认继承于什么，并且你把什么放在它们里面。

## INI 文件

作为一个简单的例子，我们将考虑解析 INI 文件。

带有值的键，可能按章节排列。

    name = Animal Facts
    author = jnthn

    [cat]
    desc = The smartest and cutest
    cuteness = 100000

    [dugong]
    desc = The cow of the sea
    cuteness = -10

## 整体方法

grammar 包含一组用关键字 `token`, `rule` 或 `regex` 声明的规则。真的，他们就像方法一样，但是用规则语法写成。

    token integer { \d+ }       # one or more digits
    token sign    { <[+-]> }    # + or - (character class)

更复杂的规则由调用现有规则组成：

    token signed_integer { <sign>? <integer> }

这些对其他规则的调用可以被量化，放在备选分支中，等等。
## 旁白：grammar和正则表达式

在这一点上，你可能想知道 grammar 和正则表达式是如何关联的。毕竟，grammar 似乎是由正则表达式那样的东西组成的。

还有一个 `regex` 声明符，可以在 grammar 中使用。

    regex email { <[\w.-]>+ '@' <[\w.-]>+ '.' \w+ }

关键的区别是 **`regex` 会回溯**，而 `rule` 或 `token` 不会。支持回溯涉及保持大量状态，并且对于复杂的 grammar 解析大量输入，这将快速消耗大量内存！大语言往往在解析器中避免回溯。

## 旁白: NQP 中的正则表达式

对于较小规模的东西，NQP 确实也在普通场景中为正则表达式提供支持。

    if $addr ~~ /<[\w.-]>+ '@' <[\w.-]>+ '.' \w+/ {
        say("I'll mail you maybe");
    }
    else {
        say("That's no email address!");
    }

这被求值为匹配对象。

## 解析条目

一个条目有一个键（一些单词字符）和一个值（直到行尾的所有内容）：

    token key   { \w+ }
    token value { \N+ }

合在一起，它们组成了一个条目:

    token entry { <key> \h* '=' \h* <value> }

`\h` 匹配任何水平空白（空格，制表符等）。 `=` 号必须加引号，因为任何非字母数字都被视为 Perl 6 中的正则表达式语法。

## 从 `TOP` 开始

grammar 的入口点是一个特殊的规则, “TOP”。现在，我们查找整个文件是否含有包含条目的行，或者只是没有。

    token TOP {
        ^
        [
        | <entry> \n
        | \n
        ]+
        $
    }

注意在 Perl 6 中, 方括号是非捕获组(Perl 5 的 `(?:...)`), 而非字符类.

## 尝试我们的 grammar

我们可以通过在 grammar 上调用 `parse` 方法来尝试我们的 grammar。这将返回一个**匹配对象**。

    my $m := INIFile.parse(Q{
    name = Animal Facts
    author = jnthn
    });

![40%](eps/example-match-object.eps)

## 迭代结果

每个 rule 调用都产生一个 match 对象, `<entry>` 调用语法会把它捕获到 match 对象中。

因为我们匹配了很多 **entrie**s, 所以我们在 match 对象中的 `entry` 键下面得到一个数组。

因此, 我们能够遍历它以得到每一个 entry:

    for $m<entry> -> $entry {
        say("Key: {$entry<key>}, Value: {$entry<value>}");
    }

## 追踪我们的 grammar

NQP 自带一些内置支持，用于跟踪 grammars 的去向。它不是一个完整的调试器，但用它来查看 grammar 在失败之前走的有多远是有用的。它使用 trace-on 函数开启：

    INIFile.HOW.trace-on(INIFile);

并且产生的结果像下面这样:

    Calling parse
      Calling TOP
        Calling entry
          Calling key
          Calling value
        Calling entry
          Calling key
          Calling value

## token vs. rule

当我们使用 `rule` 代替 `token` 时, 原子后面的任何空白被转换为对 `ws` 的**非捕获**调用。即:

    rule entry { <key> '=' <value> }

等价于:

    token entry { <key> <.ws> '=' <.ws> <value> <.ws> } # . = non-capturing

我们继承了一个默认的 `ws`, 但是我们也能提供我们自己的:

    token ws { \h* }

## 解析 sections (1)

一个 section 有一个标题和许多条目。但是，顶层也可以有条目。因此，把这个分解是有意义的。

    token entries {
        [
        | <entry> \n
        | \n
        ]+
    }

然后 TOP 规则可以变为:

    token TOP {
        ^
        <entries>
        <section>+
        $
    }

## 解析 sections (2)

最后但并非最不重要的是 section token：

    token section {
        '[' ~ ']' <key> \n
        <entries>
    }

这个 `~` 语法很漂亮. 第一行就像这样:

    '[' <key> ']' \n

然而，如果找不到闭合 `]` 就会产生一个描述性的错误消息，而不只是失败匹配。

## Actions

解析 grammar 可以使用 **actions 类**; 它的方法具有所匹配 grammar 中的某些或所有规则的名字。

在**相应规则成功匹配之后**，actions 方法被调用。

![20%](eps/top-down-bottom-up.eps)

在 Rakudo 和 NQP 编译器中，**actions构造QAST树**。对于这个例子，我们将做一些更简单的事情。

## Actions 示例: aim

给定像下面这样的 INI 文件:

    name = Animal Facts
    author = jnthn

    [cat]
    desc = The smartest and cutest
    cuteness = 100000

我们想使用 actions 类来建立一个**散列的散列**。顶级哈希将包含键 `cat` 和 `_`（下划线收集不在 section 中的任何键）。其值是该 section 中键/值对的哈希。

## Actions 示例: entries

Action 方法将刚刚匹配过的 rule 的 Match 对象作为参数。
把这个 Match 对象放到 `$/` 里很方便, 所以我们能够使用 `$<entry>` 语法糖 (它映射到 `$/<entry>` 上)。这个语法糖看起来像普通的标量, 第一眼看上去的时候有点懵, 再看一眼发现它有一对 `<>` 后环缀, 而这正是散列中才有的, `<entry>` 相当于 `{'entry'}`, 不过前者更漂亮。 

    class INIFileActions {
        method entries($/) { # Match Object 放在参数 $/ 中
            my %entries;
            for $<entry> -> $e {
                %entries{$e<key>} := ~$e<value>;
            }
            make %entries;
        }
    }


最后, **`make`** 将生成的散列**附加**到 `$/` 上。这就是为什么 'TOP` action 方法能够在构建顶级哈希时检索它。

## Actions example: TOP

`TOP` action 方法在由 `entries` action 方法创建的散列中构建顶级散列。当 `make` 将某个东西附加到 `$/` 上时, **`.ast`** 方法检索附加到其他匹配对象上的东西。

    method TOP($/) {
        my %result;
        %result<_> := $<entries>.ast;
        for $<section> -> $sec {
            %result{$sec<key>} := $sec<entries>.ast;
        }
        make %result;
    }

因此, 顶层散列通过 section 名获取到由安装到其中的 `entries` action 方法产生的散列。

## Actions 示例: 用 actions 解析

actions 作为具名参数传递给 `parse`:

    my $m := INIFile.parse($to_parse, :actions(INIFileActions.new));

结果散列可以使用 `.ast` 从结果匹配对象中获得, 正如我们已经看到的。

    my %sections := $m.ast;
    for %ini -> $sec {
        say("Section {$sec.key}");
        for $sec.value -> $entry {
            say("    {$entry.key}: {$entry.value}");
        }
    }

## Actions 示例: 输出

上一张幻灯片上的转储代码产生如下输出：

    Section _
        name: Animal Facts
        author: jnthn
    Section cat
        desc: The smartest and cutest
        cuteness: 100000
    Section dugong
        desc: The cow of the sea
        cuteness: -10

## 练习 2

使用 gramamrs 和 actions 进行小练习的一次机会。

目标是解析 Perl 6 IRC 日志的文本格式; 例如, 参见 http://irclog.perlgeek.de/perl6/2013-07-19/text

## 另外一个例子: SlowDB

解析 INI 文件这个例子是一个很好的开端, 但是离编译器还差的远。作为那个方向的进一步深入, 我们会使用查询解释器创建一个小的, 无聊的, 在内存中的数据库。 

它应该像下面这样工作:

    INSERT name = 'jnthn', age = 28
    [
        result: Inserted 1 row
    ]
    SELECT name WHERE age = 28
    [
        name: jnthn
    ]
    SELECT name WHERE age = 50
    Nothing found

## 查询解释器 (1)

我们解析 `INSERT` 或 `SELECT` 查询的任意之一.

    token TOP {
        ^ [ <insert> | <select> ] $
    }
    
    token insert {
        'INSERT' :s <pairlist>
    }
    
    token select {
        'SELECT' :s <keylist>
        [ 'WHERE' <pairlist> ]?
    }

注意 `:s` 开启了自动 `<.ws>` 插入.

## The query parser (2)

`pairlist` 和 `keylist` rules 的定义如下:

    rule pairlist { <pair>+ % [ ',' ] }
    rule pair     { <key> '=' <value>  }
    rule keylist  { <key>+ % [ ',' ] }
    token key     { \w+ }

这儿有一个有意思的新的语法是 **`%`**. 它附件到末尾的量词上, 表明某些东西(这里是逗号)应该出现在每个量词化的元素**之间**。

逗号字面值周围的方括号是为了确保 `<.ws>` 调用被生成为分割符的一部分。

## 查询解析器 (3)

最后, 这儿是关于值是这样被解析的。

    token value { <integer> | <string> }
    token integer { \d+ }
    token string  { \' <( <-[']>+ )> \' }

注意 **`<(`** 和 **`)>`** 语法的使用。这些表明通过 `string` token 整体应该捕获什么的限制。意味着引号字符不会被捕获。

## 备选分支和 LTM (1)

回忆一下 top rule:

    token TOP {
        ^ [ <insert> | <select> ] $
    }

如果我们追踪 `SELECT` 查询的解析, 我们会看到像下面这样的东西:    

    Calling parse
      Calling TOP
        Calling select
          Calling ws
          Calling keylist

所以它怎么知道不去麻烦尝试 `<insert>` 呢?

## 备选分支和 LTM (2)

答案是**可传递的最长Token匹配**(Transitive Longest Token Matching). grammar 引擎创建了一个 NFA (状态机), 一旦遇到一个备选分支(alternation), 就按照这个**备选分支能够匹配到的字符数**对分支进行排序。然后 Grammar 引擎在这些分支中首先尝试匹配最多字符的那个, 而不麻烦那些它认为不可能的分支。

## 备选分支和 LTM (3)

Gramamr 引擎不会仅仅孤立地看一个 rule。相反, 它 **可传递性地考虑 subrule 调用** (considers subrule calls transitively). 这意味着导致某种不可能的整个调用链可以被忽略。

![20%](eps/ltm-transformation.eps)

它由非声明性构造（如向前查看，代码块或对默认`ws`规则的调用）或递归 subrule 调用界定。

## 轻微的痛点

令我们讨厌的一件事情就是我们的 `TOP` action 方法最后看起来像这样:

    method TOP($/) {
        make $<select> ?? $<select>.ast !! $<insert>.ast;
    }

显而易见, 一旦我们添加 `UPDATE` 和 `DELETE` 查询,  维护起来将会多么痛苦

我们的 `value` action 方法类似:

    method value($/) {
        make $<integer> ?? $<integer>.ast !! $<string>.ast;
    }

## Protoregexes

令我们痛苦的答案是 **protoregexes**。 它们提供了**一个更可扩展的方式来表达备选分支**

    proto token value {*}
    token value:sym<integer> { \d+ }
    token value:sym<string>  { \' <( <-[']>+ )> \' }

本质上, 我们引入了一个新的语法类别, `value`, 然后定义这个类别下不同的案例(cases)。像 `value` 这样的调用会使用 LTM 来对候选者进行排序和尝试 - 就像备选分支所做的那样。

## Protoregexes 和 action 方法 (1)

回到 actions 类, 我们需要更新我们的 action 方法来匹配 rules 的名字:

    method value:sym<integer>($/) { make ~$/ }
    method value:sym<string>($/)  { make ~$/ }

然而, 我们**不需要 `value` 自身这个 action 方法**。任何查看 `$<value>` 的东西会被提供一个来自成功候选者的匹配对象 - 并且 `$<value>.ast` 因此会获得正确的东西。

## Protoregexes and action methods (2)

例如, 在我们重构查询之后:

    token TOP { ^ <query> $ }
    
    proto token query {*}
    token query:sym<insert> {
        'INSERT' :s <pairlist>
    }
    token query:sym<select> {
        'SELECT' :s <keylist>
        [ 'WHERE' <pairlist> ]?
    }

`TOP` action 方法可以简化为:

    method TOP($/) {
        make $<query>.ast;
    }

## keylist 和 pairlist

这两个是无聊的 action 方法, 包含完整性。

    method pairlist($/) {
        my %pairs;
        for $<pair> -> $p {
            %pairs{$p<key>} := $p<value>.ast;
        }
        make %pairs;
    }

    method keylist($/) {
        my @keys;
        for $<key> -> $k {
            nqp::push(@keys, ~$k)
        }
        make @keys;
    }

## 解释查询

那么我们如何运行查询呢？好吧, 下面是  `INSERT` 查询的 action 方法:

    method query:sym<insert>($/) {
        my %to_insert := $<pairlist>.ast;
        make -> @db {
            nqp::push(@db, %to_insert);
            [nqp::hash('result', 'Inserted 1 row' )]
        };
    }

在这里，我们不使用数据结构。我们 `make` 了一个闭包来接收当前数据库状态(一个散列的数组, 其中每一个散列是一行)并把由 `pairlist` action 方法产生的散列推到那个数组中。

## SlowDB 类自身

    class SlowDB {
        has @!data;
        
        method execute($query) {
            if QueryParser.parse($query, :actions(QueryActions.new)) -> $parsed {
                my $evaluator := $parsed.ast;
                if $evaluator(@!data) -> @results {
                    for @results -> %data {
                        say("[");
                        say("    {$_.key}: {$_.value}") for %data;
                        say("]");
                    }
                } else {
                    say("Nothing found");
                }
            } else {
                say('Syntax error in query');
            }
        }
    }

## Exercise 3

一次练习 protoregexes 的机会并自己学习我们已经复习过的。

拿 SlowDB 这个我们已经思考过的例子来说。给这个例子添加 `UPDATE` 和 `DELETE` 查询支持。


## 限制和与完整 Perl 6 的其它区别

这儿有一个值得了解的其它事情的杂烩。

* 有一个 `use` 语句，但它期望它使用已经预编译的任何东西。
* 没有数组展平; `[@a, @b]` 总是两个元素的数组
* 散列构造 `{}` 符只对空散列有效; 除了它之外的任何一个 `{}` 都会被当作一个 block
* `BEGIN` 块存在，但在我们能看见的外部作用域中是高度受限的（只有类型，而不是变量）

## 后端的区别

JVM 和 MoarVM 上的 NQP 相对比较一致。Parrot 上的 NQP 有点古怪: 不是所有的东西都是 6model 对象。即虽然在 JVM 和 MoarVM 上, NQP  中的 `.WHAT` 或 `.HOW` 会工作良好, 但是在 Parrot 上它会失败。这发生在整数, 数字和字符串字面值, 数组和散列, 异常和某些种类的代码对象身上。 

Exception handlers also work out a bit differently. Those on JVM and MoarVM run on the stack top at the point of the exception throw, as is the Perl 6 semantics. Those in NQP on Parrot will unwind then run, with resumption being provided by a continuation. Note that Rakudo is consistent on this on all  backends.
异常处理程序也有所不同。那些在 JVM 和 MoarVM 上运行的堆栈顶部的异常抛出点，就像 Perl 6 语义那样。那些在 Parrot 上的 NQP 将解开然后运行，恢复由继续提供。注意，Rakudo 在所有后端都是一致的。

## 总而言之...

NQP 尽管是 Perl 6 的一个相对较小的子集，但仍然包含相当多强大的语言特性。

一般来说，对他们的需求是由在 Rakudo 中工作的人的需求所驱动的。因此，NQP 功能集是由编译器编写需求定义的。

我们所覆盖的 grammar 和 action 方法资料可能是最重要的，因为这是理解 NQP 和 Perl 6 是如何编译的起点。

# 编译管道

*一步一步我们编译完了那个程序...*

## 从开始到完成

现在我们了解了一点作为语言的 NQP，是时候潜入到下面，看看当我们喂给 NQP 一个程序运行时会发生什么。

首先，我们将考虑这个简单的例子...

    nqp -e "say('Hello, world')"

...一路从 NQP 的 sub `MAIN` 到出现输出。

我们将选择 JVM 后端来检查这个。

## "stagestats" 选项

我们可以通过使用 `--stagestats` 选项来了解 NQP 内部发生的情况，该选项显示了编译器每个阶段经历的时间。

nqp --stagestats -e "say('Hello, world')"

    Stage start      :   0.000      # 启动
    Stage classname  :   0.010      # 计算类名
    Stage parse      :   0.067      # 解析源文件，构造 AST
    Stage ast        :   0.000      # 获取 AST
    Stage jast       :   0.106      # 转换成 JVM AST
    Stage classfile  :   0.032      # 转换成 JVM 字节码
    Stage jar        :   0.000      # 可能创建一个 JAR
    Stage jvm        :   0.002      # 真正地运行该代码

## 倾倒解析树

我们可以得到一些转储的阶段。例如，`--target = parse` 将产生一个解析树的转储。

    - statementlist: say('Hello world')
      - statement: 1 matches
        - EXPR: say('Hello world')
          - deflongname: say
            - identifier: say
          - args: ('Hello world')
            - arglist: 'Hello world'
              - EXPR: 'Hello world'
                - value: 'Hello world'
                  - quote: 'Hello world'
                    - quote_EXPR: 'Hello world'
                      - quote_delimited: 'Hello world'
                        - quote_atom: 1 matches
                        - stopper: '
                        - starter: '

## 倾倒 AST

有时有用的是 `--target = ast`，它转储了 QAST（下面的输出已经被简化）。

    - QAST::CompUnit
      - QAST::Block
        - QAST::Var(lexical @ARGS :decl(param))
        - QAST::Stmts
          - QAST::Var(lexical GLOBALish :decl(static))
          - QAST::Var(lexical $?PACKAGE :decl(static))
          - QAST::Var(lexical EXPORT :decl(static))
        - QAST::Stmts say('Hello world')
          - QAST::Stmts
            - QAST::Op(call &say) 'Hello world'
              - QAST::SVal(Hello world)

## 倾倒 JVM AST

你甚至可以得到一些代表性的低级 AST，它使用 `--target = jast` 变成 Java 字节码，但它完全令人脑抽（下面的一小部分用以说明）。 :-)

    .push_sc Hello world
    58 __TMP_S_0
    .push_sc &say
    .push_idx 1
    43 
    25 __TMP_S_0
    .try
    186 subcall_noa org/perl6/nqp/runtime/IndyBootstrap subcall_noa 0
    :reenter_1
    .catch Lorg/perl6/nqp/runtime/SaveStackException;
    .push_idx 1
    167 SAVER
    .endtry

## 一窥究竟

我们的旅程从 NQP 的 `MAIN` sub 开始，它位于 `src/NQP/Compiler.nqp`。
这里是一个略微简化的版本（剥离设置命令行选项和其它细节）。

    class NQP::Compiler is HLL::Compiler {
    }
    
    # 创建并配置编译器对象
    my $nqpcomp := NQP::Compiler.new();
    $nqpcomp.language('nqp');
    $nqpcomp.parsegrammar(NQP::Grammar);
    $nqpcomp.parseactions(NQP::Actions);
    
    sub MAIN(*@ARGS) {
        $nqpcomp.command_line(@ARGS, :encoding('utf8'));
    }

## HLL::Compiler 类

`command_line` 方法继承自位于 `src/HLL/Compiler.nqp` 中的 `HLL::Compiler`。此类包含协调编译过程的逻辑。

其功能包括：

* 参数处理（委托给　HLL::CommandLine）
* 从磁盘读取源文件
* 调用每个阶段，如果指定，停在`--target`
* 提供 REPL
* 提供可插拔的方式来处理未捕获的异常

## 通过 HLL::Compiler 的路径

**`command_line`** 解析参数，然后调用 `command_eval`

**`command_eval`** 工作，基于参数，如果我们应该从磁盘加载源文件，从 `-e` 获取源或进入 REPL。路径调用了一系列方法，但是所有方法都在 `eval` 中收敛。

**`eval`** 调用 `compile` 来编译代码，然后调用它

**`compile`** 循环遍历这些阶段，将前一个的结果作为下一个的输入

## 简化版的编译

Big takeaway：阶段是编译器对象或后端对象的方法。

    method compile($source, :$from, *%adverbs) {
        my $target := nqp::lc(%adverbs<target>);
        my $result := $source;
        for self.stages() {
            if nqp::can(self, $_) {
                $result := self."$_"($result, |%adverbs);
            }
            elsif nqp::can($!backend, $_) {
                $result := $!backend."$_"($result, |%adverbs);
            }
            else {
                nqp::die("Unknown compilation stage '$_'");
            }
            last if $_ eq $target;
        }
        return $result;
    }

## 阶段管理

编译器可以在管道中插入额外的阶段。例如，Rakudo 插入其优化器。

    $comp.addstage('optimize', :after<ast>);

之后, 在 `Perl6::Compiler` 中, 它提供了一个 `optimize` 方法:

    method optimize($ast, *%adverbs) {
        %adverbs<optimize> eq 'off' ??
            $ast !!
            Perl6::Optimizer.new.optimize($ast, |%adverbs)
    }

## 前端和后端

Earlier, we saw that `compile` looks for stage methods on the current compiler object, then on a backend object.

The **compiler object** is **about the language** that we are compiling (NQP,Perl 6, etc.) We collectively call these stages the **frontend**.

The **backend object** is **about the target VM** that we want to produce code for (Parrot, JVM, MoarVM, etc.) It is not tied to any particular language. We collectively call these stages the **backend**.
早些时候，我们看到`compile'在当前编译器对象上，然后在后端对象上寻找阶段方法。

**编译器对象**是关于我们正在编译的语言**（NQP，Perl 6等）的**。我们共同称这些阶段为**前端**。

**后端对象**是关于目标VM的**，我们想为（Parrot，JVM，MoarVM等）生成代码。它不与任何特定语言绑定。我们将这些阶段统称为**后端**。

![30%](eps/frontend-backend.eps)

## 前端, 后端和它们之间的 QAST

![30%](eps/frontend-backend-qast-between.eps)

前端的最后一个阶段总是给出一个 **QAST树**，后端的第一个阶段总是期望一个QAST树。

一个**交叉编译器设置**只是有一个不同于我们正在运行的当前 VM 的后端。

## 在 NQP 中解析

解析阶段在语言的 grammar（对于我们的例子，NQP::Grammar）上调用 `parse`，传递源代码和 `NQP::Actions`。它也可以打开跟踪。

    method parse($source, *%adverbs) {
        my $grammar := self.parsegrammar;
        my $actions;
        $actions    := self.parseactions unless %adverbs<target> eq 'parse';
        $grammar.HOW.trace-on($grammar) if %adverbs<rxtrace>;
        my $match   := $grammar.parse($source, p => 0, actions => $actions);
        $grammar.HOW.trace-off($grammar) if %adverbs<rxtrace>;
        self.panic('Unable to parse source') unless $match;
        return $match;
    }

## NQP::Grammar.TOP (1)

正如在我们已经看到的 grammar 中，执行从 `TOP` 开始。在 NQP 中，我们发现它实际上是一个 `方法`，而不是 `token` 或 `rule`！

    method TOP() {
        # Various things we'll consider in a moment.
        ...
        
        # Then delegate to comp_unit
        self.comp_unit;
    }

这实际上是 OK 的，只要它最终返回一个 `Cursor` 对象。因为 `comp_unit` 会返回一个，所有的都会工作的很好。

这是一个方法，因为它不做任何解析，只是做设置工作。

## NQP::Grammar.TOP (2)

`TOP` 做的第一件事是建立一个**语言编织**。

    my %*LANG;
    %*LANG<Regex>         := NQP::Regex;
    %*LANG<Regex-actions> := NQP::RegexActions;
    %*LANG<MAIN>          := NQP::Grammar;
    %*LANG<MAIN-actions>  := NQP::Actions;

虽然我们没有太早地区分，当我们开始解析一个 `token`，`rule` 或 `regex` 时，我们实际上是**切换语言**。嵌套在正则表达式内部的块将轮流切换回主语言。

因此，`%*LANG` 会跟踪我们在解析中使用的当前语言集，纠缠在一起编织美丽的头发。 Rakudo 在其穗带中有第三种语言：`Q`，即引用语言。

## NQP::Grammar.TOP (3)

接下来，设置当前**元对象**的集合。每个包声明器（`class`，`role`，`grammar`，`module`，`knowhow`）被映射到一个实现这种包的对象。

    my %*HOW;
    %*HOW<knowhow>      := nqp::knowhow();
    %*HOW<knowhow-attr> := nqp::knowhowattr();

我们只有一个内置函数 - `knowhow`。它支持拥有方法和属性，但不支持角色组合或继承。

所有更有趣的元对象都是用 KnowHOW 编写的，并且是在启动时加载的模块中。我们将在第2天更详细地回到这个主题。

## NQP::Grammar.TOP (4)

接下来，创建一个 `NQP::World` 对象。这表示一个程序的**声明方面**（如类声明）。

    my $file := nqp::getlexdyn('$?FILES');
    my $source_id := nqp::sha1(self.target()) ~
        (%*COMPILING<%?OPTIONS><stable-sc> ?? '' !! '-' ~ ~nqp::time_n());
    my $*W := nqp::isnull($file) ??
        NQP::World.new(:handle($source_id)) !!
        NQP::World.new(:handle($source_id), :description($file));

每个编译单元需要有一个**全局唯一句柄**。由于 NQP 引导，我们通常必须使用比源更多的东西，否则运行的编译器和正被编译的编译器将有重叠的句柄！

（当移植到新的 VM 时需要交叉编译 NQP 本身时，`--stable-sc` 选项禁止这种情况，）

## NQP::Grammar.comp_unit 

接下来，我们到达 `comp_unit`。在这里，剥离出本质。

    token comp_unit {
        :my $*UNIT := $*W.push_lexpad($/);
        
        # Create GLOBALish - the current GLOBAL view.
        :my $*GLOBALish := $*W.pkg_create_mo(%*HOW<knowhow>,
                                             :name('GLOBALish'));
        {
            $*GLOBALish.HOW.compose($*GLOBALish);
            $*W.install_lexical_symbol($*UNIT, 'GLOBALish', $*GLOBALish);
        }
        
        # This is also the starting package.
        :my $*PACKAGE := $*GLOBALish;
        { $*W.install_lexical_symbol($*UNIT, '$?PACKAGE', $*PACKAGE); }
        
        <.outerctx>
        <statementlist>
        [ $ || <.panic: 'Confused'> ]
    }

## 剖析 comp_unit: 作用域

`$*W` 上有与作用域相关的各种方法。

**`$*W.push_lexpad($/)`** 用于输入一个新的词法作用域，嵌套在当前词法作用域的里面。它返回一个新的 `QAST::Block` 对象来表示它。

**`$*W.pop_lexpad()`** 用于退出当前词法作用域，返回它。

**`$*W.cur_lexpad()`** 用于获取当前作用域。

正如名字所暗示的，它只是一个堆栈。

## 剖析 comp_unit: pkg_create_mo

`NQP::World` 上的各种方法都是关于包的。 `pkg_create_mo` 方法用于创建表示新包的类型对象和元对象。

    :my $*GLOBALish := $*W.pkg_create_mo(%*HOW<knowhow>, :name('GLOBALish'));

由于**单独编译**，NQP 中的所有内容都以 `GLOBAL` 的干净的，空白视图开始，我们称之为 `GLOBALish`。这些在模块加载时是统一的。

`pkg_create_mo` 方法也用于处理像 `class` 这样的关键字; 在这种情况下，它使用 `%*HOW<class>`。

## 剖析 comp_unit: install_lexical_symbol

请考虑以下 NQP 代码段。

    for @acts {
        my class Act { ... }
        my $a := Act.new(:name($_));
    }

这个词法作用域将清楚地拥有符号 `Act` 和 `$a`。然而，它们在一个重要的方面有所不同。 `Act` 在**编译时是固定的**，而 `$a` 在每次循环时都是新的。编译时词法作用域中固定的符号安装有：

    $*W.install_lexical_symbol($*UNIT, 'GLOBALish', $*GLOBALish);

## 剖析 comp_unit: outer_ctx

`outer_ctx` token 看起来像这样:

    token outerctx { <?> }

嗯？这是一个"永远成功"的断言！然而，成功会触发 `NQP::Actions` 中的 `outer_ctx` 动作方法。其最重要的行是：

    my $SETTING := $*W.load_setting(
        %*COMPILING<%?OPTIONS><setting> // 'NQPCORE');

它加载 NQP 设置（默认为`NQPCORE`），这反过来会引入元对象（`class`，`role`等），以及类似于 `NQPMu` 和 `NQPArray` 的类型。

## statementlist

`comp_unit` token 的最后一件事是调用 `statementlist`，它做了它名字所暗示的：解析语句列表。

    rule statementlist {
        | $
        | [<statement><.eat_terminator> ]*
    }

`eat_terminator` 规则将匹配分号，但也处理闭合花括号的使用来终止语句。注意它之后是一个空格所以一个 `<.ws>` 将被插入。

## 语句

`statement` 规则期望找到一个 `statement_control`（像 `if`，`while` 和 `CATCH` 这样的东西 - 这是一个 protoregex！）或一个表达式，其后跟着一个语句修饰条件和/或循环。

    # **0..1 is like Perl 5 {0,1}; forces an array, which ? does not.
    token statement {
        <!before <[\])}]> | $ >
        [
        | <statement_control>
        | <EXPR> <.ws>
            [
            || <?MARKED('endstmt')>
            || <statement_mod_cond> <statement_mod_loop>**0..1
            || <statement_mod_loop>
            ]**0..1
        ]
    }

## 旁白: 表达式解析

当我们需要解析类似下面这样的东西时...

    $x * -$grad + $c

...我们需要注意优先级。尝试将优先级编码为一堆互相调用的规则将是非常低效的（表中每个级别一个调用！）并且很难维护。

因此，`EXPR` 实际上调用了一个**运算符优先级解析器**。它的实现存在于 `HLL::Grammar` 中，虽然我们不会在这个课程中研究它; 它稍微有点可怕，不是你可能需要改变的东西。

然而，我们会在之后看到如何配置它。

## Terms

`EXPR` 中的运算符优先级解析器不仅对运算符感兴趣，
而且对运算符所应用的**项**也感兴趣。当它想要一个术语，
它调用 `termish`，它反过来调用 `term`，另一个是 proto-regex。

对于我们的 `say('Hello, world')` 例子，有趣的术语是解析一个函数调用：

    token term:sym<identifier> {
        <deflongname> <?[(]> <args>  # <?[(]> is a lookahead
    }

现在我们到那里了！我们只需要解析一个名字和一个参数列表。

## deflongname

解析标识符（这里没有什么小聪明），后面跟一个可选的 colonpair（因为像 `infix<+>` 这样的东西是有效的函数名）。

    token deflongname {
        <identifier> <colonpair>**0..1
    }

我们解析这个之后，我们（终于！）以调用我们的第一个动作方法结束：

    method deflongname($/) {
        make $<colonpair>
             ?? ~$<identifier> ~ ':' ~ $<colonpair>[0].ast.named 
                    ~ '<' ~ colonpair_str($<colonpair>[0].ast) ~ '>'
             !! ~$/;
    }

它的目的是规范化 colonpair，如果有的话。无论如何，它 `make` 出了一个简单的字符串结果。

## Parsing arguments

Parses parentheses, then delegates off to the operator precedence parser again
to parse either a single argument or a comma separated list of arguments.

    token args {
        '(' <arglist> ')'
    }
    
    token arglist {
        <.ws>
        [
        | <EXPR('f=')>
        | <?>
        ]
    }

`f=` indicates loosest allowed precedence level

## Parsing values

Once again, the operator precedence parser calls `term`, and this time we end
up reaching `term:sym<value>`.

    token term:sym<value> { <value> }
    token value {
        | <quote>
        | <number>
    }

We have a quoted string, and thus end up in the `quote` protoregex, which in
turn puts us in the candidate that parses a single quoted string.

    token quote:sym<apos> { <?[']> <quote_EXPR: ':q'>  }

## Actions all the way up!

We've actually bottomed out in the parsing of this statement now. However, we
did not yet build any QAST nodes, which will indicate what the program should
actually *do* when we run it.

The `quote_EXPR` action inherited from `HLL::Actions` does the hard work with
regard to our quoted string:

    method quote:sym<apos>($/) { make $<quote_EXPR>.ast; }

It produces a `QAST::SVal` node, which represents a string literal:

    QAST::SVal.new( :value('Hello, world!') )

## value actions

The `value` action method simply checks if we parsed a quote or a number, and
then calls `make` with the AST of what we parsed.

    method value($/) {
        make $<quote> ?? $<quote>.ast !! $<number>.ast;
    }

And the value case of a `term` simply passes the value QAST on upwards:

    method term:sym<value>($/) { make $<value>.ast; }

## arglist actions

The `arglist` action method makes a `QAST::Op` node that represents a `call`.
The name will be attached later. It has to handle 3 cases: zero arguments (so
`$<EXPR>` was not matched), a single argument, or a comma-separated list of
arguments.

    method args($/) { make $<arglist>.ast; }
    method arglist($/) {
        my $ast := QAST::Op.new( :op('call'), :node($/) );
        if $<EXPR> {
            my $expr := $<EXPR>.ast;
            if nqp::istype($expr, QAST::Op) && $expr.name eq '&infix:<,>' {
                for $expr.list { $ast.push($_); }
            }
            else { $ast.push($expr); }
        }
        make $ast;
    }

## Function call actions

Now we have canonicalized the name and built a QAST node that represents a
call. Therefore, the action method for a term that is a function call takes
the call QAST node, sets its name (prepending an `&`) and passes it on up.

    method term:sym<identifier>($/) {
        my $ast := $<args>.ast;
        $ast.name('&' ~ $<deflongname>.ast);
        make $ast;
    }

Action methods higher up tend to combine together ASTs, generated by action
methods lower in the parse, into bigger ASTs.

## statement actions

Here's a simplified version. There's nothing really new to see in here. The
real thing is only more complex because it's handling the statement modifying
conditionals and loops.

    method statement($/, $key?) {
        my $ast;
        if $<EXPR> { $ast := $<EXPR>.ast; }
        elsif $<statement_control> { $ast := $<statement_control>.ast; }
        else { $ast := 0; }
        make $ast;
    }

The `0` simply means "we didn't find anything to parse here" - probably due
to reaching the end of the source.

## statementlist actions

Slightly simplified, but not much. A `QAST::Stmts` node indicates a set of
things to do sequentially. We push the QAST node for each statements (in our
case, one) onto it.

    method statementlist($/) {
        my $ast := QAST::Stmts.new( :node($/) );
        if $<statement> {
            for $<statement> {
                $ast.push($_.ast);
            }
        }
        else {
            $ast.push(default_for('$'));
        }
        make $ast;
    }

The `else` ensures we never produce an empty QAST::Stmts that would evaluate
to `null`, but rather evaluate to `NQPMu`.

## comp_unit actions

Finally, we reach the top! The `comp_unit` action method - again slightly
simplified - pushes the `QAST::Stmts` on to the `QAST::Block` node, making
these the statements to be executed by that block. Everything is then wrapped
in a `QAST::CompUnit`, which also specifies which language the code is from.

    method comp_unit($/) {
        # Push mainline statements into UNIT.
        my $mainline := $<statementlist>.ast;
        my $unit     := $*W.pop_lexpad();
        $unit.push($mainline);
        
        # Wrap everything in a QAST::CompUnit.
        make QAST::CompUnit.new(
            :hll('nqp'),
            # Much elided here; details later.
            $unit
        );
    }

## The end of the frontend

At this point, stage `parse` is completed! We have successfully executed the
grammar, which produced us a `Match` object. And attached to this match object
is a QAST tree that represents the semantics of the program.

Therefore, stage `ast` is rather straightforward.

    method ast($source, *%adverbs) {
        my $ast := $source.ast();
        self.panic("Unable to obtain AST"
            unless $ast ~~ QAST::Node;
        $ast;
    }

From here, we now enter the backend.

## Aside: why interleave parsing and AST building?

One may wonder why parsing is not completed in full, and then an AST built.
The answer is that in many cases, we need to evaluate pieces of the AST as we
go about the parsing. For example, in:

    BEGIN { say("OMG I'm alive!") }
    1 2

That BEGIN block should actually run and produce its output, even though there
is a syntax error right after it.

BEGIN-time things can have side-effects that actually influence the parse that
follows on from them.

## Code generation: a quick overview

The job of a backend is to take a QAST tree and produce code for the target
runtime. This, once again, is organized by a set of stages. Their names will
vary depending on if you are targetting Parrot, the JVM, MoarVM, etc.

We shall postpone looking at the details of any of those stages until later,
and even then shall not dive too deeply into them. Much of the code that is
contained within them is unlikely to change much in the future, and to make
sense of much of it needs an intimate knowledge of the backend in question.

For now, we'll treat these stages as a magical black box. :-)

## Building a tiny language from scratch

So, that's diving in to NQP. It can be a little overwhelming, and so it's good
to practice on something a bit smaller.

Therefore, we'll build ourselves a couple of small compilers. I'll do one here,
and you'll do one in the exercise.

The funny thing is that mine will be a Ruby subset.

The funnier thing is that yours will be a PHP subset.

We'll start by achieving "Hello, world", then in the next section - as we learn
more about QAST - start to add language features.

## Stubbing a compiler

Just subclass three things from the NQPHLL library.

    use NQPHLL;
    
    grammar Rubyish::Grammar is HLL::Grammar {
    }
    
    class Rubyish::Actions is HLL::Actions {
    }
    
    class Rubyish::Compiler is HLL::Compiler {
    }
    
    sub MAIN(*@ARGS) {
        my $comp := Rubyish::Compiler.new();
        $comp.language('rubyish');
        $comp.parsegrammar(Rubyish::Grammar);
        $comp.parseactions(Rubyish::Actions);
        $comp.command_line(@ARGS, :encoding('utf8'));
    }

## We already have a REPL

If we run the code from the previous slide, we find we already have ourselves
a simple REPL (Read Eval Print Loop).

Predictably, trying to run things doesn't work:

    > puts "Hello world"
    Method 'TOP' not found for invocant of class 'Rubyish::Grammar'

Of course, that also tells us exactly what we should do next...

## A basic grammar

Rubyish is line-oriented, so each statement is separated by a newline, and
only horizontal whitespace is allowed between tokens.

    grammar Rubyish::Grammar is HLL::Grammar {
        token TOP          { <statementlist> }
        
        rule statementlist { [ <statement> \n+ ]* }
        
        proto token statement {*}
        token statement:sym<puts> {
            <sym> <.ws> <?["]> <quote_EXPR: ':q'>
        }
        
        # Whitespace required between alphanumeric tokens
        token ws { <!ww> \h* || \h+ }
    }

## What have we now?

With this, we can now parse our simple program, but it fails when trying to
obtain the AST:

    > puts "Hello world"
    Unable to obtain AST from NQPMatch

Which, again, tells us what we need to do next: actions!

## Basic actions

    class Rubyish::Actions is HLL::Actions {
        method TOP($/) {
            make QAST::Block.new( $<statementlist>.ast );
        }
        
        method statementlist($/) {
            my $stmts := QAST::Stmts.new( :node($/) );
            for $<statement> {
                $stmts.push($_.ast)
            }
            make $stmts;
        }
        
        method statement:sym<puts>($/) {
            make QAST::Op.new(
                :op('say'),
                $<quote_EXPR>.ast
            );
        }
    }

## It works!

Recall that the backends are language independent; they simply expect a QAST
tree as input. And our actions produce one. As such, we now have a working
compiler for our very, very simple language.

    > puts "Hello world"
    Hello World

We can also dump the AST:

    - QAST::Block
      - QAST::Stmts puts \"Hello, world\"\n
        - QAST::Op(say)
          - QAST::SVal(Hello, world)

## In summary...

We've walked our way through the flow of control from invoking NQP at the
command line, seeing it parse our program, build up a QAST tree for it and
pass it off to the backend for compilation.

We then used this same technology to build a tiny compiler up from scratch.

Since it's built on the same technology as NQP and Rakudo, it gets the same
benefits. For example, out of the box our compiler already works both on
Parrot and the JVM.

## Exercise 4

In this exercise, you'll build the PHPish equivalent of my Rubyish.

The main difference is that the keyword you want is `echo`, and the lines are
separated by semicolons rather than newline characters.

# QAST

*Between frontend and backend: the Q Abstract Syntax Tree*

## Digging deeper into QAST

We've already built some very simple QAST trees so far. However, they have
barely scratched the surface of what is available in QAST.

In this section of the course, we will look at a much wider range of node
types and the options they support.

To provide concrete examples, Rubyish will be extended to support a wider
range of language features.

## QAST::Node: children

All of the QAST node types inherit from the base class `QAST::Node`.

All QAST nodes support having **child nodes**. The initial set of child nodes may
be passed as positional arguments to `new`. On any node, it's possible to:

    my $first := $ast[0];       # get first child
    $ast[0] := $child;          # set first child
    $ast.push($child);          # push a child
    $child := $ast.pop();       # pop a child
    $ast.unshift($child);       # unshift a child
    $child := $ast.shift();     # shift a child
    @children := $ast.list();   # get underlying children list

## QAST::Node: annotations

All QAST nodes can be given arbitrary **annotations** by using hash indexing
on the node.

    $var<used> := 1;

This can be very useful, but it's easy to overuse and create a tangled mess.
Yes, I learned this the hard way.

All annotations can be obtained using the `hash` method:

    my %anno := $var.hash();

## QAST::Node: return type

There are two other important things you can do with a QAST node. All nodes
can be annotated with the type that they will evaluate to.

    $ast.returns($some_type);

Note that you **specify a type object** to represent the type, *not* a string
name of the type! In some cases, the type set here is used in code generation
(for example, natively typed variables get their native storage allocated by
virtue of this).

This can also be set when creating a node in the first place:

    QAST::Var.new( ..., :returns(int) )

## QAST::Node: node

One other important thing we may wish to do is associate a QAST node with a
source location. This information is persisted by the backend code generation,
such that it can be used to produce meaningful backtraces when runtime errors
occur.

The `node` method expects to be given a match object:

    $ast.node($/);

Once again, it can be specified (typically on QAST::Stmts nodes) as a node
constructor argument.

    my $ast := QAST::Stmts.new( :node($/) );

## The top of the tree

At the top level, a QAST tree must have either a `QAST::CompUnit` or a
`QAST::Block`.

A **`QAST::CompUnit`** represents a compilation unit. It should have a single
child which is a `QAST::Block`. However, it can also specify many other bits
of configuration; we'll see more later.

A **`QAST::Block`** represents a lexical scope. Whenever one `QAST::Block` is
nested inside another, it represents a nested lexical scope that can see the
variables in the outer one. Combined with cloning, this also facilitates
closure semantics.

## Literals: QAST::IVal, QAST::NVal and QAST::SVal

These three node types represent integer, floating point and string literals.
If we update our grammar to parse different kinds of value:

    proto token value {*}
    token value:sym<string>  { <?["]> <quote_EXPR: ':q'> }
    token value:sym<integer> { '-'? \d+ }
    token value:sym<float>   { '-'? \d+ '.' \d+ }

Then we can write the actions as:

    method value:sym<string>($/) {
        make $<quote_EXPR>.ast;
    }
    method value:sym<integer>($/) {
        make QAST::IVal.new( :value(+$/.Str) )
    }
    method value:sym<float>($/) {
        make QAST::NVal.new( :value(+$/.Str) )
    }

## Trying our literals

After a small tweak to `puts` parsing...

    token statement:sym<puts> {
        <sym> <.ws> <value>
    }

...and the matching tweak in the actions, we can now do:

    > puts 42
    42
    > puts 0.999
    0.999
    > puts "It's not a bacon tree, it's a hambush!"
    It's not a bacon tree, it's a hambush!

## Operations: QAST::Op

The `QAST::Op` node is the gateway to an incredible number of operations. They
are the same ones available through the `nqp::op(...)` syntax.

Typically, a QAST::Op node looks something like this:

    QAST::Op.new(
        :op('add_n'),
        $left_child_ast,
        $right_child_ast
    )

The operation is specified with the `:op(...)` named argument, and operands
are the node's children.

## Parsing some mathematical operators (1)

Let's add addition, subtraction, multiplication and division. For these, we
need to set up the operator precedence parser, configuring two precedence
levels.

    INIT {
        # Steal precedence level names from Perl 6 grammar
        Rubyish::Grammar.O(':prec<u=>, :assoc<left>', '%multiplicative');
        Rubyish::Grammar.O(':prec<t=>, :assoc<left>', '%additive');
    }

Note that the `O` method we call here is inherited from `HLL::Grammar`. The
first argument specifies precedence level and associativity. The second then
saves this particular configuration by name, so we can refer to it when we
declare operators.

## Parsing some mathematical operators (2)

With the precedence levels in place, we can add some operators into the
grammar. This is done by adding them to the `infix` protoregex, which we
inherit from `HLL::Grammar`.

    token infix:sym<*> { <sym> <O('%multiplicative, :op<mul_n>')> }
    token infix:sym</> { <sym> <O('%multiplicative, :op<div_n>')> }
    token infix:sym<+> { <sym> <O('%additive, :op<add_n>')> }
    token infix:sym<-> { <sym> <O('%additive, :op<sub_n>')> }

The **`:op<...>`** syntax instructs the `EXPR` action method we inherit from
`HLL::Actions` to construct a `QAST::Op` node of that op for us!

## Terms

We are nearly ready to use the operator precedence parser, but not quite. We
must also instruct it on how to **obtain a term**. We inherit a `term` protoregex
from `HLL::Grammar`, and so need only add candidates for it.

For us, that means a candidate for a term that is a value:

    token term:sym<value> { <value> }

And the matching action method:

    method term:sym<value>($/) { make $<value>.ast; }

## Wiring it all up

The final thing we need to do is update the grammar rule for `puts`:

    token statement:sym<puts> {
        <sym> <.ws> <EXPR>
    }

Along with the action method:

    method statement:sym<puts>($/) {
        make QAST::Op.new(
            :op('say'),
            $<EXPR>.ast
        );
    }

## Trying out our operators

Basic arithmetic now works, and precedence is done correctly.

    > puts 10 * 9 + 1
    91

We may also inspect the AST to see the QAST::Op nodes:

    - QAST::Block
      - QAST::Stmts puts 10 * 9 + 1\n
        - QAST::Op(say)
          - QAST::Op(add_n &infix:<+>) +
            - QAST::Op(mul_n &infix:<*>) *
              - QAST::IVal(10)
              - QAST::IVal(9)
            - QAST::IVal(1)

## Sequencing: QAST::Stmts and QAST::Stmt

There are two node types that represent running each of their children in
order

**`QAST::Stmts`** does, quite literally, nothing more than that

**`QAST::Stmt`** has the added effect of stating that any temporaries that
are created during code generation will not be needed beyond the end of this
node's execution

Generally, using them in places where the language user would think of having
a set of statements vs. a single statement makes sense.

## Block structure

A common idiom, though in no way enforced, is for a `QAST::Block` to have two
`QAST::Stmts` nodes within it

The first one is used to hold **declarations**, for example of variables or of
nested routines

The second one is used to hold **the statements parsed by statementlist** for
that block

This idiom is used in both NQP and Rakudo; for example:

    $block[0].push(QAST::Var.new(:name<$/>, :scope<lexical>, :decl<var>));

## Variables

It's time to add variables to Rubyish! In Rubyish, variables aren't declared
explicitly. Instead, they are declared in the current scope on their first
assignment.

First, let's add a precedence level for assignment:

    Rubyish::Grammar.O(':prec<j=>, :assoc<right>',  '%assignment');

And parse the assignment operator, using the `bind` NQP operation which will
bind the expression on the right to a variable on the left:

    token infix:sym<=> { <sym> <O('%assignment, :op<bind>')> }

## Expressions as statements

One thing you may recall from the NQP grammar is that an expression was also a
valid statement. We need to do that in Rubyish too.

This means adding to the grammar:

    token statement:sym<EXPR> { <EXPR> }

And the actions:

    method statement:sym<EXPR>($/) { make $<EXPR>.ast; }

## Identifier parsing

For now, we'll treat all identifiers as if they were variables. We parse them
like this:

    token term:sym<ident> {
        :my $*MAYBE_DECL := 0;
        <ident>
        [ <?before \h* '=' [\w | \h+] { $*MAYBE_DECL := 1 }> || <?> ]
    }

Notice how this looks ahead to see if we can find an assignment operator with
whitespace around it or an identifier right after it (must not treat `==` as
if it were an assignment!)

A dynamic variable is used to convey if an assignment happens, which may mean
we have a declaration.

## Identifier actions

Here is a first, cheating attempt at the actions for an identifier.

    method term:sym<ident>($/) {
        if $*MAYBE_DECL {
            make QAST::Var.new( :name(~$<ident>), :scope('lexical'),
                                :decl('var') );
        }
        else {
            make QAST::Var.new( :name(~$<ident>), :scope('lexical') );
        }
    }

This does allow us to run:

    a = 7
    b = 6
    puts a * b

## The problem

Things come unstuck fairly quickly, unfortunatley. Every assignment is now
taken to be a declaration. Thus:

    a = 1
    puts a
    a = 2
    puts a

Fails with:

    Error while compiling block: Error while compiling op bind:
    Lexical 'a' already declared

## The symbol table

Every `QAST::Block` comes with a symbol table that can be used to store extra
information about the symbols declared within it.

Really, it's just a hash of hashes, the first hash keyed on the symbol and the
inner hashes storing whatever information we wish.

We can add to or update a symbol's entries by doing:

    $block.symbol($ident, :declared(1));

We can get hold of the current information held on a symbol by doing:

    my %sym := $block.symbol($ident);

We can use this to track declaredness!

## Next challenge: keeping track of the block

We need to have access to the current block we are declaring things in before
we can use `symbols`. This is most easily handled by placing it in a dynamic
variable, creating it in the `TOP` grammar rule:

    token TOP {
        :my $*CUR_BLOCK := QAST::Block.new(QAST::Stmts.new());
        <statementlist>
        [ $ || <.panic('Syntax error')> ]
    }

With the `TOP` action method becoming:

    method TOP($/) {
        $*CUR_BLOCK.push($<statementlist>.ast);
        make $*CUR_BLOCK;
    }

## Using symbol

Now, we can use `symbol` to track what was already declared and not re-declare
it.

    method term:sym<ident>($/) {
        my $name := ~$<ident>;
        my %sym  := $*CUR_BLOCK.symbol($name);
        if $*MAYBE_DECL && !%sym<declared> {
            $*CUR_BLOCK.symbol($name, :declared(1));
            make QAST::Var.new( :name($name), :scope('lexical'),
                                :decl('var') );
        }
        else {
            make QAST::Var.new( :name($name), :scope('lexical') );
        }
    }

## Other scopes

The `QAST::Var` node isn't just for lexical scoping. The available scopes are:

    lexical         visible to nested blocks
    local           like lexical, but not visible to nested blocks
    contextual      dynamically scoped lookup of a lexical
    attribute       object attribute (children: invocant, package)
    positional      array indexing (children: array, index)
    associative     hash indexing (children: hash, key)

Note that only the first 3 make sense as a declaration. Also note that Rakudo
does not use the last 2 (its array and hash handling is factored differently),
though NQP does.

## Routines

To demonstrate lexical scoping a little more, let's add routines. The syntax
for declaring and calling them is as follows:

    def greet
        puts "hello"
    end
    greet()

We'll keep things simple by not handling the other forms of calling.

## Parsing a routine declaration

Nothing especially new in here. We take care to start a new lexical scope, so
any declarations made will not polluate the surrounding scope. The split is so
the first token's action method can see the `$*CUR_BLOCK` to install into.

    token statement:sym<def> {
        'def' \h+ <defbody>
    }
    rule defbody {
        :my $*CUR_BLOCK := QAST::Block.new(QAST::Stmts.new());
        <ident> \n
        <statementlist>
        'end'
    }

NQP and Rakudo do pretty much the same, the only difference being that they
abstract the pushing/popping of the blocks and keep a stack of them.

## Parsing calls

A call is an identifier followed by some parentheses. We also take care to
avoid keywords.

    token term:sym<call> {
        <!keyword>
        <ident> '(' ')'
    }

The `<!keyword>` is also applied to `term:sym<ident>`.

## Actions for routine declaration

`defbody` finishes up the `QAST::Block`, and it is installed as a lexical by
`statement:sym<def>`.

    method statement:sym<def>($/) {
        my $install := $<defbody>.ast;
        $*CUR_BLOCK[0].push(QAST::Op.new(
            :op('bind'),
            QAST::Var.new( :name($install.name), :scope('lexical'),
                           :decl('var') ),
            $install
        ));
        make QAST::Op.new( :op('null') );
    }
    method defbody($/) {
        $*CUR_BLOCK.name(~$<ident>);
        $*CUR_BLOCK.push($<statementlist>.ast);
        make $*CUR_BLOCK;
    }

## Invocation

Calling is an operation, and therefore done with `QAST::Op`. By default, the
name of the thing to call - which will be resolved lexically - is specified in
the `name` named argument.

    method term:sym<call>($/) {
        make QAST::Op.new( :op('call'), :name(~$<ident>) );
    }

Any case where `name` is not specified will take the first child of the node
as the thing to invoke. Thus we could have written:

    method term:sym<call>($/) {
        make QAST::Op.new(
            :op('call'),
            QAST::Var.new( :name(~$<ident>), :scope('lexical')
        );
    }

But (at least on JVM) that thwarts an optimization, so don't.

## Parameters and arguments

Argument and parameter handling involve no node types we haven't seen before.
Arguments are just children to the `QAST::Op` call node, and parameters are
simply `QAST::Var` nodes with the `decl` set to `param`.

First, let's add parsing for parameters.

    rule defbody {
        :my $*CUR_BLOCK := QAST::Block.new(QAST::Stmts.new());
        <ident> <signature>? \n
        <statementlist>
        'end'
    }
    rule signature {
        '(' <param>* % [ ',' ] ')'
    }
    token param { <ident> }

## Parameter actions

The `param` action method looks like this:

    method param($/) {
        $*CUR_BLOCK[0].push(QAST::Var.new(
            :name(~$<ident>), :scope('lexical'), :decl('param')
        ));
        $*CUR_BLOCK.symbol(~$<ident>, :declared(1));
    }

Interestingly, it never does `make`. This may seem odd at first, as the action
methods elsewhere have done so. But it has no reason to; what we really wish
to do is install the declared parameter into the current block. It's easier to
just get at it contextually.

## Passing arguments

Here's a quick and easy way to parse the arguments:

    token term:sym<call> {
        <!keyword>
        <ident> '(' :s <EXPR>* % [ ',' ] ')'
    }

Then we update the actions:

    method term:sym<call>($/) {
        my $call := QAST::Op.new( :op('call'), :name(~$<ident>) );
        for $<EXPR> {
            $call.push($_.ast);
        }
        make $call;
    }

## So far...

So far, we have used the following QAST node types:

    QAST::Block     A lexical scope
    QAST::Stmts     A sequence of things to execute
    QAST::Stmt      As above, but also a temporaries boundary
    QAST::Op        An operation of some kind
    QAST::Var       A variable or parameter usage/declaration
    QAST::IVal      Integer literal
    QAST::NVal      Floating point literal
    QAST::SVal      String literal

We'll consider a few more node types today; we'll put off some (`QAST::WVal`
and `QAST::Regex`) until tomorrow.

## Block references with QAST::BVal

A QAST::Block should only ever appear once inside a QAST tree. Where it is
placed defines its lexical scope.

So what if you want to **refer to a `QAST::Block`** elsewhere in the tree?
That's what a `QAST::BVal`, short for Block Value, is for. For example, it is
used when emitting code to make the CORE setting be a program's outer lexical
scope.

    my $set_outer := QAST::Op.new(
        :op('forceouterctx'),
        QAST::BVal.new( :value($*UNIT) ),
        QAST::Op.new(
            :op('callmethod'), :name('load_setting'),
            # stuff left out here
        ));

## Boxed vs. unboxed, void vs. non-void context

As the backend code generation takes place, it may need to box and/or unbox
things, or it may determine that something will be in a void (sink) context.

While it can reliably produce working code, it may not be efficient. Consider
the integer constant handling here:

    my int $x = 42;     # Needs an unboxed native int
    my $x = 42;         # Needs a boxed Int object

When we write the action method for integer literals, we have a dilemma. Should
we emit a `QAST::IVal`, which will have to be boxed in the second case? Or
should we put an Int constant 42 into the constants pool and reference it with
a `QAST::WVal` (more on this node type tomorrow)?

## QAST::Want to the rescue

Rather than choosing, we can present both options, and let the code generator
pick whichever will be most efficient. This is done through the `QAST::Want`
node.

    QAST::Want.new(
        QAST::WVal.new( :value($boxed_constant) ),
        'Ii', QAST::IVal.new( :value($the_value) )
    )

The first child is the default thing. It is followed by a set of selectors for
different contexts we may be in.

    Ii      native integer
    Nn      native floating point number
    Ss      native string
    v       void

## The backend escape hatch: QAST::VM (1)

Sometimes, there's a need to **do things conditionally by backend**, or to do
some **VM-specific** operation. The QAST::VM node handles this need.

For example, here is some code from NQP that loads the NQP module loader. It
needs to know what filename to look for by backend.

    QAST::Op.new(
        :op('loadbytecode'),
        QAST::VM.new(
            :parrot(QAST::SVal.new( :value('ModuleLoader.pbc') )),
            :jvm(QAST::SVal.new( :value('ModuleLoader.class') ))
        ))

If there's no applicable option for the current backend, an exception will be
thrown by the code generator.

## The backend escape hatch: QAST::VM (2)

The `QAST::VM` node type is also behind the `pir::op_SIG(...)` syntax that is
available in NQP and Rakudo. Here is how `pir::op` is parsed and implemented
in NQP.

    token term:sym<pir::op> {
        'pir::' $<op>=[\w+] <args>**0..1
    }

    method term:sym<pir::op>($/) {
        my @args := $<args> ?? $<args>[0].ast.list !! [];
        my $pirop := ~$<op>;
        $pirop := join(' ', nqp::split('__', $pirop));
        make QAST::VM.new( :pirop($pirop), :node($/), |@args );
    }

## At the top: QAST::CompUnit

QAST trees produced by Rakudo and NQP have a `QAST::CompUnit` at the top.

![30%](eps/qast-compunit-and-blocks.eps)

## What we can do with QAST::CompUnit

Here's a look at some of what `QAST::CompUnit` can do (we'll see it again
tomorrow).

    my $compunit := QAST::CompUnit.new(
        # Set the language this contains.
        :hll('nqp'),
        
        # What to do if the compilation unit is loaded as a module.
        :load(QAST::Op.new(
            :op('call'),
            QAST::BVal.new( :value($unit) )
        )),
        
        # What to do if the compilation unit is invoked as the main,
        # top-level program.
        :main(...),

        # 1 child, which is the top-level QAST::Block
        $unit
    );

## Exercise 5

In this exercise, you'll add a few features to PHPish, in order to explore
the QAST nodes we've been studying.

Looking at the NQP grammar and actions to understand how they work - or even
stealing from them wholesale and cargo-culting - is encouraged! `:-)`

# Exploring nqp:: ops

*Learn all the operations!*

## Just a glimpse

There are literally hundreds of available `nqp::op`s. They range from
arithmetic to string manipulation, from flow control (like looping) to type
creation.

We've already seen some of the operations. Tomorrow, we'll see a bunch more as
we look at 6model and serialization contexts, which have a bunch of `nqp::ops`
associated with them.

In this section, we'll take an overview of "the rest". The overview is not
exhaustive, as that would be exhausting.

Remember they can be used in the `nqp::op` form *or* in a `QAST::Op` node, so
this knowledge is reusable for both!

## Arithmetic

These come in native integer form:

    add_i   sub_i   mul_i   div_i   mod_i
    neg_i   abs_i

As well as native float form:

    add_n   sub_n   mul_n   div_n   mod_n
    neg_n   abs_n

To help with implementing rationals, we also have:

    lcm_i   gcd_i

## Numerics

The basic stuff:

    pow_n       ceil_n      floor_n
    ln_n        sqrt_n      log_n
    exp_n       isnanorinf  inf
    neginf      nan
    
Trigometric:

    sin_n   asin_n  cos_n   acos_n  tan_n
    atan_n  atan2_n sinh_n  cosh_n  tanh_n
    sec_n   asec_n  sech_n

## Relational

For comparing native integers, native floats and native strings (the code
generator will unbox as needed). For example, the native integer forms are:

    cmp_i       compare; returns -1, 0, or 1
    iseq_i      non-zero if equal
    isne_i      non-zero if non-equal
    islt_i      non-zero if less than
    isle_i      non-zero if less than or equal to
    isgt_i      non-zero if greater than
    isge_i      non-zero if greater than or equal to

The `_n` and `_s` forms all exist too.

## Array operations

There are various operations for manipulating arrays:

    atpos       atpos_i     atpos_n     atpos_s
    bindpos     bindpos_i   bindpos_n   bindpos_s
    push        push_i      push_n      push_s
    pop         pop_i       pop_n       pop_s
    shift       shift_i     shift_n     shift_s
    unshift     unshift_i   unshift_n   unshift_s
    splice      existspos   elems       setelems

Note that the natively typed versions are **not coercive**, but only work on
a natively typed array.

## Hash operations

Don't look too different from the array operations.

    atkey       atkey_i     atkey_n     atkey_s
    bindkey     bindkey_i   bindkey_n   bindkey_s
    existskey   deletekey   elems

These all assume that the keys are strings; any non-string key will be coerced
to a string first.

## Aside: Perl 6 use of the array/hash ops

In Perl 6, something like:

    @a[0] = 42;

Actually uses `atpos` to get the scalar container bound into the underlying
array storage, and then assigns to that container. `bindpos` is only used for
doing:

    @a[0] := 42;

Also, you never do this directly on a Perl 6 `Array` or `Hash` object. These
objects *contain* a lower-level array or hash as an attribute, and methods use
this ops on that.

## Creating lists and hashes

The `nqp::list` op creates a (low level) array with the elements passed to
it. As a result, it is a variable argument op.

    nqp::list($foo, $bar, $baz)

Natively typed lists can be created with `list_i`, `list_n` and `list_s`.

There is a similar `nqp::hash`, which expects a key, a value, ...

    nqp::hash('name', $name, 'age', $age)

Finally, `islist` and `ishash` tell you if something is a low-level array or
hash.

## String

String operations are mostly named as Perl 6 does.

    chars       uc          lc          x
    concat      chr         join        split
    flip        replace     substr      ord
    index       rindex      codepointfromname

There are also operations for checking character class membership. These are
mostly emitted when compiling regexes or in the regex-related classes, but may
be used elsewhere. They are:

    nqp::iscclass(class, str, index)
    nqp::findcclass(class, str, index, limit)
    nqp::findnotcclass(class, str, index, limit)

Where `class` is one of the `nqp::const::CCLASS_*`.

## Conditionals

The `if` and `unless` ops expect two or three children: a condition, a "then",
and an optional "else". Note that `elsif` in NQP and Perl 6 is compiled by
nesting `if` `QAST::Op` nodes.

    # AST for '$/.ast ?? $/.ast !! $/.Str'
    QAST::Op.new(
        :op('if'),
        QAST::Op.new(
            :op('callmethod'), :name('ast'),
            QAST::Var.new( :name('$/'), :scope('lexical') )
        ),
        QAST::Op.new(
            :op('callmethod'), :name('ast'),
            QAST::Var.new( :name('$/'), :scope('lexical') )
        ),
        QAST::Op.new(
            :op('callmethod'), :name('Str'),
            QAST::Var.new( :name('$/'), :scope('lexical') )
        )
    )

## Conditionals and arity-1 blocks

Both NQP and Perl 6 supporting things like:

    if %core_ops{$name} -> $mapper {
        return $mapper($qastcomp, $op);
    }

This evaluates `%core_ops{$name}`, then passes it in to `$mapper` if it's a
truthy value.

At QAST level, this is represented by the second child of the `if` op being a
`QAST::Block` whose `arity` is set to a non-zero value.

## Loops

There are four related loop constructs:

                                Loop while true    Loop while false
                                ---------------    ---------------
    Condition, then body      | while              until
    Body, then condition      | repeat_while       repeat_until               

They take two or three children:

* The condition
* The body
* Optionally, something to do after the body

If a `redo` control exception is thrown, the second child is re-evaluated. The
third is only evaluated after any `redo`s have taken place. It's used by the
Perl 6 (C-style) `loop` construct.

## Loop example

The Perl 6 `let` and `temp` keywords keep a list of containers and their
original values (a container, a value, etc.) This is the loop that goes
through this list at block exit to do restoration.

    $phaser_block.push(QAST::Op.new(
        :op('while'),
        QAST::Var.new( :name($value_stash), :scope('lexical') ),
        QAST::Op.new(
            :op('p6store'),
            QAST::Op.new(
                :op('shift'),
                QAST::Var.new( :name($value_stash), :scope('lexical') )
            ),
            QAST::Op.new(
                :op('shift'),
                QAST::Var.new( :name($value_stash), :scope('lexical') )
            ))));

## Other control structures

There are three others that are worth knowing about:

* **`for`** takes two children, something iterable (typically a low level
  array or list) and a block. It invokes the block for each thing in the
  iterable. Used in NQP only; Rakudo does iterators completely differently.
* **`ifnull`** takes two children. It evaluates the first. If it is not null,
  it just produces this value. If it *is* null, it evaluates the second child.
* **`defor`** is the same as `ifnull`, but considers definedness rather than
  nullness

## Throwing exceptions

There are various operations for creating and throwing an exception:

    newexception    Creates a new, empty, exception object
    setextype       Sets the exception category (nqp::const::CONTROL_*)
    setmessage      Sets the exception message (string)
    setpayload      Sets the exception payload (object)
    throw           Throws an exception object
    die             Makes/throws an exception with a string message

There is an easier way to throw some of the common control exceptions:

    QAST::Op.new( :op('control'), :name('next') )

Other valid names here are `redo` and `last`.

## Handling exceptions

The `handle` op is used to express exception handling. The first child is the
code to protect with the handler(s). The handlers are then specified as a
string specifying the kind of exception to handle, followed by the QAST to
run to handle it.

NQP and Rakudo keep a per-block `%*HANDLERS`, and build the `handle` op out of
it when the block is fully parsed.

    my $ast := $<statementlist>.ast;
    if %*HANDLERS {
        $ast := QAST::Op.new( :op('handle'), $ast );
        for %*HANDLERS {
            $past.push($_.key);
            $past.push($_.value);
        }
    }

## Working with exception objects

Within a handler, the following operations can be used. Except the first, they
all take an exception object.

    exception       Gets the current exception object
    getextype       Gets the exception category (nqp::const::CONTROL_*)
    getmessage      Gets the exception message (string)
    getpayload      Gets the exception payload (object)
    rethrow         Re-throws the exception
    resume          Resumes the exception, if possible

Finally, there are two more operations that relate to backtraces; `backtrace`
returns an array of hashes, each hash describing a backtrace entry, while
`backtracestrings` simply returns an array of strings describing the entries.

## Context introspection

Various operations are available to introspecting the symbols in a lexical
scope, or walk the dynamic (caller) or static (lexical) chain of scopes. They
are typically used to implement features such as the `CALLER` and `OUTER`
pseudo-packages in Perl 6.

    ctx             get an object representing the current context
    ctxouter        take a context and return its outer context, or null
    ctxcaller       take a context and return its caller contxt, or null
    ctxlexpad       take a context and return its lexpad
    curlexpad       get the current lexpad
    lexprimspec     given a lexpad and a name, get the name's primitive type

The lexpad itself can be used with the appropriate hash operations (`atkey`,
`bindkey`) to manipulate the symbols contained within it.

## Big integers

Perl 6 needs big integer support for its `Int` type. Therefore, it is provided
for in the NQP operations. The big integer operations are only valid on an
object with the `P6bigint` representation (more on representations tomorrow)
or something that boxes it.

Those operations that have a big integer result differ from their native
relatives by taking an extra operand, which is the type object for the result.
The following `multi`s from the Perl 6 setting illustrate this.

    multi infix:<+>(Int:D \a, Int:D \b) returns Int:D {
        nqp::add_I(nqp::decont(a), nqp::decont(b), Int);
    }
    multi infix:<+>(int $a, int $b) returns int {
        nqp::add_i($a, $b)
    }

The `_I` suffix is used for big integer ops.

## Exercise 6

If time allows, you can explore some of the nqp::ops by adding support to
PHPish for (take them in order, or pick those you'd find most fun):

* Basic numeric relational operators (`<`, `>`, `==`, etc.)
* `if`/`else if`/`else`
* `while` loops

See the exercise sheet for a few hints.

## That's all for today

Today, we've covered a lot of ground, starting out with the NQP language and
then building up to how it can be used to implement a simple compiler.

That's a good start, but we're still missing several very important pieces
that both NQP and Rakudo depend heavily on. This includes objects and the
concept of serialization contexts. We'll take these on tomorrow.

Any more questions?

(FIXUP)
