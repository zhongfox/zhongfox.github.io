---
layout: post
tags : [ruby, 面向对象, 元编程]
title: Ruby 元编程学习笔记

---

最近又在翻看《Ruby元编程》，觉得有些东西记下来印象会更深刻。决定用这篇文章来记录Ruby元编程的相关笔记，持续更新。

as time goes by ... 本笔记已经不局限于元编程，除了元编程笔记，还有很多是和ruby相关的一些知识和技巧。

1.  send

  ~~是BasicObject的公用类方法, 但却不是它的单键方法，因为BasicObject已无超类，怀疑是BasicObject mixin 了什么模块？~~ Kernel的公共实例方法, BasicObject.public_methods 能调用send是因为BasicObject是Class的实例, 也是Object的实例.

    `BasicObject.public_methods.grep(/^send/) => [:send]`

    `BasicObject.public_methods(false).grep(/^send/) => []`

    send的一个用处是用来调用某一对象的私有方法

    <s>include 是 Kernel 的私有方法</s>，include是Module的私有实例方法(因为Kernel是一个Module实例，所有Kernel也有该方法)。

    因此如果在模块/类外对其加入mixin，可以使用send：

        module Validations

          def self.included(base)
            base.send :include, ActiveSupport::Callbacks
          end
        end

2.  Module有一个私有实例方法 define_method

    `Module.private_instance_methods(false).grep(/define_method/) => [:define_method]`

    该方法成为每个模块/类的类方法，所以任何模块/类都可以用 define_method 为本模块/类增加实例方法

    如果要为一个类动态增加类方法（方法名是动态的字符串），可以考虑在该类的单件类中, 调用define_method(只要保证调用者是该类的单件类)

    经常在Kernel上使用该类方法，来定义一个**内核方法**： `Kernel.send :define_method, :test do puts 'Im a test' end` 因为define_method是Kernel的私有方法，所有用send来调用。

    Class是Module的subclass，Object是Class的实例，所以：

    `Object.private_methods.grep(/define_method/) => [:define_method]`

    但是这是为什么？ `BasicObject.private_methods.grep(/^define_method/) => [:define_method]`

3.  `class` `module` `def` 关键字会开启新的作用域

    与之对应的通过传递block来扁平化作用域的办法有：`Class.new` `Module.new` `Module#define_method`

    其他常见的扁平化作用域的：`Object#instance_eval` `Module#class_eval` `module_eval`

4.  可调用对象：Proc对象, lambda, method对象

    可调用对象的一个使用是实现延迟执行，这个在rails的scope中经常用到（scope中的lambda的Time.now不会在class加载的时候固化）

    方法中获得传入代码块转化的对象是Proc对象

    `Proc.new => proc`

    `proc {block} => proc`

    `lambda {block} => lambda`

    `->(arg) {block} => lambda`

    proc 和 lambda 的区别：

    * proc中的return会从**定义**proc的作用域返回，lambda会更合理的从lambda中返回

    * proc自适应传递的参数个数，lambda严格要求参数个数

    可以使用多个代码块来共享一个局部变量，但是我更喜欢这种类似javascript的闭包的方式：

        lambda {
          share = 10

          Kernel.send :define_method, :increase_a do
            share += 1
          end

          Kernel.send :define_method, :show_a do
            puts share
          end

        }.call

    这和js最佳实践中，每个文件都定义一个马上运行的闭包如出一辙：

        (function () {
          var share = 10;

          function increaseA()  {
            share++;
          }

          function showA()  {
            alert(share);
          }

          window.increaseA = increaseA;
          window.showA = showA

        } ());

    **method 对象**

    * `Method`

      已经有绑定对象, 可以调用call, 可以调用`Method#unbind` 转化为 UnboundMethod

      如`object.method :my_method` 绑定对象就是object

    * `UnboundMethod`

      没有绑定对象, 不可以call, 调用bind(对象)方法转化为Method

      如`MyModule.instance_method(:my_method)`

