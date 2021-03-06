# wasm的编译

# 一、WASM的编译器介绍
Webassembly出现的时间并不长，从目前来看，主流的编译器有以下几类：Emscripten工具链，binaryen ，AssemblyScript。其实他们主要的还是要依赖于LLVM，不管前台用什么来编译，当到达IR这一层面时，目前来看，LLVM的处理仍然是主流。
</br>
从目前提供的编译方式来看，有本地的编译环境，也有在线的编译环境。但它们背后使用的编译器基本上不会逃脱上面的几种编译工具。主流的几个在线编译器在前面提供了：
</br>
https://webassembly.studio/
</br>
https://wasdk.github.io/WasmFiddle/
</br>
本地工具链的安装可以采用官方文档：
</br>
https://emscripten.org/docs/getting_started/downloads.html
</br>
下面有一个整理好的方法：
</br>

```shell
$ git clone https://github.com/juj/emsdk.git
$ cd emsdk
$ ./emsdk install latest
$ ./emsdk activate latest
```
在老的版本中，曾经提供过在Windows上的离线安装版本，但在新版本里，统一使用相关的安装方式了。
# 二、相关编译技术
编译器的作用简单说来就是一个，把人类看懂的语言翻译成机器可以执行的语言。至于在编译过程应用的各种技巧和方法，都是为了使编译更高效、安全、快捷。
</br>
大家都知道现在代的编译器基本上分为两大类，即静态编译器和动态解释器。它们各有优势，目前在前端开发使命的语言中，以解释器为主，但随着技术的进步，二者之间的界限大有弥合的趋势。包括JAVA在内的虚拟机技术都提供了JIT（Just-In-Time）编译器。
</br>
在传统的解释器中，最初的工作方式如下：
</br>
代码-记法分析-语法分析-语法树（AST）-AST遍历解释器-执行结果。
这样的工作方式简单明了，但是却有一个重大的弊病，那就是效率太低下了，举一个简单的例子，有十个函数都调用了一个处理算法，那么这个算法会被解释十次，如果有更多的调用，会有更多的解释情况，这使得执行的效率成级数的下降。
</br>
解决这个问题的方法就是提供一个虚拟机（VM）入在语法树后，如果有一些公共的热点代码经常被使用就编译成字节码在VM中执行。
</br>
但是无论怎么样，效率的问题，仍然是解释器的一个致命的缺点，如果对效率要求不是很高的情况下，这个缺点倒也无所谓，但当引入一些执行效率很高的场景时，它显然就无法完成这项工作了，至少是不能很好的完成。
而JIT就相对来说又进一步，它将虚拟机的执行分成了三部分，即解释执行，监视器监视，在解释器执行的过程中，如果发现一些代码被执行了多次，就会被标记为Warm，然后送往基线编译器（baseline compiler）进行编译，开成一个桩（stub）,并给与其一个相关的标记索引（行号+变量类型）。这样，下次再执行这段代码时，就可以直接操作这个编译结果而不用去解释。
</br>
基线编译器在应用时是有限制的，最主要的是其编译的时间不能过长，这就意味着基线编译器不能对代码进行优化。
</br>
监视器在整个代码的解释过程中，如果发现某些代码形成了热点（即比Warm执行的次数还要多）那么这段代码会被送入优化编译器（optimizer compiler）进行优化。形成一个更高效的执行结果。
</br>
但是这又引出了一个问题，JS等解释型语言，它的类型是动态确定的，如果在优化的过程中，发现优化的并不对，这就会引起所谓的“去优化”，即重新回到基线编译器甚至解释执行的过程。如果这种现象反复出现的话，就会导致执行效率反而更低。如何解决这种情况呢？
</br>
有两大阵营解决了这种缺陷，一个是将JS等语言静态化，形成一种类似于c++/Java等静态编译语言的机制，但这种方式有开历史倒车的嫌疑；另外一种是将语言的类型固定了有限的几种，比如asm.js只有32整形和64位浮点两种数据类型。
</br>
但解决的方法越多，说明碎片化越严重，所以几大巨头推出了WASM，它干脆去掉了动态语言的解释过程，直接将其搞成了字节码形式（也就是说直接在机器架构的机器码上抽象了一层）。当然为了兼容，可以通过工具（polyfill）将其转换成前面的asm.js（JIT）等进行执行。
但是，它更大的优势在于，它可以使用AOT（Ahead-Of-Time）直接转到机器码执行，那它的执行效率，应该和静态编译语言的执行效率，没有了量级上的差距了。
</br>
在Wasm中，由于已经不需要对类型进行假设，所以上面的两大解决方案自然就被统一，所以目前来看，Wasm的编译优化上天然要比前面提到的技术要强。

# 三、WASM的编译过程
一段代码是如何从源码被编译器翻译成字节码的呢？或者用一句不太准确的话来描述，源码是如何翻译成WASM类型的汇编格式的呢（WAST格式）？
</br>
它的一个基本流程如下：
</br>
源码（c++/C,rust,go等）——LLVM（IR）——字节码——机器码
也就是说，源码通过各种前端（如Clang等）编译成LLVM IR，在这个过程中，LLVM自然会对相关的代码进行各种优化。在到达IR后，如果想将其转换为JS可执行的代码还需要LLVM的一个后端工具（Fastcomp）。
从这里看，WASM的相关编译器很多仍然使用LLVM做为一种核心的编译工具，同时，LLVM正在开发一种专门的后端编译工具来处理这种情况。
</br>
下面看一下代码的汇编文件：

```c++
wasm-function[0]:
  sub rsp, 8                            ; 0x000000 48 83 ec 08
  mov ecx, esi                          ; 0x000004 8b ce
  mov eax, ecx                          ; 0x000006 8b c1
  add eax, edi                          ; 0x000008 03 c7
  nop                                   ; 0x00000a 66 90
  add rsp, 8                            ; 0x00000c 48 83 c4 08
  ret                                   ; 0x000010 c3

wasm-function[1]:
  sub rsp, 8                            ; 0x000000 48 83 ec 08
  mov eax, 0x2a                         ; 0x000004 b8 2a 00 00 00
  nop                                   ; 0x000009 66 90
  add rsp, 8                            ; 0x00000b 48 83 c4 08
  ret                                   ; 0x00000f c3

```
这段汇编代码正是前面反复用到的那个简单的例子的汇编代码。这已经和普通的静态编译器编译出来的代码没有明显的区别了。

# 四、WASM的编译器发展
从目前的情况来看，WASM仍然没有一款专属于自己的全链编译器，当然，现在也不敢肯定这种需求是必然的，不过针对WASM开发出相关的适配的编译器仍然是必然的。如果不是这样，包括LLVM也不会开展这方面的开发工作。WASM的编译器开发还有很多路要走，举一个明显的例子，现在都在做内存的自动管理，包括c++这种静态语言都推出了相关的智能指针，那么WASM这方面是不是也要有推进的脚步。这就涉及到了几乎所有虚拟机头疼的GC问题。这个不是说光靠编译器就能解决的。

# 五、总结
在初步分析了WASM的编译器的应用技术后，可以看到一个很明显的特点，那就是WASM目前的编译技术还没有真正成熟起来，当然由于本身的标准制定都在进行中，所以这种想法也有一些苛求。
编译工具链还有些复杂，虽然有了一些在线IDE支持相关的编译调试，但是目前来看，还有很大的改进的空间。
