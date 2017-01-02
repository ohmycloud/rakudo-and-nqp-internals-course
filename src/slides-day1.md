标题: Rakudo and NQP Internals
子标题: The guts tormented implementers made
作者: Jonathan Worthington

## 关于这个课程

Perl 6 是一种大型语言, 包含许多要求正确实现的功能。

这样的软件项目很容易被难控制的复杂性淹没。
Rakudo 和 NQP 项目的早期阶段已经遭受了这样的困难, 因为我们学到了 - 艰难的方式 - 关于复杂性, 出现并可能在实现过程中不受限制地扩散。

本课程将教您如何使用 Rakudo 和 NQP 内部。 在他们的设计中编码是一个大量学习的过程, 关于如何(以及如何不)写一个 Perl 6 实现, 这个过程持续了多年。 因此, 本课程还将教你事情的来龙去脉。

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

接收直截了当的输入(文本)并产生直截了当的输出(文本或二进制), 但**内部的数据结构很丰富**

像字符串那样处理东西, 通常是最后的手段

## 运行时做什么

运行像 Perl 6 这样的语言不仅仅是将它转换为低级代码。 此外, 它需要**运行时支持**来提供:

* 内存管理
* I/O, IPC, OS 交互
* 并发
* 动态优化

## 构建我们需要的东西来构建东西

我们已经以现有的编译器构造技术进行了各种尝试来构建 Perl 6。 编译器的早期设计至少有一部分是基于常规假设的。

这样的尝试是信息性的, 但从长远来看还不够好。

Perl 6 提出了一些有趣的挑战...

## Perl 6 是使用 Perl 6 解析的

Perl 6 的标准文法(grammar)是用 Perl 6 写的。它依赖于...

* 可传递的 **最长 token 匹配** (我们会在之后看到更多关于它的东西)
* 能够在不同语言之间来回切换 (主语言, 正则表达式语言, 引用(quoting)语言)
* 能够**动态地派生出新语言** (新运算符, 自定义引用构造)
* 在自下而上的表达式解析和自顶向下的更大的结构解析之间无缝集成
* 保持**令人惊叹的错误报告**的各种状态

所有这些本质上代表了解析中的新范例。

## 非静态类型或动态类型

Perl 6 是一种 **渐进类型化**的语言。

    my int $distance = distance-between('Lund', 'Kiev');
    my int $time = prompt('Travel time: ').Int;
    say "Average speed: { $distance / $time }";

我们想利用 `$distance` 和 `$time` 原生整数来产生更好的代码, 如果我们不知道类型(应该只是输出代码中的原生除法指令)。

## 模糊编译时和运行时

运行时可以做一些编译时:

    EVAL slurp @demos[$n];

编译时可以做一些运行时:

    my $comp-time = BEGIN now;

**注意编译时计算的结果必须持久化直到运行时, 这它们之间可能有一个处理(process)边界!**

## NQP 作为语言

Perl 6 文法显然需要用 Perl 6 表示。这反过来将需要集成到编译器的其余部分。用 Perl 6 编写整个编译器是很自然的。

然而, 一个完整的 Perl 6 太大, 给它写一个好的优化器花费太多时间。
因此,  **NQP (Not Quite Perl 6)** 语言诞生了:它是 Perl 6 的一个子集, 用于实现编译器。NQP 和 Rakudo 大部分都是用 NQP 写的。

## NQP 作为编译器构造工具链

NQP `src` 目录中不仅仅是 NQP 本身。

* **NQP, how, core**: 这些包含 NQP 编译器, 元对象(它指定了 NQP 的类和 roles 的工作方式)和内置函数。
* **HLL**: 构造高级语言编译器的通用结构, 在 Rakudo 和 NQP 之间共享。
* **QAST**: Q 抽象语法树的节点。代表着程序语法的树节点。(即, 当它执行时会做什么)。
* **QRegex**: 解析和执行 regexes 和 grammars 时所提到的对象。
* **vm**: 虚拟机抽象层。因为 NQP 和 Rakudo 可以运行在 Parrot, JVM 和 MoarVM 上。

## QAST

QAST 树是 NQP 和 Rakudo 内部最重要的数据结构之一。

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

编译器工具链的另一个重要部分是 `nqp::op` 指令集。你会有两种方式遇到它, 并且了解它们之间的差异很重要！

您可以在 **NQP 代码** 中使用它们, 在这种情况下, 您说您希望在程序中的那个点上执行该操作:

    say(nqp::time_n())

在代表着正在编译的程序的 **QAST树** 中也使用完全相同的指令集:

    QAST::Op.new(
        :op('call'), :name('&say'),
        QAST::Op.new( :op('time_n') )
    )

## Bootstrapping in a nutshell

人们可能会想知道 NQP 几乎完全用 NQP 编写时是如何编译的。

在每个 `vm` 子目录中都有一个 `stage0` 目录。它包含一个编译的 NQP(Parrot上是PIR文件, JVM上是JAR文件等)然后:

![40%](eps/nqp-bootstrapping-stages.eps)

因此, 你 `make test` 的 NQP 是可以重新创建自身的 NQP。

通常, 我们使用最新版本来更新 `stage0`

## How Rakudo uses NQP

Rakudo 本身不是一个自举编译器, 这使得它的开发容易一点。大部分 Rakudo 是用 NQP 编写的。这包括:

* **编译器本身的核心**, 它解析 Perl 6 源代码, 构建 QAST, 管理声明并进行各种优化
* **元对象**, 它指定了不同类型(类, roles, 枚举, subsets)是如何工作的
* **bootstrap**, 它将足够的 Perl 6 核心类型组合在一起, 以便能够在 Perl 6 中编写内置的类, 角色和例程

因此, 虽然一些 Rakudo 是可访问的, 如果你知道 Perl 6, 知道 NQP  - 既作为一种语言又作为一种编译器工具链 - 是和大部分 Rakudo 的其他部分工作的入口。

# NQP 语言

*它不完全是 Perl 6(Not Quite Perl 6), 但是能很好地构建 Perl 6*

## 设计目标

NQP 被设计为……

* 理想的编写编译器相关的东西
* 几乎是 Perl 6 的一个子集
* 比 Perl 6 更容易编译和优化

注意, 它避免了

* 赋值
* Flattening and laziness
* 操作符的多重分派(因此没有重载)
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

在 NQP 中这些总是需要圆括号:

    say('Mushroom, mushroom');

像 Perl 6 中一样, 这为子例程的名字添加 & 符号并对该例程做词法查询。

然而, 没有列表操作调用语法:

    plan 42;    # "Confused" parse error
    foo;        # Does not call foo; always a term

这可能是 NQP 初学者最常见的错误。

## 变量

可以是 `my` (lexical) 或 `our` (package) 作用域的:

    my $pony;
    our $stable;

常用的符号集也是可用的:

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

目前, NQP 并不真正支持对变量的类型约束。唯一的例外是它会注意**原生类型**。

    my int $idx := 0;
    my num $vel := 42.5;
    my str $mug := 'coffee'; 

**注意:** 在 NQP 中, 绑定用于原生类型！这在 Perl 6 中是非法的, Perl 6 中原生类型只能被赋值。尽管这是非常武断的, 目前 Perl 6 中原生类型的赋值实际上被编译到 `nqp::bind(...)` op 中！

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

与 Perl 6 中子例程的声明很像, 但是 NQP 中即使没有参数, 参数列表也是强制的。你可以 `return` 或使用最后一个语句作为隐式返回值。

    sub mean(@numbers) {
        my $sum;
        for @numbers { $sum := $sum + $_ }
        return $sum / +@numbers;
    }

Slurpy 参数也是可用的, 就像 `|` 用来展开参数列表那样。

**注意:** 参数可以获得类型约束, 但是与变量一样, 当前只有原生类型。(例外:多重分派;以后会有更多。)

## Named arguments and parameters

支持命名参数:

    sub make_op(:$name) {
        QAST::Op.new( :op($name) )
    }
    
    make_op(name => 'time_n');  # 胖箭头语法
    make_op(:name<time_n>);     # Colon-pair 语法
    make_op(:name('time_n'));   # The same

**注意:** NQP 中没有 `Pair` 对象！Pairs - colonpairs 或 fat-arrow 对儿 - 仅在参数列表上下文中有意义。