5. 寻找当前类，在代码任何地方，我们都需要留意当前的self是什么，但有的时候也需要留意当前class是什么，因为在使用关键字def定义方法时，是为某个class增加实例方法：

        class MyClass
          def method_one
            def method_two; "hello!"; end
          end
        end

        obj = MyClass.new
        obj.method_one
        obj.method_two #=> "hello!"

    其实method_two 也成为了Myclass的实例方法

    **结论：在def的时候，**当前`class = self.is_a? Class ? self : self.class` 下面的 instance_eval 例外。

    关于`class_eval`， 它会修改self和当前class，class_eval比关键字class更灵活，它可以用在常量和变量上。

    关于`instance_eval`, 它除了修改self外，还修改了接收者的单件类，也就是说在 instance_eval 中使用def定义的方法，并不是该对象的类的实例方法，而是该对象的单件方法。

6. 类的实例变量和类变量：

   当前环境定义的实例变量都属于当前self，如 `class MyClass; @my_var = 1; end` Myclass就拥有了一个实例变量，类的实例变量仅仅是这个类对象的实例变量，因此**类的实例变量和继承链无关**

   当前环境定义的类变量都属于当前class，类变量是可以继承的，而在最顶层环境定义的类变量都属于当前class：Object，看看下面诡异的结果：

        @@v = 1
        class MyClass
          @@v = 2
        end

        @@v #=> 2

   原因是全局定义@@v是属于Object的，MyClass继承了Object，所以上面的代码出现了子类复写类变量。

   So最佳实践是：避免使用类变量，尽量使用类的实例变量。

7. 匿名类：使用 Class.new 可以创建一个匿名类，传一个参数作为该匿名类的超类，以及一个代码块。每个类对象都有name方法返回自己的类名（字符串），但是匿名类的name是nil，直到把该匿名类赋值给一个常量，该常量的字符串表示将成为这个类的name：

        c = Class.new(Object) {}
        c.name # nil
        MyClass = c
        c.name # "MyClass"

8. Duck Typing: ruby在乎的不是一个对象是什么类，而是这个对象能相应的方法，方法可以是普通方法（实例方法），单键方法或者幽灵方法。

9. 类方法就是类的单件方法。

10. 对象扩展：类include一个模块，是把模块的实例方法变成类的实例方法, `A include B` B 必须是一个模块, 不能是类

    类扩展：在类/对象的单件类中include一个模块，就把模块的实例方法变成了类/对象的单件方法。

        module MyModule
          def my_method; 'hi'; end
        end

        class MyClass
          class << self
            include MyModule
          end
        end

        MyClass.my_method #hi

    类扩展的快捷方式是extend，如

        module MyModule
          def my_method; 'hi'; end
        end

        class MyClass
          extend MyModule
        end

        MyClass.my_method #hi

    `base_class.extend feature_class`不会改变base_class的继承链，但是会改变base_class的单件类的继承链：

        module A
        end

        class B
          extend A
        end

        B.ancestors #=>[B, Object, Kernel, BasicObject]
        B.singleton_class.ancestors #=> [A, Class, Module, Object, Kernel, BasicObject]

11. 使用`alias`来实现环绕命名

    alias是ruby的关键字而不是方法，所以使用是新旧方法名间无逗号： alias new_method_name old_method_name

    Module#alias功能与关键字alias功能相同，新旧方法名间有逗号。

    环绕命名使用场景：希望把旧的方法功能重新升级，但是新功能的实现依赖旧方法（如重写Fixnum#+使1+1=3）

    步骤：

    1. alias 旧方法代理名 旧方法名

    2. 重新定义 旧方法，在重写定义中调用旧方法代理; 或者：

       定义新方法名，在新方法中调用旧方法代理，然后 alias 旧方法名 新方法名; 这种方式比上面一种多了一个新方法名可以用而已。

    总之可以实现调用旧方法名，实现了新功能。

12. 关于方法访问：

    * `public`

    * `protected` 可以继承，只能在类内部访问，可以指定接收对象

    * `private` 可以继承，只能在类内部访问，不能指定接收对象（只能隐式使用self）

13. `attr_accessor`

    在类定义中，使用该方法为类的实例声明读方法和写方法, 每对写方法内部对应着一个实例变量可以使用`object.instance_variables`获得

    该操作同样可以对类对象实施：

        class Client
          class << self
            attr_accessor :host, :port
          end
        end

    类Client可以使用host，port读写方法

    `Client.instance_variables` => [:@host, :@port]

