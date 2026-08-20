---
title: 集合论类型与渐进式类型检查：Elixir 类型系统设计原则详解
date: 2026-08-20 11:50:00
tags: [Technique]
mathjax: true
---

> 本文解读 Giuseppe Castagna、Guillaume Duboc 与 José Valim 的论文《The Design Principles of the Elixir Type System》。论文提出了一套面向 Elixir 的渐进式类型系统，重点处理集合论类型、语义子类型、模式匹配、guard、map、动态代码与 BEAM 运行时检查之间的关系。

论文的核心判断是：Elixir 需要静态类型检查，但类型系统不能脱离 Elixir 的语言习惯重新设计一套陌生语言。它必须理解多子句函数、函数元数、模式匹配、guard、map、协议、动态代码和 BEAM 的运行时行为。作者因此没有选择一套简单的“类型标签”，而是把集合论类型、局部类型推断、类型收窄和渐进式类型检查组合在一起，构成一套逐步集成到 Elixir 编译器中的设计方案。[1]

## 一、论文讨论的不是 Typespec 的小修补

Elixir 已经有 Typespec，也可以使用 Dialyzer 分析代码。问题在于，Typespec 的声明并不会由 Elixir 编译器完整验证，Dialyzer 则采用 success typing：它只在能够证明某处存在问题时报告警告，从而尽量避免误报。这种策略适合分析规模庞大的既有代码，却会放过一部分潜在错误。

论文提出的系统追求更强的类型安全保证。它希望在编译期报告更多类型错误，同时允许项目继续保留未标注的动态代码。类型系统因此必须同时完成两件事：一方面精确表达函数和数据结构的约束，另一方面控制迁移成本，不要求开发者一次性改写整个代码库。[1]

这项工作属于语言设计和类型理论研究，不是一篇以 benchmark 为中心的实验论文。论文给出了 Core Elixir 的形式化类型规则，介绍了原型实现，并列出未来的编译器集成路线；作者也明确承认，大规模代码库上的性能、警告质量和社区接受度仍需要后续实现验证。[1]

## 二、为什么简单的并集类型不够用

先看一个函数：

```elixir
$ (integer() or boolean()) -> (integer() or boolean())
def negate(x) when is_integer(x), do: -x
def negate(x) when is_boolean(x), do: not x
```

这个声明只说明输入可以是整数或布尔值，输出也可以是整数或布尔值。它没有表达输入和输出之间的对应关系。因此，当下面的代码调用 `negate/1` 时，类型检查器无法确认结果一定是整数：

```elixir
integer_value + negate(integer_value)
```

论文改用函数箭头类型的交集：

```elixir
$ (integer() -> integer()) and
  (boolean() -> boolean())
```

这个类型表达了两个子行为：整数输入得到整数输出，布尔值输入得到布尔值输出。它比前一个并集类型更精确。

集合论类型把类型理解为值的集合：

- `integer()` 表示所有整数构成的集合；
- `boolean()` 表示 `{true, false}`；
- `t1 or t2` 表示两个集合的并集；
- `t1 and t2` 表示两个集合的交集；
- `not t` 表示集合补集；
- `none()` 表示空集合；
- `term()` 表示所有 Elixir 值。

这种解释使类型运算服从集合论的交换律、结合律和分配律。比如：

```elixir
{integer() or string(), boolean()}
```

与：

```elixir
{integer(), boolean()} or {string(), boolean()}
```

表示同一个值集合。语义子类型通过集合包含关系定义，因此类型检查器可以识别这种等价性，而不需要依赖类型表达式的表面语法。[1]

## 三、函数元数必须进入类型语义

在 Elixir 中，函数元数不是附属信息。`foo/1` 和 `foo/2` 是两个不同的函数，`is_function(value, 2)` 也可以在运行时检查函数是否接受两个参数。

传统的一元函数类型系统往往把多参数函数编码为接受 tuple 的函数。例如，二元函数可能被表示成接受 `{x, y}` 的一元函数。这种编码无法准确表达 Elixir 的 arity，也无法正确处理 `is_function/2`。

论文因此直接把元数写入函数类型：

```text
(t1, ..., tn) -> t
```

形式化定义中的函数空间使用 $n$ 元输入，而不是单个输入集合。类型子型关系首先要求元数相同，然后比较各个参数域和返回域。不同元数的函数类型交集为空集，这与 Elixir 的运行时语义一致。[1]

