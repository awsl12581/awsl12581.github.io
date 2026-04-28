[[Activate Environments in Terminal Using Environment Variables](https://aka.ms/vscodePythonTerminalActivation)](https://aka.ms/vscodePythonTerminalActivation)

[[Extension fails to activate unnamed conda environments · Issue #24095 · microsoft/vscode-python](https://github.com/microsoft/vscode-python/issues/24095)](https://github.com/microsoft/vscode-python/issues/24095)

具体来说为：在使用`ctrl+shift+``切出终端时，vscode并没有激活已经选定的Python解释器。

按照如上说法，需要把settings中的**native locator**从native切换至js

![](https://cdn.nlark.com/yuque/0/2025/png/21764230/1739775063299-11c67f34-db02-4611-9d7b-475463a2ee06.png)