14. `attr`

    在类定义中，使用该方法为类的实例声明读方法，但是没有写方法

    可以传递true设置写方法：`attr_accessor :a` 等于 `attr :a, true`

15. module被include时的钩子：

    可以在included钩子中对base extend other，以实现添加类方法(这样一个include既添加了实例方法，又添加了类方法)

        module A
          def self.included(base)
            puts 'do something for base'
            puts "base is #{base}"
          end
        end

        class B
          include A
        end

16. 查看module下的所有常量：`#constants` 如`User.constants`

17. 关于代码块参数的定义，以及代码块的调用：

    * block to a Proc

      方法定义时用&b，如`def do_something(x, y, z, &b)` 调用时可以用`b.call(参数)` 或者 `yield(参数)`

    * Proc to a Block

      获得一个Proc对象, 可以将其作为代码块传递给需要代码块的方法, 如: `b = -> {puts 'Im block object'}`,  方法调用时`do_something(x, y, z, &b)`

    * 方法定义时没有代码块参数，方法中调用代码块只能用 yield

    * 方法定义时用b，如`def do_something(x, y, z, b)` 调用时可以用`b.call(参数)` 这其实是传递普通对象

    * 总结： &b是表示代码块，b是表示 Proc 对象

18. ruby 特殊常量：

    * $: 或者 `$LOAD_PATH`  default search path (array of paths)
    * $" 已经加载过的文件
    * $0 Ruby脚本的文件名
    * `$*` 命令行参数
    * $$ 解释器进程ID
    * $! 最近一次的错误信息
    * $@ 错误产生的位置
    * `$_` gets最近读的字符串
    * $. 解释器最近读的行数(line number)
    * $& 最近一次与正则表达式匹配的字符串
    * $~ 作为子表达式组的最近一次匹配
    * $n 最近匹配的第n个子表达式(和$~[n]一样)
    * $= 是否区别大小写的标志
    * $/ 输入记录分隔符
    * $\ 输出记录分隔符
    * $? 最近一次执行的子进程退出状态

19. require load autoload

    * require:

      若是相对路径，则查找$:

      若省略扩展名则按以下顺序查找[.rb,.so,.dll]

      rb文件作为源文件载入，其它作为扩展载入

      已经载入的文件存放于$"中

      require不会载入已存在于$"中的文件，如果载入成功返回true，否则false

    * require_relative

      这个相对关系是相对于调用`require_relative`的文件，因此如果在irb(不在文件里)里调用会得到`LoadError: cannot infer basepath`

      加载成功后返回true, 把文件放入$"

      重复加载返回false

      文件不存在报错

      ruby1.9.2 中因为安全的原因从$LOAD_PATH移除了当前目录，因此不能用require去加载当前目录文件

      引入的require_relative 则是用来加载当前目录文件的方法

    * require 和require_relative的区别在于引入类似`require 'mytest.rb'` 这种形式，对于`require './mytest.rb'` 都可以成功加载相对路径文件

    * load:

      load加载文件

      不记录在$" 中，可以多次加载

      必须要加扩展名

      load的作用：在development模式下，当你修改一段代码后，不用重启服务器，你的代码更改会被自动reload，这就是load的作用 而如果你使用require的话，多次require并不会起作用

      You use load to execute code, and you use require to import libraries. load 和 require都使自己定义的常量残留下来, load提供了一个解决方案`load(file_name, true)` If you load a file this way, Ruby creates an anonymous module, uses that module as a Namespace to contain all the constants from motd.rb , and then destroys the module

    * autoload(module, filename): 已经不推荐使用

      module 可以是字符串或者符号

      注册以后将会被加载的文件，当module被首次使用时

      加载时使用的是 Kernel::require

20. 给Hash中不存在的key加上默认值技巧

        1.9.3p125 :008 > a = Hash.new {|h,k| h[k] = []}
        => {}
        1.9.3p125 :009 > a[:no_exist_key]
        => []

    关于Hash.new

        h = Hash.new("Go Fish") #设置默认值
        h["a"] = 100
        h["b"] = 200
        h["a"]           #=> 100
        h["c"]           #=> "Go Fish" #没设置过的键会返回默认值，但是此键仍然不存在！
        # The following alters the single default object
        h["c"].upcase!   #=> "GO FISH" #修改没显示设置过的key，实际上会修改默认值
        h["d"]           #=> "GO FISH"
        h.keys           #=> ["a", "b"] #没设置过的键，虽然会返回默认值，但是key还是没设置过

        # While this creates a new default object each time
        h = Hash.new { |hash, key| hash[key] = "Go Fish: #{key}" } #在block里设置了key，因此key存在。key的值将是block的返回值
        h["c"]           #=> "Go Fish: c"
        h["c"].upcase!   #=> "GO FISH: C"
        h["d"]           #=> "Go Fish: d"
        h.keys           #=> ["c", "d"]


