Date: 2011-07-19 03:23
Title: Monad in Scala
Tags: Scala
Category: Programming Language

关于什么是Monad(单子), [All about Monad][1]里是这样介绍的:

> A monad is a way to structure computations in terms of values and sequences of
> computations using those values. Monads allow the programmer to build up
> computations using sequential building blocks, which can themselves be
> sequences of computatpions. The monad determines how combined computations
> form a new computation and frees the programmer from having to code the
> combination manually each time it is required.

> It is useful to think of a monad as a strategy for combining computations into
> more complex computations.


听起来很玄乎. 网络上大部分讨论Monad的文章, 大部分都挤满了希腊字母组成的各种公式,
似乎需要完全的理解Monad, 还需要较为深厚的数学背景, 而从程序设计的角度来讨论Monad
的文章, 也大多都是使用Haskell作为示例来进行的(比如上面引用的这篇). 而以我熟悉的
Scala为示例来讨论的, 我找到了如下的两篇:

* [Monads are Elephants][2]
* [Scala实现的单子(Monad)例子(1)][3]

两篇文章都是以实例来讲解Monad, 这也是我为什么能看懂的原因. 

第一篇实际上是一个系列, 共有4篇, 主题就是"学习单子如盲人摸象, 若期望以实际的应用
来理解抽象的Monad概念, 那么必然只能只知其一而不知其二. 但如此并非无可救药. 正如盲
人摸象,若盲人们能聚在一起做个总结, 那么亦可以从片面理解概念. 如果能够了解Monad的
各种应用实例, 那么离理解Monad也就不远了.
   
第二篇javaeye上的Arbow的作品, 例子改写自[單子(monad)入門(一)][4]这篇文章, 相对于
第一篇来说, 这一篇要更为小巧, 作为理解单子的开始, 再合适不过.

作为抽象思维很糟糕的我来说, 学习Monad的时候就只能做盲人, 本文会借上面两篇文章的例
子来说明Monad的实际应用, 并以应用入手, 希望能讲清楚Monad到底要干什么.

## 简单的例子 -- 求值四则运算表达式

Arbow的文章里举的例子非常简单, 甚至不使用Monad的实现都不会太烦复, 这个例子的任务
是完成一个简单的四则运算器.

    :::scala
    Div(Add(Neg(Num(5)), Num(1)), Num(2))


当不需要考虑Div的时候, 实现的逻辑是很简单的, Arbow给出了下面的方案:摘录自[Scala实
现的单子(Monad)例子(1)][2]:

    #!scala
    trait Expr { def eval:Int }  
    case class Num(n:Int) extends Expr { 
      def eval = n
    }  
      
    case class Neg(e:Expr) extends Expr { 
      def eval = - e.eval
    }  
  
    case class Add(e1:Expr,e2:Expr) extends Expr { 
      def eval = e1.eval + e2.eval 
    }  
  
    Add(Neg(Num(5)), Num(1)).eval // -4

每一种运算被实现为一个case class, class内部定义了eval函数来进行求值. 每一个case
class都可以看成是一个计算过程(computation), case class的eval函数接收Expr作为参数,
进行计算并返回结果, 比如Add.eval接收两个Expr,首先分别对两个Expr求值，然后去取和.
随后这些computation被组合起来完成"-5 + 1"的功能.

注意这里用到了组合起来的计算(Combined Computation), 正好符合Monad需要解决的问题,
遗憾的是目前的computation都非常简单, 组合起来也非常方便, 实际上没有使用Monad的必
要.

但需要注意的是, 我们目前还没有引入Div操作. 当我们引入Div运算之后, 情况就比原来要
复杂了. 因为我们需要判断除数为0的情况, 并且在除数为0时返回一个错误, 而不是继续求
值进行除运算. 因为除数为0会引起JVM抛出异常, 而我们希望表达式计算的实现可以合理的
方式处理这样的错误. 一般来说, Div.eval需要返回特定的值来指示这样的错误, 而这个错
误信息需要在组合的computation中层层传递, 直到最上层被告知给调用者, . 在大多数的程
序设计语言中, 这样的工作通常以异常来实现, 在除数为0的时候抛出异常, 然后运行时或者
解释器会将异常层层传递, 直到异常被处理.

当使用Scala时, 我们更喜欢使用的是Option, 于是就有了Arbow加入Div之后的方案(这里不
再摘录Arbow的源代码, 请访问[这里][3]查看代码). Option可以被看成是一个容器, 子类
Some用来包装正确的值,而None则用来指示错误的发生. 而使用Pattern Match可以很容易的
对两者进行区别, 以进行不同的操作. 当使用Option这样的容器作为computation之间的参数
传递时,可以通过判断为Some还是None来进行不同的操作, 最终完成计算和错误处理.
  
在例子里面, 我们可以看到有下面的几个变化:

+ Expr.eval的返回值变成了Option[Int]
+ 当Div.eval的除数为0时, 返回值为None
+ 当某一个操作数的值为None时, 任意一个操作的返回值都是None
      
