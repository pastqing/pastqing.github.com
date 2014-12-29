---
layout: post
title:  Java String类源码解析（一）
category: java
date: 2014-12-10

---

# JAVA String类源码解析（一）


## PASS.1 不可改变的String ?

###众所周知java中的String对象时不可改变的， 实际中我们经常做类似于**String a = "abc" + new String(b) ;** 的操作， 实际上都是创建了一个全新的String对象， 这个新的对象包含了修改后的字符串内容。 下面来看看**String**构造函数.

{% highlight java %}
    public final class String 
        implements java.io.Serializable, Comparable<String>, CharSequence {
            private final char value[];
         private int hash; // Default to 0
    }
{% endhighlight %}


<!-- more -->

###从上面的代码中，我们可以看出String的本质就是char[]， 然而这个字符数组被**final**关键字修饰， 那么问题就来了， **final**是个么？
> **补充**： **final**关键字的几个意义：
    1. 如果final修饰类的话， 那么这个类是不能被继承的。
    2. final修饰方法的话， 那么这个方法是不能被重写。
    3. final修饰变量的话， 那么这个变量在运行期间时不能被修改的。
    **这里应该重点理解第三点**

{% highlight java %}
public static void main(String[] args) {
		final StringBuffer a = new StringBuffer("松哥");
		final StringBuffer b = new StringBuffer("songgeb");
		System.out.println("未改变： " + a);
		System.out.println("a的地址： " + a.hashCode());
		a = b; //这里编译就会报错了～～
		a.append("帅") ;
		System.out.println("改变： " + a);
	}
	/*
	    通过上面的例子， 我们对第三点的理解应该是这样：
	    1. 若变量是基本类型， 则它的值是不能变的 
	    2. 若变量是对象， 则它的引用所指的堆栈区中的内存地址是不能变得， 而内容时可以变滴。
	*/
{% endhighlight %}

###以上也就解决了， 为啥String是不可改变的问题。 String里面的成员变量是私有的而且被**final**修饰， 因此对String本身是无法改变的， 只能重新new一个出来了。。。其实想改变还是有办法的， 只要想办法访问私有变量value就行， 至于怎么访问， 需要用的java中的**反射技术**了， 以后再谈

## PASS.2 警惕toString()无意识的递归 

###这里我完全引用一下thinking in java 中的例子：
{% highlight java %}
    /*
        java中的每个类从根本上都是继承自object， 因此容器类都有toString这个方法， 例如ArrayList, 调用它的toStirng方法会遍历其中包含的每一个对象， 调用每个对象的toString方法。 下面的例子是要打印每个对象的内存地址， 这时就会陷入无意识的递归。
     */
    public class demo_3 {
		public String toString() {
			return "address: " + this + "\n" ;
		}
	public static void main(String[] args) {
		// TODO Auto-generated method stub
		List<demo_3> a = new ArrayList<demo_3>();
		for( int i =0 ; i < 5; ++i){
			a.add(new demo_3());
		}
		System.out.println(a);
	}
}
{% endhighlight %}

###发生以上错误的原因就是在这句 “ + this ” 上， 编译器看到一个String对象后面跟着一个 +  再后面对象不是String， 于是他就会调用this的toString方法了， 然后就递归了。。。。怎么解决呢？是的用**super.toString()**

##PASS.3 codePoints 与 codeUnit
- codePoints是代码点， 表示的是例如'A', '王' 这种字符
- codeUnit是代码单元， 它根据编码不同而不同， 可以理解为是字符编码的基本单元， 比如一个char(8位1个字节）就是一个单元
我们知道， java中的char是两个字节， 也就是16位的。这样也反映了一个char只能表示从**u+0000～u+FFFF**范围的unicode字符, 在这个范围的字符也叫BMP（basic Multiligual Plane ）， 超出这个范围的叫增补字符。
看下面这个例子：
{% highlight java %}
    public class demo_4 {
	public static void main(String[] args) throws UnsupportedEncodingException {
		/*
		 * \u1D56属于增补字符范围
		 */
		String a = "\u1D56B";
		String b = "\uD875\uDD6B";	
		System.out.println(a);
		System.out.println(a.length());
		System.out.println(b);
		System.out.println(b.length());
		//也就是说length返回的是代码单元的数量
	
	}
}
{% endhighlight %}
下面是String其中的一个构造函数：
{% highlight java %}
     public String(int[] codePoints, int offset, int count) {
        if (offset < 0) {
            throw new StringIndexOutOfBoundsException(offset);
        }
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        // Note: offset or count might be near -1>>>1.
        if (offset > codePoints.length - count) {
            throw new StringIndexOutOfBoundsException(offset + count);
        }
        final int end = offset + count;
        // Pass 1: 计算字符数组的大小， 以分配空间
        int n = count;
        for (int i = offset; i < end; i++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))//这里是判断这个字符是不是属于bmp范围
                continue;
            else if (Character.isValidCodePoint(c))//判断是否越界
                n++;
            else throw new IllegalArgumentException(Integer.toString(c));
        }
        // Pass 2: 填充value
        final char[] v = new char[n];

        for (int i = offset, j = 0; i < end; i++, j++) {
            int c = codePoints[i];
            if (Character.isBmpCodePoint(c))
                v[j] = (char)c;
            else
                Character.toSurrogates(c, v, j++);
        }

        this.value = v;
    }
    //可以看出， 构造的思路很简单， 但是必须要考虑的很全面。 像以前写c++时， 要考虑一个类能不能被复制， 赋值， 等等情况。
{% endhighlight %}
##PASS.4 equals还是==？