21. 对象类型判断

    `o.instance_of? SomeClass` 不考虑继承链 等价于`o.class == String`

    `o.is_a? SomeClass` 等价于 `o.kind_of? SomeClass` 等价于`SomeClass === o` 都会考虑继承，也包括继承链上的module

22. 对象实例变量操作

    `Object#instance_variable_set(symbol, obj)`

    `Object#instance_variable_get(symbol) → obj`

    `Object#instance_variable_defined?(symbol) → true or false`

23. super

    super 调用当前类的祖先类中的同名方法

    super不带括号表示调用父类的同名函数，并将本函数的所有参数传入父类的同名函数；

    super()带括号则表示调用父类的同名函数，但是不传入任何参数；

24. Module#ancestors

    对Module的实例（模块）来说，因为没有superclass方法，ancestors 只返回他include的module和自己

    对Class的实例（类）来说，包括自身，include的module和继承的祖先类

25. 对象相等判断

    * `equal?`  判断是否是同一个对象，等价于判断`a.object_id == b.object_id`， 最佳实践:用于不要重写`equal?`

    * `==` Object类中它是equal?的同义词，大多数类重新实现此方法

      * Hash 遍历键值对比较，键通过`eql?` 值通过`==`，如果全等，即为相等
      * Array 遍历相同位置元素，通过`==`比较，如果全等，即为相等

    * `eql?` Object类中它是equal?的同义词，大多数类重新实现此方法(为严格相等，不允许类型转换)

    * `===` Object类中它是==的同义词, 条件性相等，大多数类重新实现此方法

      * 范围匹配 `(1..10) === 5`
      * 正则匹配 `/\d+/ === '123'`
      * 类的实例判断 `String === 'ss'`
      * Symbol 判断同内容的字符串 `:s === 's'`
      * case 里隐式判断

    * `=~` 只对String和正则的对象有用

      * '123' =~ /\d+/ 返回字符串中第一匹配的index
      * /\d+/ =~ '123' 同上

26. `Array` 和 `Hash` 居然还是Kernel中定义的方法：

        #Returns arg as an Array. First tries to call arg.to_ary, then arg.to_a
        Array(1..5)   #=> [1, 2, 3, 4, 5]

    这个方法有一个好处，参数可以是数组或者单个值，最后都会转换成数组

        irb(main):048:0* Array(1)
        => [1]
        irb(main):049:0> Array([1])
        => [1]
        irb(main):050:0> Array([1,2])
        => [1, 2]

        Hash(a: 1) #=> {:a=>1}

27. 关于method的类

  * Method类: 是绑定了调用对象的方法对象, 如`1.method(:to_s) => #<Method: Fixnum#to_s>`

    可以直接调用(用call或者[]): `1.method(:to_s).call(参数)` `1.method(:+)[2]`

    转换为UnboundMethod: `1.method(:to_s).unbind`

  * UnboundMethod类: 没有绑定调用对象的方法对象, 如`Fixnum.instance_method(:to_s) => #<UnboundMethod: Fixnum#to_s>`

    不可以直接调用

    转换为Method类: `Fixnum.instance_method(:to_s).bind(1)`, 目标对象必须是同类对象

28. binding

  binding 代表了代码执行的上下文环境

  You can think of Binding objects as “purer” forms of closures than blocks because these objects contain a scope but don’t contain code

  * 常量`TOPLEVEL_BINDING`代表顶层binding对象

  * `Kernel#binding`可以获得当前的Binding对象

  * `Proc#binding`可以获得代码块的闭包Binding对象

        # 例1
        def var_from_binding(&b)
          eval('var', b.binding)
        end
        var = 123
        var_from_binding {} #123

        # 例2
        class A
          def abc
            binding
          end

          private
          def xyz
            'xyz'
          end
        end

        eval "xyz",  A.new.abc # => "xyz"