## Blocks 和 pointy blocks

尖尖块提供了熟悉的 Perl 6 语法:

    sub op_maker_for($op) {
        return -> *@children, *%adverbs {
            QAST::Op.new( :$op, |@children, |%adverbs )
        }
    }

从这个例子可以看出, 它们有闭包语义。

**注意:** 普通块也可用作闭包, 但不像 Perl 6 那样使用隐式的 `$_` 参数。

## Built-ins 和 nqp::ops

NQP 具有相对较少的内置函数。但是, 它提供了对 NQP 指令集的完全访问。这里有几个常用的指令, 知道它们会很有用。

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

`try` 和 `CATCH` 指令也是可用的, 尽管不像完全的 Perl 6, 你没有期望去智能匹配 `CATCH` 的内部。一旦你到了那儿, 就认为异常被捕获到了(弹出一个显式的 `nqp::rethrow`)。

    try {
        something();
        CATCH { say("Oops") }
    }

## 类, 属性和方法

就像在 Perl 6 中一样, 使用 `class`, `has` 和 `method` 关键字来声明。类可以是词法(`my`)作用域的或包(`our`)作用域的(默认)。

    class VariableInfo {
        has @!usages;
        
        method remember_usage($node) {
            nqp::push(@!usages, $node)
        }
        
        method get_usages() {
            @!usages
        }
    }

`self` 关键字也是可用的, 方法可以具有像 subs 那样的参数。

## More on attributes

NQP 没有自动的存取器生成, 所以你不能这样做:

    has @.usages; # 不支持

支持原生类型的属性, 并且将直接有效地存储在对象体中。任何其他类型都被忽略。

    has int $!flags;

与 Perl 6 不同, 默认构造函数可用于设置私有属性, 因为这是我们所拥有的。

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

或者, MOP 可用于将 role 混合到单个对象中:

    method set_compile_time_value($value) {
        self.HOW.mixin(self, QAST::CompileTimeValue);
        self.set_compile_time_value($value);
    }

## 多重分派

支持基本多重分派。它是 Perl 6 语义的一个子集, 使用更简单(但兼容)的候选排序算法版本。

与完全的 Perl 6 不同, 你**必须写一个 `proto` ** sub 或方法; 没有自动生成。

    proto method as_jast($node) {*}
    
    multi method as_jast(QAST::CompUnit $cu) {
        # compile a QAST::CompUnit
    }
    
    multi method as_jast(QAST::Block $block) {
        # compile a QAST::Block
    }

## 练习 1

有机会熟悉基本的 NQP 语法, 如果你还没有这样做。

还有机会学习常见的错误看起来是什么样的, 所以如果你在实际工作中遇到他们, 你可以认出它们。 :-)

## Grammars

虽然在许多领域 NQP 相比完全的 Perl 6 相当有限, 但是 grammar 几乎支持相同的水平。这是因为 NQP 语法必须足够好以处理解析 Perl 6 本身。

Grammars 是一种类, 并且使用 `grammar` 关键字引入。

    grammar INIFile {
    }


事实上, grammars 太像类了, 以至于在 NQP 中, 它们是由相同的元对象实现的。区别是它们默认继承于什么, 并且你把什么放在它们里面。

## INI 文件

作为一个简单的例子, 我们将考虑解析 INI 文件。

带有值的键, 可能按章节排列。

    name = Animal Facts
    author = jnthn

    [cat]
    desc = The smartest and cutest
    cuteness = 100000

    [dugong]
    desc = The cow of the sea
    cuteness = -10

## 整体方法

grammar 包含一组用关键字 `token`, `rule` 或 `regex` 声明的规则。真的, 他们就像方法一样, 但是用规则语法写成。

    token integer { \d+ }       # one or more digits
    token sign    { <[+-]> }    # + or - (character class)

更复杂的规则由调用现有规则组成:

    token signed_integer { <sign>? <integer> }

这些对其他规则的调用可以被量化, 放在备选分支中, 等等。
## 旁白:grammar和正则表达式

在这一点上, 你可能想知道 grammar 和正则表达式是如何关联的。毕竟, grammar 似乎是由正则表达式那样的东西组成的。

还有一个 `regex` 声明符, 可以在 grammar 中使用。

    regex email { <[\w.-]>+ '@' <[\w.-]>+ '.' \w+ }

关键的区别是 **`regex` 会回溯**, 而 `rule` 或 `token` 不会。支持回溯涉及保持大量状态, 并且对于复杂的 grammar 解析大量输入, 这将快速消耗大量内存！大语言往往在解析器中避免回溯。

## 旁白: NQP 中的正则表达式

对于较小规模的东西, NQP 确实也在普通场景中为正则表达式提供支持。

    if $addr ~~ /<[\w.-]>+ '@' <[\w.-]>+ '.' \w+/ {
        say("I'll mail you maybe");
    }
    else {
        say("That's no email address!");
    }

这被求值为匹配对象。

## 解析条目

一个条目有一个键(一些单词字符)和一个值(直到行尾的所有内容):

    token key   { \w+ }
    token value { \N+ }

合在一起, 它们组成了一个条目:

    token entry { <key> \h* '=' \h* <value> }

`\h` 匹配任何水平空白(空格, 制表符等)。 `=` 号必须加引号, 因为任何非字母数字都被视为 Perl 6 中的正则表达式语法。

## 从 `TOP` 开始

grammar 的入口点是一个特殊的规则, “TOP”。现在, 我们查找整个文件是否含有包含条目的行, 或者只是没有。

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

NQP 自带一些内置支持, 用于跟踪 grammars 的去向。它不是一个完整的调试器, 但用它来查看 grammar 在失败之前走的有多远是有用的。它使用 trace-on 函数开启:

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

一个 section 有一个标题和许多条目。但是, 顶层也可以有条目。因此, 把这个分解是有意义的。

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

最后但并非最不重要的是 section token:

    token section {
        '[' ~ ']' <key> \n
        <entries>
    }

这个 `~` 语法很漂亮. 第一行就像这样:

    '[' <key> ']' \n

然而, 如果找不到闭合 `]` 就会产生一个描述性的错误消息, 而不只是失败匹配。

## Actions

解析 grammar 可以使用 **actions 类**; 它的方法具有所匹配 grammar 中的某些或所有规则的名字。

在**相应规则成功匹配之后**, actions 方法被调用。

![20%](eps/top-down-bottom-up.eps)

在 Rakudo 和 NQP 编译器中, **actions构造QAST树**。对于这个例子, 我们将做一些更简单的事情。

## Actions 示例: aim

给定像下面这样的 INI 文件:

    name = Animal Facts
    author = jnthn

    [cat]
    desc = The smartest and cutest
    cuteness = 100000

我们想使用 actions 类来建立一个**散列的散列**。顶级哈希将包含键 `cat` 和 `_`(下划线收集不在 section 中的任何键)。其值是该 section 中键/值对的哈希。

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

上一张幻灯片上的转储代码产生如下输出:

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

它由非声明性构造(如向前查看, 代码块或对默认`ws`规则的调用)或递归 subrule 调用界定。

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

那么我们如何运行查询呢?好吧, 下面是  `INSERT` 查询的 action 方法:

    method query:sym<insert>($/) {
        my %to_insert := $<pairlist>.ast;
        make -> @db {
            nqp::push(@db, %to_insert);
            [nqp::hash('result', 'Inserted 1 row' )]
        };
    }

在这里, 我们不使用数据结构。我们 `make` 了一个闭包来接收当前数据库状态(一个散列的数组, 其中每一个散列是一行)并把由 `pairlist` action 方法产生的散列推到那个数组中。

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

## 练习 3

一次练习 protoregexes 的机会并自己学习我们已经复习过的。

拿 SlowDB 这个我们已经思考过的例子来说。给这个例子添加 `UPDATE` 和 `DELETE` 查询支持。


## 限制和与完整 Perl 6 的其它区别

这儿有一个值得了解的其它事情的杂烩。

* 有一个 `use` 语句, 但它期望它使用已经预编译的任何东西。
* 没有数组展平; `[@a, @b]` 总是两个元素的数组
* 散列构造 `{}` 符只对空散列有效; 除了它之外的任何一个 `{}` 都会被当作一个 block
* `BEGIN` 块存在, 但在我们能看见的外部作用域中是高度受限的(只有类型, 而不是变量)

