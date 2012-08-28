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
* 一个学生信息系统。它就是一个庞大的数据库，里面记载着所有的学生信息，包括他们当前正在进行的课程，以及以后需要完成的课程;还有他们的笔记，这份笔记可以是他们对一门所完成课程的高度总结。当然，这个信息系统也可以提供其他的管理功能，比如跟踪一个学生是否上缴了学费。
* 一个文档库（比如使用 Alfresco）。它用来存储文件，以及跟踪用户合作维护文件时的工作流。  
* 一个电子档案袋（ePortfolio）。 学生可以在这里存放他们自己的资料(assets)并且吧它们组合起来形成其他文档。例如利用这些资料编写一篇CV（简历），或者用来证明档案所有者已经满足了一门实践课的选修条件。  


Moodle专注于为所有参与到教学中的人提供一个在线平台，而不是为某一个教育组织特定设计的某个系统。Moodle仅仅为其他功能提供了最基本的实现，所以它可以单独的作为一个应用，或者与其它系统进行集成。Moodle扮演的角色被正式地称为虚拟教学环境（VLE），或者是教学/课程管理系统（LMS，CMS，甚至LCMS）。

Moodle是一个用PHP编写的开源免费软件(GPL)。它可以在绝大多数的Web服务器和平台上运行。它需要一个数据库，目前支持MySQL，PostgreSQL，MS SQL Server以及Oracle。

Moodle从哪里来？
---------------
Moodle项目由Martin Dougiamas在1999年开创，当时他正在澳大利亚的科廷大学工作。1.0版本于2002年发布，当时使用的语言和数据库版本是PHP 4.2和MySQL 3.23。当时的版本从一开始就限制了可能采用的框架，然而从那以后，整个软件都发生了很大变化。现在发布的版本是Moodle 2.2.x系列。

13.1. Moodle运作方式综述
=================
安装Moodle的三个部分
------------------
Moodle安装由三部分组成：  

1. 代码，通常在一个类似 `/var/www/moodle` 或者 `~/htdocs/moodle` 的目录里。Web服务器应该具有这个目录的写权限。  
2. 数据库，由上面提到过的几种RDMS（RDBMS，关系性数据库管理系统）管理。实际上，moodle给所有的表名增加了一个前缀，所以如果需要的话，它可以和其他应用共用一个数据库。  
3. `moodledata` 目录。这个目录用于存储用户上传的文件以及系统生成的文件，同样Web服务器需要有这个目录的写权限。出于安全考虑，这个目录应该设置于Web根目录之外。

这些可以完全部署在一台服务器上。或者，采用负载均衡的设置，在每台Web服务器上都部署代码，但是仅仅共用一个数据库和一个很有可能在其他服务器上的`moodledata`目录。

当Moodle安装完毕后，这三个部分的配置信息被存储在`moodle`根目录下的`config.php`文件中。

请求调度
-------
Moodle是一个Web应用，所以用户通过浏览器来与之交互。从Moodle自己的视角来看，这就意味着它要响应HTTP请求。Moodle的一个重要设计考量就是URL的名字空间，以及URLs如何被调度到不同的脚本上。

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

每一个插件都必需包含一个叫做`version.php`的文件，这个文件定义了关于这个插件本身的元数据。Moodle的插件安装系统会使用它来对插件进行安装和升级。例如`local/greet/version.php`包含代码：

    <?php
    $plugin->component    = 'local_greet';
    $plugin->version      = 2011102900;
    $plugin->requires     = 2011102700;
    $plugin->maturity     = MATURITY_STABLE;	