当使用Option的时候, 我们发现eval函数的逻辑变得更为复杂了, 原因就在于需要检查传入
的参数是None的情况. 而事实上, 这样额外的逻辑是可以用Monad来简化的. 因为Monad的作
用就是"组合已有的combined computation来形成新的computation, 从而简化程序员手动组
合computation的工作".

## Functor

在介绍使用Monad的方案之前, 我们先来探究以下到底什么是Moand呢? 在函数式的程序设计
语言中, Monad这个概念正是由定义在其上的函数来表达的.
"[Monads are Elephants][2]"这篇文章里用Scala语言定义了Monad需要满足的条件.

首先, 每一个Monad都是一个Functor. 在Scala中, Functor是指一个定义了map方法的类.
而该map方法的定义则需要遵守一定的规则. map方法的signature应该是下面的
样子:

    :::scala
    class M[A] {
      def map[B](f: A => B): M[B] = ...
    }


如果我们把Monad想象成为一个Container, 类似于List. 那么map操作的含义就是对容易里面
的每一个值, 通过函数f进行处理, 得到类型为B的结果, 然后这些结果将被打包在这样一个
同样的Monad Container里. 而map方法则需要满足下面的两个规则:

- **规则1: Identify **:
    
	    :::scala
        // if we have identify defined to be
        def identify[A](x: A) = x

        // then map should 
        m map identify == m

    
- **规则2: Composition **:

        :::scala
        m map g map f = m map { x => f(g(x)) }

     
第一条规则涉及到另外一个函数identify, 顾名思义， identify被定义为接收一个参数x，
并原封不动的返回那个参数。 而当map接收这样的一个函数时， 它必须返回一个和m这个
functor相等的值。注意， 这里只需要相等，而不一定要是同一个对象。

第二条规则则是说当map被组合调用时，相当于传入参数的组合来做一次单独的map调用。这
里如果两次调用传入的参数分别为g和f， 那么相当于只做一次map调用，而传入 /x =>
f(g(x))/ 作为参数.
      
## Monad        
    
而相对于Functor， Monad则还需要定义unit, flatMap两个函数: 


    :::scala
    def unit[A](x: A): M[A] = ...

    class M[A} {
      def flatMap[B](f: A => M[B]): M[B] = ...
    }

    
unit可以看成是constructor或者是factory method, 它用来创建一个Monad的实例.
flatMap和map函数类似, 不同的是它接收的参数f返回的是M[B]. 因此和map不同的是,
flatMap会对Monad Container里面的每一个元素都产生一个新的Monad Container. 然后所有
这些Container里的元素会被取出而组合到一个Monad Container里面. 这个过程一般叫作
Flatten.

unit, map, flatMap是Scala Monad的三个必要函数, 而它们直接存在下面的关系:

    :::scala
    m map f = m flatMap { x => unit(f(x)) }

这个关系使得map和flatMap的定义存在一定的联系.
    
而作为Scala Monad, 需要满足的三个规则:

- **规则1: Identify :**

        :::scala
        m flatMap { x => unit(x) } == m

+ **规则2: Unit :**

        :::scala
        unit(x) flatMap f == f(x)

+ **规则3: Composition :**

        :::scala
        m flatMap g flatMap f = m flatMap { x => g(x) flatMap f }

无论是Monad需要满足的Functor的特性还是Monad额外的特性, 看起来都比较抽象. 实际上这
样抽象的内容不需要强行进行理解, 理解抽象概念最好的方式还是从实际的例子出发, 来看
看实际应用中定义的Monad是什么样的.

## Option

Option实际上就是Scala中的一个Monad实现. 下面是scala中对Option的定义(有所简化):
  
    #!scala
    sealed abstract class Option[+A] {
      def isEmpty: Boolean      
      
      def get: A
      
      def map[B](f: A => B): Option[B] =
        if(isEmpty) None else Some(f(this.get))
	
      def flatMap[B](f: A => Option[B]): Option[B] =
        if(isEmpty) None else f(this.get)
    }

    final case class Some[+A](x: A) extends Option[A] {
      def isEmpty: false
       
      def get = x
    }

    final case class None extends Option[Nothing] {
      def isEmpty = true
      
      def get = throw new NoSuchElementException("None.get")
    }

其中有一些Scala中的对象和类型定义的细节, 这里略过. 重点是unit, map和flatMap的定义.
因为Some和None都被定义为case Class, 所以其名字就是constructor, 我们无需再定义
unit函数. 相对来说, map和flatMap的定义都很简单, 这里我们可以把Option看成是一个只
有一个元素的Container, 那么map的工作就是对这仅有的一个元素进行f调用,然后再把结果
打包成Option. 而flatMap的工作则是对这个元素进行f调用,产生的结果已经是Option实例,
而且没有其它的结果需要进行flatten, 因此可以直接返回这个Option实例.

