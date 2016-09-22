# 番外特别篇之 为什么我不建议你直接使用UIImage传值?--从一个诡异的相册九图连读崩溃bug谈起

## 关于"番外特别篇"

所谓"番外特别篇",就是系列文章更新期间内,随机插入的一篇文章.目前我正在更新的系列文章是 [实现iOS图片等资源文件的热更新化](https://github.com/ios122/ios_assets_hot_update).但是,这两天,被一个自己App中诡异的相册读取的Bug困扰,暂时延缓了文章的更新进度.这个BUG,诡异而又有趣,既然花了10个小时才理清,不妨再投入1个小时,晒出来供大家鉴赏,品玩.

## Bug 的详细描述

### 诡异的画风

![bug_img](https://github.com/ios122/why_not_uiimage/blob/master/bug_img.jpg?raw=true)

此Bug仅在操作多张高像素图片时才会触发,所谓高像素就是图片本身并不算大,但是图片宽高非常大的图片.这次触发这个问题的是一组 5701 * 3171 的图片.画风大家可以点击链接查看原图自行感受下 --[https://github.com/ios122/why_not_uiimage/blob/master/bug_img.jpg?raw=true](https://github.com/ios122/why_not_uiimage/blob/master/bug_img.jpg?raw=true)

### 当BOSS刚好是一个摄影爱好者

在大多数情况下,是很少有用户触发这个问题的,但是BOSS是一个摄影爱好者,手机里有许多高像素图,一天他想往自己公司的App上传分享几张图片时,他竟然没法把一次性地从相册选取九张图,每次选中后,点击"确定",都会理解Crash.是的,就是那九张图,其他图片是没问题的,8张图,也是OK的,他还强调了下是用的最新版本的App.

### 关于 BUG 的预处理

首先,我的第一反应是肯定是他的手机太烫了吧,重启下,就好了.恩,肯定是这样.发布作品的逻辑,好几个版本都没动过.模拟器,手机,我自己试了下,都是OK的.也没有其他用户反馈过,fabric也看不到任何log.对,手机太烫了.我稍后,再联系他,肯定就OK了.

稍后,再直接联系BOSS,竟然还是会Crash,他甚至给我录屏演示了一下,真的每次都会crash.而且我还无法复现.而且BOSS手机iPhone6 plus,自身内存不足的原因非常非常小.

形势,瞬间变得很紧张,这个问题的优先级瞬间被提到了最高!再次尝试了各种可能的情况.图片大小?它是9张1.5M的图,我就用9张3M的图,也是OK的呀!选取时,顺序有问题?我试着按照录屏中演示的顺序去选取图片,也是OK的.一股深深地无力感!竟然连复现都无法复现不了!

最后的最后,说是会拿手机给我测试.不过,最后BOSS的手机,还是没有拿到,只是拿到了开篇那张画风诡异的图片.没错,就是它,连续选取9张,就Crash了.

至少,我现在能复现问题了.下面的,需要的就只是时间,耐心还有大开的脑洞了.

## Bug 分析思路的简要描述

我不觉得,分析Bug真的有什么思路可言.Bug产生的原因,是有许多可能性的,可能行验证的顺序,方式和深度很大程度上取决于coder本身已有的经验,天赋,甚至还有些许的运气!我能描述的,可能仅仅是我处理这个问题的一个相对的完整脑洞过程.部分分析过程间,明显不是有逻辑性的.越是诡异的问题,越是不能循规蹈矩,要时刻尝试去问自己最可能地问题是什么,而不是沿着一条路,一条道走到黑.

### 1.排除通用逻辑问题

Coder有些许高傲,有时候是有利于自己更冷静地处理问题的.稍微不自信点的童鞋,可能就会怀疑:我代码是不是有什么特殊的临界判断没有加?不行,我得去看看.一行一行,看代码,从天黑到天亮,从期待到绝望...其实,稍微有一些对比实验常识的人,都很容易猜到: 两种情况,唯一的变量是 图片素材本身,那 **最可能** 的原因肯定是 图片本身的问题.一种高大上的说法,这某种程度上,也暗合了所谓的"贪心算法".每次,都只从最可能的原因入手,管他谁是谁,我的代码就算有问题,那触发这个问题的可能性,也是远小于 图片素材本身的.---多么朴素的真理呀!

### 2.确定是相册选取图片内存过高

这个问题,在真机上,并不好确定,因为连续读取9张高像素图时,内存是瞬间飙升的,你几乎没有机会去观察内存占用,给人一种因为某种逻辑判断而导致的Crash的错觉.如果换做模拟器,会很容易看到,这个内存占用,是飙升到G单位的.当然,我也没那么睿智,我是单个N个断点,最终确认了Crash的代码的准确位置.一个for循环,每次step 1,这下很明显地看到内存,几乎是 100M/张的速度在飙升,而图片本身的大小只有 1.5M/张.此处我想说的是,打断点也是有技巧的,最后没有办法的办法也是讲究办法的.可是试着注释掉可能的引起的代码,然后逐步放开注释,这要观察,会比直接打断点快些.--意会!

### 3.确定是PHImageManager 的问题requestImageForAsset:方法引起的高内存占用

当你通过注释法,配合断点,很容易就可以引起内存高占用的代码.此处,我的App中,是读取相册原图,用的是 *PHImageManager* 的 *requestImageForAsset:targetSize:contentMode:options: resultHandler:* 方法.此处接下来的解决思路,有大坑呀!你可能会想,是UIImage加载的问题吧?那就研究下UIImage渲染机制吧.然后1天过去了,等你学成归来,蓦然发现 *PHImageManager* 是一个系统方法,它加载的图片机制,你无力干涉!我可能运气比较好些吧,研究UIImage的渲染机制,想想都头疼,抱着试一试的态度,我google了下: **PHImageManager requestImageForAsset memory high**,然后第一条链接的第二个回答就是我要到答案: [http://stackoverflow.com/questions/33274791/high-memory-usage-looping-through-phassets-and-calling-requestimageforasset](http://stackoverflow.com/questions/33274791/high-memory-usage-looping-through-phassets-and-calling-requestimageforasset)  

是的,我运气,似乎总是很好~

### 4.使用requestImageDataForAsset:替换的问题requestImageForAsset:

答案原文是:

```
I found that if i switch from

- requestImageForAsset:targetSize:contentMode:options:resultHandler:

to

- requestImageDataForAsset:options:resultHandler:

i will receive the image with the same dimension {5376, 2688} but the size in byte is much smaller. So the memory issue is solved.

hope this help !!

(note : [UIImage imageWithData:imageData] use this to convert NSData to UIImage)
```

简单说,就是用 - requestImageDataForAsset:options:resultHandler: 替换 requestImageForAsset:targetSize:contentMode:options:resultHandler: 就可以了,前者是直接返回二进制数据,不渲染.

但是,这里有一个可能不是问题的问题, 这个方法调用是位于一个名为第三方库 *TZImagePickerController* 内,我方便直接改吗? 我是直接给改了.此处,将来必成大患,以后再用到,肯定还会有相同问题,还不如直接把原来的实现直接替换掉.当然,这也是成本最小的方法.这个库,本身,已经在App内,深度定制和重写了,如果一些成熟的第三方库,这么做,最好先备份或备注下.

### 5.使用imageWithData:兼容原来的调用

为了和原来的Api接口调用兼容,用imageWithData:将NSData转换为 UIImage 传出,同时扩展方法,使支持同时传出 UIImage和原始的 NSData对象.传出NSData对象的原因是,是因为高像素图片,会引起一些列的问题,故事到此远远没有结束,详见衍生问题部分.

### 6.变更前后的代码对比

还是来段代码感受下吧,一码剩千言:

```oc
/*原来的代码*/
[[PHImageManager defaultManager] requestImageForAsset:asset targetSize:PHImageManagerMaximumSize contentMode:PHImageContentModeAspectFit options:option resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
            BOOL downloadFinined = (![[info objectForKey:PHImageCancelledKey] boolValue] && ![info objectForKey:PHImageErrorKey]);
            if (downloadFinined && result) {
                result = [self fixOrientation:result];
                if (completion) completion(result,info);
            }
        }];
```

```oc
/*优化后代码*/
[[PHImageManager defaultManager] requestImageDataForAsset:asset options:option resultHandler:^(NSData * _Nullable imageData, NSString * _Nullable dataUTI, UIImageOrientation orientation, NSDictionary * _Nullable info) {
            UIImage * result = [UIImage imageWithData:imageData];

            BOOL downloadFinined = (![[info objectForKey:PHImageCancelledKey] boolValue] && ![info objectForKey:PHImageErrorKey]);
            if (downloadFinined && result) {
                result = [self fixOrientation:result];
                if (completion) completion(result,info,imageData);
            }
        }];
```

## 此类Bug的可能的通用解决思路

首先,我要说明下,我解决的思路和方式,很大程度上依赖也受限于我已有的经验,此处的解法,可能不是最优解,最多只能算是个通用解.说不定,将来等我再研究下渲染机制一类的技术,会有一个新的更简单的方法.欢迎大神补充!

未来遇到UIImage内存问题的童鞋,至少能从此处获取的一个至少验证可用的解决策略.

回到问题本身,用一句概括就是:永远不要直接传递UIImage对象.在需要传递UIImage的场景中,请使用图片名或者NSData二进制对代替.

## 衍生问题应用与解决

故事,真的还没有完结.从相册顺利读取这张诡异的高像素图后,我发现我没有办法将它上传,也无法在轮播图上,连续显示.简要概括如下.

### 无法直接以UIImage格式,连续把九张图保存到缓存目录

图片选取后,并不是立即上传的,为了能实现"重发"功能,需要在缓存目录保留副本.原来是将 UIImage 转换为 NSData写入.在此过程中,又一次引起了巨额的内存开销.解决方法,就是直接缓存原始获取的 NSData 的对象,而不要 NSData --> UIImage --> NSData.

### 无法直接以UIImage格式,连续在轮播图上显示九张图

此处对应的是一个本地大图预览功能,实现是在前一个页面把九张本地图的UIImage传递给轮播预览组件.此处的坑是: 把一个存放在 数组中的UIImage对象传递给 UIImageView的 image属性,当UIImageView加载到父视图时,会引起巨额的内存占用.原因初步猜测是 UIImage 对象显示到 UIImageView 会有一个特殊的耗费内存的操作,如果原始的 UIImage 对象一直存在,这一块内存那就无法释放.这一步,困扰了我很久很久,好几个小时!我真没想到,**一个UIImage对象,竟然会二次引起高内存占用**.最终的解决方法,就是在前一个页面传递 NSData数组,在赋值处,再使用imageWithData:转换为 UIImage.这样,内存使用基本没什么起伏.

或许,我应该研究下 一个UIImage对象,竟然会二次引起高内存占用 的原因.欢迎大神完善!

## 参考链接

* [http://stackoverflow.com/questions/33274791/high-memory-usage-looping-through-phassets-and-calling-requestimageforasset](http://stackoverflow.com/questions/33274791/high-memory-usage-looping-through-phassets-and-calling-requestimageforasset)

* [http://stackoverflow.com/questions/31898326/memory-issue-when-using-large-uiimage-array-causes-crashing-swift](http://stackoverflow.com/questions/31898326/memory-issue-when-using-large-uiimage-array-causes-crashing-swift)


---
* github:[https://github.com/ios122/why_not_uiimage](https://github.com/ios122/why_not_uiimage)
