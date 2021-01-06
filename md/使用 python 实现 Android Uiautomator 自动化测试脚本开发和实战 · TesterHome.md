> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [testerhome.com](https://testerhome.com/articles/21317)

在 Android 自动化测试中，UIautomator 是 Google 官方提供的一个 Android 自动化测试框架。该框架可以实现获取屏幕控件和相关操作的功能，十分强大。但是，该框架有两个主要的缺点：  
1、只支持 java 语言进行脚本开发  
2、测试脚本要打包成 jar 或者 apk 包上传到设备上才能运行，环境准备和搭建都比较繁琐

为解决以上问题，我们希望快速编写自动化测试脚本，并且直接在电脑上运行，操作设备，所见即所得的进行测试。对此我们选用了 python 作为脚本语言，并利用第三方开源的库 uiautomator2 进行脚本开发。  
Uiautomator2 原理是在手机上运行了一个 http rpc 服务，将 uiautomator 中的功能开放出来，然后再将这些 http 接口封装成 Python 库。目前该项目在 github 上仍然持续更新中。下面将对该库的使用方法做一下介绍和实战操作。

一、Uiautomator2 模块的安装和环境准备
-------------------------

安装过程比较简单，python 环境准备好之后，通过 pip 即可安装，执行命令：  
pip install uiautomator2

安装之后，我们在电脑上连接一台要测试的 Android 手机，然后执行  
python -m uiautomator2 init

该命令是用于初始化我们的测试手机，该过程将安装包含 http rpc 服务的 apk 到手机主要有：atx-agent, minicap, minitouch 等。

我们测试一下环境是否准备 ok，在脚本中执行如下代码：

```
import uiautomator2 as u2
d = u2.connect()
print(d.info)


```

输出：  
[](https://testerhome.com/uploads/photo/2019/3f835a13-fe46-499a-84b2-7fa57f9cd6a0.jpeg!large)

[![](https://testerhome.com/uploads/photo/2019/3f835a13-fe46-499a-84b2-7fa57f9cd6a0.jpeg!large)](https://testerhome.com/uploads/photo/2019/3f835a13-fe46-499a-84b2-7fa57f9cd6a0.jpeg!large)

能正确输出信息证明环境和设备准备 ok

### 1、几种连接测试机的操作

通过 usb 数据线连接  
假设设备的 deviceid 是 "1234567abc"  
则连接语句是  
[](https://testerhome.com/uploads/photo/2019/12051678-cd08-4853-802e-e68db2e3b8e0.png!large)

[![](https://testerhome.com/uploads/photo/2019/12051678-cd08-4853-802e-e68db2e3b8e0.png!large)](https://testerhome.com/uploads/photo/2019/12051678-cd08-4853-802e-e68db2e3b8e0.png!large)

通过 adbWi-Fi 无线连接  
手机与电脑在同一局域网下，并且手机的 ip 地址是 “168.0.0.1”  
可以通过以下方式进行无线连接：  
[](https://testerhome.com/uploads/photo/2019/4cea49fe-4049-463d-b2f1-8ffcdcdc15bc.png!large)

[![](https://testerhome.com/uploads/photo/2019/4cea49fe-4049-463d-b2f1-8ffcdcdc15bc.png!large)](https://testerhome.com/uploads/photo/2019/4cea49fe-4049-463d-b2f1-8ffcdcdc15bc.png!large)

### 2、安装、启动和停止一个 app

**安装**  
可以通过 url 或直接安装一个 apk 包：

```
# 通过url安装apk包
d.app_install('http://some-domain.com/some.apk')

# 直接安装apk包
d.app_install('some.apk')

```

**启动 app**

```
# 默认的这种方法是先通过atx-agent解析apk包的mainActivity，然后调用am start -n $package/$activity启动
d.app_start("com.example.hello_world")

# 使用 monkey -p com.example.hello_world -c android.intent.category.LAUNCHER 1 启动
# 这种方法有个附带的问题，它自动会将手机的旋转锁定给关掉
d.app_start("com.example.hello_world", use_monkey=True) # start with package name

# 通过指定main activity的方式启动应用，等价于调用am start -n com.example.hello_world/.MainActivity
d.app_start("com.example.hello_world", ".MainActivity")

```

**停止 app**

```
# 相当于使用命令：`am force-stop`
d.app_stop("com.example.hello_world")
# 相当于使用命令：`pm clear`
d.app_clear('com.example.hello_world')


```

### 3、上传文件到手机 / 从手机取得文件

**上传文件到手机**  
上传文件使用 push() 方法：

```
# 上传文件夹到指定目录
d.push("foo.txt", "/sdcard/")
# 上传文件并重命名
d.push("foo.txt", "/sdcard/bar.txt")
# 上传一个文件对象
with open("foo.txt", 'rb') as f:
     d.push(f, "/sdcard/")
# 上传并且更改文件的权限
d.push("foo.sh", "/data/local/tmp/", mode=0o755)


```

**从手机上下载文件**  
下载文件使用 pull() 方法：

```
# 从手机上下载目标文件
d.pull(src="/sdcard/tmp.txt", dst="tmp.txt")


```

### 4、操控手机的基本事件

操控手机的基本事件主要有亮屏、灭屏、获取屏幕状态、按键、解锁屏幕、点击屏幕、截图等操作。

**亮屏 / 灭屏**

```
d.screen_on()  # 点亮屏幕
d.screen_off()  # 关闭屏幕


```

**获取屏幕状态**

```
d.info.get('screenOn') # 获取当前屏幕是否是点亮状态，返回True/False


```

**按键点击**  
可以通过按键名使用 press() 方法来进行点击，目前支持的按键名有：home、back、left、right、up、down、delete（del）、volume_up、volume_down、volume_mute 等。

```
d.press("home")  # 通过按键名点击，home键
d.press("back")  # 通过按键名点击，back


```

**解锁屏幕**

```
d.unlock() # 解锁操作


```

**点击屏幕**  
根据坐标点击：

```
d.click(x, y) # 根据坐标点击屏幕


```

**双击：**

```
d.double_click(x, y, 0.1) # 双击点（x，y），中间间隔0.1s


```

**长按：**

```
d.long_click(x, y, 0.5) # 长按点（x,y),时长0.5s


```

**滑动：**

```
d.swipe(sx, sy, ex, ey, 0.5) # 开始点是（sx，sy），结束点是（ex,ey),从开始滑动到结束，速度是0.5s


```

**拖拽：**

```
d.drag(sx, sy, ex, ey, 0.5) #从（sx，sy)拖拽到（ex,ey),速度是0.5s


```

**在点之间滑动（可用于实现九宫格解锁操作）**

```
d.swipe_points([(x0, y0), (x1, y1), (x2, y2)], 0.2)) # 从点（x0,y0）滑动到点（x1,y1）再滑动到（x2,y2）,中间间隔0.2s


```

**截图**  
截图功能需要额外安装 python 模块：pillow

```
image = d.screenshot() # 默认使用pillow模块
image.save("home.jpg") # 支持保存jpg或者png格式图片


```

**5、元素定位和操作**  
我们通过选择器（selector）来获取屏幕上的控件，这个操作与原生 uiautomator 类似，比如我们可以通过 text 和 className 来获取一个控件：

```
selector_clock = d(text='Clock', className='android.widget.TextView') #通过text和className获取控件


```

**目前可支持的选择器有：**

```
text, textContains, textMatches, textStartsWith
className, classNameMatches
description, descriptionContains, descriptionMatches, descriptionStartsWith
checkable, checked, clickable, longClickable
scrollable, enabled,focusable, focused, selected
packageName, packageNameMatches
resourceId, resourceIdMatches
index, instance


```

还可以通过子节点、兄弟节点方式来定位元素：

**获取子节点**

```
d(class)


```

**获取兄弟节点**

```
d(text="Google").sibling(class


```

**通过子节点的 text 或 description 属性获取子节点（child_by_text、child_by_description）**

```
d(class)


```

二、实战：获取百度音箱 app 上的对话信息
----------------------

我们为了自动化测试百度音箱的问答效果，准备了很多音频文件，需要通过脚本控制自动向百度音箱播放音频（问题），音箱接收音频信息后，会在百度音箱的 app 上显示对话的内容（问题和答案），我们通过抓取百度音箱 app 上显示的对话列表的内容，并将结果保存到 excel 文档中，完成整个自动化测试流程。

我们可以直接通过编写一个 python 脚本在电脑上运行来完成上述全部功能，其中核心就是使用上面介绍的 uiautomator2 模块，核心代码如下：

```
def get_asr_and_tts():
    d = u2.connect(device_id) # 连接手机
time.sleep(1)
    duihua_list = d(description='对话历史', className='android.view.View')  # 获取对话列表控件
duihua_list.scroll(5)  # 添加向上滑动操作，保证页面显示的是最新内容
print('scroll stop.....')
    time.sleep(1)
    duihua_list_info = duihua_list.info
    duihua_list_child_count = duihua_list_info['childCount']  # 获取到对话列表控件下的全部子元素的个数
print(duihua_list_child_count)
    last_box = duihua_list.child(index=duihua_list_child_count-2)  # 通过index选择器获取对话列表内最新的对话
last_box_info = last_box.info
print(last_box_info)
    general_asr_box = duihua_list.child(index=duihua_list_child_count-4)  # 获取问题的控件
    general_tts_box = duihua_list.child(index=duihua_list_child_count-2)  # 获取答案的控件
    general_asr_box_height = get_box_height(general_asr_box)
    general_asr_text = general_asr_box.info['contentDescription']  # 获取问题文本
    general_tts_text = general_tts_box.info['contentDescription']  #

return general_asr_text, general_tts_text

```

**操作的页面：**  
[](https://testerhome.com/uploads/photo/2019/da5ada46-9a3a-4aa4-9bff-6e7aba48f5f7.jpeg!large)

[![](https://testerhome.com/uploads/photo/2019/da5ada46-9a3a-4aa4-9bff-6e7aba48f5f7.jpeg!large)](https://testerhome.com/uploads/photo/2019/da5ada46-9a3a-4aa4-9bff-6e7aba48f5f7.jpeg!large)

**最后获得的结果：**  
[](https://testerhome.com/uploads/photo/2019/11b23587-0c32-43ea-89c5-458591339fb8.jpeg!large)

[![](https://testerhome.com/uploads/photo/2019/11b23587-0c32-43ea-89c5-458591339fb8.jpeg!large)](https://testerhome.com/uploads/photo/2019/11b23587-0c32-43ea-89c5-458591339fb8.jpeg!large)

三、总结
----

以上是使用 python 的 uiautomator2 模块来进行 Androiduiautomator 自动化脚本开发的简单介绍，通过使用这个模块，我们可以直接使用 python 编写脚本来操作 app，简化了使用原生 uiautomator 的复杂环境搭建和打包过程，并且直接运行脚本可以更好加入自己的测试逻辑在脚本中，更直观达到我们自动化测试的目的。更详细的内容可以查看该项目的 GitHub。

参考文档：  
Uiautomator2 项目的 github 地址：[https://github.com/openatx/uiautomator2](https://github.com/openatx/uiautomator2)