这一点看似基础，实际影响很大。guard 分析、函数应用检查和多子句函数的重载行为都依赖于准确的 arity 信息。

## 四、参数多态与局部类型推断

集合论类型并不排斥参数多态。论文用 `map/2` 和 `reduce/3` 说明这一点：

```elixir
$ ([a], (a -> b)) -> [b]
  when a: term(), b: term()
def map([h | t], fun), do: [fun.(h) | map(t, fun)]
def map([], _fun), do: []
```

它的含义是：对于任意类型 `a` 和 `b`，`map/2` 接受 `a` 类型元素的列表，以及一个从 `a` 映射到 `b` 的函数，返回 `b` 类型元素的列表。

调用时不需要显式实例化类型变量：

```elixir
map([1, 4], fn x -> negate(x) end)
```

局部类型推断可以把 `a` 和 `b` 都实例化为 `integer()`，因此结果类型为 `[integer()]`。

论文还用递归类型描述嵌套列表：

```elixir
tree(a) = (a and not list()) or [tree(a)]
```

这个定义允许叶节点是 `a` 类型且不是 list 的值，也允许节点是由其他树组成的 list。于是 `flatten/1` 可以获得如下类型：

```elixir
tree(a) -> [a]
```

这种类型表达能力对处理 Elixir 的通用集合函数很重要。单纯把所有值都近似成 `term()`，会迅速丢失输入元素与输出元素之间的关系。

## 五、guard 是类型信息，而不只是运行时条件

Elixir 程序大量依赖 guard。论文的一个核心工作，是把 guard 分析纳入类型系统，而不是只把 guard 当成无法理解的运行时黑箱。

例如：

```elixir
def get_age(person) when is_integer(person.age), do: person.age
```

类型检查器可以从 `person.age` 和 `is_integer/1` 推断：

```text
%{age: integer(), ...} -> integer()
```

这里的 `...` 表示开放 map：map 至少包含 `age` 字段，也可以包含其他字段。

这个推断同时完成了三件事：

1. `person` 必须是 map；
2. `person` 必须定义 `age` 字段；
3. `age` 的值必须是整数。

因此，类型系统可以把 guard 视为对变量类型的约束，并在后续表达式中使用这些约束。

### 5.1 类型收窄

考虑：

```elixir
def negate_alt(x) do
  if is_integer(x), do: -x, else: not x
end
```

如果 `x` 初始类型是 `integer() or boolean()`，类型检查器会在 `do` 分支把它收窄成 `integer()`，在 `else` 分支把它收窄成 `boolean()`。

论文把这种技术称为 narrowing。它也适用于被测试表达式内部的变量，例如 map 字段和 tuple 元素。

### 5.2 复杂 guard 的环境分裂

下面的 guard 有两个逻辑分支：

```elixir
def baz(x, y) when is_boolean(x) or is_integer(y), do: {y, x}
def baz(_, _), do: nil
```

第一种情况是 `x` 为 boolean，`y` 可以是任意值；第二种情况是 `y` 为 integer，`x` 可以是任意值。因此第一条子句的返回类型是：

```text
{term(), boolean()} or {integer(), term()}
```

系统不能简单地把 `x` 和 `y` 各自标成一个 union 类型，而需要为 `or` 的两个分支建立不同的类型环境，再合并分支结果。论文的 guard 分析规则从左到右处理 guard，并考虑 Elixir guard 的求值顺序和可能失败的表达式。[1]

## 六、当 guard 无法被类型精确表达时，系统使用上下近似

类型系统无法表达所有运行时谓词。例如：

```elixir
def foo(x) when map_size(x) == 2, do: Map.to_list(x)
```

“所有恰好包含两个字段的 map”是一个运行时集合，但不一定能够用当前类型语法精确表达。

论文因此区分两种近似。

### Potentially accepted type

它是一个上近似，包含所有可能通过 pattern/guard 的值，也可能包含一些实际上不会通过 guard 的值。

在上面的例子中，可以使用 `map()` 作为 potentially accepted type。它比“大小为 2 的 map”更宽，但包含后者的全部值。

### Surely accepted type

它是一个下近似，只包含确定会通过 pattern/guard 的值。

例如：

```elixir
def bar(x) when
  (is_map(x) and map_size(x) == 2) or is_list(x), do: to_string(x)

def bar(x) when length(x) == 2, do: x
```

