---
title: 修改WP Editor.md插件使WordPress支持\$...\$格式的Latex行内公式
layout: post
date: 2017-07-29
categories: 架构
tags: [架构, 分析, Markdown, WordPress]
---

修改WP Editor.md插件使WordPress支持\$...\$格式的Latex行内公式
==========
### 问题描述
WordPress博客搭建好了，第一篇文章也写好了，打开写文章页面，把Markdown格式的文章往上一贴，以为万事大吉了，就要去点发布。
忽然，发现有点奇怪，为什么这个页面是用来编辑HTML格式的文本的？再一看，默认的编辑模式是可视化和纯文本，难道不支持Markdown？

  
外事不决问Google，一看果然不行，要装插件。于是进了插件页面，随手一搜，发现有好多支持Markdown的，评价都还行，就随便找了一个兼容
当前版本的就装上了。装完回来，页面果然变了，那就赶紧发布吧。拷贝好文章往上一贴，嗯，看起来不错。
  
嗯？公式怎么没有变化？继续Google。哦，普通的Markdown还不支持Latex公式，需要找一个支持的，或者另外装一个Mathjax的工具。为一件事情装两个插件这种事我是不会干的，
那就再找找吧。好，WP Editor.md看起来不错，界面很漂亮，就它了。装好了，设置一下支持Latex，把文章再往上一贴，搞定。诶？不对，行间
的公式是出来了，行内的公式怎么没什么变化？还是老样子？继续问Google吧。再一通搜索，发现WordPress上的插件就没有支持\$...\$这种
格式写行间公式的，最多也就支持个\$latex 公式\$。这就尴尬了，我要调整成这种格式，普通的查找替换还不行，得一个一个改，我那么多公式，
改到什么时候去啊？更何况我怎么着也是个程序员，怎么能干这种纯苦力活呢？更何况我的Subline搭配OmniMarkupPreview都能支持这种模式，
工作的棒棒的，可不能因为这点小问题就换工作环境啊！继续找……
  
找来又找去，最后找到Mathjax的官网上去了，仔细一看，从根子上人家就不建议用这个，说这种情况太常见了，容易弄错，本来好好写两个美元价格，
结果一不小心就判断成公式了，不合适。不过人家也给了一个解决方案，就是在页面的引用Mathjax的script里边加入这么一段：

	<script type="text/x-mathjax-config">
	MathJax.Hub.Config({
	    tex2jax: {
	      inlineMath: [ ['$','$'], ["\\(","\\)"] ],
	      displayMath: [ ['$$', '$$'], ["\\[", "\\]"] ],
	      processEscapes: true
	    },
	  });
	</script>

这样就支持啦。好办法，就这么干了，进了WP Editor.md的编辑页面，就开始找跟script有关的代码，发现怎么都找不到。难道WP Editor.md不是用的Mathjax？
进了它的Github页面仔细看了一下，好嘛，还真不是，用的是Katex，号称全网最快的Latex公式解析器。

### 解决方案

问题就是这么个问题，情况也就是这么个情况。Katex就Katex吧，先看看Katex是怎么干活的。把刚刚生成的网页的源代码调出来，检查了一下正确转换的公式，代码的格式是这样的：

	<p><script type="text/javascript">document.write(katex.renderToString("B(t) = P_0 + (P_1 - P_0)t = (1 - t)P_0 + tP_1, t\\in[0,1]"));</script></p>
  