而None的存在是唯一的插曲, 实际上如果把实现放在None和Some里面会更为清晰, 其实这里
map和flatMap对于None要做的额外工作就是检查自己是不是None, 如果是, 那么就直接返回
None. 因为None表示的是一个空的Container, 因为不能进行其它的计算.
    
这里None的存在还涉及到Monad里面的Monadic Zeros的概念. 本文不再讨论, 你可以阅读
[Monads are Elephants][2]一文来了解.

## 使用Option Monad来解决表达式求值问题
    
Option是Monad, 那么我们当然可以用它来解决表达式求值问题了. 事实上, 我们已经使用
Option了, 只不过前面利用的是Option中None表示错误状态这样的特点, 我们还是使用手工
的方式组合computation, 判断是否为None, 并根据判断进行不同的操作. 通过Option的源代
码, 我们知道Option的map和flatMap可以帮助我们进行这样的判断, 而且我们可以传入函数
参数来进行计算. 所以Arbow给出了Option的Monad版本.

同样的, 代码不再重复贴出, 我们挑Add操作来分析加入Monad前后的实现, 在加入
Monad之前的代码是下面的样子, 我们手动分析参数Expr进行eval之后的结果是否为
None, 并根据判断结果进行不同的操作:

    :::scala
    case class Add(e1:Expr,e2:Expr) extends Expr {  
      def eval = e1.eval match {  
        case Some(n1) => e2.eval match {  
          case Some(n2) => Some(n1 + n2)  
          case None => None  
        }  
        case None => None  
      }  
    }
    
我们来简单分析一下, Add操作需要做的是将两个Expr e1, e2的求值结果进行相加, 而当其
中任何一个是None的时候, Add操作总是返回None. 当映射到map和flatMap时, 就相当于对
e1.eval的结果map上"加上e2.eval"这样的操作. 所以第一步我们可以写成如下的样子:

    :::scala
    case class Add(e1: Expr, e2: Expr) extends Expr {
      def eval : Option[Int] = e1.eval match {
        case Some(n1) => e2.eval map { x => n1 + x }
        case None => None
      }
    }
    
当想清楚这一步之后, 既然内部的pattern match操作可以简化成一次map, 那么外面的是否
也可以呢．我们可以把. "`e2.eval map {x => n1 + x}`"看成是一个"`x: Int => Option [Int]`"
的函数. 因此可以写下最后的版本:

    :::scala
    case class Add(e1: Expr, e2: Expr) extends Expr {
      def eval : Option[Int] = 
        e1.eval flatMap (y => (e2.eval map (x => x + y ))
      }
    }

这里外层使用flatMap的原因是内部的 `(y => (e2.eval map (x = x + y))` 操作返回
的是Option[Int], 因此需要flatten.

因为map和flatMap都已经帮我们处理了参数为None的情况, 所以我们不再费神去写检测的代
码. 当所以的操作都写成如上例子中的效果时, 我们的程序瞬间简化了很多.

## for/yield syntax sugar
    
直接的调用map和flatMap看起来还是有些别扭, 如果并非对函数式程序设计烂熟于心,还是很
难看出到底在做什么样的操作. 而实际上Scala提供了for/yield syntax sugar来做
map/flatMap的操作.

    :::scala
    val qs = for(n <- ns) yield f(n)

    // is syntax sugar for
    val qs = ns map f

    // and a little bit complex    
    val qs = for(n <- ns; o <- os) yield f(n, o)

    // is syntax sugar for
    val qs = ns flatMap ( x => ns map (y => f(x, y) ))

    // or more complex 
    ... ...

从 `ns map f` 到 `for(n <- ns) yield f(n)` 较容易理解, map的含义就是对Container
中的每一个元素进行函数f操作, 然后结果被组合成在新的Container里面,这个操作和
`for(n <- ns) yield f(n)` 是一样的效果. 而map/flatMap的组合, 我们可以通过Add操作
的例子来理解, 在Add操作中, 我们通过flatMap和map的组合完成的操作,实际上就是对两个
Container中的元素的组合进行f操作. 当然, for语句的含义远比上面描述的还要复杂, 这里
不再描述, 更多内容, 请参考[Monads are Elephants][2].

当了解到这样的syntax sugar之后, 我们的Add.eval方法可以最后变成:

    :::scala
    def eval: Option[Int] = 
        for(x <- e1.eval, x <- e2.eval) yield (x + y)
    
没有烦人的None判断, 也没有map和flatMap. 


[1]: http://haskell.org/all_about_monads/html/introduction.html "All about Monad"
[2]: http://james-iry.blogspot.com/search/label/monads "Monads are Elephants"
[3]: http://arbow.javaeye.com/blog/721848 "Scala实现的单子(Monad)例子(1)"
[4]: http://www.iis.sinica.edu.tw/~scm/ncs/2009/11/a-monad-primer/ "單子(monad)入門(一)"

<!-- 
Local Variables:
mode:markdown
End:
-->
