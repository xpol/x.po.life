---
title: Visual Studio Code 视频支持折腾记
date: 2019-05-12 20:27
---

由于项目需要，打算使用Markdown作为内容制作的格式。而项目中有不少内容是需要展示音频和视频的地方。也就是说需要让Markdown支持音频和视频格式。

项目程序员的非Vim党们使用的都是Visual Studio Code（后面简称Code）。所以自然想选择Code作为Markdown编辑器了。

所以问题归结为，如何让VSCode预览的时候把Markdown里面的影视频链接变成video和audio标签。

通过一番搜索，发现Code内部Markdown预览使用的是markdown-it，并且提供了扩展markdown-it的接口。
同时markdown-it又现成的markdown-it-html5-embed插件。所以应该是写一个小插件就能了事的。

## 第一回：Code插件尝试

写个简单的Code插件，为markdown-it加载跟多的插件，只要几行代码：

```javascript
function activate(context) {
    return {
        extendMarkdownIt(md) {
            return md
                .use(require("markdown-it-html5-embed"), h5embedOptions);
        }
    }
}
```

挺简单吧，不过如果这就是结局，那也太简单了。

发布代码，测试。

什么灰色的影视频按钮？什么鬼？

继续搜索，Code和音视频支持，结果发现：Code居然在发布的时候移除了全部ffmpeg功能，而且微软不打算支持在Code中使用音视频。

等等，Code不是开源的吗？我自己编译不就好了。

## 第二回：自己编译Code

说干就干，先克隆代码： [https://github.com/microsoft/vscode.git](https://github.com/microsoft/vscode.git)

然后编译，本地运行，看起来都OK。就是运行的时候似乎有些小问题。

再试试打包成 App，结果，正如多数大型JavaScript项目打包一样，他崩溃了。

我只想要一个简单功能，即便能打包成功，每次编译Code也是太麻烦了吧。

等等，我应该不是第一个想吃螃蟹的人，先前一定有人想过这么干。Google一下，还真有。

## 第三回：VSCodium

是的VSCodium项目就是提供社区编译版本的Code。下载试试，音频视频都能播放。太好了，看来可以结案了。

过了半天，突然想给内容编辑同事展示一下VSCodium里面编辑带音视频的Markdown的情况。先在自己电脑上试试，诶，有点不对，视频这么又灰了。不对是MP4灰了，Webm格式还是可以的。

那么我们继续安装换编辑器的思路，试试Atom把，毕竟以前还用过Atom。

## 第四回：Atom

Atom里面用Markdown内嵌HTML标签的方式先试试。不错，MP4，MP3格式都可以啊。

当然还是有些不甘心，毕竟现在Atom维护和流畅都比不上Code。尝试把Atom的FFmpeg库拷贝到VSCodium里面，结果预览直接黑屏。放弃，继续验证Atom的思路走下去。

那么就是要让Markdown预览支持把音视频链接换成audio、video标签。发现比较流行的Markdown Preview Enhanced。但是问题又来了，这个插件没有直接使用markdown-it，而是使用了作者自己的mume模块。

去看看mume的代码，但 `package.json` 里面并没有依赖markdown-it，甚至根本没有第三方markdown模块的依赖。

只好去Markdown Preview Enhanced项目提功能需求。看了看作者还是深圳的，萌生了想付费让作者帮忙支持这个功能的想。打开邮箱，写邮件……

等等，mume应该不会自己写markdown解析，再仔细找找。果然，有个依赖目录，里面真有markdown-it及几个插件的源代码。

那么问题又简单了，赶紧为mume写个PR。提过去后，过了一段时间，收到作者回复，说可以接受PR，需要做些小的修改啥的。顺手为几个配置项，然后顺手把Markdown Preview Enhanced的PR也写了。

带着本地版本的插件，给内容同事演示了一把，基本上满意。

好的，接下来就等作者合并发版了。

## 第五回：峰回路转

虽然问题基本上解决了，但是Atom毕竟现在启动速度和插件方面不如Code了。而且要同时安装Code和Atom有点麻烦。然而压死骆驼的最后一根稻草是Atom的Markdown高亮插件不如Code的。

还是Debug一下为什么不能播放音视频，是在chromium内核层面还是只是剔除了相关的FFmpeg实现？
通过JavaScript接口发现，chromium还是声称能支持MP4、MP3的。

那么应该是可以替换ffmpeg库的。之前替换VSCodium失败应该是ffmpeg版本不一致导致的。

再次Google，发现[有个项目专门为NWjs编译ffmpeg](https://github.com/iteufel/nwjs-ffmpeg-prebuilt)。NWjs和Electron（Code和Atom的运行平台）原理差不多。同时记得之前编译Code的时候，在网上下载的是ffmpeg 3.8 的版本，于是下载一个 3.8.1版本的预编译的ffmpeg。替换一下，居然OK了。居然就这么样OK了。

所以最后的办法还是回到的Code，替换ffmpeg库的方式。写个 install.sh 脚本：

```bash
#!/usr/bin/env bash

curl -sL -o libffmpeg.zip https://github.com/iteufel/nwjs-ffmpeg-prebuilt/releases/download/0.38.1/0.38.1-osx-x64.zip
unzip libffmpeg.zip
mv ./libffmpeg.dylib "/Applications/Visual Studio Code.app/Contents/Frameworks/Electron Framework.framework/Versions/A/Libraries/libffmpeg.dylib"
```

当然，这里还可以通过短连接等方式让每次下载的脚本是最新的ffmpeg。具体方法这里就不展开了。

这里我吧他放到这个[gist](https://gist.github.com/xpol/483ec9967d0a3d14791374947ecf0ec8)这样就可以通过 curl pipe bash的方式安装。

```bash
curl -s https://gist.githubusercontent.com/xpol/483ec9967d0a3d14791374947ecf0ec8/raw/4083e7563f48b9de57503932fce6c2512f70f5c8/install.sh | bash
```

最后再写个文档，教非编辑同事如何打开终端运行上面这段代码。

完美！（*希望这个不是很快倒下的flag*）

## 后记

曲折的过程为最终解决方案积累信息。

不去编译一下Code，可能不知道要什么版本的ffmpeg。不先给Code写个插件，那么后面替换ffmpeg的方案可能也要曲折一些。

当收集到的信息足够且充分连接的时候，得出理想的方案只是水到渠成而已。
