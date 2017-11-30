
# ClassifyDemo


1.0

区分手机号得到相应手机号是否存在微信用户，用户是否状态异常，还有获取到微信id和地区，性别及昵称。

输入：手机号清单文本

输出：用户存在，用户状态异常，用户不存在，用户信息4个文本。

1.2
   
新增hook类。这个是利用Xposed的框架的实现的，要hoonk的是微信的方法，得到搜索时的界面信息，信息放在bundle里面，从bundle中取出我们想要的数据。对了，需求变了，现在要获取多几个数据，用户的个性签名，v1值，v2值。

然后就是将hook到的信息传输到我们的MAinActivity中，再将数据操作写入到文件中去。然后这里就是个大坑，本来以为是线程间的通信，通过同步锁来实现同步，实现数据的传输。结果发现这其实是两个进程，所以就得用进程间的通信，这个就麻烦了。

因为进程间的通信，所以同步锁就失效了，而且变量不能直接传输。所以就得用进程间的通信了，开始时考虑直接用共享文件来做，但是文件的同步很容易出问题，Hook方法所在进程创建文件，把数据储存进临时文件，同时把这个文件拿来当成是进程间的同步标识，但是Activity所在进程拿数据时，可能文件还没写入完成或者说写了一半，此时那边的拿就很有问题了。咦，突然想到，用两个文件，hook的逻辑 1、先删除标识文件	2、将数据写入储存的临时文件。  3、创建标识文件

而Activity那边的逻辑。 1、do  while 判断标识文件是否存在  2、 将储存数据的临时文件的数据读取出来，写入本地。

这样应该是没有问题的。


项目中不是用文件的方式了，是用ContentProvider的方式，因为ContentProvider是一个数据暴露的接口，可以用于进程间的通信，而SharedPreference的Multi Process的模式被抛弃了，然而ContentProvider还有一个问题，它不能储存数据的，网上很多都是将数据放入数据库中，但是我不想用数据库，所以用了SharedPreference的方式储存数据。此时需要处理的就是SharedPreference的是ContentValues数据，而ContentProvider却是操作uri。所以在其查询方法中应该自定义Cursor。利用get/set Extras的方式操作bundle来实现数据的操作。这里有个需要注意的是因为操作SharedPreference需要context对象，但是获取Activity / Service  / ContentProvider的context都没有用。要获取当前hook所在的进程的context。 这里获取的方式也是通过hook方法来获取的。


最后本来以为搞定了，结果在远程机器上跑的时候，出现了一大堆问题。zzz，网络跟cpu都下降了，一开始出现了找不到添加按钮，所以进不去判断是那种情形的方法，将do while 获取添加按钮的时间调整为20s。而且每次都重新获取根节点。然后将判断是哪种情形的逻辑并行处理，不用设为串行，这样就不会浪费时间了，然后在获取节点时可能会出现节点为null的情况，此时在函数中添加判断节点不为null即可。


1.3 

改进几个bug

1、昵称含特殊符号时换行了

2、个性签名含特殊符号时换行了

3、有的地区没有获取到

新增几个功能

1、记录每次翻译的总个数

2、记录当前翻译的个数

3、记录翻译的开始和结束时间

4、自定义异常类，实现超时（例如网络引起）、等待辅助功能类、等待hook函数超时的异常获取。在主界面中显示异常的类型，还有停止当前程序。


1.4 

1、新增将数据存储在数据库的功能（为了后面的跟服务端交互）

2、新增toast打印当前翻译进度

3、新增停止按钮

4、添加保存进度的功能，可以把前面没做完的任务继续执行下去


2.0

改革系统的设计了。

直接从后台获取待分类的手机号，首先将数据插入数据库，然后在开始分类，分类完成后再更新数据状态，最后将数据一起上传到服务器。


其中有很多细节需要处理，比如一开始就是先判断数据库有没有还没做完的任务，如若有就先翻译再上传，然后还有判断数据库有没有没上传的数据，有的话也是自动处理；上传过程是while循环的，便于异常的处理。

整个逻辑过程都需要异常处理，然后在主界面中显示，还有做到网络中断后的处理，还有模拟器被关闭的处理。

实现开机自启动，让其自己不断地做任务。


2.1

1、当app获取到异常信息时，利用sharedPrefrences来存储数据，点击app就可以在ui界面中再次从sharedPrefrences中拿出来复原

2、辅助功能类获取超时的处理，利用while循环来保证可靠性，特别的输入手机号后按两下回车的时候

3、动态获取userid，这个是用于与后台交互时需要的函数参数

4、整理整体逻辑，下次进来时会把未翻译的翻译完成后，未提交到服务器的先提交


2.2    

1、保证手机号码的填入

2、缩短每次任务失败到重新启动的时间间隔

3、修改等待hook类的代码逻辑

4、新增每次获取根节点去操作前都会判断是否为空

