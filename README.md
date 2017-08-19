# Chat
一个社交APP
# 概述
整体实现是按照传统的Android+Servlet+Json+MySql方案进行的，实时聊天借助Socket在服务端建立队列连接并通信，动态的请求和发送借助Servlet+Json实现，Android方面使用了Okhttp、Glide、circleimageview一些常用的框架，具体实现将在下面说明。
# 数据库
数据库的设计包含用户信息、用户关系、动态、评论、消息、二次评论和赞，具体直接上图<br>
![](https://github.com/onceabu/chat/raw/master/picture/database.jpg)<br>
图中同种颜色为主外键关系，粗线框为主键，细线框为外键，其他细节这里不再过多描述。要说明的一点就是图片的存储在数据库中只是索引字符，发布动态后服务端将照片读取并存储到文件夹中，Android端从数据库获取索引后借助Glide加载图片。
# 服务端
服务端Tomcat+MyEclipse，具体的数据获取和发送以及逻辑处理这里不过多描述，主要说明一下实时聊天的实现，在服务启动时开启一条子线程，该线程主要负责通信队列的建立和管理，在Android端登录时会与服务端建立Socket连接并以ID为索引加入队列中。
```
...
mSS=new ServerSocket(30000);
while(true){
	Socket socket=mSS.accept();
	bufferedReader=new BufferedReader(new InputStreamReader(s.getInputStream(),"utf-8"));
	try{
		String ID=bufferedReader.readLine();
		if(ID!=null){
			socketList.put(ID,socket);
			new Thread(new ServerThread(socket)).start();
		}
	}catch(IOException e){
		e.printStackTrace();
	}
}
...
```
ServerThread主要负责对消息的处理，当用户A向用户B发送一条消息，ServerThread会接收相关的ID以及消息内容，然后在通信队列中查找用户B的Socket连接，如果存在直接向其发送消息，如果不存在说明用户B不在线，此时存储消息等待用户B上线后再次获取，当用户退出应用时Socket连接也会断开退出通信队列。
# Android端
功能展示只有截图和gif，但截图只能展示页面布局，gif受大小限制也只是一些主要的东西，整体详细情况可通过录屏video获知，这里不做上传。
## 一、整体结构
应用主要包括登录与注册、好友动态、寻找好友、个人信息和好友聊天五个模块，具体结构直接上图。
![](https://github.com/onceabu/chat/raw/master/picture/dxc.png)
在第一次进入应用时选择登录或注册，在成功登录后进入主页面，主页面构成为ViewPager+Fragment+自定义选项菜单。自定义选项菜单以线性布局结合相对布局，与ViewPager联动，在主页面初始化时与服务端连接从数据库获取未读消息或者好友请求等信息数，信息数在自定义选项菜单中显示，Fragment包含好友动态、寻找好友、个人信息和好友聊天，整体预览图如下。
![](https://github.com/onceabu/chat/raw/master/picture/main.gif)
## 二、登录与注册
* UI<br>
登录界面布局方面以线性布局结合相对布局，邮箱输入框基于AutoCompleteTextView的自定义控件，通过监听自动完成邮箱地址的拓展，左侧CompoundDrawables根据焦点高亮显示，右侧CompoundDrawables根据Touch事件的坐标判断进行内容的删除。
注册界面以帧布局为主，邮箱和密码的输入完成后对邮箱是否重复和密码格式的准确性进行判断，符合后隐藏邮箱密码输入页面，显示个人信息输入页面
![](https://github.com/onceabu/chat/raw/master/picture/lasa.png)<br>
* 功能实现<br>
注册中对邮箱、密码、手机号等不同格式的信息输入通过正则表达式进行判断，同时也可以通过图片选择器上传头像和个人主页背景，关于图片选择器的具体实现会在好友动态中详细说明。注册过程中字符和图片数据通过OkHttp发送至服务端并存储到mysql用户表中，同样图片文件存储到文件夹中，数据库只保存图片的字符索引。注册成功后借助sharePreference将用户的相关信息保存在本地，便于以后登录的直接使用。
## 三、好友动态
* UI<br>
好友动态中的UI包括基于RecyclerView实现的动态列表、基于旋转动画的圆形菜单、基于Loader+Glide+RecyclerView的图片选择器以及自定义的动态详情页面。<br>
![](https://github.com/onceabu/chat/raw/master/picture/CircleMenu.gif)<br>
![](https://github.com/onceabu/chat/raw/master/picture/picSelector.gif)
![](https://github.com/onceabu/chat/raw/master/picture/comment.gif)<br>
* 功能实现<br>
1、圆形菜单最初的想法是通过对屏幕上的各种高度宽度的计算得到圆心位置，然后通过半径和三角函数计算圆的轨迹，但多次尝试后发现效果不佳，然后考虑到RotationAnimation，起初测试发现效果不错，计算出正确的圆心后Button便可以进行任意角度的圆形轨迹旋转，但测试时Button上为纯色没有添加图标，当添加图标后发现Button在绕圆心旋转的同时也在自转，进一步思考后将Button改为一个轨迹点，这样点的轨迹就是一个单纯的圆形，然后Button只需要在轨迹点转动时随之转动即可。<br>
参考项目：无<br>
2、图片选择器主要借助Loader+Glide+RecyclerView，所以与这三方面如何使用的细节不做过多描述，主要说明一下选择的顺序逻辑实现，在单独页面中时不管如何切换图片文件夹和变换图片选择顺序对顺序的记录都没有太多的难度，关键问题在于发送动态页面和图片选择页面之间来回传递，因为两个页面都可以对相关信息进行改变，而且为了保证图片选择页面中数标的正确显示需要多个序列记录相关信息，这样如果想要借助Intent进行信息的传递不仅增加了代码上的复杂性和出错的概率，而且会使得程序的耦合性增高。综合考虑后决定使用Application，Application优点就是全局共享，效率高，不必考虑Activity之间的传递，而且这些序列信息本身都是一些临时数据不需要持久化，Application完全符合这些要求，也降低了实现的复杂程度，唯一要说明的是使用Application可能会占用更多的内存空间，当内存紧张进行回收时可能会造成数据的丢失，所以只有在处理像这样需要频繁修改并且占用空间小具有实时性的数据时才适合使用Application。<br>
参考项目：无<br>
3、自定义动态详情页面的实现参考了很多类似的项目，但是很多项目多多少少都有一些滑动上的小bug，或者不能完全达到需要的效果，同时也考虑到使用谷歌官方提供的解决方案CoordinatorLayout+AppBarLayout但使用这一方案后有些效果就会受到影响，比如Toolbar布局的自定义和响应都会受到一定的限制，后来基于github一个stickyViewpager的项目进行修改和拓展逐渐实现了需要的效果，项目的主要的设计理念是借助帧布局把需要展示的主内容（ScrollerView/ListView/RecyclerView）和Header以及下拉刷新页面按照显示逻辑叠加在一起，然后对主内容加入滑动监听，通过一系列的高度计算和逻辑判断实现Header与主内容的联动和悬停。同样下拉刷新的效果是借助Activity的dispatchOnTouchEvent事件，从整个Touch事件的开始位置进行监听并进行各个方面的处理，包括相关组件的布局属性和自身属性的修改，对滑动事件的多种判断，刷新组件的动画监听等等方面，从而实现下拉刷新的效果。<br>
参考项目：https://github.com/xmuSistone/stickyViewpager<br>
4、关于数据的逻辑处理主要还是借助OkHttp+服务端的HttpServlet+Json，不管是动态的发布还是动态的获取都是以Json的格式进行传递，图片数据借助OkHttp上传后存储到服务端指定文件夹中，逻辑上的处理完全交由服务端处理，处理后的数据保存到Mysql中。<br>
5、这里还需要说明的一点是动态中图片的适配，当图片超过一张时直接按照九宫格的形式，通过屏幕宽度和各个组件之间的间距计算得到九宫格图片的大小，这个比较统一，跟图片本身关系不大。当只有一张图片的时候为了得到合适的显示效果就必须先得到图片的相关信息，至少要有图片的长和宽，因为加载图片需要一个过程，等待的这个过程由适配图负责，所以没有图片的长宽信息适配图的存在也就没有意义，动态的加载过程看起来也会很难受，也就是说图片的相关信息要在加载图片之前获取。所以这里涉及到的问题就是不论通过Glide还是别的形式加载图片都必须先获得图片然后才能知道它的尺寸大小，除非图片源提供了尺寸等相关信息的接口，所以这里只能在动态发布的时候将图片的相关信息一同上传至数据库中，然后在加载图片时先获取其相关信息，然后对其大小比例做适配，最后按照适配好的尺寸加载图片。
## 四、寻找好友
* UI<br>
好友列表项同样基于RecyclerView，RecyclerView的Adapter贯穿了整个项目，侧边导航栏参考了github上的WaveSideBar<br>
![](https://github.com/onceabu/chat/raw/master/picture/searcha.gif)<br>
![](https://github.com/onceabu/chat/raw/master/picture/searchb.gif)
![](https://github.com/onceabu/chat/raw/master/picture/searchc.gif)<br>
* 功能实现<br>
寻找好友模块的实现大部分是数据上的处理，包括好友列表的排序，好友信息区别展示，发送好友请求，获取请求消息以及添加好友等，大部分都是通过服务端处理，这里说一下好友列表的排序，从数据库获取到的好友信息包含CID信息，即好友的name+id，这样即使有好友name重复也可以根据id决定其排列顺序，主要在于中英文的混合排序和归类，这里借助了第三方包pinyin4j，在正则表达式匹配首字符为中文时通过pinyin4j获取不带声调的拼音首字母并添加在CID首位，以特殊字符连接（系统不允许用户使用的特殊字符），然后直接借助Arrays.sort进行排序，排序完成后根据首字母的ASC码做逻辑判断对其添加标识和归类，这样就得到了A~Z的标题栏数据，之后对排序后的CID数据进行特殊字符的去除并适配到RecyclerView中即可。<br>
关于好友信息区别展示，发送好友请求，获取请求消息以及添加好友等方面多为服务端的逻辑处理，篇幅原因这里不过多描述。
## 五、个人信息
个人信息主要包括个人主页、个人信息修改、系统消息和设置方面，由于时间原因,系统消息和设置方面功能还未完全实现。<br>
* UI<br>
![](https://github.com/onceabu/chat/raw/master/picture/account.png)<br>
![](https://github.com/onceabu/chat/raw/master/picture/homepage.gif)
![](https://github.com/onceabu/chat/raw/master/picture/edit.png)<br>
* 功能实现<br>
1、个人主页的实现与动态详情页面的实现原理一样，同样是借助帧布局、滑动监听和Touch事件，这种实现方式有很大的自定义性，受的限制也比较少，可以实现更多想要的的效果。借助沉浸式和自定义View及其属性与主内容联动的方式让个人主页变得更加炫酷自然，同时也增强了阅读空间，背景图也可以根据需要设置为同速滚动和差速滚动。<br>
2、我的资料的修改方面主要还是与服务端的数据传递和处理，包括头像、主页背景、昵称、个性签名、手机号等。这里主要补充一下拍照上传和图片的裁剪，拍照上传直接调用系统相机，并在启动相机前传递一个应用关联缓存目录中的Uri过去，用于存放拍摄的照片，之后通过Uri进行上传。裁剪使用的是第三方库，在选择本地图片后获取Uri进行裁剪，裁剪之前除了把这个Uri传递过去，同时像拍照一样也传递一个应用关联缓存目录中的Uri，用于存储裁剪后的图片并且上传。