29. respond_to? will return false for protected methods in Ruby 2.0 <http://tenderlovemaking.com/2012/09/07/protected-methods-and-ruby-2-0.html>

---

### 再读元编程(第二版)

1. ruby 2 还提供了类似`include`的`prepend`, 区别在于继承链中的位置:

        module M3
          prepend M1
          include M2
        end
        M3.ancestors # => [M1, M3, M2]

2. Kernel

   class Object includes Kernel , so Kernel gets into every object’s ancestors chain.

   Every line of Ruby is always executed inside an object(main)

   So you can call the instance methods in Kernel from anywhere (比如print)

3. Every line of Ruby code is executed inside an object

4. Refinements TODO

5. 复写`method_missing`

   * `def method_missing(method, *args)`, 在此方法中`block_given?`可以得到原始方法是否有传代码块
   * `def method_missing(message, *args, &block)` 需要调用block的话

6. remember to override `respond_to_missing?`  every time you override `method_missing`

7. 想防止方法干扰, 白板类

   * 继承 BasicObject `BasicObject.instance_methods(false)  => [:==, :equal?, :!, :!=, :instance_eval, :instance_exec, :__send__, :__id__]`

   * Object 没有新加实例方法, 全靠include Kernel `Object.instance_methods(false) => [] `

   * 移除方法:

     * `Module#undef_method` 移除包括继承链上的方法
     * `Module#remove_method` 只移除接收者的方法, 不会影响继承链的方法

8. `instance_eval` 代码块无参数, `instance_exec`有参数, 有些不常用的微妙区别

9. a lambda is evaluated in the scope it’s defined in, while a Method is evaluated in the scope of its object

10. 类宏`attr_*` 是 类 Module 提供的实例方法, 因此module也可以使用, 生成读写方法用于被其他class混入

11. singleton_class

    singleton classes have only a single instance (that’s where their name comes from), and they can’t be inherited.

    More important, a singleton class is where an object’s Singleton Methods live:

        def obj.my_singleton_method; end
        singleton_class.instance_methods.grep(/my_/)
        # => [:my_singleton_method]

12. When you redefine a method, you don’t really change the method. Instead, you define a new method and attach an existing name to that new method. You can still call the old version of the method as long as you have another name that’s still attached to it

13. eval(statements, @binding, file, line)

    在irb里可以嵌套开启另一个irb, 还可以指定binding对象, 如`irb a_object` 新的irb会话是在`a_object`里的(类似instance_eval) irb的实现代码就死eval

    参数file, line用于: 当出eval的代码执行现异常时, 报错的stack中打印文件和行数


    eval always requires a string, `instance_eval` and `class_eval` can take either a String of Code or a block

    **最佳实践**

    you should probably avoid Strings of Code whenever you have an alternative.

14. Hooks

    Class#inherited

    Module#included

    Module#extended

    Module#prepended

    实例方法hooks: Module#method\_added Module#method\_removed Module#method\_undefined

    单件方法hooks: BasicObject#singleton\_method\_added , singleton\_method\_removed , and singleton\_method\_undefined


---

### 特殊方法来源汇总

* `define_method`

  `Module.private_instance_methods(false).grep(/define_method/) => [:define_method]`

* `include`

  `Module.private_instance_methods(false).grep /include/ => [:included, :include]`

* `send`

  `Kernel.public_instance_methods(false).grep /send/ => [:send, :public_send]`

* `method_missing`

  `BasicObject.private_instance_methods(false).grep /method_missing/ => [:method_missing]`

* `respond_to?`

  `Kernel.public_instance_methods(false).grep /respond_to\?/ => [:respond_to?]`

* `const_missing`

  `Module.public_instance_methods(false).grep /const_missing/ => [:const_missing]`

* `undef_method`

  `Module.private_instance_methods(false).grep /undef_method/`

* `instance_eval`

  `BasicObject.public_instance_methods(false).grep /eval/ => [:instance_eval]`

* `class_eval`

   `Module.instance_methods(false).grep /eval/ => [:module_eval, :class_eval]`

* Object#define_singleton_method

* Kernel#eval

* `private` `Kernel.private_methods(false).grep /private/   => [:private]`
