# OCEval

## 需求

目前流行的 JSPatch/RN 基于JavaScriptCore提供了iOS的热修复和动态化方案。但是都必须通过下发Javascript脚本来调用Objective-C。 尤其是JSPatch，编写大量的JS代码来调用OC的方法，开发效率较低(目前可以借助语法转换器)，运行效率也会打折扣。
更好的方案是直接编写Objective-C代码，来实现热修复或者动态化方案。开发效率更高，代码的执行效率也更高。

在python和javascript等脚本语言里，有类似eval()函数来直接动态执行代码。所以我实现了[OCEval](https://github.com/lilidan/OCEval) 这个库，让我们能直接动态执行Objective-C代码。例子如下:

```objc
NSString *inputStr = @"return 1 + 3 <= 4 && [NSString string] != nil;";
NSNumber *result = [OCEval eval:inputStr]; // result: @(YES)
```

为了实现跟JSPatch类似的热修复功能，增加了方法替换。我们就可以通过下发Objective-C代码进行现有App的方法替换，来进行热修复的功能。

In order to fix bugs dynamically,we should be able to replace original Objective-C method. We could deliver Objective-C code through network,like this:


```objc
//call original invocation in new implementation
NSString *viewDidLoad2 = @"{\
[originalInvocation invoke];\
";

[OCEval hookClass:@"ViewController"
         selector:@"viewDidLoad"
         argNames:@[]
          isClass:NO
   implementation:viewDidLoad2];
```

Theoretically we could make a whole application written by Objective-C and deliver it through the network. I had finished a simple ViewController in the iOS demo.


OCEval甚至可以用来完整的编写一个页面或者App，并动态下发。我在iOS的Demo里实现了一个简单的页面，具体见源码。

## 实现原理

C和C++的性能高，是因为编译型语言在编译期就已经生成了机器码，运行时只需要执行机器码所以执行效率高，但是动态性差。
js的性能差，是因为js的runtime引擎通常是在实际执行前进行的编译的。优点是动态性好。
像Dart和python等等都可以编译打包执行或者JIT(Just in time)执行。

不同于C和C++，Objective-C是动态化的语言，Objective-C的runtime利用消息发送和转发可以动态地执行任何方法。
同时Objective-C又不同于javascript等完全动态化的语言。 因为大多数调用是在编译期就已经决定的，编译出可执行文件(mach-O)。


所以在OCEval里实现了一个轻量级的解释器，动态地解释Objective-C代码，同时利用OC的runtime消息转发来动态执行Objective-C的代码，就可以实现类似eval()函数的完全动态化方式。


#### 解释器

Objective-C在LLVM下的编译过程:

源码 -> AST -> LLVM IR(中间语言) -> LLVM Bytecode -> ASM -> Native

LLVM的前端是Clang，Clang的工作是把源码变成AST语法生成树。

Clang的前端编译过程：
- ``Preprocesser``: 包括#include #import等预处理, #if,#ifdef 等条件，#define等宏定义
- ``Lexer``:词法分析，把文本变成token(Tokenizer)
- ``Parser``:语法分析，把token变成AST

但是在OCEval里，没有做得那么复杂，因为只是为了能够执行。所以只实现了词法分析和语法分析，得到语法生成树AST。

#### Runtime

执行的时候递归下降地执行每一条指令。这里利用的runtime主要是NSInvocation，利用methodSignature封装方法的调用惯例，跟JSPatch/RN的最终调用方式如出一辙。

```objc
+ (id)invokeWithCaller:(id)caller selectorName:(NSString *)selectorName argments:(NSArray *)arguments
{
    SEL selector = NSSelectorFromString(selectorName);
    NSInvocation *invocation;
    NSMethodSignature *methodSignature = [caller methodSignatureForSelector:selector];
    invocation= [NSInvocation invocationWithMethodSignature:methodSignature];
    [invocation setTarget:caller];
    [invocation setSelector:selector];
    NSUInteger numberOfArguments = methodSignature.numberOfArguments;
    NSInteger inputArguments = [arguments count];
    if (inputArguments > numberOfArguments - 2) {
        id result = invokeVariableParameterMethod(arguments, methodSignature, caller, selector); //转而调用objc_msgsend
        return result;
    }
    return [self invokeWithInvocation:invocation argments:arguments];
}
```

参考JSPatch，在类似``[NSString stringWithFormat:]``这样可变参数的方法里使用``objc_msgsend``。因为``NSInvocation``不支持不确定的参数个数的情况。

## 性能

因为省去了跟JavascriptCore进行参数传递的过程，单个方法调用比JSPatch/RN快100%，耗时只有JSPatch一半，多个方法调用优势更大，耗时可能只有30%以下。

NSInvocation的调用跟Native速度差不多。但是因为动态调用很麻烦，入参出参和调用惯例都需要动态定义，同时上下文参数在内存的传递也比较慢，所以整体是比原生慢很多(动态化必要的牺牲)。

## 审核

我没有尝试过提交AppStore审核，但是鉴于JSPatch屡次被拒绝，被拒绝的可能性极大。我们的App确实也没有热修复的需求。

## 感谢

感谢JSPatch,libff,Aspect