## 后端的区别

JVM 和 MoarVM 上的 NQP 相对比较一致。Parrot 上的 NQP 有点古怪: 不是所有的东西都是 6model 对象。即虽然在 JVM 和 MoarVM 上, NQP  中的 `.WHAT` 或 `.HOW` 会工作良好, 但是在 Parrot 上它会失败。这发生在整数, 数字和字符串字面值, 数组和散列, 异常和某些种类的代码对象身上。 

异常处理程序也有所不同。那些在 JVM 和 MoarVM 上运行的堆栈顶部的异常抛出点, 就像 Perl 6 语义那样。那些在 Parrot 上的 NQP 将解开然后运行, 恢复由continuation提供。注意, Rakudo 在所有后端都是一致的。

## 总而言之...

NQP 尽管是 Perl 6 的一个相对较小的子集, 但仍然包含相当多强大的语言特性。

一般来说, 对他们的需求是由在 Rakudo 中工作的人的需求所驱动的。因此, NQP 功能集是由编译器编写需求定义的。

我们所覆盖的 grammar 和 action 方法资料可能是最重要的, 因为这是理解 NQP 和 Perl 6 是如何编译的起点。

# 编译管道

*一步一步我们编译完了那个程序...*

## 从开始到完成

现在我们了解了一点作为语言的 NQP, 是时候潜入到下面, 看看当我们喂给 NQP 一个程序运行时会发生什么。

首先, 我们将考虑这个简单的例子...

    nqp -e "say('Hello, world')"

...一路从 NQP 的 sub `MAIN` 到出现输出。

我们将选择 JVM 后端来检查这个。

## "stagestats" 选项

我们可以通过使用 `--stagestats` 选项来了解 NQP 内部发生的情况, 该选项显示了编译器每个阶段经历的时间。

nqp --stagestats -e "say('Hello, world')"

    Stage start      :   0.000      # 启动
    Stage classname  :   0.010      # 计算类名
    Stage parse      :   0.067      # 解析源文件, 构造 AST
    Stage ast        :   0.000      # 获取 AST
    Stage jast       :   0.106      # 转换成 JVM AST
    Stage classfile  :   0.032      # 转换成 JVM 字节码
    Stage jar        :   0.000      # 可能创建一个 JAR
    Stage jvm        :   0.002      # 真正地运行该代码

## 倾倒解析树

我们可以得到一些转储的阶段。例如, `--target = parse` 将产生一个解析树的转储。

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

有时有用的是 `--target = ast`, 它转储了 QAST(下面的输出已经被简化)。

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

你甚至可以得到一些代表性的低级 AST, 它使用 `--target = jast` 变成 Java 字节码, 但它完全令人脑抽(下面的一小部分用以说明)。 :-)

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

我们的旅程从 NQP 的 `MAIN` sub 开始, 它位于 `src/NQP/Compiler.nqp`。
这里是一个略微简化的版本(剥离设置命令行选项和其它细节)。

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

其功能包括:

* 参数处理(委托给 HLL::CommandLine)
* 从磁盘读取源文件
* 调用每个阶段, 如果指定, 停在`--target`
* 提供 REPL
* 提供可插拔的方式来处理未捕获的异常

## 通过 HLL::Compiler 的路径

**`command_line`** 解析参数, 然后调用 `command_eval`

**`command_eval`** 工作, 基于参数, 如果我们应该从磁盘加载源文件, 从 `-e` 获取源或进入 REPL。路径调用了一系列方法, 但是所有方法都在 `eval` 中收敛。

**`eval`** 调用 `compile` 来编译代码, 然后调用它

**`compile`** 循环遍历这些阶段, 将前一个的结果作为下一个的输入

## 简化版的编译

Big takeaway:阶段是编译器对象或后端对象的方法。

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

编译器可以在管道中插入额外的阶段。例如, Rakudo 插入其优化器。

    $comp.addstage('optimize', :after<ast>);

之后, 在 `Perl6::Compiler` 中, 它提供了一个 `optimize` 方法:

    method optimize($ast, *%adverbs) {
        %adverbs<optimize> eq 'off' ??
            $ast !!
            Perl6::Optimizer.new.optimize($ast, |%adverbs)
    }

## 前端和后端

