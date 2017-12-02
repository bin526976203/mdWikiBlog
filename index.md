莫贤彬的MarkdownBlog
=========

前言
------------
Hint: 本人是互联网金融领域研发人员，深耕互联网与金融领域多年，深度挖掘工作甚至商业中有高度价值的技术，系统整理，然后分享给大家。本人博客based Markdown,所有的核心内容都是markdown文件，大家如有兴趣可以fork我右上角的GitHub项目。本人CSDN博客:http://blog.csdn.net/u012712901 .如需联系，请发邮件，邮箱在网页底部。

Note: 本人博客使用100%html,js,css进行对markdown语法文件进行渲染,100%运行在客户端中。以下是本人博客使用的MarkDown语法的简介。

代码块
------------

```java
public class Person{
    private int age;
    private String name;
    public Person(int age, String name){
        this.age = age;
        this.name = name;
    }
}
```
    ```java
    public class Person{
        private int age;
        private String name;
        public Person(int age, String name){
            this.age = age;
            this.name = name;
        }
    }
    ```

Gimmicks标签
------------

Gimmicks是语法小便签，带来大量的动态功能到网页中。例如，您可以使用它们来嵌入的YouTube视频，图像幻灯片或脸谱网喜欢按钮。
所有你需要做的是包括一些特制的链接到你的Markdown文件。例如，如果你想嵌入一个YouTube视频（而不是链接它），你只需要插入一个视频链接：

    [](http://www.youtube.com/watch?v=RMINSD7MmT4)

Warning: Gimmicks标签是需要加载网上资源，如果脱网状态，将无法起效。


Gimmicks标签汇总
===================
* * *

Alerts提示标签
------

每当你用一个特殊的触发词来启动一个段落时，就会自动地将此段落进行颜色修饰。

触发词不区分大小写，必须是下列内容之一：

**类型**    | **触发词**
-----------|---------
Warning    |warning, achtung, attention, warnung, atención, guarda, advertimiento, attenzione
Note       |note, beachte, nota 
Hint       |hint, tip, tipp, hinweis, suggerimento

效果:

Attention: This text is important.

Note! This is a note.

Hint: This is a hint.

* * *

UML 图表 based yUML.me
-----

基于优秀的yUML.me进行UML图表创作 [yUML.me](http://yuml.me)  (see their website for documentation).

_**例子:**_

[gimmick:yuml]( [HttpContext]uses -.->[Response] )

    [gimmick:yuml]( [HttpContext]uses -.->[Response] )
- - -
[gimmick:yuml (type: 'class')]([User|+Forename;+Surname;+HashedPassword;-Salt])

    [gimmick:yuml]([User|+Forename+;Surname;+HashedPassword;-Salt|+Login();+Logout()])
- - -
[gimmick:yuml (type: 'activity', style: 'plain') ]( `Make Coffee´->`want more coffee´ )

    [gimmick:yuml (type: 'activity', style: 'plain') ]( `Make Coffee->`want more coffee )

- - -
[gimmick:yuml (type: 'usecase', scale: 150) ]( [Customer]-`Sign In, [Customer]-`Buy Products )

    [gimmick:yuml (diag: 'usecase', scale: 150) ]( [Customer]-`Sign In, [Customer]-`Buy Products )

Arguments

* **direction**
  * is one of [ 'TB', 'LR' ]
  * direction of the diagram: top-to-bottom or left-to-right
* **scale**
  * is an integer percentage value, i.e. 150 or 200
  * defines the scaling applied to the diagram in percent, 100% = no scaling
* **type**
  * is one of [ 'class', 'activity', 'usecase' ]
  * type of the UML diagram
* **style**
  * is one of [ 'plain', 'scruffy' ]
  * defines the applied theme, *plain* for clean and *scruffy* for comic-style look


Math
-----
[gimmick: math]()
Math formulas are realized through the [MathJax](http://www.mathjax.org) library. To enable math formulas on a page, the `math` gimmick must be loaded by adding this link anywhere in the file:

    [gimmick: math]()

To enable math for all sites, put the above link into the `navigation.md` file. Putting this link onto the site will load MathJax dynamically from a CDN provider.

Note: The MathJax script is very large and loads some more dependencies like fonts. Using the math gimmick might result in slow page loads.

### Math inserted to new paragraph  

You can add math formulas by putting them between to `$$` signs and use LateX syntax:

    $$ x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a} $$

$$ x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a} $$

### Inline equations

Inline equations can be inserted by surrounding them with the delimiters `\\(` and `\\)`:

    Diameter \\( d \\) of a circle given area \\( A \\) can by obtained via \\(d=\sqrt{\frac{4A}{\pi}}\\)

Will show as: Diameter \\( d \\) of a circle given area \\( A \\) can by obtained via \\(d=\sqrt{\frac{4A}{\pi}}\\)

### Some examples

$$ \frac{\partial \phi}{\partial x} \vert_b = \frac{1}{\Delta x/2}(\phi_0-\phi_b) $$

$$ \int u \frac{dv}{dx}\,dx=uv-\int
\frac{du}{dx}v\,dx\lim_{n\rightarrow \infty }
\left (  1 +\frac{1}{n} \right )^n
$$

Chart
-----

Adds a chart to the screen using data from your Markdown table.

#### Options

- labelColumn: This is a string that indicates which column will be used to label the data points. This String must be a direct match to your table header fro the column.
- dataColumns: This is an array of strings that indicated the column to be plotted. This String must be a direct match to your table header fro the column.
- canvasId: This is an ID for the generated chart. Defaults to a random number between 1-1000.
- chartOptions: This is an object that is passed to chartjs to configure its options. Refer to chartjs for documentation on available options.
- chartType:   This string is the type of chart render. Bar, Line, or Radar. Defaults to Line.

Note: Currently only support a single table on a page. You CAN have multiple charts from the same table.

    | #  | Sprint          | Points | Sum | Avg  | Note |
    | -  | --------        |------- | --- | ---- | ---- |
    | 1  | Sprint 1        | 6      | 6   | 6.0  | |
    | 2  | Sprint 2        | 6      | 12  | 6.0  | |
    | 3  | Sprint 3        | 15     | 27  | 9.0  | |
    | 4  | Sprint 4        | 9      | 36  | 9.0  | |
    | 5  | Sprint 5        | 6      | 42  | 8.4  | |
    | 6  | Sprint 6        | 9      | 51  | 8.5  | |
    
    [gimmick:chart ({dataColumns: ['Avg'], labelColumn: "Sprint", chartType: 'Line', width: '660px', height: '300px'})]()
    
    [gimmick:chart ({dataColumns: ['Avg'], labelColumn: "Sprint", chartType: 'Bar', width: '660px', height: '300px'})]()

Example:

| #  | Sprint          | Points | Sum | Avg  | Note |
| -  | --------        |------- | --- | ---- | ---- |
| 1  | Sprint 1        | 6      | 6   | 6.0  | |
| 2  | Sprint 2        | 6      | 12  | 6.0  | |
| 3  | Sprint 3        | 15     | 27  | 9.0  | |
| 4  | Sprint 4        | 9      | 36  | 9.0  | |
| 5  | Sprint 5        | 6      | 42  | 8.4  | |
| 6  | Sprint 6        | 9      | 51  | 8.5  | |

[gimmick:chart ({dataColumns: ['Avg'], labelColumn: "Sprint", chartType: 'Line', width: '660px', height: '300px'})]()

[gimmick:chart ({dataColumns: ['Avg'], labelColumn: "Sprint", chartType: 'Bar', width: '660px', height: '300px'})]()

* * *

GitHub Gists
------------

Gists on github can be embedded by passing their numeric id:

    [gimmick:gist](5641564)

效果:

[gimmick:gist](5641564)

* * *
Fork me on GitHub - Binke
--------------------------

The popular github binke that is also present on this page. 

Example:

    [gimmick:ForkMeOnGitHub](https://github.com/bin526976203/mdWikiBlog/)

or with options:

    [gimmick:ForkMeOnGitHub (position: 'left', color: 'darkblue') ](http://www.github.com/Dynalon/mdwiki)

Arguments:

* **color**
  * is one of [ 'red', 'darkblue', 'green', 'orange', 'white', 'gray' ]
  * defines the color of the ribbon
* **position**
  * is one of [ 'left', 'right' ]
  * defines the upper-corner position of the ribbon

Note: To display the ribbon on every page, put the gimmick link into the `navigation.md` file.

* * *

Disqus
------

Adds comment / forum style functionality to your website. You first need to [signup with disqus](http://disqus.com) and use your disqus shortname as the link target.

    [gimmick:Disqus](your_disqus_shortname)

Preview:

[gimmick:Disqus](mdwiki)

Disqus is always embedded at the bottom of a page, so scroll down this site to see a preview.