第一条子句可能接受两字段 map，也接受所有 list。系统无法精确表达两字段 map，但可以确定所有 list 都已经被第一条子句捕获。因此第一条子句的：

```text
potentially accepted type = map() or list()
surely accepted type = list()
```

这足以判定第二条子句是冗余的，因为 `length/1` 只对 list 有意义，而所有 list 已经被前一条子句处理。[1]

## 七、穷尽性检查和冗余分支检查

模式匹配的价值不只在于解构数据，也在于它能够给出完整的控制流信息。

论文定义：

```elixir
result() =
  %{output: :ok, socket: socket()} or
  %{output: :error, message: :timeout or {:delay, integer()}}
```

函数却只处理：

```elixir
def handle(r) when r.output == :ok, do: "Msg received"
def handle(r) when r.message == :timeout, do: "Timeout"
```

仍有一种输入没有实现：

```elixir
%{output: :error, message: {:delay, integer()}}
```

类型检查器可以报告函数定义不完整，并直接指出缺失的输入类型。相比只说“可能存在未匹配分支”，这种警告更适合重构大型代码库。

反过来，如果函数输入类型只允许 map，却增加了一个匹配 tuple 的子句，类型检查器可以报告该分支永远不会执行。

这两种能力分别对应：

- exhaustivity checking：检查是否覆盖所有可能输入；
- redundancy checking：检查是否存在永远无法匹配的分支。

## 八、统一理解 record 与 dictionary

Elixir 的 map 有两个常见用途：

```elixir
person.age
```

把 map 当成 record；

```elixir
person[key]
```

把 map 当成 dictionary。

两种访问方式的运行时语义不同：

- `person.age` 要求 `age` 一定存在，否则抛出异常；
- `person[:age]` 允许 key 缺失，此时返回 `nil`。

论文用 required 和 optional 字段标记这种差异：

```elixir
%{required(:age) => integer(), optional(term()) => term()}
```

也可以使用更直观的开放 map 写法：

```elixir
%{age: integer(), ...}
```

如果字段可能缺失：

```elixir
%{optional(:age) => integer()}
```

此时：

```elixir
person.age
```

会触发类型警告，因为 `age` 不一定存在；而：

```elixir
person[:age]
```

的结果类型是：

```text
integer() or nil
```

如果 map 中的字段被声明为必需字段：

```elixir
%{foo: integer(), bar: integer()}
```

那么：

```elixir
m[:foo] + m[:bar]
```

可以被推断为整数运算，因为两个字段都不可能返回 `nil`。

### 8.1 混合记录字段与字典键域

论文允许一个 map 类型同时包含固定字段和动态键域：

```elixir
type t() = %{foo: atom(),
              optional(:bar) => atom(),
              optional(atom()) => integer()}
```

它表示：

- `foo` 必须存在，并且是 atom；
- `bar` 可以缺失，存在时是 atom；
- 其他 atom key 对应 integer。

固定 singleton key 的声明优先于更宽泛的 key domain。这一点使类型系统能够表达结构化数据与动态字典混合的实际用法。

## 九、渐进式类型检查：dynamic() 的作用

Elixir 已经存在大量动态代码。若要迁移这些代码，类型系统必须允许静态部分和动态部分共存。

论文引入：

```elixir
dynamic()
```

它不是普通的顶层类型。`term()` 表示所有值，而 `dynamic()` 表示类型检查器暂时不知道某个表达式的具体类型，并允许它在运行时以不同类型出现。

例如：

```elixir
$ dynamic() -> _
def foo1(x), do: ...
```

函数可以接受任意类型的参数，函数体中每次使用 `x` 时也可以处于不同的动态类型环境。

但 `dynamic()` 不会让类型检查完全失去意义：

```elixir
$ (dynamic() -> dynamic()) -> _
def foo2(fun), do: ...
```

这里仍然要求 `fun` 具有函数类型。下面的调用会被拒绝：

```elixir
foo2({7, 42})
```

因为 tuple 不是函数，即使函数参数的其他细节是动态的。

## 十、普通函数箭头与 strong arrow

这是论文最关键、也最容易被忽略的部分。

考虑两个身份函数：

```elixir
$ integer() -> integer()
def id_weak(x), do: x

$ integer() -> integer()
def id_strong(x) when is_integer(x), do: x
```

