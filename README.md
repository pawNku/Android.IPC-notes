# Android.IPC-notes
Android之IPC的一些理解和整理。结合了入门的‘’第一行代码‘’和进阶的‘’Android开发艺术探索‘’两本书和一个实例Demo~

## 开发艺术探索之IPC机制
> IPC即Inter-Process Communication 含义很明显为进程间通信 也就是说两个进程直接交换数据的过程 说明几点：
> * 1.线程是最小的调度单元 是被包含于进程当中的 而进程是拥有资源的执行单元 一般意味在Android上就是一个app程序 而线程就是UI线程
> * 2.在任何的操作系统中都离不开跨进程通信 在Linux中的命名管道、共享内容和信号量等 在Android里有Binder和socket 另外还有CP做为底层的进程间通信 很明显提供了CURD一样的给别的程序 跨进程了
> * 3.应用场景最经常的就是 应用需要向其他应用获取数据  然后因为一些特殊的原因有些模块在单独的进程 这样就需要了

### 1. Android中的多进程模式
**1.1 开启多进程模式**
> 首先在Andorid中多进程指一个应用中存在多个进程的情况 所以不是两个应用之间的多进程情况   
> 然后使用多进程的方法也很唯一：only在Manifest中指定android:process 属性 这个属性就是进程名  假如一个应用里有MainActivity、SecondActivity、ThirdActivity三个活动 现在要为后两者增加两个新进程
* 情况1：` android：process = “：remote” `   
这样为secondActivity启动时 系统会为其假如一个进程名为“com.ryg.chapter_2:remote”的进程 这是当前应用的私有进程但是依旧不同于默认进程  其他应用的组件不可以和它跑在一个进程中 也可以发现“：” 的含义是指在当前的进程名前面附加上当前的包名
* 情况2：` android：process = “com.example.c2.remote” `    
这是一种完整完整完整的命名方法  是属于全局进程 也是一个新的进程 其他应用可以通过ShareUID方式来跑在同一进程
* 而MainActivity因为没有指定 所以默认进程 默认进程就是包名  就是com.ryg.chapter_2 所以一共三个进程 可以通过使用 adb shell ps 或 adb shell ps|grep 包名 来查看一个包中的具体进程信息 也就是ID：645、659、672 三个进程了     

**1.2 多线程模式的运行机制**
> 首先多进程只要一开启就会很多问题 比如静态变量讲道理是共享的比如在上面的SecondActivity 中有个变量你设为了2但是你因为给SecondActivity单独的一个进程了所以这里就失效了 即使你在MainActivity中修改了这个变量的值 重新打印SecondActivity也还是原来的值。
> * 具体的原因：因为Android是运行在Linux的基础上的 而其中为每个进程都会分配一个单独的虚拟机 不同的虚拟机都会在内存上分配不同的地址 这样导致了不同的虚拟机在访问同一个类的时候会产生很多副本 举栗子：比如上面在com.ryg.chapter_2和com.ryg.chapter_2:remote两个进程上会分别产生一个Manager类 但是这两个类是互不影响的 在SecondActivity中修改静态变量值 也只会在com.ryg.chapter_2:remote这个进程上产生影响 对MainActivity的么得影响 所以！！！只要通过内存共享数据 都会失败失败失败！
* 多进程会带来以下问题：   
> 1.静态成员和单例模式的完全失效  不解释哦    
> 2.线程同步机制也会完全失效 因为即使锁对象其实还是对象在多进程的情况下 已经不是同一个对象了还同步啥。。
> 3.SP的可靠性下降 因为底层是XML的键值对 并不支持多进程的并发 哪怕两个进程都不行的 有几率丢包
> 4.Application对象会多次的创建  因为说了呀是不同的虚拟机 所以每次启动也是启动一个应用的过程 换句话说不同进程拥有不同的虚拟机+Application+内存空间   
* 实现跨进程传递数据的方式  
> 1.Intent传递数据   
> 2.共享文件 不是即使的 和 SharedPreferences  稳定性不行哦      
> 3.最强最强最强的基于Binder的Messenger和AIDL    
> 4.Socket 还行

### 2. IPC基础概念介绍
> 主要介绍Serializable、Parcelable、Binder Serializable、Parcelable是用来给对象完成序列化的  序列化就是说给对象实现变成可以传输的状态 比如我们在使用Intent和Binder传输数据的时候就用的到  或者是那种对象持久化也就是存储了也需要对对象完成序列化

**2.1 Serializable接口**
> Serializable接口是Java提供的一个空接口序列化接口  为对象提供序列化和反序列化操作 只要一个类去实现并且去声明 serialVersionUID 即可实现序列化    
` private static final long serialVersionUID = 8711368828010083044L `  
* 其实这个UID也可以不声明 但是有个问题就是如果不声明 在反序列化的时候这个类内部有些CURD的变量改变了 系统会重新计算当前类的hash值并更新 serialVersionUID 这个时候当前类的 serialVersionUID 就和序列化数据中的serialVersionUID 不一致，导致反序列化失败，程序就出现crash就凉了   
* 序列化是针对对象的  所以静态成员变量就很high不受影响也不参加序列化过程因为属于类啊
* 通过重写writeObject和readObject方法就可以轻轻松松的实现序列化过程咯  如下如下：
```
 7 //序列化
 8  
 9 FileOutputStream fos = new FileOutputStream("object.out");
10  
11 ObjectOutputStream oos = new ObjectOutputStream(fos);
12  
13 User user1 = new User("xuliugen", "123456", "male");
14  
15 oos.writeObject(user1);
16  
17 oos.flush();
18  
19 oos.close();
20  
21 //反序列化
22  
23 FileInputStream fis = new FileInputStream("object.out");
24  
25 ObjectInputStream ois = new ObjectInputStream(fis);
26  
27 User user2 = (User) ois.readObject();
28  
29 System.out.println(user2.getUserName()+ " " +
30  
31 user2.getPassword() + " " + user2.getSex());
32  
33 //反序列化的输出结果为：xuliugen 123456 male
```
**2.2 Parcelable接口**
> Parcelable接口是在Android中的其实就是内部包装了可序列化的数据，也是个接口反正实现了也可以一手Intent和Binder 毕竟Android中  
* 序列化功能由 writeToParcel 方法完成,最终是通过 Parcel 的一系列writer方法来完成
```
 @Override
    public void writeToParcel(Parcel out, int flags) {
    out.writeInt(code);
    out.writeString(name);
    }
```   
* 反序列化功能由 CREATOR 来完成，其内部表明了如何创建序列化对象Parcel和数组，通过 Parcel 的一系列read方法来完成
```
public static final Creator<Book> CREATOR = new Creator<Book>() {
@Override
public Book createFromParcel(Parcel in) {
return new Book(in);
} 
@Override
public Book[] newArray(int size) {
return new Book[size];
}
};
protected Book(Parcel in) {
code = in.readInt();
name = in.readString();
}
```
**2.3 对比**
