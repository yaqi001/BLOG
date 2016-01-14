# Matplotlib 中文显示问题

希望给像我一样刚开始使用 Matplotlib 的同学们一个准确的参考:)。
BTW，我使用的 linux 发行版是 Fedora 22。

1. **查看你系统中的字体**

   我使用的是 *WenQuanYi Zen Hei*，如果你有自己习惯使用的中文字体也可，不过一定要知道该字体究竟是有衬还是无衬，下面会详细说。
   ~~~ bash
   $ fc-list | grep WenQuanYi
   ~~~

   如果没有输出就说明你的 linux 系统中并不存在该字体，下面我们来安装该字体。
   如果有输出，请忽略第 2 步。

2. **安装中文字体**
   ~~~ bash
   $ sudo yum -y install wqy-zenhei-fonts.noarch
   ...
   $ fc-list | grep WenQuanYi
   /usr/share/fonts/wqy-zenhei/wqy-zenhei.ttc: 文泉驿点阵正黑,文泉驛點陣正黑,WenQuanYi Zen Hei Sharp:style=Regular
   /usr/share/fonts/wqy-zenhei/wqy-zenhei.ttc: 文泉驿等宽正黑,文泉驛等寬正黑,WenQuanYi Zen Hei Mono:style=Regular
   /usr/share/fonts/wqy-zenhei/wqy-zenhei.ttc: 文泉驿正黑,文泉驛正黑,WenQuanYi Zen Hei:style=Regularv
   ~~~
   
3. **查看字体文件的出处**

   安装了 `WenQuanYi` 之后，你可能会再次执行 matplotlic 代码，但是你会发现中文字体仍然没有显示出来。你需要继续往下看。
   查看 `~/.cache/matplotlib/fontList.cache`（`~/.cache/matplotlib/` 目录下面的两个文件是在执行代码的时候产生的）文件，我们会发现，matplotlib 主要会从一下几个目录中寻找字体文件：
   * `/usr/share/matplotlib/mpl-data/fonts/ttf/***.ttf`
   * `/usr/share/fonts/.../***.ttf`
   * `/usr/share/fonts/default/.../***.afm`

   我们猜想也许是因为 `matplotlib` 会自动忽视 `.ttc` 的文件，所以我用下面的链接将 `wqy-zenhei.ttc` 文件转换成了 `wqy-zenhei.ttf` 文件，链接在此:[http://www.files-conversion.com/font-converter.php](http://www.files-conversion.com/font-converter.php)

4. **有衬字体 & 无衬字体**

   根据相关资料的查询，我了解到 WenQuanYi Zen Hei 是 sans-serif（无衬字体）。serif 是有衬字体。所以，当你下载了自己喜欢的字体后，应该先查询一下它究竟是有衬还是无衬，因为这一点浪费了我好多时间啊！！！

5. **修改 matplotlibrc 文件**

   在我的操作系统下 `matplotlibrc` 文件在 `/etc` 目录下，打开该文件，修改以下几个内容：
   ~~~ bash
   font.family   : sans-serif   # 也就是去掉了注释符号而已，如果你下载的字体是有衬字体，这里要填写 serif
   font.sans-serif    : WenQuanYi Zen Hei, Bitstream Vera Sans, Lucida Grande, Verdana, Geneva, Lucid, Arial, Helvetica, Avant Garde, sans-serif   # 同样，这里也是去掉注释符号，但是如果你下载的字体是有衬字体，就要将 font.serif 的注释去掉，而不是现在这个，然后在添加上 WenQuanYi Zen Hei。
   ~~~

6. **删除 `~/.cache/matplotlib` 目录下的两个文件，然后重新执行你的代码**

我想应该成功了吧，如果没有成功，请将你的详细情况告知于我，我的邮箱是：zhangyingyun001@gmail.com

参考文章:
[http://www.cnblogs.com/skabyy/p/3461229.html](http://www.cnblogs.com/skabyy/p/3461229.html)
[http://blog.sciencenet.cn/blog-43412-343002.html](http://blog.sciencenet.cn/blog-43412-343002.html)
[http://www.jinbuguo.com/gui/linux_fontconfig.html](http://www.jinbuguo.com/gui/linux_fontconfig.html)
