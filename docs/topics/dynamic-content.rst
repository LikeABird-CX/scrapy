.. _topics-dynamic-content:

====================================
选择动态加载的内容
====================================

当你在网络浏览器中加载网页时，有些网页显示了所需的数据。
然而，当你用Scrapy下载它们时，你使用 :ref:`selectors <topics-selectors>` 却无法得到所需的数据.

当这种情况发生时，推荐的方法是 :ref:`找到数据源<topics-finding-data-source>` 并从中提取数据。

如果你没有做到这一点，但你仍然可以通过 :ref:`DOM <topics-livedom>` 从你的网络浏览器
访问所需的数据,请参阅 :ref:`topics-javascript-rendering` 。

.. _topics-finding-data-source:

寻找数据源
=======================

为了提取所需的数据，你必须首先找到它的源位置。

如果数据是基于非文本的格式，如图像或PDF文档。
使用你的网络浏览器的 :ref:`network tool <topics-network-tool>` 来查找相应的请求，
并 :ref:`复制请求 <topics-reproducing-requests>`。

如果你的网络浏览器允许你选择所需的数据作为文本，这些数据可能是在嵌入的JavaScript代码中定义，
或者以基于文本的格式从外部资源加载。

在这种情况下，你可以使用像 wgrep_ 这样的工具来找到该资源的URL。

如果事实证明数据来自原始的URL本身，你必须 :ref:`检查网页的源代码<topics-inspecting-source>` 以确定数据的位置。

如果数据来自一个不同的URL，你将需要 :ref:`复制相应的请求<topics-reproducing-requests>` 。

.. _topics-inspecting-source:

检查网页的源代码
=======================================

有时你需要检查一个网页的源代码（而不是 :ref:`DOM <topics-livedom>` ），以确定一些所需数据的位置。

使用Scrapy的 :command:`fetch` 命令来下载网页的内容，就像Scrapy看到的那样的内容:

::
  
  scrapy fetch --nolog https://example.com > response.html

如果所需的数据是在 ``<script/>`` 元素内的嵌入式JavaScript代码中, 请参见 :ref:`topics-parsing-javascript` 。

如果你找不到所需的数据，首先要确定不仅仅Scrapy是这样的。
用 curl_ 或 wget_ 等HTTP客户端下载网页，看看是否能在它们的响应中找到该信息。

如果他们得到一个带有所需数据的响应，请修改你的Scrapy :class:`~scrapy.http.Request` ，
使之与其他HTTP客户端的匹配。例如尝试使用相同的用户代理字符串（ :setting:`USER_AGENT` ）或
相同的 :attr:`~scrapy.http.Request.headers` 。

如果他们也得到一个没有所需数据的响应，你需要采取措施，使你的请求与网络浏览器的请求更加相似。
参见 :ref:`topics-reproducing-requests` 。

.. _topics-reproducing-requests:

复制请求
====================

有时我们需要按照我们的网络浏览器执行的方式来重现一个请求。

使用你的网络浏览器的 :ref:`网络工具<topics-network-tool>` 来查看你的网络浏览器如何执行所需的请求,
并尝试用Scrapy重现该请求。

它可能足以yield一个 :class:`~scrapy.http.Request` ，具有相同的HTTP方法和URL。
然而，你可能还需要重现body、headers和表单参数（见 :class:`~scrapy.http.FormRequest` ）的请求。

由于所有主要的浏览器都允许以 `cURL <https://curl.haxx.se/>`_ 格式的请求，
Scrapy采用了以下方法 :meth:`~scrapy.http.Request.from_curl()` 来生成一个
等效的 :class:`~scrapy.http.Request` 来自cURL命令。要获得更多信息
请访问 :ref:`request from curl <requests-from-curl>` 内的网络工具部分。

一旦你得到了预期的响应，你可以 :ref:`从中提取所需的数据 <topics-handling-response-formats>` 。

你可以用Scrapy重现任何请求。然而，有些时候重现所有必要的请求，在开发人员的时间里可能显得不那么有效。
如果这是你的情况，而且抓取速度对你来说不是一个主要的问题，你可以选择
考虑 :ref:`用JavaScript进行预渲染 <topics-javascript-rendering>` 。

如果你"有时"得到预期的响应，但并不总是如此，那么问题就在于
可能不是你的请求，而是目标服务器的问题。目标服务器可能是
错误，超载，或者 :ref:`禁止<bans>` 你的一些请求。

注意，要把cURL命令翻译成Scrapy请求。
你可以使用 `curl2scrapy <https://michael-shub.github.io/curl2scrapy/>`_ .

.. _topics-handling-response-formats:

处理不同的响应格式
===================================

一旦你有一个带有所需数据的响应，你如何从其中提取所需的
数据取决于响应的类型:

- 如果响应是HTML或XML，使用 :ref:`selectors <topics-selectors>` ，和平常一样。

- 如果响应是JSON，使用 :func:`json.load()` 来加载想要的数据，
  从 :attr:`response.text <scrapy.http.TextResponse.text>` :

  ::

    data = json.load(response.text)

  如果所需的数据是在HTML或XML代码内嵌入JSON数据,
  你可以把这些HTML或XML代码加载到一个 :class:`~scrapy.selector.Selector` ，
  然后 :ref:`使用它<topics-selectors>` 像往常一样。

  ::
    
    Selector = Selector(data['html'])

- 如果响应是JavaScript，或带有 ``<script/>`` 元素的HTML，其中包含所需的数据。
  请参阅 :ref:`topics-parsing-javascript` 。

- 如果响应是CSS，使用 :doc:`正则表达式 <library/re>` 从 :attr:`response.text <scrapy.http.TextResponse.text>` 提
  取需要的数据。

