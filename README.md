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
> 1.平台的不同 S是java自带的序列化接口 而P是Android的序列化接口   
2.原理的不同  S将一个对象变成可传输的状态 而P是将其分解 然后分成序列化对象Parcel和数组 每部分都支持可传递的数据
3.S是简单但是效率低代价大 因为ObjectInputStream过程需要大量的IO操作 但是P也麻烦 但是高效    
4.S适合存储到设备sd卡 或者序列化通过网络存储 而P是主要用在内存的序列化上    

### 3. Binder
> Binder是Android中的一个类，实现了 IBinder 接口。1.跨进程通信方式 2.各种Manager和ManagerService的桥梁 3.CS的桥梁 也就是AIDL类似于了   
1.从IPC角度说，Binder是Andoird的一种跨进程通讯方式。    
2.从Android Framework角度来说，Binder是 ServiceManager 连接各种Manager(ActivityManager·、WindowManager)和相应ManagerService的桥梁       
3.从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService时，服务端返回一个包含服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务器端提供的服务或者数据（ 包括普通服务和基于AIDL的服务）     

**3.1 具体解释**     
![图片](http://gityuan.com/images/binder/prepare/IPC-Binder.jpg)    
* 如图所示
> 1.在Android Framework也就是框架层 可以看见是连接各种SM的桥梁    
  2.在Android应用层  可以看见是连接CS的桥梁   
* 具体解释   
> 1.首先组件方面 包括：Client、Server的CS家走 其次是SM各种用来在框架层连接AM的 最后还要个binder驱动就是刚刚连接用的桥梁 SM是用来提供服务的   
  2.虚线：因为C/S和SM其实明显都不是直接交互的 要不然要Binder驱动干嘛    
  3.层次：C/S、SM是在用户控件的也就是应用层和框架层的  而Binder驱动 却是内核控件     
  4.开发人员只需要自定义client、server端 借助上面安卓的自带的SM和Binder也就是Android平台层就可以进行IPC通信了    
  
**3.2 由系统根据AIDL文件自动生成.java文件**    
* Book.java:定义了图书信息的实体类 并且实现Parcelable接口    
* Book.aidl：Bookai AIDL中的声明
* IBookManager.aidl：！用户自定义的一个管理管理管理Book类的一个接口  里面一般都是空的接口方法 但是依旧要导入Book类    
* IBookManager.java：！自动生成的生成的生成的  系统为我们哈哈是个.java文件 这就是aidl的好处    
> IBookManager.java继承了 IInterface 接口  所以的在Binder中传输的接口这里就是管理类都得继承IInterface 接口   
  1.不仅声明了getBookList 和 addBook 方法，还声明了两个整型id分别标识这两个方法 用于在onTransact中标识客户端到底请求哪个方法   
  2.声明了一个Stub类就是Binder类 当CS在同一进程 则不会走跨进程的 transact 不同进程 则走跨进程的 transact 逻辑由里面的Proxy代理类来完成   
  3.接口的核心实现 由内部的Stub和Proxy实现     
  
**3.3 Stub和Proxy类的内部方法和定义**    
![图片](http://upload-images.jianshu.io/upload_images/1944615-3c92d9d160957e78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)      

```
ServiceConnection conn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        IMyAidlInterface imai = IMyAidlInterface.Stub.asInterface(iBinder); 
        /**
         * 拿到AIDL就可以跨进程调用代理的自己定义的方法了.showProgress()
         */
        imai.showProgress();
```    
* asInterface：在客户端最后解析数据的！ 也就是Bindert-->Clien 通过Binder对象传入并且转换为客户端所需的AIDL接口类型的对象 拿到这个aidl就可以也就是返回的Stub对象本身 也就是接口可以调用自己的方法了showProgress()  真正的实现在ServiceDemo中   
* onTransact：用来回头的！ 也就是Servver-->Binder这个方法运行在服务端的Binder线程池中，由客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。该方法的原型是 `public Boolean onTransact(int code,Parcelable data,Parcelable reply,int flags)` 
> 可以发现首先通过code来确定客户端调用了哪个方法 从data中取数据执行方法 ok后通过reply写入返回值再通过asBinder返回Binder对象也就是各种Stub哦 这样就完成了回头了  最后的flag是boolean看看是不是请求失败了哦   
* ！！！整体来回：首先在客户端创建各种Pacel序列化输入输出的对象然后把这些参数信息写进data  然后通过data读取要执行哪个方法 通过onTransact在服务端中会调用方法 最后在线程池中写入结果到reply中 最后通过asBinder返回成Binder对象 最后在客服端的asInterface把这个Binder传递进入转换成客户端用的aidl接口获取数据    
* ps：AIDL文件不是必须的，之所以提供AIDL文件，是为了方便系统为我们生成IBookManager.java，但我们完全可以自己写一个但是我就是不写哎~
     
### 4. Android中的IPC方式   
> 主要有以下方式：    
Intent中附加extras-Bundle   
共享文件   
Binder   
ContentProvider   
Socket   

**4.1 使用Bundle--Intent中**   
> 四大组件中的三大组件（ Activity、Service、Receiver） 都支持在Intent中传递 Bundle 数据   
Bundle实现了Parcelable接口，因此可以方便的在不同进程间传输。当我们在一个进程中启动了另一个进程的Activity、Service、Receiver，可以再Bundle中附加我们需要传输给远程进程的消息并通过Intent发送出去。被传输的数据必须能够被序列化   

**4.2 使用文件共享--低并发**   
> 有点神类似S我们可以序列化一个对象到文件系统中的同时从另一个进程中恢复这个对象   
* 通过 ObjectOutputStream / ObjectInputStream 序列化一个对象到文件中，或者在另一个进程从文件中反序列这个对象。注意：反序列化得到的对象只是内容上和序列化之前的对象一样，本质是两个对象。其中是IS还是OS是针对内存来说的哦   读到sd卡的话通过小车byte[]数组按len来搞   
* 问题所在 因为不是及时的 所以大量的并发不适合对数据同步的处理凉凉    
* SP：是xml的键值对 存在大概5M的缓存机制  所以内存中的缓存也导致了高并发凉凉    
* 所以IPC基本上都不能文件共享 除非低并发    

**4.3 使用Messenger--Handler串行**   
> 是一种轻量级的IPC方案底层反手一个AIDL 进行了封装 可以翻译成信使不同进程间传递Message对象   因为是一次处理一次请求的 所以不用考虑线程同步的问题 也因为服务端不存在并发执行的情形   如下像AIDL：因为Imessenger.Stub.asInterface(IBinder)这个就很像传递进Binder最后返回的是客户端用的接口 其实也是个Binder mTarget
```
public Messenger(Handler target){
    mTarget = target.getImessenger();  //都是Binder这里的mTarget 通过Handler
}
public Messenger(IBinder target){
    mTarget = Imessenger.Stub.asInterface(target);   //通过Binder返回Binder 极像AIDL
}
```
![图片](https://img-blog.csdn.net/20160828161207521)        
* 具体使用时，分为服务端和客户端两大流程：    
> 1.服务端：创建一个Service来处理客户端请求，同时创建一个Handler并通过它来创建一个Messenger，`private final Messenger mMessenger = new Messenger (new xxxHandler());` 类似于上面的构造方法一  然后再Service的onBind中通过Messenger.getBinder返回Messenger对象底层的Binder即可     
```
//示例服务端代码  1.Handler 2.Messenger 3.Binder
public class myService extends Service {
    private static class myHandler extends Handler{
        public void handleMessage(Message msg){}   //用来处理客户端消息的
    }
    private final Messenger mMessenger = new Messenger (new xxxHandler());   //与myHandler相关联没有Handler就没有mMessenger
    
    public IBinder onBind(Intent intent){
       return mMessenger.getBinder();     //与mMessenger相关联没有mMessenger就没有IBinder对象用来最终返回给客户端的
    }
}
```
> 2.客户端：绑定服务端的Sevice==IBinder对象，利用服务端返回的IBinder对象来创建一个Messenger，通过这个Messenger就可以向服务端发送消息了，消息类型是 Message    
> 如果需要服务端响应，则需要创建一个Handler并在内部来创建一个Messenger（ 和服务端一样） ，并通过 Message 的 replyTo 参数传递给服务端 服务端通过Message的 replyTo 参数就可以回应客户端了
```
//1.简单的不需要响应 iBinder就是服务端返回的Service绑定
public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
    mMessenger = new Messenger(service);   //1.1 利用服务端返回的IBinder对象来创建一个Messenger
    Message msg = Message.obtain();    //1.2创建一个传递的消息类型msg 都是事先了Parcelable接口的
    Bundle data...data.putString...msg.setData    //Bundle支持大量的数据类型
    mMessenger.send(msg);    //1.3通过msg来给服务端发消息
}

//2.服务端不仅接受 还要提供响应
//2.1服务端的修改
public class myService extends Service {
    private static class myHandler extends Handler{
        public void handleMessage(Message msg){
           Messenger Client = msg.replyTo;   //服务端中通过 message.replyTo 来获得对方的Messenger
           Message msg = Message.obtain();
           Bundle data...data.putString...msg.setData 
           Client.send(msg); 
        }   
    }
    ...
}
```

* 总而言之，就是客户端和服务端 拿到对方的Messenger来发送 Message .send(msg);    
* 只不过客户端通过bindService==也就是`mMessenger = new Messenger(service); 此处的service就是Binder对象`    
* 而服务端通过 message.replyTo 来获得对方的Messenger==Messenger Client = msg.replyTo;    
* 最后：Messenger中有一个 Hanlder 以串行的方式处理队列中的消息。不存在并发执行，因此我们不用考虑线程同步的问题。   

**4.4 使用AIDL--高并发跨进程**   
> 首先与Messenger不同的是 因为也说过了 M是Handler串行的所以有大量的消息到服务端就凉了 其实M的主要作用就是传递消息   
* 1.服务端：服务端需要创建Service来监听客户端请求，然后创建一个AIDL文件一定一定一定是在服务端中创建的AIDL文件，将暴露给客户端的接口也就是需要提供的方法在AIDL文件中声明，最后在Service中实现这个AIDL接口即可  也就是实现具体方法的逻辑   
* 2.客户端：首先绑定服务端的Service，绑定成功后，将服务端返回的Binder对象通过asInterface转成AIDL接口所属的类型，接着就可以通过这个Binder调用AIDL中的方法了 
* AIDL文件    
```
// IMyAidlInterface.aidl
package com.example.servicedemo;

// Declare any non-default types here with import statements

interface IMyAidlInterface {
    /**
     * 在服务器方创建自定义AIDL 接口
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);

     //定义自己所需要的方法：显示当前服务的进度
     /*
     * 仅将方法定义出来 在接口中不需要实现
     */
     void showProgress();
}
```    
> AIDL方法是在服务端的Binder线程池中执行的，因此当多个客户端同时连接时，管理数据的集合直接采用 CopyOnWriteArrayList 来进行自动线程同步  类似的还有 ConcurrentHashMap 

* 远程服务端Service提供的方法的具体实现   
```
    //绑定
    //IBinder：在android中用于远程操作对象的一个基本接口
    @Override
    public IBinder onBind(Intent intent) {
        // TODO: Return the communication channel to the service.
        Log.e("TAG","服务绑定了");
        //Binder
        /**
         * 通过服务端方的AIDL接口的.Stub来实现具体的逻辑方法
         */
        return new IMyAidlInterface.Stub(){
            @Override
            public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, String aString) throws RemoteException {
            }

            @Override
            public void showProgress() throws RemoteException {
                Log.e("TAG","当前进度是" + i);
            }
        };
    }

    //对于onBind方法而言，要求返回IBinder对象
    //实际上，我们会自己定义一个内部类，集成Binder类

    class MyBinder extends Binder{
        //定义自己需要的方法（实现进度监控）
        public int getProcess(){
            return i;
        }
    }

    //解绑
    @Override
    public boolean onUnbind(Intent intent) {
        Log.e("TAG","服务解绑了");
        return super.onUnbind(intent);
    }

    //摧毁
    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.e("TAG","服务销毁了");
    }
}
```    

* 客户端调用服务的具体实现     
> 绑定远程服务 绑定成功后将Binder对象通过IMyAidlInterface.Stub.asInterface传入转换成aidl接口 就可以调用了
```
    ServiceConnection conn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            /**
             * 1.调用方转移通过AIDL来建立联系从解释权方到调用方中的代理类中的Stub
             * 2.在调用方也有onServiceConnected方法
             * 3.通过自定义的IMyAidlInterface接口.Stub也就是通信类.asInterface可以传递进
             * ServiceDemo中的MyService.java中onBind中通过Stub代理类返回的IBinder可以传递进去
             * 4.返回的是接口对象
             * 5.进而调用的是代理类Stub的showProgress方法 实现咯 only代理 完成是在ServiceDemo中的
             */
            IMyAidlInterface imai = IMyAidlInterface.Stub.asInterface(iBinder);
            try {
                /**
                 * 拿到AIDL就可以跨进程调用代理的自己定义的方法了.showProgress()
                 */
                imai.showProgress();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
```

**4.5 使用ContentProvider**      
> ContentProvider是四大组件之一，天生就是用来进程间通信。和Messenger一样，其底层实现是用Binder。

* 系统预置了许多ContentProvider，比如通讯录、日程表等。要RPC访问这些信息，只需要通过ContentResolver的query、update、insert和delete方法即可。

* 创建自定义的ContentProvider，只需继承ContentProvider类并实现 onCreate 、 query 、 update 、 insert 、 getType 六个抽象方法即可。getType用来返回一个Uri请求所对应的MIME类型，剩下四个方法对应于CRUD操作。这六个方法都运行在ContentProvider进程中，除了 onCreate 由系统回调并运行在主线程里，其他五个方法都由外界调用并运行在Binder线程池中。

* ContentProvider是通过Uri来区分外界要访问的数据集合，例如外界访问ContentProvider中的表，我们需要为它们定义单独的Uri和Uri_Code。根据Uri_Code，我们就知道要访问哪个表了。

* query、update、insert、delete四大方法存在多线程并发访问，因此方法内部要做好线程同步。若采用SQLite并且只有一个SQLiteDatabase，SQLiteDatabase内部已经做了同步处理。若是多个SQLiteDatabase或是采用List作为底层数据集，就必须做线程同步。

