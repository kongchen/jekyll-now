---
layout: post
title: '[Tanslation] THE SINGLETON PATTERN'
date: 2006-02-14 14:34:00.000000000 +08:00
categories:
- 技术
tags:
- Pattern
- 翻译
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  dsq_thread_id: '2986055540'
author:
  login: admin
  email: kongchen@gmail.com
  display_name: KONG
  first_name: ''
  last_name: ''
---
【**出处**】[THE SINGLETON PATTERN][0]

【**作者**】John Zukowski, president of JZ Ventures, Inc. ([http://www.jzventures.com][1]). 

---

在软件设计中，设计模式是解决一般问题的一种通用方式。具体来说就是把解决方法翻译成代码，并且使得代码能轻松应对在新问题出现时的不同的情况。  
关于设计模式的讨论源于Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides的《[_Design Patterns: Elements of Reusable Object-Oriented Software_][2]》一书。这四位被称作是"Gang of Four" 简称 "GoF"。在书中，Gof将各种各样的模式划分为三种主要的类别：creational模式, structural模式和 behavioral模式。

Creational模式描述了对象是如何创建的（或"instantiated" ps:字典对这个词的解释是：specifically defined object through the replacement of some variables with values，其实就是实例化的意思，呵呵。）

Structural模式对怎样连接(connect)和联合(combine)对象offer help。Behavioral模式描述了算法或通信机制。

常用的模式主要有Singleton模式－－Creational；Observer模式－－Behavoral；Facade－－Structural；本篇文章就严格按照java上下文来描述一下Singleton模式。

Singleton模式作为Creational类中最常用的一种，描述了确保一个类永远只有一个实例的技术。本质上，这项技术遵循下面的原则：不允许在类的外部创建该对象的实例。典型地来说，Singleton的类是直到需要时才被创建，以此来降低内存的需求。你可以通过多种不同的方式来实现这一原则。

如果你知道一个将要被创建的实例将会是一个子类，那么就构造其父类为abstrcat并且提供一个得到当前实例的方法。上述说法的一个例子就是AWT包中的Toolkit类。Toolkit类的构造函数是公有的：

public Toolkit()  

这个类有一个getDefaultToolkit()方法用来得到指定的子类------此种情况下，子类是平台指定的：

public static Toolkit getDefaultToolkit()  

在安有Sun Java runtime的Linux平台上，得到的子类是**sun.awt.X11.XToolkit**。但是你无需知道它，因为你只是通过它的通用虚父类Toolkit来获取并使用它。

Collator类是Singleton模式的另外一个例子，与上面的例子稍微有点不同。它提供了两个getInstance()方法。没有参数的getInstance()更具默认的场所得到Collator。你也可以将你的场所传给Collator的getInstance()方法。向Collator多次请求获取相同场所的实例，你得到的返回将是相同的Collator实例。Collator的构造函数是protected的。类似的这种限制类创建的方式遍布在J2SE标准库中。

就这一点来说，你可能认为限制访问类的构造函数就使该类成为Singleton。错！Calendar 类有一个反例。Calendar 类的构造函数是protected的，并且该类提供一个getInstance()方法来得到它的实例。但是，每次调用getInstance()方法都得到的是一个新的实例。所以它不是Singleton的。

当你要创建你自己的Singleton类时，确保永远只有单个实例：

public class MySingleton {  
private static final MySingleton INSTANCE =  new MySingleton();  

private MySingleton() {  
}  

public static final MySingleton getInstance() {  
return INSTANCE;  
}  
}  

静态的方法getInstance()，返回类的单个实例。注意到即使这单个实例需要成为一个子类，你也无需改变API

理论上来说，你无需getInstance()方法，因为你可以把INSTANCE变量声名称public的。但是，getInstance()方法为今后系统的改变提供了更好的灵活性。一个好的虚拟机实现必须内嵌对静态getInstance()方法的调用。

上述并非创建一个Singleton的全部，如果你需要创建一个序列化（Serializable）的Singleton类，那么你就必须提供一个readResolve方法：

/\*\*  
\* Ensure Singleton class  
\*/  
private Object readResolve() throws ObjectStreamException {  
return INSTANCE;  
}  

readResolve()方法构造好之后，反序列化结果就是一个（仅一个）对象------与通过调用getInstance()方法得到的是相同的对象。但如果你不提供readResolve()方法，每当你反序列化一个对象时，都将产生一个该对象的新实例。

Singeton模式在你知道你仅有一个资源并且需要共享访问该资源的语句时是相当有用的。在设计的时候为Singleton模式明确需要可以简化日后的开发。但是，有时你直到当执行出现问题导致你需要重构代码、稍晚使用模式时才知道你需要什么。比如说，你可能发现因为你的程序在不断重复的创建相同的一个类的实例to pass along state information而使你的系统性能降低。通过改造成Singleton模式，你可以避免重复创建相同的对象。这节省了系统重建对象的时间，同时也节省了垃圾回收器回收这些实例的时间。

简而言之，只要你确保你永远都只需要创建一个（仅一个）类的实例，那就用Singleton模式。如果你的构造函数不需要任何参数，那就提供一个空的私有的构造函数（或者一个protected，如果你想要有子类）。否则，系统会默认地提供一个公有的构造函数------这就不是Singleton了。

注意，Singleton模式只保证在一个类装载器中的唯一性。如果你在多分布的企业级容器中使用Singelton类，你将会在每个容器里得到一个实例。

Singleton模式经常与另外一种叫做Factory模式的模式一块使用。和Singleton模式一样，Factory模式也是Creational模式。它描述了特定对象的子类们，或者更典型地------特定接口的实现们来实现对象的创建。Factory模式的一个不错的例子是Swing BorderFactory类。这个类有一系列返回不同类型Border对象的静态方法。它隐藏了子类实现的细节，允许factory为接口的实现直接调用构造函数。下面是一个BorderFactory使用的例子:  

Border line =  BorderFactory.createLineBorder(Color.RED);  
JLabel label = new JLabel("Red Line");  
label.setBorder(line);  

这里，BorderFactory创建了一个LineBorder,至于BorderFactory是怎样做到的，是被开发者隐藏起来的。在这个特殊的例子里，你可以直接调用LineBorder的构造函数，但是在使用Factory模式的许多实例中，你不能这样。

要得到Singleton的工厂，你可以调用PopupFactory的getSharedInstance()方法:

PopupFactory factory = PopupFactory.getSharedInstance();

接着,你可以通过调用factory的getPopup()方法来创建一个Popup对象。需要传入父组件，内容，位置作为参数：

Popup popup = factory.getPopup(owner, contents, x, y);

你会发现Factory模式在安全的文章中被频繁使用。在下面的例子中，一个认证工厂存在于特别的算法中，得到了对流的认证：

FileInputStream fis = new FileInputStream(filename);  
CertificateFactory cf = CertificateFactory.getInstance("X.509");  
Collection c = cf.generateCertificates(fis);

就像刚才在BorderFactory说的那样，Factory模式不一定要和Singleton模式一块使用。但是这两种模式经常在一起被使用。

---

[0]: http://java.sun.com/developer/JDCTechTips/2006/tt0113.html#1
[1]: http://www.jzventures.com/
[2]: http://www.amazon.com/exec/obidos/ASIN/0201633612/