早些时候, 我们看到`compile'在当前编译器对象上, 然后在后端对象上寻找阶段方法。

**编译器对象**是关于我们正在编译的语言**(NQP, Perl 6等)的**。我们共同称这些阶段为**前端**。

**后端对象**是关于目标VM的**, 我们想为(Parrot, JVM, MoarVM等)生成代码。它不与任何特定语言绑定。我们将这些阶段统称为**后端**。

![30%](eps/frontend-backend.eps)

## 前端, 后端和它们之间的 QAST

![30%](eps/frontend-backend-qast-between.eps)

前端的最后一个阶段总是给出一个 **QAST树**, 后端的第一个阶段总是期望一个QAST树。

一个**交叉编译器设置**只是有一个不同于我们正在运行的当前 VM 的后端。

## 在 NQP 中解析

解析阶段在语言的 grammar(对于我们的例子, NQP::Grammar)上调用 `parse`, 传递源代码和 `NQP::Actions`。它也可以打开跟踪。

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

正如在我们已经看到的 grammar 中, 执行从 `TOP` 开始。在 NQP 中, 我们发现它实际上是一个 `方法`, 而不是 `token` 或 `rule`！

    method TOP() {
        # Various things we'll consider in a moment.
        ...
        
        # Then delegate to comp_unit
        self.comp_unit;
    }

这实际上是 OK 的, 只要它最终返回一个 `Cursor` 对象。因为 `comp_unit` 会返回一个, 所有的都会工作的很好。

这是一个方法, 因为它不做任何解析, 只是做设置工作。

## NQP::Grammar.TOP (2)

`TOP` 做的第一件事是建立一个**语言编织**。

    my %*LANG;
    %*LANG<Regex>         := NQP::Regex;
    %*LANG<Regex-actions> := NQP::RegexActions;
    %*LANG<MAIN>          := NQP::Grammar;
    %*LANG<MAIN-actions>  := NQP::Actions;

虽然我们没有太早地区分, 当我们开始解析一个 `token`, `rule` 或 `regex` 时, 我们实际上是**切换语言**。嵌套在正则表达式内部的块将轮流切换回主语言。

因此, `%*LANG` 会跟踪我们在解析中使用的当前语言集, 纠缠在一起编织美丽的头发。 Rakudo 在其穗带中有第三种语言:`Q`, 即引用语言。

## NQP::Grammar.TOP (3)

接下来, 设置当前**元对象**的集合。每个包声明器(`class`, `role`, `grammar`, `module`, `knowhow`)被映射到一个实现这种包的对象。

    my %*HOW;
    %*HOW<knowhow>      := nqp::knowhow();
    %*HOW<knowhow-attr> := nqp::knowhowattr();

我们只有一个内置函数 - `knowhow`。它支持拥有方法和属性, 但不支持角色组合或继承。

所有更有趣的元对象都是用 KnowHOW 编写的, 并且是在启动时加载的模块中。我们将在第2天更详细地回到这个主题。

## NQP::Grammar.TOP (4)

接下来, 创建一个 `NQP::World` 对象。这表示一个程序的**声明方面**(如类声明)。

    my $file := nqp::getlexdyn('$?FILES');
    my $source_id := nqp::sha1(self.target()) ~
        (%*COMPILING<%?OPTIONS><stable-sc> ?? '' !! '-' ~ ~nqp::time_n());
    my $*W := nqp::isnull($file) ??
        NQP::World.new(:handle($source_id)) !!
        NQP::World.new(:handle($source_id), :description($file));

每个编译单元需要有一个**全局唯一句柄**。由于 NQP 引导, 我们通常必须使用比源更多的东西, 否则运行的编译器和正被编译的编译器将有重叠的句柄！

(当移植到新的 VM 时需要交叉编译 NQP 本身时, `--stable-sc` 选项禁止这种情况, )

## NQP::Grammar.comp_unit 

接下来, 我们到达 `comp_unit`。在这里, 剥离出本质。

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

**`$*W.push_lexpad($/)`** 用于输入一个新的词法作用域, 嵌套在当前词法作用域的里面。它返回一个新的 `QAST::Block` 对象来表示它。

**`$*W.pop_lexpad()`** 用于退出当前词法作用域, 返回它。

**`$*W.cur_lexpad()`** 用于获取当前作用域。

正如名字所暗示的, 它只是一个堆栈。

## 剖析 comp_unit: pkg_create_mo

`NQP::World` 上的各种方法都是关于包的。 `pkg_create_mo` 方法用于创建表示新包的类型对象和元对象。

    :my $*GLOBALish := $*W.pkg_create_mo(%*HOW<knowhow>, :name('GLOBALish'));

由于**单独编译**, NQP 中的所有内容都以 `GLOBAL` 的干净的, 空白视图开始, 我们称之为 `GLOBALish`。这些在模块加载时是统一的。

`pkg_create_mo` 方法也用于处理像 `class` 这样的关键字; 在这种情况下, 它使用 `%*HOW<class>`。

## 剖析 comp_unit: install_lexical_symbol

请考虑以下 NQP 代码段。

    for @acts {
        my class Act { ... }
        my $a := Act.new(:name($_));
    }

这个词法作用域将清楚地拥有符号 `Act` 和 `$a`。然而, 它们在一个重要的方面有所不同。 `Act` 在**编译时是固定的**, 而 `$a` 在每次循环时都是新的。编译时词法作用域中固定的符号安装有:

    $*W.install_lexical_symbol($*UNIT, 'GLOBALish', $*GLOBALish);

## 剖析 comp_unit: outer_ctx

`outer_ctx` token 看起来像这样:

    token outerctx { <?> }

嗯?这是一个"永远成功"的断言！然而, 成功会触发 `NQP::Actions` 中的 `outer_ctx` 动作方法。其最重要的行是:

    my $SETTING := $*W.load_setting(
        %*COMPILING<%?OPTIONS><setting> // 'NQPCORE');

它加载 NQP 设置(默认为`NQPCORE`), 这反过来会引入元对象(`class`, `role`等), 以及类似于 `NQPMu` 和 `NQPArray` 的类型。

## statementlist

`comp_unit` token 的最后一件事是调用 `statementlist`, 它做了它名字所暗示的:解析语句列表。

    rule statementlist {
        | $
        | [<statement><.eat_terminator> ]*
    }

`eat_terminator` 规则将匹配分号, 但也处理闭合花括号的使用来终止语句。注意它之后是一个空格所以一个 `<.ws>` 将被插入。

## 语句

`statement` 规则期望找到一个 `statement_control`(像 `if`, `while` 和 `CATCH` 这样的东西 - 这是一个 protoregex！)或一个表达式, 其后跟着一个语句修饰条件和/或循环。

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

...我们需要注意优先级。尝试将优先级编码为一堆互相调用的规则将是非常低效的(表中每个级别一个调用！)并且很难维护。

因此, `EXPR` 实际上调用了一个**运算符优先级解析器**。它的实现存在于 `HLL::Grammar` 中, 虽然我们不会在这个课程中研究它; 它稍微有点可怕, 不是你可能需要改变的东西。

然而, 我们会在之后看到如何配置它。

## Terms

`EXPR` 中的运算符优先级解析器不仅对运算符感兴趣, 
而且对运算符所应用的**项**也感兴趣。当它想要一个术语, 
它调用 `termish`, 它反过来调用 `term`, 另一个是 proto-regex。

对于我们的 `say('Hello, world')` 例子, 有趣的术语是解析一个函数调用:

    token term:sym<identifier> {
        <deflongname> <?[(]> <args>  # <?[(]> is a lookahead
    }

现在我们到那里了！我们只需要解析一个名字和一个参数列表。

## deflongname

解析标识符(这里没有什么小聪明), 后面跟一个可选的 colonpair(因为像 `infix<+>` 这样的东西是有效的函数名)。

    token deflongname {
        <identifier> <colonpair>**0..1
    }

我们解析这个之后, 我们(终于！)以调用我们的第一个动作方法结束:

    method deflongname($/) {
        make $<colonpair>
             ?? ~$<identifier> ~ ':' ~ $<colonpair>[0].ast.named 
                    ~ '<' ~ colonpair_str($<colonpair>[0].ast) ~ '>'
             !! ~$/;
    }

它的目的是规范化 colonpair, 如果有的话。无论如何, 它 `make` 出了一个简单的字符串结果。

## 解析参数

解析圆括号, 然后再次委派给运算符优先解析器来解析单个参数或逗号分隔的参数列表。

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

`f=` 表示允许的最松散的优先级。

## 解析值

再一次, 运算符优先级解析器调用 `term`, 这次我们来到了 `term:sym<value>`。

    token term:sym<value> { <value> }
    token value {
        | <quote>
        | <number>
    }

我们有一个带引号的字符串, 因此结束在 `quote` protoregex 中, 这反过来使我们成为解析单个引号字符串的候选人。

    token quote:sym<apos> { <?[']> <quote_EXPR: ':q'>  }

## 动作一路向上!

我们现在已经到达了解析这个语句的底部。 然而, 我们还没有构建任何 QAST 节点, 这些节点将指示程序应该实际**做**什么, 当我们运行它时。

从 `HELL::Actions` 继承的 `quote_EXPR` 动作对我们引用的字符串做了很多工作:

    method quote:sym<apos>($/) { make $<quote_EXPR>.ast; }

它产生一个 `QAST::SVal` 节点, 其代表一个字符串字面值:

    QAST::SVal.new( :value('Hello, world!') )

## value actions

`value` 动作方法只是检查我们是否解析了一个引号或一个数字, 然后用我们解析的 AST 调用 `make`。

    method value($/) {
        make $<quote> ?? $<quote>.ast !! $<number>.ast;
    }

而一个 `term` 的 value case 只是向上传递值 QAST:

    method term:sym<value>($/) { make $<value>.ast; }

## arglist actions

`arglist` 动作方法创建一个代表 `call` 的 `QAST::Op` 节点。
名称将在以后附加。 它必须处理 3 种情况: 零参数(因此 `$<EXPR>` 不匹配), 单个参数或逗号分隔的参数列表
参数。

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

## 函数调用 actions

现在我们已经规范化了名称并构建了代表调用的 QAST 节点。 因此, 作为函数调用的术语的action方法使用 call QAST 节点, 设置其名称(在前面加上`&`)并将其传递。

    method term:sym<identifier>($/) {
        my $ast := $<args>.ast;
        $ast.name('&' ~ $<deflongname>.ast);
        make $ast;
    }

更高的 action 方法倾向于将由解析中较低的 action 方法产生的 AST 组合成更大的 AST。

## statement actions

这是一个简化版本。 在这里看不到什么真正新的东西。 真正的事情只是更复杂, 因为它处理语句修改条件和循环。

    method statement($/, $key?) {
        my $ast;
        if $<EXPR> { $ast := $<EXPR>.ast; }
        elsif $<statement_control> { $ast := $<statement_control>.ast; }
        else { $ast := 0; }
        make $ast;
    }

`0` 只是意味着“我们在这里没有找到任何东西来解析” - 可能是由于到达了源的末端。

## statementlist actions

稍微简化下, 但不是很多。 `QAST::Stmts` 节点表示一组顺序执行的操作。 我们将每个语句(在我们的例子中, 一个)的 QAST 节点推送到它上面。

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

`else` 确保我们永远不会生成一个求值为“null”的空的 `QAST::Stmts`, 而是被计算为“NQPMu”。

## comp_unit actions

最后, 我们到达顶部！ `comp_unit` 动作方法 - 再次略微简化 - 将 `QAST::Stmts` 推送到 `QAST::Block` 节点上, 使这些语句由该块执行。 然后, 所有一切都被包装在一个 `QAST::CompUnit` 中, 它还指定了代码所来自的语言。

    method comp_unit($/) {
        # Push mainline statements into UNIT.
        my $mainline := $<statementlist>.ast;
        my $unit     := $*W.pop_lexpad();
        $unit.push($mainline);
        
        # Wrap everything in a QAST::CompUnit.
        make QAST::CompUnit.new(
            :hll('nqp'),
            # 这里省略很多, 稍后详细介绍。
            $unit
        );
    }

## 前端的结束

在这个时候, 阶段 `parse` 完成了！ 我们已经成功地执行了这个 grammar, 它产生了一个 `Match` 对象。 而附加到这个匹配对象上的是一个表示程序语义的 QAST 树。

因此, 阶段 `ast` 是相当简单的。

    method ast($source, *%adverbs) {
        my $ast := $source.ast();
        self.panic("Unable to obtain AST"
            unless $ast ~~ QAST::Node;
        $ast;
    }

从这儿开始, 我们现在进入后端。

## 旁白:为什么交错解析和AST创建呢?

你可能想知道为什么解析没有完全完成, 然后就构建了AST。
答案是在许多情况下, 当我们着手解析时我们需要计算AST片段。 例如, 在:

    BEGIN { say("OMG I'm alive!") }
    1 2

那个 BEGIN 块应该实际运行并产生它的输出, 即使它之后有一个语法错误。

BEGIN-time 的东西可能有副作用, 实际上影响从他们的那儿的解析。

## 代码生成:快速概览

后端的工作是接收一个 QAST  树并为目标运行时(target runtime)生成代码。这, 再一次, 是由一组阶段(stages)组织的。它们的名字会根据你是在 Parrot, JVM, MoarVM, etc 上而不同。


我们会推迟查看那些阶段中的任何一个的详情, 直到之后, 我们甚至还不能太深入它们。它们中包含的大部分代码在将来都不会改变, 为了掌握它们的大部分东西, 我们需要详细了解一些后端的知识。

现在, 我们会把这些阶段当作神奇的黑盒子。 :-)


## 从头开始创建一个小语言

所以, 这是深入到 NQP。 它有点汹涌, 所以它适合练习一些更小的东西。

因此, 我们将自己建立几个小的编译器。 我会在这里做一个, 你会在练习中做一个。

有趣的是, 我的将是一个 Ruby 子集。

更有趣的是, 你的将是一个 PHP 子集。

我们将从实现 “Hello, world” 开始, 然后在下一部分 - 当我们更多地了解 QAST 后 - 开始添加语言功能。

## Stubbing a compiler

只需从 NQPHLL 库中继承的三个东西。

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

## 我们已经拥有了 REPL

如果我们运行前一张幻灯片中的代码, 我们发现我们已经有了一个简单的 REPL(Read Eval Print Loop)。

不出所料, 试图运行的东西不能工作:

    > puts "Hello world"
    Method 'TOP' not found for invocant of class 'Rubyish::Grammar'

当然, 它也准确地告诉了我们下一步该做什么 。。。

## 一个基本的 grammar

Rubyish 是面向行的, 所以每个语句由换行符分隔, 并且在 tokens 之间只允许水平空格。

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

## 我们现在有什么?

有了这个, 我们现在可以解析我们的简单程序, 但是当尝试获取 AST 时却失败了:

    > puts "Hello world"
    Unable to obtain AST from NQPMatch

这再一次告诉我们下一步需要做什么:actions！

## 基本的 actions

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

## 有效果了!

回想一下, 后端是独立于语言的; 他们只需要一个 QAST 树作为输入。 我们的 actions 产生一个。 因此, 我们现在有一个非常非常简单的能工作的语言编译器。

    > puts "Hello world"
    Hello World

我们还可以输出这个 AST:

    - QAST::Block
      - QAST::Stmts puts \"Hello, world\"\n
        - QAST::Op(say)
          - QAST::SVal(Hello, world)

## 总之...

我们在命令行中调用 NQP 的控制流程, 看到它解析我们的程序, 构建一个 QAST 树, 并将其传递给后端进行编译。

然后我们使用这种技术从头开始构建一个小编译器。

因为它建立在与 NQP 和 Rakudo 相同的技术之上, 它获得相同的好处。 例如, 已经工作在 Parrot 和 JVM 上开箱即用的编译器。

## 练习 4

在本练习中, 您将构建相当于我的Rubyish 编译器的 PHPish 编译器 。

主要的区别是, 你想要的关键字是 “echo”, 并且行是用分号而不是换行符分隔语句。

# QAST

*在前端和后端之间: Q Abstract Syntax Tree*

## 进一步深入 QAST

到目前为止, 我们已经构建了一些非常简单的 QAST 树。 然而, 他们几乎都只留于 QAST 的表面。

在本课程的这一部分中, 我们将讨论更广泛的节点类型及它们支持的选项。

为了提供具体的例子, Rubyish 将被扩展以支持更广泛的语言功能。

## QAST::Node: children

所有的 QAST 节点类型都继承自基类 `QAST::Node`。

所有 QAST 节点都支持拥有**孩子节点**。 初始孩子节点集可以作为位置参数传递给 “new”。 在任何节点上, 可以:

    my $first := $ast[0];       # get first child
    $ast[0] := $child;          # set first child
    $ast.push($child);          # push a child
    $child := $ast.pop();       # pop a child
    $ast.unshift($child);       # unshift a child
    $child := $ast.shift();     # shift a child
    @children := $ast.list();   # get underlying children list

## QAST::Node: annotations

通过在节点上使用散列索引, 可以给所有 QAST 节点提供任意的**注解**。

    $var<used> := 1;

这可能非常有用, 但它很容易过度使用, 并造成混乱。
是的, 我费了一番功夫才学会它。

所有注解都可以使用 `hash` 方法获取:

    my %anno := $var.hash();

## QAST::Node: 返回类型

使用 QAST 节点你还可以做其他两件重要的事情。 所有节点都可以使用他们将被求值的类型进行注解。

    $ast.returns($some_type);

注意, 你**指定一个类型对象**来表示类型, *不是*类型的字符串名字！ 在某些情况下, 这里设置的类型用于代码生成(例如, 原生类型的变量通过这个来分配它们的原生存储)。

这也可以在创建节点时首先设置:

    QAST::Var.new( ..., :returns(int) )

## QAST::Node: node

我们可能希望做的另一个重要的事情是将 QAST 节点与源位置相关联。 该信息由后端代码生成来持久化, 使得当发生运行时错误时, 它可以用于产生有意义的回溯。

`node` 方法需要接收一个匹配对象:

    $ast.node($/);

再次, 它可以被指定(通常在QAST::Stmts节点上)为节点构造函数的参数。

    my $ast := QAST::Stmts.new( :node($/) );

## 树的顶部

在顶层, QAST 树必须要么具有 `QAST::CompUnit`, 要么具有 `QAST::Block`。

**`QAST::CompUnit`** 代表一个编译单元。 它应该有一个单独的 `QAST::Block` 孩子。 然而, 它也可以指定许多其他位的配置; 我们将在后面看到。

**`QAST::Block`** 代表词法作用域。 每当一个 `QAST::Block` 嵌套在另一个中时, 它代表一个嵌套的词法作用域, 它可以看到外部的变量。 结合克隆, 这也有利于闭包语义。

## 字面值: QAST::IVal, QAST::NVal 和 QAST::SVal

这三个节点类型表示整数, 浮点和字符串字面值。
如果我们更新我们的 grammar 来解析不同类型的值:

    proto token value {*}
    token value:sym<string>  { <?["]> <quote_EXPR: ':q'> }
    token value:sym<integer> { '-'? \d+ }
    token value:sym<float>   { '-'? \d+ '.' \d+ }

然后我们可以把 actions 写为:

    method value:sym<string>($/) {
        make $<quote_EXPR>.ast;
    }
    method value:sym<integer>($/) {
        make QAST::IVal.new( :value(+$/.Str) )
    }
    method value:sym<float>($/) {
        make QAST::NVal.new( :value(+$/.Str) )
    }

## 尝试我们的字面值

经过一个小小的调整, 使`puts` 能够解析...

    token statement:sym<puts> {
        <sym> <.ws> <value>
    }

...加上 actions 中的匹配调整, 我们现在可以做:

    > puts 42
    42
    > puts 0.999
    0.999
    > puts "It's not a bacon tree, it's a hambush!"
    It's not a bacon tree, it's a hambush!

## Operations: QAST::Op

`QAST::Op` 节点是到达令人难以置信数量的运算符的网关。 它们同样是通过 `nqp::op(...)` 语法可用的。

通常, QAST::Op 节点看起来像这样:

    QAST::Op.new(
        :op('add_n'),
        $left_child_ast,
        $right_child_ast
    )

该运算符由 `:op(...)` 命名参数指定, 操作数是节点的孩子。

## 解析一些数学运算符 (1)

让我们添加加法, 减法, 乘法和除法运算符。 对于这些, 我们需要设置运算符优先级解析器, 配置两个优先级别。

    INIT {
        # 从 Perl 6 Grammar 窃取优先级别的名称
        Rubyish::Grammar.O(':prec<u=>, :assoc<left>', '%multiplicative');
        Rubyish::Grammar.O(':prec<t=>, :assoc<left>', '%additive');
    }

注意, 我们在这里调用的 `O` 方法继承自 `HLL::Grammar`。 第一个参数指定优先级别和结合性。 然后第二个按照名称保存这个特定的配置, 所以我们可以在声明操作符时引用它。

## 解析一些数学运算符 (2)

就地使用优先级别, 我们可以向 grammar 中添加一些运算符。 这是通过将它们添加到 `infix` protoregex 中来完成的, 它是我们继承自 `HLL::Grammar` 的。

    token infix:sym<*> { <sym> <O('%multiplicative, :op<mul_n>')> }
    token infix:sym</> { <sym> <O('%multiplicative, :op<div_n>')> }
    token infix:sym<+> { <sym> <O('%additive, :op<add_n>')> }
    token infix:sym<-> { <sym> <O('%additive, :op<sub_n>')> }

**`:op<...>`** 语法指示我们从 `HLL::Actions` 继承的 `EXPR` action 方法来为我们构造该运算符的 `QAST::Op` 节点!

## Terms

我们几乎准备好使用运算符优先级解析器了, 但还不完全是。 我们还必须指导它如何**获得一个 term**。 我们从 `HLL::Grammar` 继承了一个 `term` protoregex, 因此只需要为它添加候选者。

对我们来说, 这意味着一个term的候选者是一个值:

    token term:sym<value> { <value> }

和匹配的 action 方法:

    method term:sym<value>($/) { make $<value>.ast; }

## 把所有的东西连接到一块

最后我们需要做的是更新 `puts` 的 grammar 规则:

    token statement:sym<puts> {
        <sym> <.ws> <EXPR>
    }

还有 action 方法:

    method statement:sym<puts>($/) {
        make QAST::Op.new(
            :op('say'),
            $<EXPR>.ast
        );
    }

## 试验我们的运算符

基本的算术运算现在工作了,并且运算符的优先级也被正确地处理了。

    > puts 10 * 9 + 1
    91

我们还可以检查 AST 以查看 `QAST::Op` 节点:

    - QAST::Block
      - QAST::Stmts puts 10 * 9 + 1\n
        - QAST::Op(say)
          - QAST::Op(add_n &infix:<+>) +
            - QAST::Op(mul_n &infix:<*>) *
              - QAST::IVal(10)
              - QAST::IVal(9)
            - QAST::IVal(1)

## Sequencing: QAST::Stmts 和 QAST::Stmt

有两种按顺序运行每个子节点的节点类型。

**`QAST::Stmts`**, 确实, 没有什么比这更多的了。

**`QAST::Stmt`** 具有附加的效果, 它规定在代码生成期间创建的任何临时值在该节点执行结束之后都不再需要了。

一般来说, 在语言使用者认为的那样, 具有一组语句和单个语句的地方使用它们是有意义的。

## Block 结构

一个常见的用法, 虽然没有强制性, 是一个 `QAST::Block` 拥有两个 `QAST::Stmts` 节点。

第一个用于保存**声明**, 例如变量或嵌套例程。

第二个用于保存**由该块的 statementlist 解析的语句**。

这个惯用法在 NQP 和 Rakudo 中都可用; 例如:

    $block[0].push(QAST::Var.new(:name<$/>, :scope<lexical>, :decl<var>));

## 变量

现在是时候添加变量到 Rubyish 中了！ 在 Rubyish 中, 变量不是显式地声明的。 相反, 它们在首次赋值时在当前作用域中被声明。

首先, 让我们为赋值添加一个优先级:

    Rubyish::Grammar.O(':prec<j=>, :assoc<right>',  '%assignment');

并解析赋值运算符, 使用 `bind` NQP 运算符, 它将右侧的表达式绑定给左侧的变量:

    token infix:sym<=> { <sym> <O('%assignment, :op<bind>')> }

## 表达式作为语句

你可能还记得在 NQP grammar 中, 表达式也是一个有效的语句。 我们需要在 Rubyish 也这样做。

这意味着将表达式添加到 grammar:

    token statement:sym<EXPR> { <EXPR> }

还有相应的 actions:

    method statement:sym<EXPR>($/) { make $<EXPR>.ast; }

## 标识符解析

现在, 我们将所有标识符视为变量一样。 我们这样解析标识符:

    token term:sym<ident> {
        :my $*MAYBE_DECL := 0;
        <ident>
        [ <?before \h* '=' [\w | \h+] { $*MAYBE_DECL := 1 }> || <?> ]
    }

注意这里使用了向前查看以查看我们是否可以找到一个赋值运算符, 在其周围有空格或标识符紧跟其后(不能将`==`视为赋值！)

动态变量用于传达是否发生了赋值, 这可能意味着我们有一个声明。

## 标识符 actions

Here is a first, cheating attempt at the actions for an identifier.
这是第一个, 欺骗尝试的标识符的 action。

    method term:sym<ident>($/) {
        if $*MAYBE_DECL {
            make QAST::Var.new( :name(~$<ident>), :scope('lexical'),
                                :decl('var') );
        }
        else {
            make QAST::Var.new( :name(~$<ident>), :scope('lexical') );
        }
    }

这允许我们运行:

    a = 7
    b = 6
    puts a * b

## 问题

不幸的是, 事情变化得相当快。 每个赋值现在都被视为声明。 因此:

    a = 1
    puts a
    a = 2
    puts a

失败了:

    Error while compiling block: Error while compiling op bind:
    Lexical 'a' already declared

## 符号表

每个 `QAST::Block` 都有一个符号表, 可以用来存储其中声明的符号的额外信息。

真的, 它只是一个散列哈希, 第一个散列键在符号和内部散列存储任何我们希望的信息。

我们可以通过执行以下操作来添加或更新符号的条目(entries):

    $block.symbol($ident, :declared(1));

我们可以通过做以下事情来获取符号上保存的当前信息:

    my %sym := $block.symbol($ident);

我们可以用这个来跟踪声明！

## 下一个挑战:跟踪块

我们需要访问我们正在声明的当前块, 然后才能使用符号。 将它放在一个动态变量中是最容易处理的, 在 `TOP` grammar 规则中创建它:

    token TOP {
        :my $*CUR_BLOCK := QAST::Block.new(QAST::Stmts.new());
        <statementlist>
        [ $ || <.panic('Syntax error')> ]
    }

相应的 `TOP` action 方法变为:

    method TOP($/) {
        $*CUR_BLOCK.push($<statementlist>.ast);
        make $*CUR_BLOCK;
    }

## 使用符号

现在, 我们可以使用 `symbol` 跟踪已经声明过的内容, 而不是重新声明它。

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

## 其它作用域

`QAST::Var` 节点不只是用于词法作用域。 其可用作用域如下:

    lexical         对嵌套块可见
    local           像 lexical, 但对嵌套块不可见
    contextual      动态作用域词法的查找
    attribute       对象属性 (children: invocant, package)
    positional      数组索引 (children: array, index)
    associative     散列索引 (children: hash, key)

注意只有前 3 个作为声明是有意义的。 还要注意, Rakudo 不使用最后 2 个(它的数组和散列处理因素不同), 虽然 NQP 使用。

## 例程

为了更多的说明词法作用域, 让我们添加例程。 声明和调用例程的语法如下:

    def greet
        puts "hello"
    end
    greet()

我们将通过不处理其他形式的调用来保持简单。

## 解析例程声明

这里没有什么特别新的东西。 我们注意启动一个新的词法作用域, 所以任何声明都不会污染周围的作用域。 split 因此是第一个 token 的 action 方法可以看到 `$* CUR_BLOCK` 安装进去。

    token statement:sym<def> {
        'def' \h+ <defbody>
    }
    rule defbody {
        :my $*CUR_BLOCK := QAST::Block.new(QAST::Stmts.new());
        <ident> \n
        <statementlist>
        'end'
    }

NQP 和 Rakudo 做的几乎相同, 唯一的区别是, 他们抽象了块的推入/弹出并保持追踪。

## 解析调用

调用是一个标识符, 后跟一些圆括号。 我们也会小心以避免使用关键字。

    token term:sym<call> {
        <!keyword>
        <ident> '(' ')'
    }

`<!keyword>` 也适用于 `term:sym<ident>`.

## 例程声明的 Actions

`defbody` 完成了 `QAST::Block`, 它通过 `statement:sym<def>` 被安装为词法。

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

## 调用

调用是一个操作, 因此使用 `QAST::Op` 来完成。 默认情况下, 要调用的东西的名称(将在词法中解析)在 `name` 命名参数中指定。

    method term:sym<call>($/) {
        make QAST::Op.new( :op('call'), :name(~$<ident>) );
    }

任何没有指定 `name` 的情况都会把节点的第一个子节点作为要调用的东西。 因此, 我们本可以这样写:

    method term:sym<call>($/) {
        make QAST::Op.new(
            :op('call'),
            QAST::Var.new( :name(~$<ident>), :scope('lexical')
        );
    }

但是(至少在JVM上)这阻碍了优化, 所以不要这样做。

## 形参和实参

实参和形参处理涉及到我们以前没有见过的节点类型。 形参只是 `QAST::Op` 调用节点的子节点, 参数只是 `decl` 设置为 `param` 的 `QAST::Var` 节点。

首先, 让我们为参数添加解析。

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

`param` action  方法看起来像这样:

    method param($/) {
        $*CUR_BLOCK[0].push(QAST::Var.new(
            :name(~$<ident>), :scope('lexical'), :decl('param')
        ));
        $*CUR_BLOCK.symbol(~$<ident>, :declared(1));
    }

有趣的是, 它从来不做 `make`。 这可能看起来奇怪, 因为其他地方的 action 方法已经这样做。 但它没有理由那样做; 我们真正想做的是将已声明的参数安装到当前块中。 它更容易在上下文中获取。

## 参数传递

这里有一个快速和简单的方法来解析参数:

    token term:sym<call> {
        <!keyword>
        <ident> '(' :s <EXPR>* % [ ',' ] ')'
    }

之后我们更新相应的 actions:

    method term:sym<call>($/) {
        my $call := QAST::Op.new( :op('call'), :name(~$<ident>) );
        for $<EXPR> {
            $call.push($_.ast);
        }
        make $call;
    }

## 目前为止...

到目前为止, 我们使用了以下的 QAST 节点类型:

    QAST::Block     一个词法作用域
    QAST::Stmts     要执行的一系列东西
    QAST::Stmt      同上, 还是临时边界
    QAST::Op        某种操作
    QAST::Var       变量或参数使用/声明
    QAST::IVal      整数字面值
    QAST::NVal      浮点数字面值
    QAST::SVal      字符串字面值

我们现在将考虑更多的节点类型; 我们将推迟一些(`QAST::WVal` 和 `QAST::Regex`), 直到明天。

## 用 QAST::BVal 引用 Block

QAST::Block 应该只在一个 QAST 树中出现一次。 它放在哪里就在哪里定义它的词汇作用域。

那么, 如果你想**在树中的其他地方引用 `QAST::Block`** 呢?
这就是 `QAST::BVal`, Block Value的缩写, 的作用所在。 例如, 它在发出代码时使用, 使CORE设置成为程序的外部词法作用域。

    my $set_outer := QAST::Op.new(
        :op('forceouterctx'),
        QAST::BVal.new( :value($*UNIT) ),
        QAST::Op.new(
            :op('callmethod'), :name('load_setting'),
            # stuff left out here
        ));

## Boxed vs. unboxed, void vs. non-void 上下文

当后端代码生成发生时, 它可能需要装载和/或解包东西, 或者它可以确定某个东西是否会在空(sink)上下文中。

虽然它可以可靠地生成工作代码, 但它可能不高效。 在这里考虑整数常量的处理:

    my int $x = 42;     # Needs an unboxed native int
    my $x = 42;         # Needs a boxed Int object

当我们写整数字面值的 action 方法时, 我们有一个困境。 我们应该发出一个 `QAST::IVal` 吗, 这将在第二种情况下被装箱。 或者我们应该将一个Int常量42放入常量池, 并使用 `QAST::WVal`(明天更多的讨论这个节点类型)来引用它吗?

## QAST::Want 来救场

与其选择, 我们倒不如把这两个选项都呈现出来, 让代码生成器选择最有效率的那个。 这是通过 `QAST::Want` 节点来完成的。

    QAST::Want.new(
        QAST::WVal.new( :value($boxed_constant) ),
        'Ii', QAST::IVal.new( :value($the_value) )
    )

第一个孩子是默认的东西。 它后面是一组选择器, 用于我们可能所处的不同上下文中。

    Ii      原生整数
    Nn      原生浮点数
    Ss      原生字符串
    v       void

## 后端逃生舱口: QAST::VM (1)

有时, 有必要**通过后端有条件地做事情**, 或做一些**特定于虚拟机的**操作。 QAST::VM 节点处理这一需求。

例如, 下面是来自 NQP 的一些代码, 用于加载 NQP 模块加载器。 它需要知道后端要查找的文件名。

    QAST::Op.new(
        :op('loadbytecode'),
        QAST::VM.new(
            :parrot(QAST::SVal.new( :value('ModuleLoader.pbc') )),
            :jvm(QAST::SVal.new( :value('ModuleLoader.class') ))
        ))

如果当前后端没有适用的选项, 代码生成器将抛出异常。

## 后端逃生舱口: QAST::VM (2)

`QAST::VM` 节点类型也在 `pir::op_SIG(...)` 语法的后面, 它在 NQP 和 Rakudo 中可用。 这里是 `pir::op` 在 NQP 中是如何解析和实现的。

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

Rakudo 和 NQP 生成的 QAST 树在顶部有一个 `QAST::CompUnit`。

![30%](eps/qast-compunit-and-blocks.eps)

## 我们可以用 QAST::CompUnit 做什么

这里看看 `QAST::CompUnit` 能做些什么(我们明天会再看一下)。

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

## 练习 5

在本练习中, 您将向 PHPish 添加一些功能, 以便探索我们一直在研究的 QAST 节点。

看看 NQP grammar 和 actions, 了解他们的工作原理 `:-)`

# 探索 nqp::ops

*学习所有的运算符!*

## 仅仅瞄一眼

有几百个可用的 `nqp::op`s。 它们的范围从算术到字符串操作, 从流控制(如循环)到类型创建。

我们已经看到了一些运算符。 明天, 我们将看到一堆更多的运算符, 因为我们看看 6model 和序列化上下文, 它们有一堆与它们相关的 `nqp::ops`。

在本节中, 我们将概述“其余的”。 概述不是详尽无遗, 因为那会令人精疲力尽。

记住它们可以以 `nqp::op` 形式*或*在 `QAST::Op` 节点中使用, 因此这些知识可以重复使用！

## 算术

这些是以原生整数形式:

    add_i   sub_i   mul_i   div_i   mod_i
    neg_i   abs_i

 还有原生浮点数形式:

    add_n   sub_n   mul_n   div_n   mod_n
    neg_n   abs_n

为了帮助实现有理数, 我们还有:

    lcm_i   gcd_i

## 数字

基础的东西:

    pow_n       ceil_n      floor_n
    ln_n        sqrt_n      log_n
    exp_n       isnanorinf  inf
    neginf      nan
    
三角函数:

    sin_n   asin_n  cos_n   acos_n  tan_n
    atan_n  atan2_n sinh_n  cosh_n  tanh_n
    sec_n   asec_n  sech_n

## 关系

为了比较原生整数, 原生浮点数和原生字符串(如有必要代码生成器将会拆箱)。 例如, 原生整数形式是:

    cmp_i       compare; returns -1, 0, or 1
    iseq_i      non-zero if equal
    isne_i      non-zero if non-equal
    islt_i      non-zero if less than
    isle_i      non-zero if less than or equal to
    isgt_i      non-zero if greater than
    isge_i      non-zero if greater than or equal to

`_n` 和 `_s` 形式也都存在。

## 数组操作

有各种运算符操作数组:

    atpos       atpos_i     atpos_n     atpos_s
    bindpos     bindpos_i   bindpos_n   bindpos_s
    push        push_i      push_n      push_s
    pop         pop_i       pop_n       pop_s
    shift       shift_i     shift_n     shift_s
    unshift     unshift_i   unshift_n   unshift_s
    splice      existspos   elems       setelems

请注意, 原生类型的版本是**非强制的**, 但只适用于原生类型的数组。

## 散列操作

看起来跟数组操作并无二至。

    atkey       atkey_i     atkey_n     atkey_s
    bindkey     bindkey_i   bindkey_n   bindkey_s
    existskey   deletekey   elems

这些都假定键是字符串; 任何非字符串键将首先被强制转换为字符串。

## 旁白: Perl 6 数组/散列 ops 的用法

在 Perl 6 中, 像下面这样的东西:

    @a[0] = 42;

实际上使用 `atpos` 来使标量容器绑定到底层的数组存储, 然后分配给该容器。 `bindpos` 只用于做:

    @a[0] := 42;

此外, 你永远不会直接在 Perl 6 的 `Array` 或 `Hash` 对象上这样做。 这些对象*包含*一个较低级别(lower-level)的数组或散列作为属性, 而方法在那上面使用这个 ops。

## 创建列表和散列

`nqp::list` op 创建一个(低级)数组, 其元素传递给它。 因此, 它是一个可变参数 op。

    nqp::list($foo, $bar, $baz)

原生类型的列表可以使用 `list_i`, `list_n` 和 `list_s` 创建。

有一个类似的 `nqp::hash`, 它期望一个键, 一个值 ...

    nqp::hash('name', $name, 'age', $age)

最后, `islist` 和 `ishash` 告诉你某个东西是否是低级的数组或散列。

## 字符串

字符串操作大多数命名为 Perl 6 那样。

    chars       uc          lc          x
    concat      chr         join        split
    flip        replace     substr      ord
    index       rindex      codepointfromname

还有用于检查字符类成员资格的操作。 这些主要是在编译正则表达式或正则表达式相关的类时发出的, 但可以在其他地方使用。 他们是:

    nqp::iscclass(class, str, index)
    nqp::findcclass(class, str, index, limit)
    nqp::findnotcclass(class, str, index, limit)

其中 `class` 是 `nqp::const::CCLASS_*` 其中之一。

## 条件

`if` 和 `unless` ops 期望两个或三个孩子: 一个条件, 一个 "then" 和一个可选的 "else"。 注意, NQP 和 Perl 6 中的 `elsif` 是通过嵌套 `if` `QAST::Op` 节点来编译的。

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

## 条件和一元块

NQP 和 Perl 6 都支持像下面这样的东西:

    if %core_ops{$name} -> $mapper {
        return $mapper($qastcomp, $op);
    }


这将计算 `%core_ops{$name}`, 然后将它传递给 `$mapper`, 如果它是一个真值的话。

在 QAST 级别, 这由 `if` op 的第二个孩子代表 `QAST::Block`, 它的元数 `arity` 被设置为一个非零值。

## 循环

有四个相关的循环结构:

                                Loop while true    Loop while false
                                ---------------    ---------------
    Condition, then body      | while              until
    Body, then condition      | repeat_while       repeat_until               

它们接收两个或三个孩子:

* 条件
* 循环体
* 可选的, 在循环体之后要做的事情

如果抛出 `redo` 控制异常, 则重新计算第二个子节点。 第三个只有在任何 `redo` 发生后才被计算。 它由 Perl 6(C风格)的 `loop` 结构使用。

## Loop example

Perl 6 的 `let` 和 `temp` 关键字保存容器及其原始值的列表(容器, 值等)。这是在块退出时遍历此列表进行恢复的循环。

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

## 其它控制结构

还有三个值得了解的控制结构:

* **`for`** 需要两个孩子, 一个可迭代(通常是一个低级数组或列表)和一个块。 它为迭代中的每个东西调用该块。 仅用于 NQP; Rakudo 中处理迭代是完全不同的。
* **`ifnull`** 需要两个孩子。 它计算第一个。 如果它不为 null, 它只是产生这个值。 如果*是* null, 它将计算第二个孩子。
* **`defor`** 与 `ifnull` 相同, 但是考虑的是定义而不是 nullness

## 抛出异常

有各种创建和抛出异常的操作:

    newexception    创建一个新的, 空的异常对象
    setextype       设置异常类别 (nqp::const::CONTROL_*)
    setmessage      设置异常信息 (string)
    setpayload      设置异常有效负载(object)
    throw           抛出异常对象
    die             执行/抛出带有字符串信息的异常

有一个更简单的方法来抛出一些常见的控制异常:

    QAST::Op.new( :op('control'), :name('next') )

这里其它有效的名字有 `redo` 和 `last`。

## 处理异常

`handle` op 用于表示异常处理。 第一个孩子是用处理程序保护的代码。 然后处理程序被指定为一个字符串, 指定要处理的异常的类型, 后面跟着要运行的 QAST 来处理它。

NQP 和 Rakudo 保留每个块 `%*HANDLERS`, 并且当块被完全解析时在块外面构建它的 `handle` op。

    my $ast := $<statementlist>.ast;
    if %*HANDLERS {
        $ast := QAST::Op.new( :op('handle'), $ast );
        for %*HANDLERS {
            $past.push($_.key);
            $past.push($_.value);
        }
    }

## 使用异常对象

在处理程序内, 可以使用以下操作。 除了第一个之外, 他们都接受一个异常对象。

    exception       获取当前的异常对象
    getextype       获取异常类别 (nqp::const::CONTROL_*)
    getmessage      获取异常信息 (string)
    getpayload      获取异常有效负载 (object)
    rethrow         重新抛出异常
    resume          如果可能的话, 恢复异常

最后, 还有两个与回溯相关的操作; `backtrace` 返回一个散列数组, 每个散列描述一个回溯条目, `backtracestrings` 只返回一个描述条目的字符串数组。

## 上下文自省

可以使用各种操作来对词法作用域中的符号进行内省, 或者遍历动态(调用者)或静态(词法的)作用域链。 它们通常用于实现诸如 Perl 6 中的 `CALLER` 和 `OUTER` 伪包功能。

    ctx             获取一个代表当前上下文的对象
    ctxouter        接收一个上下文并返回它的外部上下文或 null
    ctxcaller       接收一个上下文并返回它的调用者上下文或 null
    ctxlexpad       接收一个上下文并返回它的 lexpad
    curlexpad       获取当前的 lexpad
    lexprimspec     给定一个 lexpad 和 名字, 获取名字的原始类型

lexpad 本身可以与适当的散列操作(`atkey`, `bindkey`)一起使用, 以操作其中包含的符号。

## 大整数

Perl 6 需要大的整数支持其 `Int` 类型。 因此, 它在 NQP 操作中提供。 大整数操作仅对具有 `P6bigint` 表示(明天更多地表示)的对象有效, 或者对其进行封装。

具有大整数结果的那些操作与它们的原生亲属不同, 通过采用额外的操作数(这是结果的类型对象)。
下面的来自于 Perl 6 设置的 `mult`s 说明了这一点。

    multi infix:<+>(Int:D \a, Int:D \b) returns Int:D {
        nqp::add_I(nqp::decont(a), nqp::decont(b), Int);
    }
    multi infix:<+>(int $a, int $b) returns int {
        nqp::add_i($a, $b)
    }

`_I` 后缀用于大整数 ops.

## 练习 6

如果时间允许, 您可以通过添加对 PHPish 的支持来探索一些 nqp::ops(按顺序, 或选择你觉得最有趣的):

* 基本的数字关系运算 (`<`, `>`, `==`, etc.)
* `if`/`else if`/`else`
* `while` 循环

参见练习表得到一些提示。

## 今天就到这儿了

今天, 我们已经涉及了很多领域, 从 NQP 语言开始, 然后建立如何使用它来实现一个简单的编译器。

这是一个好的开始, 但我们仍然缺少几个 NQP 和 Rakudo 严重依赖的非常重要的部分。这包括对象和序列化上下文的概念。 我们明天再来看看。

还有什么问题吗?

(修理)
