# 在 macOS 上配置 VSCode 与 Skim 的 LaTeX 正反跳转

https://liam.page/2018/04/24/Working-with-VSCode-on-macOS-configuration-LaTeX-workshop-and-Skim/

VSCode 是微软主导开发的新一代编辑器。自其开发之初，就与 Sublime Text 以及 GitHub 主导开发的 Atom 对标。几年前，VSCode 中的 LaTeX 支持还很不完善，考虑到我个人对 LaTeX 的强需求，当时没有从 Sublime Text 切换到 VSCode 上。时至今日，VSCode 发展得已经很不错。前些日子，东升在他的新主页上发布了一篇博文，讲解[如何在当前的 VSCode 上配置 LaTeX IDE](http://ddswhu.me/posts/2018-04/vs-code-for-latex/)。看过之后，我就心动了，立即配置好来使用。

不过，由于东升不在 macOS 下工作，他的博文中没有提到如何让 VSCode 在 macOS 上与诸如 Skim 的外部 PDF 浏览器配合工作——特别是 LaTeX 的正反跳转。检索互联网之后，也没有完整可用的方法。甚至 LaTeX workshop 官方的说法也是不支持，需要用户自己想办法绕过。故此有这篇文章。



## 正向搜索

正向搜索指的是从 `.tex` 源文件向编译生成的 PDF 文件的跳转。这部分我们主要需要寻找到合适的方式，在命令行状态打开 Skim 应用。

根据 [Skim 官网的介绍](https://sourceforge.net/p/skim-app/wiki/TeX_and_PDF_Synchronization/)，Skim 提供了名为 `displayline` 的脚本，用于和 TeX 正反同步协同工作。其脚本位于

```
/Applications/Skim.app/Contents/SharedSupport/displayline
```

具体用法则是

```
/Applications/Skim.app/Contents/SharedSupport/displayline %line "%pdffile" "%texfile"
```

此处 `%line` 表示 `.tex` 文件的行号，`%pdffile` 表示编译生成的 PDF 文件的完整路径，`%texfile` 则是对应的 `.tex` 源文件。考虑到我们对反向搜索也有需求，按照官方文档，我们还需要加上 `-r` 参数。

因此，在 VSCode 的配置里，我们需要给出这样的 JSON 配置。

```
"latex-workshop.view.pdf.viewer": "external",
"latex-workshop.view.pdf.external.synctex": {
    "command": "/Applications/Skim.app/Contents/SharedSupport/displayline",
    "args": [
        "-r",
        "%LINE%",
        "%PDF%",
        "%TEX%"
    ]
},
```

LaTeX workshop 插件的配置中，除了 `latex-workshop.view.pdf.external.synctex`，还有一项也和打开外部 PDF 浏览器有关。它是 `latex-workshop.view.pdf.external.command`，表示简单地打开 PDF 文件，而忽略正向搜索功能。Skim 提供的命令行脚本默认不支持这样的操作，我们需要根据 `displayline` 脚本自行修改。

`displayline` 脚本调用了 Apple Script 来实现与 Skim.app 的交互，此处我们照葫芦画瓢，模拟一个即可。我们可以将如下脚本保存为 `displayfile` 并以 `chmod u+x displayfile` 使其变为可执行脚本，而后将其放在环境变量 `PATH` 包含的目录中。

> 你可以在终端（Terminal.app）中执行 `echo ${PATH}` 来查看你的环境变量。通常，命令会返回以冒号 `:` 分隔的若干个目录。你可以从中选择一个合适的目录，或者干脆只是你喜欢的目录，保存这个文件。

```
#!/bin/bash

# displayfile (Skim)
#
# Usage: displayfile [-r] [-g] PDFFILE
#
# Modified from "displayline" to only revert the file, not jump to a given line
#

if [ $# == 0 -o "$1" == "-h" -o "$1" == "-help" ]; then
  echo "Usage: displayfile [-r] [-g] PDFFILE
Options:
-r, -revert      Revert the file from disk if it was open
-g, -background  Do not bring Skim to the foreground"
  exit 0
fi

# get arguments
revert=false
activate=true
while [ "${1:0:1}" == "-" ]; do
  if [ "$1" == "-r" -o "$1" == "-revert" ]; then
    revert=true
  elif [ "$1" == "-g" -o "$1" == "-background" ]; then
    activate=false
  fi
  shift
done
file="$1"
#shopt -s extglob
#[ $# -gt 2 ] && source="$3" || source="${file%.@(pdf|dvi|xdv)}.tex"

# expand relative paths
[ "${file:0:1}" == "/" ] || file="${PWD}/${file}"

# pass file arguments as NULL-separated string to osascript
# pass through cat to get them as raw bytes to preserve non-ASCII characters
/usr/bin/osascript \
  -e "set theFile to POSIX file \"$file\"" \
  -e "set thePath to POSIX path of (theFile as alias)" \
  -e "tell application \"Skim\"" \
  -e "  if $activate then activate" \
  -e "  if $revert then" \
  -e "    try" \
  -e "      set theDocs to get documents whose path is thePath" \
  -e "      if (count of theDocs) > 0 then revert theDocs" \
  -e "    end try" \
  -e "  end if" \
  -e "  open theFile" \
  -e "end tell"
```

至此，我们需要更新 VSCode 的配置。

```
"latex-workshop.view.pdf.external.command": {
    "command": "displayfile",
    "args": [
        "-r",
        "%PDF%"
    ]
},
```

## 反向搜索

再反向搜索中，我们需要让 Skim 能够打开 VSCode，并将文件名和相应的行号正确地传给 VSCode。根据 [Skim 官网的介绍](https://sourceforge.net/p/skim-app/wiki/TeX_and_PDF_Synchronization/)，它只会从 `/usr/bin` 和 `/usr/local/bin` 中检索可执行程序。为此，我们需要将 VSCode 的可执行程序软链到这两个目录中的一个当中。

```
ln -sf /Applications/Visual\ Studio\ Code.app/Contents/MacOS/Electron /usr/local/bin/vscode
```

至此，我们在命令行中执行 `vscode -g "%file":%line` 即可打开 `%file` 文件并定位到第 `%line` 行。于是，我们只需要在 Skim 的同步配置中将预设改为「自定义」，并将命令和参数分别设置如下：

- Command: `vscode`
- Arguments: `-g "%file":%line`

如此即完成了配置。