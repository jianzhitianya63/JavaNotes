## 深入理解Java虚拟机 
*** 
### 第二章 java内存区域与内存溢出异常  
#### 2.2 运行时数据区域
- 程序计数器  
为了线程切换后能恢复到正确位置,每条线程都有一个独立的程序计数器,各线程之间计数器互不影响,独立储存,这类内存区域为"线程私有内存".
这是唯一一个在java虚拟机规范中没有规定任何OutOfMemoryError情况的区域  
- java虚拟机栈  
线程私有, 生命周期与线程相同.  
栈中每个方法对应一个栈帧,用于存储局部变量表,操作数栈,方法出口等信息,方法从调用到完成过程,就对应着一个栈帧在虚拟机栈中入栈到出栈的过程  
如果线程请求的栈深度大于虚拟机所允许的深度,将抛出StackOverflowError异常;  
如果虚拟机栈中可以动态扩展,进行扩展时无法申请到足够的内存,就会抛出OutOfMemoryError异常  
- 本地方法栈  
与虚拟机栈类似,本地方法栈为虚拟机使用到的Native方法服务  
***
####2.4 HotSpot虚拟机对象探秘
- java堆  
是垃圾收集器管理的主要区域,用于存放对象实例  
如果在堆中没有内存完成实例分配,并且堆也无法再扩展时,将会抛出OutOfMemoryError异常  
- 方法区
各线程共享区域,存储类信息、常量、静态变量等数据  
这区域的内存回收目标主要是针对常量池的回收和对类型的卸载,这个区域回收比较困难,尤其是类型的卸载.
当方法区无法满足内存分配的需求,将抛出OutOfMemoryError异常  
- 运行时常量池  
方法去的一部分, 存放编译期生成的各种字面量和符号的引用  
当常量池无法再申请到内存时会抛出OutOfMemoryError异常  
- 直接内存  
不属于虚拟机规范中定义的内存区域  
在根据实际内存设置-Xmx等参数信息,经常会忽略直接内存,使得各个区域内存总和大于物理内存限制(包括物理和操作系统级的限制),从而导致动态扩展时出现OutOfMemoryError异常  
- 对象的创建  
1.检查符号引用代码的类是否已经被加载,解析和初始化过.没有则执行类加载过程  
2.为新生对象分配内存,并且分配到的内存空间都将初始化为零值  
3.对对象进行必要设置,例如设置对象头(元数据,哈希码,对象GC分代年龄等)  
4.按照程序员意向执行对对象的初始化
- 对象的内存布局  
1.对象头  
存储对象对身的运行时数据  
2.实例数据  
对象存储的真正存储有效信息  
3.对其补充  
JVM要求对象的大小必须是8字节的整倍数  
当对象实例数据部分没有对齐时,需求通过对齐补充来补全  
- 对象的访问定位  
1.句柄访问  
效率相对2低
2.指针访问方式  
效率高  
***
####实战: OutOfMemoryError异常  
- Java堆溢出  
参数
>AM Args: -Xms20M -Xmx20M -XX:+HeapDumpOnOutOfMemoryError
-Xms 最小堆内存 -Xmx 最大堆内存 -XX:+HeapDumpOnOutOfMemoryError 出现内存溢出Dump当前内存转储快照,位于项目根目录
<pre>
  ArrayList<OOMObject> list = new ArrayList<>();
        while (true){
            list.add(new OOMObject());
   		}
</pre>  
抛出:OutOfMemoryError:java heap spece
解决: 1. 先搞清楚是内存泄漏还是内存溢出  
1.1 如果是内存泄漏,可通过工具进行查看例如jdk自带工具VisualVM,将生成堆dump导入查看  
1.2 如果不存在内存泄漏,就是内存中这些对象必须活着,可以检查虚拟机的堆参数 -Xms与-Xms是否还可以调大,从代码上检查是否有些对象生命期过长等  

- 虚拟机栈和本地方法栈溢出  
1.如果线程请求深度大于虚拟机所允许的最大深度,将抛出StackOverflowError异常  
<pre>
	//-Xss128k 设置栈容量
	private int stackLength = 1;
	public void stackLeak(){
		stackLength ++:
		stackLeank;//无限递归
	}
</pre>  
抛出:StackOverFlowError异常    
在单个线程下无论是栈帧太小或者虚拟机栈容量小,当内存无法分配的时候,都会抛出此异常  
  
2.如果虚拟机栈在扩展时无法申请到足够的内存,抛出OutOfMemoryError:unable to create new native thread异常    
如果要测此异常,可以通过不断的建立线程方式产生内存溢出.  
原因:每个线程分配到的栈容量越大,可以建立的线程数量自然就越少,建立线程时就容易把剩下的线程耗尽  
如果是在不能减少线程数量或者更换64位虚拟机的情况下,就只能通过减少最大堆和减少栈容量来换取更多的线程  

- 方法区和运行时常量池溢出  
常量池溢出
<pre>
	//-XX:PermSize=10M 设置大小 -XX:MaxPermSize=10设置最大容量
	List<String> list = new ArrayList<String>();
	int i = 0;
	while(true){
		list.add(String.valueOf(i++).intern)
	}
</pre>
抛出:OutOfMemoryError:PermGen space  

方法区溢出  
要测试方法区可使用CGlib和反射等创建大量类.一个类要被垃圾收集器回收判定条件是非常苛刻的.需要特别注意  

- 本机直接内存溢出  
可通过-XX: MaxDirectMemorySize指定,如果不指定.则默认与Java最大值(-Xms指定一样)  
<pre>
 private static final int_1MB = 1024 * 1024;

 public static void main(String[] args) {
	Field unsafeField = Unsafe.class.getDeclaredFields.get()[0];
	unsafeField.setAccessible(true);
	unsafe unsafe = (Unsafe) unsafeField.get(null);
	while(true)}{
		unsafe.allocateMemory(_1MB);
	}
 }
</pre>
抛出: OutOfMemoryError  
由DirectMemory导致的内存溢出,一个明显的特征是在Heap dump文件中不会看见明显的异常,如果发现OOM之后Dump文件很小,而程序又直接或间接的使用NIO那就可以考虑检查是不是这方面的原因