从静态输入输出关系看，它们都像是：

```text
integer() -> integer()
```

运行时行为却不同：

- `id_weak/1` 没有检查输入，传入字符串时可能原样返回字符串；
- `id_strong/1` 通过 guard 检查输入，非整数会匹配失败。

因此，论文把第二种函数视为 strong function，把第一种函数视为 weak function。

strong arrow 的语义是：函数即使收到不属于声明 domain 的动态输入，也必须满足以下条件之一：

1. 返回符合 codomain 的结果；
2. 通过 BEAM 或程序员写出的运行时检查失败；
3. 发散而不返回。

于是：

```elixir
$ dynamic() -> {dynamic(), integer()}
def foo3(x), do: {id_weak(x), id_strong(x)}
```

第一项仍然是 `dynamic()`，因为 `id_weak/1` 没有运行时检查；第二项可以被确定为 `integer()`，因为 `id_strong/1` 要么返回整数，要么在输入错误时失败。

这套设计解决了一个工程问题：传统 sound gradual typing 通常需要编译器在动态代码与静态代码交界处插入 cast。本文方案要求类型系统不改变 Elixir 的编译结果，因此它改为分析现有的 guard、模式匹配和 BEAM 检查，利用已经存在的运行时行为完成安全性推断。[1]

## 十一、形式化核心：Core Elixir 与双向类型检查

论文没有直接形式化整个 Elixir，而是定义了一个 Core Elixir。其表达式包括：

- 常量和变量；
- lambda 与函数应用；
- tuple 与 tuple projection；
- 带类型注解的 let；
- 加法；
- 带 pattern 和 guard 的 case。

类型语法包括：

```text
b ::= int | atom | 1fun | 1tup

t ::= b | c | α | t -> t | {t} | t or t | not t
```

交集可以由否定和并集编码：

$$
t_1 \mathbin{and} t_2 = not\,(not\,t_1 \mathbin{or} not\,t_2)
$$

顶层类型和空类型分别编码为：

$$
term() = int \mathbin{or} atom \mathbin{or} 1fun \mathbin{or} 1tup
$$

$$
none() = not\,term()
$$

类型检查采用双向思想：一部分表达式合成类型，另一部分表达式在给定期望类型的条件下进行检查。case 表达式的类型规则会计算每个 pattern/guard 能够处理的输入区域，并用这些区域生成分支环境。

论文把 guard 的类型分析拆成多个子环境，以处理 `or` 分支以及前置 guard 失败后的求值路径。类型系统还区分能够保证穷尽性的规则和只能给出 warning 的近似规则：如果只能使用 potentially accepted type 判断覆盖范围，系统会提示 case 可能在运行时没有匹配分支。[1]

## 十二、编译器集成原则

论文提出了几项明确的集成要求。

### 12.1 不改变 Elixir 表达式语法

类型系统不能要求每个函数参数都增加新语法。Elixir 已经把大量语言结构写在自身宏系统中，因此类型语法也必须遵守现有语言的表达习惯。

论文选择：

```elixir
or
and
not
```

表达集合论中的并集、交集和否定。作者认为，Elixir 开发者更容易理解：

```elixir
JSON.Encoder.t() and XML.Encoder.t()
```

而不是另外引入一套完全不同的符号。

### 12.2 先从隐式类型信息开始

作者规划了三个里程碑。

第一步，类型只在编译器内部使用。开发者暂时不能书写完整类型注解，系统先从 pattern 和 guard 中提取类型信息，用于发现字段名错误和基本类型不匹配。

第二步，为 struct 引入类型注解。struct 是命名且静态定义的 record，能够提供比普通 map 更稳定的字段信息。

第三步，引入函数类型注解。没有显式类型的参数默认使用 `dynamic()`，从而保持旧代码可编译。

### 12.3 类型检查优先于完整类型重建

论文认为，优先支持大多数 Elixir 习惯用法，比立即实现高成本的完整类型重建更实际。类型重建会带来复杂的约束求解和多轮代码分析，因此应当在语言常用结构得到支持后再推进。

## 十三、尚未完成的部分

论文没有把类型系统包装成已经完成的工程产品。作者列出了一系列尚未完成的研究方向。

### 13.1 类型重建与 occurrence typing

