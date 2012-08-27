Moodle
=======
作者：[Tim Hunt](http://www.aosabook.org/en/intro2#hunt-tim)  
译者：[Li Shijian](http://lishijian.com)

Moodle是一款为教育系统设计的Web应用。我会对Moodle各个部分如何运作做一个综述，同时我将专注于介绍几个我认为特别有趣的设计：  

1. 用插件分割应用的方法；  
2. 权限系统 —— 它控制着什么用户可以在系统不同的地方做什么事情；  
3. 产生输出的方式 —— 它使得应用不同的主题来更改外观，将UI接口分离出来；  
4. 数据库抽象层。

什么是Moodle?
------------
Moddle （http://moodle.org/） 提供了一个师生之间的在线教学平台。一个Moodle站点被划分为不同的课程。特定的用户以不同的身份参与到一个课程中去，比如学生或者老师。每一个课程都由一系列的资源和活动组成。一个资源可以是一份PDF文档，Moodle的一个HTML页面，或者干脆是一个指向网络上其他位置的链接。一个活动或许是一个论坛，一次测验，或者是一个Wiki。在一个课程中，这些资源和活动以某种方式被组织起来。例如，它们可以按照逻辑上的话题，或者是日程上特定的周目被分配到一起。  
![course.png](http://www.aosabook.org/images/moodle/course.png)  
图13.1： Moodle课程

Moodle可以作为一个单独的应用来使用。比如说，如果你只是想为软件架构课程构建一个网站，你只要在你的网站托管那里下载Moodle并且安装它，而后创建课程，等待学生自己来注册就可以了。或者说，你为一个庞大的机构工作，Moodle可以仅仅成为你所运行的众多系统中的一个。你很有可能已经拥有了:  
![university.png](http://www.aosabook.org/images/moodle/university.png)  
图13.2: 大学系统典型架构  
* 一个管理跨系统的用户帐号身份认证服务（例如使用LDAP）。  
* 一个学生信息系统。它就是一个庞大的数据库，里面记载着所有的学生信息，包括他们当前正在进行的课程,以及以后需要完成的课程;还有他们的笔记，这份笔记可以是他们对一门所完成课程的高度总结。当然，这个信息系统也可以提供其他的管理功能，比如跟踪一个学生是否上缴了学费。
* 一个文档库（比如使用 Alfresco）。它用来存储文件，以及跟踪用户合作维护文件时的工作流。  
* 一个电子档案袋（ePortfolio）。 学生可以在这里存放他们自己的资料(assets)并且吧它们组合起来形成其他文档。例如利用这些资料编写一篇CV（简历），或者用来证明档案所有者已经满足了一门实践课的选修条件。  


Moodle专注于为所有参与到教学中的人提供一个在线平台，而不是为某一个教育组织特定设计的某个系统。Moodle仅仅为其他功能提供了最基本的实现，所以它可以单独的作为一个应用，或者与其它系统进行集成。Moodle扮演的角色被正式地称为虚拟教学环境（VLE），或者是教学/课程管理系统（LMS，CMS，甚至LCMS）。

Moodle是一个用PHP编写的开源免费软件(GPL)。它可以在绝大多数的Web服务器和平台上运行。它需要一个数据库，目前支持MySQL，PostgreSQL，MS SQL Server以及Oracle。

Moodle从哪里来？
---------------
Moodle项目由Martin Dougiamas在1999年开创，当时他正在澳大利亚的科廷大学工作。1.0版本于2002年发布，当时使用的语言和数据库版本是PHP 4.2和MySQL 3.23。当时的版本从一开始就限制了可能采用的框架，然而从那以后，整个软件都发生了很大变化。现在发布的版本是Moodle 2.2.x系列。

Moodle运作方式综述
=================
安装Moodle的三个部分
------------------
Moodle安装由三部分组成：  

1. 代码，通常在一个类似 `/var/www/moodle` 或者 `~/htdocs/moodle` 的目录里。Web服务器应该具有这个目录的写权限。  
2. 数据库，被上面提到过的RDMSs之一。实际上，moodle给所有的表名增加了一个前缀，所以如果需要的话，它可以和其他应用共用一个数据库。  
3. `moodledata` 目录。这个目录用于存储用户上传的文件以及系统生成的文件，同样Web服务器需要有这个目录的写权限。出于安全考虑，这个目录应该设置于Web根目录之外。

这些可以完全部署在一台服务器上。或者，采用负载均衡的设置，在每台Web服务器上都部署代码，但是仅仅共用一个数据库和一个很有可能在其他服务器上的`moodledata`目录。

当Moodle安装完毕后，这三个部分的配置信息被存储在`moodle`根目录下的`config.php`文件中。

请求调度
-------
Moodle是一个Web应用，所以用户通过浏览器来与之交互。从Moodle自己的视角来看，这就意味着它要相应HTTP请求。Moodle的一个重要设计考量就是URL的名字空间，以及URLs如何被调度到不同的脚本上。

Moodle在这里采用PHP标准方法。浏览一个课程的主页时，URL可能像 `.../course/view.php?id=123`，这里`123`就是这门课程在数据库中的唯一标识。浏览一个论坛讨论时，URL可能是 `.../mod/forum/discuss.php?id=456789`。 也就是说，这些特定的脚本，`course/view.php` 或者 `mod/forum/discuss.php` 会来处理这些请求。这对于开发者来说是非常简单的。想要理解Moodle是怎么处理一个特定的请求，你可以观察URL，从阅读那份php文件的代码开始。这从用户的角度来看是十分丑陋的，因为这些URL是永久不变的。如果一个课程改了名字，或者一个管理员把一个讨论转移到另一个论坛中，这些URL都不会变。（这对于URL来说是一个非常好的性质，正如Tim Berners-Lee在他的文章[Cool URIs don't change](http://www.w3.org/Provider/Style/URI.html)中提到的）

另一种可以采用的方法是建立一个唯一入口 `.../index.php/[其他使请求唯一确定的信息]`。这个单独的`index.php`脚本会通过某种方式将请求进行调度。这个方法添加了一个大多数软件开发者都喜欢用的间接层。缺少了这个间接层并不会影响到Moodle的使用。

插件
----
和许多其它成功的开源项目一样，Moodle由许多和系统内核协同工作的插件构建起来。这是一个绝妙的主意，因为它可以使用户按照他们定制的方法来增强Moodle的功能。一个开源系统的重要优势在于，你可以根据自己的特定需求来更改它。然而，为代码增加高可定制性的同时，会在系统升级的时候引入大麻烦，即使我们已经采用了很好的版本控制系统。Moodle的插件通过定义好的API与内核交互，所以在自包含的插件中，可以允许尽可能多的用户定制与新特性被开发出来。这也方便了用户根据需求定制自己的Moodle，分享这些定制内容，同时也便于对Moodle系统内核进行升级。

有许多不同的方法可以将一个系统构建成插件化的。Moodle具有一个相对庞大的内核，并且插件是强类型的。我所说的相对庞大的内核，指的是内核提供了大量的功能。这违反了那类，由一个小型的插件启动存根(stub)引导，其余都是插件的架构设计。

当我提及插件是强类型的时候，我指的是根据你想要实现的具体功能，你可能需要写完全不同的插件，实现不同的API。比如，一个新的活动模块插件会和一个新的认证插件或者是提问插件截然不同。根据最后统计，现在我们一共有35种不同的插件（这里有一个[Moodle插件类型完全列表](http://docs.moodle.org/dev/Plugins/)）。这违背了那类，所有插件通过使用最基本的API，通过注册它们感兴趣的钩子和事件的架构设计。

通常来说，Moodle现在有尝试把更多的功能移到插件中以减小内核的趋势。可是这并没有带来巨大的成功，因为当前一个逐渐增长的特性集趋于去扩展内核。另一个趋势是尽可能将不同种类的插件进行规范化。这样在许多公共功能上，比如安装和升级，所有类型的插件都能够按照统一的方式运行。

一个Moodle中的插件其实就是一个包含许多文件的目录。每一个插件都有一个类型和名字，这两个构成了这个插件的"Frankenstyle"组件名称。（"Frankenstyle"这个单词出自于开发者Jabber频道的一次讨论，所有的人都爱它，所以它就被固定下来了）插件的类型和名字确定了这个插件目录的路径。插件类型给定一个前缀，目录名称就是这个插件的名字。这里有一些例子：

<table class="table table-condensed table-bordered table-striped">
<tr><td><strong>Plugin type</strong></td><td><strong>Plugin name</strong></td><td><strong>Frankenstyle</strong></td><td><strong>Folder</strong></td></tr>
<tr><td>mod (Activity module)</td><td><code>forum</code></td><td><code>mod_forum</code></td><td><code>mod/forum</code></td></tr>
<tr><td>mod (Activity module)</td><td><code>quiz</code></td><td><code>mod_quiz</code></td><td><code>mod/quiz</code></td></tr>
<tr><td>block (Side-block)</td><td><code>navigation</code></td><td><code>block_navigation</code></td><td><code>blocks/navigation</code></td></tr>
<tr><td>qtype (Question type)</td><td><code>shortanswer</code></td><td><code>qtype_shortanswer</code></td><td><code>question/type</code>/shortanswer</td></tr>
<tr><td>quiz (Quiz report)</td><td><code>statistics</code></td><td><code>quiz_statistics</code></td><td><code>mod/quiz/report/statistics</code></td></tr>
</table>

最后的一个例子表明了每一个活动模块被允许声明子插件类型。只有活动模块才能做到这个，出于亮点原因。如果所有的插件都可以声明子插件类型，这或许会带来严重的性能问题。活动模块是Moodle中最重要的教育活动，也是插件中最终要的类型，所以它们应该具有特殊的权限。

示例插件
------

我会以一个具体的插件实例来解释Moodle架构中的大量细节。作为惯例，我选择实现一个显示"Hello world"的插件。

这个插件实际上并不适合任何一种Moodle标准插件。它只是一个简单的脚本，和其他任何东西都没有联系，所以我选择把它制作成一个'local'类型的插件。这是一个catch-all的插件类型，专门处理一些杂乱的功能，所以在这里再适合不过了。我给我的插件命名为`greet`,所以它的Frankenstyle的名字是`local_greet`，路径为`local/greet`。（[插件代码](https://github.com/timhunt/moodle-local_greet)下载）



