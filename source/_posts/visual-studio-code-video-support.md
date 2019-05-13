---
title: Visual Studio Code 视频支持折腾记
date: 2019-05-12 20:27
---

由于项目需要，打算使用 Markdown 作为内容制作的格式。而项目中有不少内容是需要展示音频和视频的地方。也就是说需要让 Markdown 支持音频和视频格式。

项目程序员中的非 Vim 党们使用的都是 Visual Studio Code（后面简称 Code ）。所以自然想选择 Code 作为 Markdown编辑器了。

所以问题归结为，如何让 Code 预览的时候把 Markdown 里面的影视频链接变成 video 和 audio 标签。

通过一番搜索，发现 Code 内部 Markdown 预览使用的是 markdown-it，并且提供了扩展 markdown-it 的接口。
同时 markdown-it 有现成的 markdown-it-html5-embed 插件可以完成渲染 video 和 audio 标签。所以应该是写一个小插件就能了事的。

## 第一回：Code 插件尝试

写个简单的 Code 插件，为 markdown-it 加载跟多的插件，只要几行代码：

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

继续搜索，Code 和音视频支持，结果发现：Code 居然在发布的时候移除了全部 FFmpeg 功能，而且微软不打算支持在 Code 中使用音视频。

思考人生……

思考人生……

Code 不是开源的吗？我自己编译不就好了。

## 第二回：自己编译Code
说干就干，先克隆代码： https://github.com/microsoft/vscode.git

然后编译，本地运行，看起来都 OK。就是运行的时候似乎有些小问题。

再试试打包成  App，结果，正如多数大型 JavaScript 项目打包一样，他崩溃了。

我只想要一个简单功能，即便能打包成功，每次编译 Code 也是太麻烦了吧。

等等，我应该不是第一个想吃螃蟹的人，先前一定有人想过这么干。Google 一下，还真有。

## 第三回：VSCodium
是的 VSCodium 项目就是提供社区编译版本的 Code。下载试试，音频视频都能播放。太好了，看来可以结案了。

过了半天，突然想给内容编辑同事展示一下 VSCodium 里面编辑带音视频的Markdown 的情况。先在自己电脑上试试，诶，有点不对，视频这么又灰了。不对是 MP4 灰了，WebM 格式还是可以的。

那么我们继续安装换编辑器的思路，试试 Atom 把，毕竟以前还用过 Atom。

## 第四回：Atom
Atom 里面用 Markdown 内嵌 HTML 标签的方式先试试。不错，MP4，MP3格式都可以啊。

当然还是有些不甘心，毕竟现在 Atom 维护和流畅都比不上 Code。尝试把 Atom 的 FFmpeg 库拷贝到 VSCodium 里面，结果预览直接黑屏。放弃，继续验证 Atom 的思路走下去。

那么就是要让 Markdown 预览支持把音视频链接换成 audio、video 标签。发现比较流行的 Markdown Preview Enhanced。但是问题又来了，这个插件没有直接使用 markdown-it，而是使用了作者自己的 mume 模块。

去看看 mume 的代码，但 `package.json` 里面并没有依赖 markdown-it，甚至根本没有第三方 Markdown 模块的依赖。

只好去 Markdown Preview Enhanced 项目提功能需求。看了看作者还是深圳的，萌生了想付费让作者帮忙支持这个功能的想。打开邮箱，写邮件……

等等，mume 应该不会自己写 Markdown 解析，再仔细找找。果然，有个依赖目录，里面真有 markdown-it 及几个插件的源代码。这不按套路出牌啊。

那么问题又简单了，赶紧为 mume 写个 PR。提过去后，过了一段时间，收到作者回复，说可以接受 PR，需要做些小的修改啥的。受到鼓舞，我当即表示说现在的代码配置的太死，要添加几个配置项，然后顺手把 Markdown Preview Enhanced 的 PR 也写了。

带着本地版本的插件，给内容同事演示了一把，基本上满意。

好的，接下来就等作者合并发版了。

## 第五回：峰回路转
虽然问题基本上解决了，但是 Atom 毕竟现在插件质量不如 Code 了。而且要同时安装 Code 和 Atom 有点难受。然而压死骆驼的最后一根稻草是 Atom 的 Markdown 高亮插件不如 Code 的精细。

还是研究一下为什么不能播放音视频，是在 Chromium 层还是只是剔除了相关的 FFmpeg 实现？
通过 JavaScript 接口发现，Chromium 还是声称能支持 MP4、MP3 的。

那么应该是可以替换 FFmpeg 库的。之前替换 VSCodium 失败应该是 FFmpeg 版本不一致导致的。

再次 Google，发现[有个项目专门为NWjs编译ffmpeg](https://github.com/iteufel/nwjs-ffmpeg-prebuilt)。NWjs 和Electron（Code 和 Atom 的运行平台）原理差不多。同时记得之前编译 Code的时候，在网上下载的是 FFmpeg 3.8 的版本，于是下载一个 3.8.1 版本的预编译的 FFmpeg。替换一下，居然 OK 了。居然就这么样 OK 了。

所以最后的办法还是回到的 Code，替换 FFmpeg 库的方式。写个 install.sh 脚本：

```bash
#!/usr/bin/env bash

curl -sL -o libffmpeg.zip https://github.com/iteufel/nwjs-ffmpeg-prebuilt/releases/download/0.38.1/0.38.1-osx-x64.zip
unzip libffmpeg.zip
mv ./libffmpeg.dylib "/Applications/Visual Studio Code.app/Contents/Frameworks/Electron Framework.framework/Versions/A/Libraries/libffmpeg.dylib"
```

当然，这里还可以通过短连接等方式让每次下载的脚本是最新的ffmpeg。具体方法这里就不展开了。

这里我吧他放到这个[gist](https://gist.github.com/xpol/483ec9967d0a3d14791374947ecf0ec8)这样就可以通过 curl pipe bash 的方式安装。

```bash
curl -s https://gist.githubusercontent.com/xpol/483ec9967d0a3d14791374947ecf0ec8/raw/4083e7563f48b9de57503932fce6c2512f70f5c8/install.sh | bash
```

最后再写个文档，教编辑同事如何打开终端运行上面这段代码。

完美！（*希望这个不是很快倒下的 flag*）

## 后记
曲折的过程是为最终解决方案积累信息。

不去编译一下 Code，可能不知道要什么版本的 FFmpeg。不先给 Code 写个插件，那么后面替换 FFmpeg 的方案可能也要曲折一些。

当收集到的信息足够且充分连接的时候，得出理想的方案只是水到渠成而已。