当前系统可以检查某些精确类型，但未必能够自动重建这些类型。例如 `filter/2` 的精确结果类型需要分析谓词函数对元素的判断，并把判断结果传递回元素变量。这需要更强的 occurrence typing 技术。

### 13.2 Row polymorphism

map 删除和更新操作需要 row polymorphism，才能表达“保留所有未知字段，同时删除或更新某一个字段”。集合论类型与 row polymorphism 的结合仍是开放问题。

### 13.3 消息传递

`receive` 已经可以利用 pattern、guard 和 narrowing，但 Elixir 进程之间的消息协议还没有被完整类型化。未来可以引入 mailbox types 或 behavioural types，描述进程接收和发送的消息集合。

### 13.4 Behaviours

论文最后用 `GenServer` 说明 behaviour 类型化的难点。理想的 `GenServer(request, reply)` 类型需要同时描述：

- 对外透明的 option 和 result 类型；
- 每个实现内部不透明的 state 类型；
- 由具体实现决定的 request 和 reply 类型；
- 必须实现或可以省略的 callback；
- `start/3` 的第一个参数必须是兼容的 behaviour 模块。

这要求类型系统处理模块类型、抽象类型、参数化 behaviour 以及更复杂的存在类型关系。

## 十四、与 Dialyzer、eqWAlizer 和 Gleam 的关系

### Dialyzer

Dialyzer 采用 success typing，优先减少误报。论文方案追求更强的 soundness，因此可能报告更多问题。二者代表不同的工程取舍：前者适合在大型动态代码库中保守分析，后者试图提供更强的编译期契约保证。[1]

### eqWAlizer

eqWAlizer 支持泛型、局部类型推断、类型收窄和渐进式类型，但论文方案进一步使用集合论类型、否定类型和 strong arrows，并把 map record 与 dictionary 放在统一类型体系中。[1]

### Gleam

Gleam 选择一套从 ML 家族继承而来的静态类型路线，拥有 Hindley–Milner 类型推断和有限的 row polymorphism。它从语言设计之初就选择静态类型；本文方案则直接面对现有 Elixir 代码库，优先解决逐步迁移与语义兼容问题。[1]

## 十五、如何评价这套设计

这套方案的理论优势很明确。交集箭头可以表达多子句函数的输入输出对应关系；guard 分析可以把 Elixir 的控制流信息转化为类型信息；开放 map 类型可以同时处理 record 和 dictionary；strong arrow 则利用已有运行时检查，避免类型系统为了保证安全而自动改变代码执行方式。

工程风险同样明确。类型表达式可能非常复杂，子类型判断可能增加编译成本，更强的 soundness 可能带来更多 warning，而宏、消息传递和 behaviour 仍需要继续研究。论文没有提供大型真实项目上的编译耗时、警告精度和迁移成本数据，因此它应当被看作一套有形式化基础的设计提案和原型路线，而不是已经完成的生产级类型检查器。

这篇论文最重要的贡献，是把问题重新表述为：如何让静态类型适应 Elixir 的运行时和编程风格。它没有要求 Elixir 放弃动态性，也没有把 Elixir 改造成另一门静态函数式语言，而是试图从 pattern、guard、map、BEAM 检查和已有函数契约中逐步提取可靠的类型信息。

## 参考资料

1. Giuseppe Castagna, Guillaume Duboc, José Valim, “The Design Principles of the Elixir Type System”, arXiv:2306.06391v3, 2024。 [论文摘要页](https://arxiv.org/abs/2306.06391)；[PDF](https://arxiv.org/pdf/2306.06391)。
2. Giuseppe Castagna, Guillaume Duboc, José Valim, “The Design Principles of the Elixir Type System”, *The Art, Science, and Engineering of Programming*, 8(2), 2024, Article 4。 [DOI](https://doi.org/10.22152/programming-journal.org/2024/8/4)。
3. 论文中提到的 Typex 原型：[typex.fly.dev](https://typex.fly.dev/)。
4. CDuce 项目：[cduce.org](https://www.cduce.org/)。
5. WhatsApp eqWAlizer：[github.com/WhatsApp/eqwalizer](https://github.com/WhatsApp/eqwalizer)。
6. Erlang Typespec 文档：[erlang.org/doc/reference_manual/typespec.html](https://www.erlang.org/doc/reference_manual/typespec.html)。
7. Gleam：[gleam.run](https://gleam.run/)。
