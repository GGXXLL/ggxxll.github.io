---
title: SublimeText3+OmniMarkupPreviewer出现404问题解决方案
tags: 
  - SublimeText3
date: 2022-08-24
categories:
  - 工具
---

某日突然发现`Sublime Text3（3.2.2）`插件`OmniMarkupPreviewer`打开页面出现404错误
```
Error: 404 Not Found
Sorry, the requested URL 'http://127.0.0.1:51004/view/26' caused an error:

'buffer_id(26) is not valid (closed or unsupported file format)'

**NOTE:** If you run multiple instances of Sublime Text, you may want to adjust
the `server_port` option in order to get this plugin work again.
```

查看`Console`窗口出现如下错误：
```
OmniMarkupPreviewer: [ERROR] Exception occured while rendering using MarkdownRenderer
  Traceback (most recent call last):
    File "%AppData%\Roaming\Sublime Text 3\Packages\OmniMarkupPreviewer\OmniMarkupLib\RendererManager.py", line 266, in render_text
    rendered_text = renderer.render(text, filename=filename)
    File "%AppData%\Roaming\Sublime Text 3\Packages\OmniMarkupPreviewer\OmniMarkupLib\Renderers/MarkdownRenderer.py", line 48, in render
    extensions=self.extensions)
    File "%AppData%\Roaming\SUBLIM~1\Packages\PYTHON~3\st3\markdown\core.py", line 390, in markdown
    md = Markdown(**kwargs)
    File "%AppData%\Roaming\SUBLIM~1\Packages\PYTHON~3\st3\markdown\core.py", line 100, in __init__
    configs=kwargs.get('extension_configs', {}))
    File "%AppData%\Roaming\SUBLIM~1\Packages\PYTHON~3\st3\markdown\core.py", line 126, in registerExtensions
    ext = self.build_extension(ext, configs.get(ext, {}))
    File "%AppData%\Roaming\SUBLIM~1\Packages\PYTHON~3\st3\markdown\core.py", line 166, in build_extension
    module = importlib.import_module(ext_name)
    File "./python3.3/importlib/__init__.py", line 90, in import_module
    File "<frozen importlib._bootstrap>", line 1584, in _gcd_import
    File "<frozen importlib._bootstrap>", line 1565, in _find_and_load
    File "<frozen importlib._bootstrap>", line 1529, in _find_and_load_unlocked
  ImportError: No module named 'xxx'
```

## 原因
`OmniMarkupPreviewer`插件使用`python-markdown@3.2.2`

`Sublime Text3`使用`python-markdown@3.1.1`

当`sys.path`中的路径下有相同包名时，`import`会优先导入路径靠前的包

所以`Sublime Text3`按照启动顺序加载`sys.path`后，会优先使用`python-markdown@3.1.1`，出现导包错误

## 解决方案
本人是windows系统，使用`%AppData%\Sublime Text 3\Packages\OmniMarkupPreviewer\OmniMarkupLib\Renderers\libs\markdown`文件夹覆盖`%AppData%\Sublime Text 3\Packages\python-markdown\st3\markdown`

然后重启`Sublime Text3`即可