.. _topics-parsing-images。

- 如果响应是一个图像或其他基于图像的格式（例如PDF）。
  从 :attr:`response.body <scrapy.http.TextResponse.body>` 中读取响应的字节数，
  并使用OCR解决方案来提取所需的文本数据。

  例如，你可以使用 pytesseract_ 。从PDF中读取一个表格, `tabula-py`_ 可能是一个更好的选择。

- 如果响应是SVG，或者是嵌入了SVG的HTML，包含了所需的
  数据，你就可以用 :ref:`selectors <topics-selectors>` 来提取所需的数据，因为SVG是基于XML的。

  另外，你可能需要将SVG代码转换为光栅图像，并 :ref:`处理该光栅图像<topics-parsing-images>` 。

.. _topics-parsing-javascript:

解析JavaScript代码
=======================

如果想要的数据是在JavaScript中硬编码的，你首先需要得到JavaScript编码:

- 如果JavaScript代码是在一个JavaScript文件中，只需读取 :attr:`response.text <scrapy.http.TextResponse.text>` 。

- 如果JavaScript代码在一个HTML页面的 ``<script/>`` 元素中。
  使用 :ref:`selectors <topics-selectors>` 来提取 ``<script/>`` 元素中的文本。

一旦你有了一个带有JavaScript代码的字符串，你就可以从中提取所需的数据:

- 你也许可以使用 :doc:`正则表达式 <library/re>` 以JSON格式提取所需的数据，然后你可以
  用 :func:`json.load()` 解析:

  例如，如果JavaScript代码包含一个单独的行，像 ``var data = {"field": "value"};`` 一样,
  你可以提取这些数据，如下所示:

  :: 

    >>> pattern = r'\bvar\s+data\s*=\s*(\{.*?\})\s*;\s*\n'
    >>> json_data = response.css('script::text').re_first(pattern)
    >>> json.loads(json_data)
    {'field': 'value'}

- chompjs_提供了一个API，将JavaScript对象解析为 :class:`dict` 。

  例如，如果JavaScript代码包含 ``var data = {field: "value", secondField: "第二个值"};`` 你可以
  按以下方式提取该数据:

  ::

    >>> import chompjs
    >>> javascript = response.css('script::text').get()
    >>> data = chompjs.parse_js_object(javascript)
    >>> data
    {'field': 'value', 'secondField': 'second value'}  

- 另外，使用 js2xml_ 将JavaScript代码转换成一个XML文档,
  这样你就可以使用 :ref:`selectors <topics-selectors>` 来解析:

  例如，如果JavaScript代码包含 ``var data = {field: "value"};`` 你可以按下面的方式提取该数据:

  ::

    >>> import js2xml
    >>> import lxml.etree
    >>> from parsel import Selector
    >>> javascript = response.css('script::text').get()
    >>> xml = lxml.etree.tostring(js2xml.parse(javascript), encoding='unicode')
    >>> selector = Selector(text=xml)
    >>> selector.css('var[name="data"]').get()
    '<var name="data"><object><property name="field"><string>value</string></property></object></var>'  

.. _topics-javascript-rendering:

预先渲染JavaScript
========================

在从其他请求中获取数据的网页上，复制那些包含所需数据的请求是首选方法。
这种努力往往是值得的：结构化的、完整的数据，解析时间和网络传输最少。

然而，有时可能真的很难重现某些请求。或者你可能需要一些任何请求都不能给你的东西，
比如在网页浏览器中看到的网页的截图。

在这些情况下，请使用 Splash_ 的JavaScript渲染服务，同时使用 `scrapy-splash`_ 进行无缝集成。

Splash以HTML形式返回网页的 :ref:`DOM <topics-livedom>` ，以便你可以用 :ref:`selectors <topics-selectors>` 来
解析它。它通过 configuration_ 或 scripting_ 提供了极大的灵活性。

如果你需要超越Splash所提供的东西，比如从Python代码中与DOM进行即时交互，而不是使用先前写好的脚本，
或者处理多个网络浏览器窗口，你可能
需要使用 :ref:`使用无头浏览器<topics-headless-browsing>` 来代替。

.. _configuration: https://splash.readthedocs.io/en/stable/api.html
.. _scripting: https://splash.readthedocs.io/en/stable/scripting-tutorial.html

.. _topics-headless-browsing:

使用无头浏览器
========================

`headless browser`_ 是一种特殊的网络浏览器，为自动化提供了一个API。

使用Scrapy的无头浏览器的最简单方法是使用 Selenium_ ，同时使用 `scrapy-selenium`_ 进行无缝集成。

.. _AJAX: https://en.wikipedia.org/wiki/Ajax_%28programming%29
.. _chompjs: https://github.com/Nykakin/chompjs
.. _CSS: https://en.wikipedia.org/wiki/Cascading_Style_Sheets
.. _curl: https://curl.haxx.se/
.. _headless browser: https://en.wikipedia.org/wiki/Headless_browser
.. _JavaScript: https://en.wikipedia.org/wiki/JavaScript
.. _js2xml: https://github.com/scrapinghub/js2xml
.. _pytesseract: https://github.com/madmaze/pytesseract
.. _scrapy-selenium: https://github.com/clemfromspace/scrapy-selenium
.. _scrapy-splash: https://github.com/scrapy-plugins/scrapy-splash
.. _Selenium: https://www.selenium.dev/
.. _Splash: https://github.com/scrapinghub/splash
.. _tabula-py: https://github.com/chezou/tabula-py
.. _wget: https://www.gnu.org/software/wget/
.. _wgrep: https://github.com/stav/wgrep 