###初学java者在比较字符串时， 常常容易混淆equals和==的不同。 下面我们讲一下两者的不同。首先来看个例子：
{% highlight java %}
public static void main(String[] args) {
        
        String x = new String("java");    //创建对象x，其值是java
        String y = new String("java");    //创建对象y，其值是java
        
        System.out.println(x == y);        // false, 使用关系相等比较符比较对象x和y
        System.out.println(x.equals(y));    // true, 使用对象的equals()方法比较对象x和y    
        
        String m = "java";    //创建对象m，其值是java
        String n = "java";    //创建对象n，其值是java
        
        System.out.println(m == n);        // true, 使用关系相等比较符比较对象m和n
        System.out.println(m.equals(n));    // true, 使用关对象的equals()方法比较对象m和n    
    }
}
{% endhighlight %}
###那么问题就来了， 相信大部分人第一个输出为flase能很明白， 但是第三个输出为true就有点晕了。 第三个输出为true是因为java常量池的原因， 类似于c++中的堆栈里的常量区。 我们先来看看equlas的源码， 再来解释常量池的东西。
{% highlight java %}
 public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) { // 判断是不是String的一个实例化
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
{% endhighlight %}
###看源码的好处就是， 能看到优雅的代码， 清晰的思路， 全面的考虑。比如里面的while语句。 以前读c++的源码的时候， 就会发现里面每个函数小算法都实现的很精美， 往往几行搞定， 效率很高。总结来说java String 的实现就是对字符数组的各种操作， 这样就可以写自己的String类了。

### 下面解释jvm对String的处理， 这里我参考了小学徒的成长历程的博文
> String a = "Hello World" ;

###如上，字符串a是常量， 同时String是不可改变的， 因此我们可以共享这个常量。 为了提高效率， 节省资源就有了常量池这个东西。常量池的一个作用就是存放**编译期间**生产的各种字面值常量和引用。同时根据jvm的垃圾回收机制吧， 在这个常量区中的对象基本不会被回收的。看下面的例子：
{% highlight java %}
    public static void main(String [] args ){ 
        String a = "Hello" ;
        String b = "Hello" + " World" ;
        String c = "Hello World" ;
    }
{% endhighlight %}
### 用javac 编译文件， 用javap -c 查看编译后的字节码， 如下： 
{% highlight java %}
    public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String Hello
       2: astore_1      
       3: ldc           #3                  // String Hello World
       5: astore_2      
       6: ldc           #3                  // String Hello World
       8: astore_3      
       9: return        
}

{% endhighlight %}
####**ldc**指的是将常量值从常量池中拿出并且压入栈中， 可以看出第3 、6行取出的是同一个常量值， 这说明了，**在编译期间，该字符串变量的值已经确定了下来，并且将该字符串值缓存在缓冲区中，同时让该变量指向该字符串值，后面如果有使用相同的字符串值，则继续指向同一个字符串值**

###不过， 一旦使用了变量或者调用了方法， 那就不一样了。 看下面的例子：
{% highlight java %}
      public static void main(String [] args ){ 
        String a = "Hello" ;
        String b = new String("Hello");
    }
{% endhighlight %}
用javap -c 返回的自己码为：
{% highlight java %}
public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String Hello
       2: astore_1      
       3: new           #3                  // class java/lang/String
       6: dup           
       7: ldc           #2                  // String Hello
       9: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
      12: astore_2     
      13: return        
}
{% endhighlight %}
###从上面可以看出， 从常量区中拿出来放到栈中， 再从栈中拿出来， 然后调用String的一个构造函数， 通过关键字new进行创建对象， 然后将新的引用赋给b。 从这也能看出来用这种构造函数初始化一个字符串， 效率是不高的， 我们尽量少用。