因为可以从路径上显然地推导出插件的名字，所以乍看之下代码里面包含组件名（component name）略显多余。而实际上，安装器需要通过组件名来验证插件是否安装在正确的位置上。版本（Version）字段定义了这个插件的版本，成熟度(Maturity）是诸如ALPHA，BETA，RC（发布候选版, release candidate）, 或者STABLE这样的标签。Requires字段标识着能和这个版本兼容的Moodle最低版本号。如果需要，你也可以记录下这个插件依赖的其他插件。

这里是这个简单插件的主要脚本（存储在`local/greet/index.php`）：

    <?php
    require_once(dirname(__FILE__) . '/../../config.php');        // 1
    
    require_login();                                              // 2
    $context = context_system::instance();                        // 3
    require_capability('local/greet:begreeted', $context);        // 4
    
    $name = optional_param('name', '', PARAM_TEXT);               // 5
    if (!$name) {
        $name = fullname($USER);                                  // 6
    }
    
    add_to_log(SITEID, 'local_greet', 'begreeted',
        'local/greet/index.php?name=' . urlencode($name));        // 7
    
    $PAGE->set_context($context);                                 // 8
    $PAGE->set_url(new moodle_url('/local/greet/index.php'),
        array('name' => $name));                                  // 9
    $PAGE->set_title(get_string('welcome', 'local_greet'));       // 10
    
    echo $OUTPUT->header();                                       // 11
    echo $OUTPUT->box(get_string('greet', 'local_greet',
        format_string($name)));                                   // 12
    echo $OUTPUT->footer();                                       // 13

Line 1：引导Moodle
-----------------
    require_once(dirname(__FILE__) . '/../../config.php');        // 1

这单独的一行是大多数工作都要首先完成的。我之前说过，`config.php`包含着Moodle如何连接数据库以及找到`metadata`目录的细节。然而，它以一行`require_once('lib/setup.php')`结束。这样：

1. 通过`require_once`加载所有Moodle标准库；  
2. 开始处理会话；  
3. 连接数据库；  
4. 初始化一系列全局变量，我们一会就将看到它们。

Line 2：检查用户是否登录
-----------------------

    require_login();

这行使得Moodle利用管理员配置过的任何认证插件来判断，当前访问用户是否已经登录。

一个与Moodle整合性更好的插件会在这里传递更多的参数，比如这个页面属于哪个课程或者活动。然后调用`require_login`仍然会检查是否当前用户是否参加了这门课程或者活动。如果是，用户就可以访问这门课程，或者观看这个活动；如果不是，那么适当的错误信息会被显示出来。

13.2. Moodle中的角色和权限系统
===========================
接下来的两行代码显示出如何检查用户是否有做某件事的权限。正如你所见，从开发者的角度来说，这些API都十分的简单。但是，实际上在这下面是一个非常复杂的接入系统。这会给管理员很大的伸缩性，以控制什么人可以做什么。

Line 3: 获得上下文
--------------------
    $context = context_system::instance();                        // 3

在Moodle中，同一个人可能在不同的地方拥有不同的权限。比如说一个用户可能在某个课程上做一名老师，也可能是另一门课程的一位学生。这些地方被称作为上下文。上下文在Moodle中构筑了一个特别像文件系统中目录结构的多层结构。

在系统的上下文中，有许多的上下文信息被构筑出来，它们负责维护那些为了组织课程而被创建的不同分类(Category)。这些上下文可以是嵌套的，比如在一个分类里面包含有其他更多的分类。分类上下文同时也包含着课程上下文。最后，每一个课程中的活动也会拥有一个自己的Moodle上下文。  
![contexts.png](http://www.aosabook.org/images/moodle/contexts.png)  
图13.3：上下文

Line 4: 检查用户是否有权执行这个脚本
--------------------------------
    require_capability('local/greet:begreeted', $context);        // 4

现在我们获得了上下文——与Moodle相关的领域——权限就能够被检查了。任何一丁点的用户能否执行某个功能的信息被称作一个能力（Capability）。 基于能力的检查可以提供比简单的`require_login`检查更加细粒度的访问检查。我们这个简单的插件，只有一个能力：`local/greet:begreeted`。

这个检查通过`require_capability`函数来完成，不过这需要这个能力的名字以及当前的上下文。就像其他`require_...`函数一样，如果用户没有这个能力，它不会正常返回，而是显示一个错误。在其他地方，非致命的`has_capability`函数，当可用的时候返回true，比如要不要在另一个网页上添加对于当前这个脚本的一个链接。

那么管理员是如何配置什么用户拥有什么权限的呢？这是在`has_capability`函数中通过计算得到的（至少理论上是这样）：

1. 从当前上下文开始；  
2. 获得这个用户在当前上下文中所扮演的所有角色；  
3. 计算出在当前上下文中，每一个角色所拥有的权限；    
4. 将这些权限整合起来获得一个最终的结果。

定义能力
-------
在下面一个例子中，一个插件可以根据它要提供的独特功能来定义新的能力。在每一个Moodle插件中都有一个子目录，叫做`db`。这个目录包含了所有安装和升级这个插件所需的信息，其中有一个`access.php`文件来定义能力。下面就是我们插件的`access.php`,它位于`local/greet/db/access.php`

    <?php
    $capabilities = array('local/greet:begreeted' => array(
        'captype' => 'read',
        'contextlevel' => CONTEXT_SYSTEM,
        'archetypes' => array('guest' => CAP_ALLOW, 'user' => CAP_ALLOW)
    ));

这里定义了对于每个能力的元信息，这些元信息会在在构造权限管理用户界面的时候被用到。它规定了对于常见角色的默认权限。

角色
----
Moodle权限系统的下一个部分就是角色了。一个角色其实就是一个权限集合的名字。当你登录到Moodle之后，你在系统上下文中就拥有了一个“Authenticated user”的角色。由于系统上下文是上下文层次结构中的根节点，所以这个角色会被应用到所有的地方。

在一个特定的课程中，你或许是一个学生，那么这个角色就会在这个课程的上下文以及其子模块的上下文中都有效。然而，再另一门课程中，你可能有一个不同的身份。例如，Gradgrind先生可以是”Facts，Facts，Facts“这门课的教师，但是他却是职业发展课程“Facts Aren't Everything”中的一名学生。最后，一个用户或许会在特定的论坛（模块上下文）中被指派为一个主持人(Moderator)的角色。

权限
-----
一个角色对每一种能力都规定了一个权限。例如，教师的角色很有可能允许（ALLOW）`moodle/course:manage`，但是学生角色就不会。然而，教师和学生都会允许`mod/forum:stardiscussion`。

角色通常是全局定义的，但是他们可以在每一个上下文中被重新定义。比方说，一个特定的wiki如果要对所有学生角色变成只读的，只需要将这个wiki（模块上下文）的上下文中对于学生的`mod/wiki:edit`能力覆盖成禁止（PREVENT）。

这里是四种权限：

* 未设置／继承（默认）  
* 允许  
* 禁止（PREVENT）  
* 戒绝（PROHIBIT）

在给定的上下文中，一个角色对每一个能力都有这四种权限之一。禁止和解决的一个重要区别是，戒绝在禁止的基础上还确保子上下文不能覆盖这个权限。

权限整合
-------
最后，一个用户在这个上下文中根据所有角色获得的权限会被整合起来。

* 如果任何角色对于一个能力给出的权限是戒绝，那么返回false。  
* 否则，如果任何角色对于这个能力给出的权限是允许，那么返回true。  
* 再否则，返回false

一个使用戒绝权限的用例如下：假设一个用户在许多的论坛中持续乱发帖子，我们想让这个家伙立刻闭嘴。那么我们可以建立一个叫做捣蛋鬼（Naughty）的角色，这个角色对于类似`mod/forum:post`这样的权限全部设置为戒绝。我们可以把这个捣蛋鬼的角色在系统上下文中分配给那个乱发帖子的用户。这样我们就能保证这个用户在所有的论坛里面都不能发帖了。（我们可以和这个学生好好谈谈，得到一个满意的答复，然后在把这个角色指派删除掉，这样他又能使用我们的系统了）

总而言之，Moodle的权限系统给了管理员很大的伸缩性。他们可以定义任何一个他们喜欢的角色，为这个角色的每一个能力指定不同权限；他们可以在子上下文中改变角色的定义；并且，他们还可以在不同的上下文中对用户赋予不同的角色。

13.3. 回到样例脚本
=================
脚本的下一个部分解释了一些繁杂的事情：

Line 5：从请求中获得数据
-----------------------
    $name = optional_param('name', '', PARAM_TEXT);               // 5

每个网络应用都会做的一件事情就是，把数据从请求中获取出来（GET或者POST变量），而不会产生SQL注入和跨站脚本攻击。Moodle提供了两种方法来完成这件事。

上面那行代码就是一个简单的方法。它通过一个参数名（在这里的`name`），一个缺省值，以及一个期望类型来获得一个单独的值。期望类型用来清理掉所有带有非法字符的输入。我们定义了许多类型，诸如`PARAM_INT`，`PARAM——ALPHANUM`，`PARAM——EMAIL`等等。

这里你也可以用类似`required_param`这样的函数。这些`require...`函数如果发现期望的参数没有找到时会停止执行并显示一个错误信息。

另一个Moodle从请求中获得数据的机制是一个非常成熟的库。它给PEAR的HTML QuickForm库包了一层。（对于非PHP程序员来说，PEAR在PHP中就相当与CPAN）这在刚开始决定的时候似乎是一个好主意，但是现在我们已经不维护它了。或许在未来的某一天，我们会迁移到一个新形势的库中，就像我们中许多人一直想做的，因为QuickForm有一些恼人的设计问题。不过现在，这个机制就已经足够了。表单就是一个字段的集合（比如text box，select drop-down，date-selector），每一个字段可能具有不同的类型用于前端和后端的验证（包括使用`PARAM_...`类型）。

Line 6：全局变量
---------------
    if (!$name) {
        $name = fullname($USER);                                  // 6
    }

这一小段展示了Moodle提供的第一个全局变量。`$USER`保存关于执行当前脚本的用户信息。其他的全局变量包括：

* $CFG：保存常用的配置  
* $DB：数据库连接  
* $SESSION：封装了PHP会话（session）  
* $COURSE：当前请求相关的课程  
还有一些其他的，我们会在下面见到它们。

你或许觉得“全局变量”这个词十分恐怖。然而，请注意，PHP每次之处理一个请求。所以这些变量根本就不是全局的。实际上，PHP的全局变量可以被视为线程安全的注册表模式（请参看Martin Fowler的企业级应用架构模式），这也是Moodle使用它们的原因。使最常用的对象始终可见是非常方便的，因为你不需要把它们作为参数传入到没一个函数和方法中去。这个方法很少被滥用。

没那么简单
--------
这一行同时也揭示出问题领域的一点：任何事都不是那么简单。显示一个用户名远比轻易地把`$USER->firstname`，`~`，`$USER->lastname`拼接起来复杂的多。学校或许规定只允许显示其中的一部分，而且许多不同的文化对于名字显示的顺序也有不同的习惯。所以，根据这些规则，才会有针对它的不同配置和一个用来组装全名的函数。

日期是一个类似的问题。不同的用户可能在不同的时区中。Moodle把所有的日期都存储成Unix时间戳，这种时间戳是一个整数，并且所有的数据库都支持它。所以，必须有一个`userdate`的函数来根据用户所在的时区和本地设置来合理的显示时间。

Line 7：日志
------------
    add_to_log(SITEID, 'local_greet', 'begreeted',
            'local/greet/index.php?name=' . urlencode($name));    // 7

Moodle所有的关键动作都会被日志记录下来。日志被写到数据库的一个表中。这是一个折中的办法。这个方法使得复杂的分析变得容易，而且Moodle基于这些日志也提供了很多翔实的报告。但是对一个大规模、高访问量的网站来说，这却带来了一个性能问题。日志表非常大，这使得数据库备份变得异常困难，而且对于日志的查询也非常的慢。在日志表上还存在者写入竞争。这些问题可以通过不同的放发得以缓解，比如批量写，存档或者删除旧的记录，把它们从主数据库中移除。

13.4. 产生输出（页面生成）
==============
输出主要通过两个全局对象来处理：
    $PAGE->set_context($context);                                 // 8

Line 8：全局变量$PAGE
--------------------
`$PAGE`存储着要被输出的页面信息。 这个信息在所有产生HTML的代码中都可以轻易获得。在这个脚本中，必须明确指明当前的上下文是什么。（在其他的情况下，它或许已经通过`require_login`被自动设置了）这个页面的URL也必须被明确。这看起来似乎很没必要，但是需要它的合理性在于你或许会使用不同的URL来获取同一个页面，但是传递给`set_url`的URL必须是一个这个页面的规范化URL —— 一个永久链接，如果你喜欢的话。页面的标题也要被设置。这样HTML的head元素就被构建了。

Line 9：Moodle URL
------------------
    $PAGE->set_url(new moodle_url('/local/greet/index.php'),
            array('name' => $name));                              // 9

顺便说一句，上面用过的`add_to_log`并没有使用这个辅助类。确实，日志API不能够接受`moodle_url`对象。这种不一致性是一个像Moodle一样老的code-base①的典型特征。  
> ① 译者注：code-base通常是指那些由人力产生的代码，许多自动生成的代码不算，比如由配置文件通过工具生成的代码就不是code-base。一般code-base的代码才有用版本控制的价值。

Line 10：国际化
--------------
    $PAGE->set_title(get_string('welcome', 'local_greet'));       // 10

Moodle使用自己的系统来支持多语言。或许现在有许多的PHP国际化库，但是在它第一次被实现的2002年当时，没有任何一个可用的库能够完成这个任务。整个系统基于`get_string`函数。字符串被一个键和插件的Frankenstyle名字唯一确定。就像你在第12行看到的，完全可以把值插入到字符串中。（多值在PHP中通过数组和对象来处理）

字符串会在一个语言文件中被查找，这个语言文件里面其实就是一个PHP数组。这里有一个我们插件的语言文件`local/greet/lang/en/local_greet.php`：

    <?php
    $string['greet:begreeted'] = 'Be greeted by the hello world example';
    $string['welcome'] = 'Welcome';
    $string['greet'] = 'Hello, {$a}!';
    $string['pluginname'] = 'Hello world example';

注意到，除了两个我们脚本中用到的字符串，这里还给某个能力了一个名字，还有这个插件显示在用户界面上的名字。

不同语言由两个字母的国家码唯一确定（这里的`en`）。语言包或许衍生于其他的语言包。比如说`fr_ca`（加拿大法语）语言包声明了`fr`（法语）作为它的母语言，所以要所搜包括它母语言包的所有语言包才能够找到这个字符串。

语言包制作以及协同翻译在[http://lang.moodle.org]上管理，Moodle用它们制作了一个可定制插件（[local_amos](http://docs.moodle.org/22/en/AMOS)）。它使用Git和数据库作为存储语言文件的后端，保留了所有的历史版本。

Line 11：开始输出
----------------
    echo $OUTPUT->header();                                       // 11

这又是一个看似平淡无奇的一行，然而它所做的工作可比看起来多得多。这里最关键的一点在于，在任何的输出之前，页面所采用的主题（皮肤）必须被计算出来。这取决于页面上下文以及用户偏好的组合。然而，`$PAGE->context`只在第8行被设置，所以`$OUTPUT`全局变量不能在脚本的一开始就初始化。为了解决这个问题，一些PHP的小技巧根据`$PAGE`的信息，在第一次调用输出方法的时候才构造合适的`$OUTPUT`。

另一件需要考虑的事情是，Moodle中的每一个界面都有可能包含块（blocks）。这些块是一部分可以额外配置的内容，通常被显示在主要内容的左侧或者右侧。（它们是一类插件）同时，到底哪些特定的块需要被显示出来，通过一种弹性的方式（管理员可控），由页面的上下文和其他页面的标识来决定。所以，输出的另一个准备工作就是调用`$PAGE->blocks->load_block()`。

当所有必要的信息都被准备好了之后，主题插件（控制页面的整体外观）被调用以产生页面的整体布局，包括任何标准需要的头部和页脚。这个调用同时也负责在HTML中对应的位置填入块中的内容。在布局的中间，这里会有一个`div`，这个页面特定的内容会显示在这里。当HTML的布局产生之后，在主要内容的`div`上一切两半。在第一半完成后，其他的部分被存储起来，由`$OUTPUT->footer()`返回。

Line 12：输出页面Body
---------------------


Line 13：结束输出
----------------


这个脚本应该混杂逻辑和显示么？
-------------------------


13.5. 数据库抽象
===============


`modle_database`类
-------------------


定义数据库结构
-------------


13.6. 本文未涉及到的
===================


13.7. 经验教训
=============