这就好办了，把WP Editor.md的工程拷贝下来，用Android Studio打开，搜一下哪里有document.write(katex.renderToString。这一搜还真搜找了，
不在期望了Katex文件夹下面，而在Jetpack文件夹下面的Latex文件夹里边的latex.php文件，看来是Jexpack调用了Katex来处理Markdown的格式转换。
找到了地方就可以开始干活了，主要负责模式提取的函数叫latex_markup，长这样：

	function latex_markup( $content ) {
		$textarr = wp_html_split( $content );
		
		$regex = '%
			\$\$(?:=*|\s+)
			((?:
				[^$]+ # Not a dollar
			|
				(?<=(?<!\\\\)\\\\)\$ # Dollar preceded by exactly one slash
			)+)
			(?<!\\\\)\$\$ # Dollar preceded by zero slashes
		%ix';
		
		foreach ( $textarr as &$element ) {
			if ( '' == $element || '<' === $element[0] ) {
				continue;
			}

			if ( false === stripos( $element, "$$" ) ) {
				continue;
			}

			$element = preg_replace_callback( $regex, 'latex_src', $element );
		}

		return implode( '', $textarr );
	}
嗯，典型的模式提取，regex已经写的很清楚了，提取的就是\$\$...\$\$这种模式的，要改成\$...\$这种模式的话，删掉一个就可以了，简单又明了。
但是删掉了不就解析不了行间模式了吗？两个都得支持呀，那怎么办？直接改这个函数吧，增加一种新的模式，发现两个\$就用行间模式，发现只有一个\$就用行内模式。
于是修改了一下函数，变成了这个样子：

	function latex_markup( $content ) {
		$textarr = wp_html_split( $content );
		
		$regex = '%
			\$\$(?:=*|\s+)
			((?:
				[^$]+ # Not a dollar
			|
				(?<=(?<!\\\\)\\\\)\$ # Dollar preceded by exactly one slash
			)+)
			(?<!\\\\)\$\$ # Dollar preceded by zero slashes
		%ix';
	    $regex_inline = '%
	        \$(?:=*|\s+)
	        ((?:
	            [^$]+ # Not a dollar
	        |
	            (?<=(?<!\\\\)\\\\)\$ # Dollar preceded by exactly one slash
	        )+)
	        (?<!\\\\)\$ # Dollar preceded by zero slashes
	    %ix';
		
		foreach ( $textarr as &$element ) {
			if ( '' == $element || '<' === $element[0] ) {
				continue;
			}

			if ( false === stripos( $element, '$$' ) ) {
			    if(false === stipos( $element, '$' )){
				    continue;
				}
				else
				{
			        $element = preg_replace_callback( $regex_inline, 'latex_src', $element );
			        continue;
				}
			}
			$element = preg_replace_callback( $regex, 'latex_src', $element );
		}

		return implode( '', $textarr );
	}
  
看起来不错吧，一口气处理两种情况，很清晰。再发布一下看看吧，一发布，失败，行间公式依然是正确的，行内公式依然没有解析。为什么呢？
带着这个问题，我开始对代码进行各种破坏性的修改，期望看到一点点变化，此处就略过不讲了，直到最后我忽然发现文件的最底下有这么两行字：
  
	add_filter( 'the_content', 'latex_markup', 9 ); // before wptexturize
	add_filter( 'comment_text', 'latex_markup', 9 ); // before wptexturize
  
看起来是把这个函数注册到了某个处理流程里边。流程化处理一般最好是一次只处理一种情况，要不然容易错乱，问题会不会出再这呢？那我是不是
可以写一个单独处理行内模式的函数注册进去？想到就做，于是我又写了一个只处理行内模式的函数：
  
	function latex_inline_markup( $content ) {
		$textarr = wp_html_split( $content );
		
		$regex = '%
			\$(?:=*|\s+)
			((?:
				[^$]+ # Not a dollar
			|
				(?<=(?<!\\\\)\\\\)\$ # Dollar preceded by exactly one slash
			)+)
			(?<!\\\\)\$ # Dollar preceded by zero slashes
		%ix';

		foreach ( $textarr as &$element ) {
			if ( '' == $element || '<' === $element[0] ) {
				continue;
			}

			if ( false === stripos( $element, '$' ) ) {
				continue;
			}

			$element = preg_replace_callback( $regex, 'latex_src', $element );
		}

		return implode( '', $textarr );
	}
看起来跟行间模式简直一模一样，然后再注册行间模式的两句之后加上了这两句。
  

	add_filter( 'the_content', 'latex_inline_markup', 9 ); // before wptexturize
	add_filter( 'comment_text', 'latex_inline_markup', 9 ); // before wptexturize
好了，看起来很优雅，再跑一下看看，成功啦！！一朝被蛇咬，十年怕井绳，因为问题出的太多了，我又不放心，又仔细检查了一遍，一检查还真发现问题了，
公式里边的\in没有转换成对应的符号$\in$，不过这下有经验了，先试着把\改成\\\\再发布一次，果然正常了，那么再回到刚才的文件里边，找到给公式做格式转换的函数
  
	function latex_entity_decode( $latex ) {
		return str_replace( array( '&lt;', '&gt;', '&quot;', '&#039;', '&#038;', '&amp;', "\n", "\r", "&#92;", "&#40;", "&#41;", "&#95;","&#33;" ), array( '<', '>', '"', "'", '&', '&', ' ', ' ', '\\\\', '(', ')', '_', '!' ), $latex );
	}

给第一个array加上一个"\\"，第二个array加上一个"\\\\",完成！
  
直到改完这个插件，我才发现，它是用php做的，可是我没学过php啊！！以后是不是我可以号称我会php了？？