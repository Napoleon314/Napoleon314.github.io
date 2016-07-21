---
layout: post
title: CryEngine渲染剖析
tags: Graphics 3D CryEngine
categories: 远古回忆
---

这是2010年我发在[OpenGPU][opengpu]上的一个帖，那个时候网上图形相关的资料还相对比较少，幸运地得到了一些关注，当时我没有博客，所以[原帖][origin]发在了论坛中的[KlayGE][klayge]专区（因为叛爷长得帅）。在六年后的今天看来，其中的很多技术都已经显得陈旧落后了，当时我稚嫩的用词现在看来也略显可爱，不过这段时光仍然是我最难忘的开发回忆。因此我想把这篇帖搬运过来作为我博客的开篇，顺便缅怀下那逝去的青春。PS:其实并没有写完，坑我是有计划填的，但是鉴于技术有点跟不上时代，以及我正在写的开源引擎（轮子的坑都还没爬出来）Venus3D([https://github.com/Napoleon314/Venus3D][venus3d]),打算跟着引擎的开发，以日记的形式将用到的技术以及细节分享出来，具体将会在以后的博客中和喜欢图形的朋友们见面:P

* TOC
{:toc}

# 正文搬运

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

这个帖可能在这儿发比较奇怪，不过我是一个信仰黑客精神的程序员，也非常支持KlayGE的研发，希望发一些我的原创资料，和作者，以及KlayGE的作用者交流，甚至有一些麻烦的问题，也希望作者可以解答，如果KlayGE的研发能够涉及到这些东西，那就最好了。

作为一个刚从业不久的引擎开发人员，为了能快速有效地提高自己引擎的渲染质量，使用了各种办法来解析CryEngine的渲染手法。在此我将以专业的视角来与您分享我的经验和成果。如果您也是专业的图形程序员，希望与您一起讨论，解读这款令人惊叹的引擎，如果您是一个刚刚入门的新人或者爱好者，这将成为您不可多得的原创中文资料。

CryEngine是目前最难窥探的引擎没有之一，比起在中国泛滥的虚幻和GB的源代码，CryEngine也显得神秘得多，很少有人说自己有这款引擎，哪怕只有一个Eval。虽然有教育免费版，但是据我所知，也只有北大才申请到了授权。花了500W左右买了授权的好像也只有畅游和久游。但是越是如此，这款引擎的魅力越让人无法抗拒。当我有PerfHUD，PIX这样的分析工具时，我如何能不想着去拆解这个神秘又神奇的引擎呢。

Crysis所伴随的往往是最高画质的赞美和硬件杀手的批评，但是，我觉得即使这样，CryEngine仍然可以称为当今世界上优化最好，负载最大的引擎。其实引擎的设计追求的主要是负载能力，追求画面的应该是游戏。但是反过来，拥有最大的画面质量，没有引擎的强大负载，肯定是做不到的。任何引擎可以被美术用成硬件杀手，但是能在同样的情况下，跑出Crysis那样的画面的引擎肯定有着强大的负载能力。

CryEngine优化的出彩之处主要有以下几个方面：

１材质排序：从PerfHUD的结果来看，CryEngine对DXAPI的调用是最为简洁的，如此多的材质，却只有如此简洁的状态切换过程，不能不令人赞叹，相比之下，OGRE或者GB之类的三流引擎做得就要差得多。材质排序已经被做到了极至，从这里省下来的时间，成为了Crysis如此复杂的宏大场景的基础。

２技术选择：这是我最佩服Crytek的一点，如此多的次世代渲染技术，竟然每个都如此地和谐，兼顾着性能与效果。我研究了它的好多渲染手法，真是深有感触，为啥能如此恰当地选择技术呢？后来看了它们LPV的一篇Paper中有这样一个预算:

![budjet]({{ site.url }}/assets/CryEngine-profile/budget.jpg)

这才恍然大悟。原来他们竟如此专业，预先做了这样的评估，难怪技术都被优化得如此高效，和谐。
3.瑕疵优化：这个通常被人忽略，或者被人认为是错的。很多人都觉得宁愿速度慢一点，也要求技术没有瑕疵。其实，这是不对的。瑕不掩瑜，在Crysis如此震撼的效果面前，有几个会对它技术的瑕疵非常关注，觉得不能忍受呢？但是在CryEngine中，所有的瑕疵都换来了宝贵的时间，以至它能对一个游戏添加如此多的渲染技术，以达到Crysis的效果。

有了这些与众不同，使CryEngine变得如此另类，以及怪物般的负载能力。当然有人要说，CryEngine不是最快的，因为相同速度的情况下，虚幻的画面可能更好。对，确实，虚幻的Irradiance技术提供了一个廉价的GI效果，但是，这个东西带来的限制却让虚幻完全没有能力大幅度改变场景光照，因为主光源是预先渲染好的。这个东西在CryEngine3中已经被支持了，同时候还有了LPV这个神奇的动态GI。这样说的话，Crysis所追求的全动态场景的理念其实也是逼真临场体验的一部分，当然需要更多的性能开销了。也就是因为Crysis有昼夜变化，才带来了这种前所未有的临场他体验。

接下来，我要对Crysis所使用的渲染技术进行一些细致的剖析，当然也带着一些不理解的问题，希望大牛们能给我一些建议和帮助。

***

![gonggod]({{ site.url }}/assets/CryEngine-profile/gongminmin.png)

你分析得很好。

1. 材质切换少，主要原因是CryEngine2用的是uber shader，切换都写在shader里面，不用通过API切换。CryEngine3用Deferred light，切换也多不了。

2. 这一点在主流商业引擎里面都可以看到，确实安排得很好。

3. 没错，Trade off永远很重要。
CryEngine强大就在于它不但能有那么多的特效，还能把那些特效有机结合起来，都安排到合理的时间内。KlayGE也是朝着这个方向发展的。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

Crysis是用CryEngine2制作的，CryEngine3无幸窥见，在这里，先说说确切的CryEngine2的渲染过程。
众所周知CryEngine2，没有使用DeferredShading，所以无法支持太多的光源，主要渲染围绕着阳光进行，而且使用了非常复杂的PixelShader来实现漂亮的材质，所以ZPass预渲染就变得很必要，不然会有很多无用象素被复杂的PixelShader渲染，当然这个和DeferredShading一样，无法支持Blend，所以只有两种，普通和AlphaTest。渲染ZPass为了开启PreZ，肯定要进行排序，先渲染普通物体，后渲染带AlphaTest（当然是使用Shader的Texkill来进行，没有状态切换）的物体。其实标准的ZPass应该是不需要ColorWrite的，但是Crysis利用这个机会，输出了深度到一张R32F，因为视距为八公里，乘一个0.000125映射到[0,1]，天空为2。有了深度的G-Buffer，其实就已经可以做很多事了，除了光照外很多的后处理都可以用这张深度缓冲来进行。有了深度后，会先进行AO计算，然后阴影，这都是屏幕空间的计算，然后渲染每个物体都会对这两张图进行采样。AO是一张RGBA，R通道用来存SSAO的结果，G通道用来存2.5DTerrainAO的结果。然后进行一次Blur。阴影是有个Mask的，平行光一般是R，剩下三个通道，大概可以存放三个点光源或者SpotLight吧。然后就是对一张R16G16B16A16F进行HR的高范围渲染。这回DepthFunc已经是Equal了，因为深度已经完全填充。当然，为了PreZ，还是先普通后AlphaTest，然后会看到大量的碎片，因为Z检测会挡掉大量像素。所有着色像素加起来也几乎就是屏幕像素了，除非有两个三角形有点是同Depth的Fighting状态，当然可能性很低。完成后会进行Blend渲染，主要是粒子，玻璃之类的东西。结束后，就要进行HDRToneMap和HDRHighLightBloom了。完成后，接下来有一个EdgeAA，当然除此之外应该还有抗齿过程，只是调试时肯定是要关掉的。EdgeAA是个抗齿的后处理，只使用Depth来提取边缘。然后还有个Glow，大概会把火焰之类的Bloom加大吧。在这之后，就是SunShaft，来加上漂亮的Ray和Beam，最后会进行ColorGrading，完成渲染的最后一步，这样，就能看到Crysis的画面了。这样说是不是有些笼统呢？没关系，如果大家喜欢，我还会继续深入剖析CryEngine渲染的一些技术点，来解读震撼画面背后的故事。今天有点晚了先写到这儿了。

　　睡前再写一些看了KlayGE后的感想，KlayGE是一款我很喜欢的开源引擎。对于开源引擎来说，没有强大的资金支持，很难开发出像商业引擎那样功能强大的实用引擎。KlayGE这样研究一些新鲜的API，支持OGL3或4，DX11，以及一些好玩的技术的实现，却是真正有帮助的。它的字体算法的设计真的是非常的独到，让我非常震撼，我正在想有什么办法能使缩小时更清晰。像Ogre，Irrlicht那种引擎中既见不到强大的实用性，又没有这样的看点，我个人还是比较无爱的。KlayGE最近又出了FFT的水，对于只能看到SigGraph1999的我来说，还是非常有帮助的，记得上回Ogre有个水体插件，也做了FFT的波动，不过实时CPU刷顶点做法还是有点过了，这了新的Demo看，真的挺兴奋的。
有一点想问下作者，你的DeferredShading为啥不支持HW的AA呢，而使用一个后处理来进行？DX10开始，MRT不是已经可以AA了吗？难道还有什么障碍？

最后发张我现在自己渲染的图，虽然比Crysis差不少，但是渲染技术是差不多的。其实如果时间允许，我真是想自己操刀写个引擎，把Ogre改成这样真不是一般的麻烦，Ogre的渲染体系太古典了，不太适应现代的技术，搞得我动不动接管过来调DX，不然要么性能莫名其妙地不见，要么就是干脆无法实现。不过因为DX9，使用了DeferredShading后只能用后处理来抗齿，其实可以用超级采样，但是舍不得那性能啊。Ogre的材质系统不灵活，导致我一怒之下统一使用了线性过滤，材质糊糊的很不爽。下一步，可能就是要尝试下LPV了，毕竟GI还是很令人Happy的。

![show]({{ site.url }}/assets/CryEngine-profile/show.jpg)

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

继续更新啦，首先谈谈CryEngine的地形技术。地形技术是CryEngine的一个较为核心的技术，为了实现八公里的超远视距，地形经过了比较细致的优化。CryEngine的地形主要使用的数据结构仍然是高度图，并且在CryEngine2中可以看到，它所支持的尺寸非常有限，好像也不是动态加载技术的无缝大场景。为此为特地下了Crysis的Sandbox一探究竟。CryEngine的地形Lod时并没有Morph的过渡而是直接突变，这让我很惊呀，一般地，地形Lod时的突变会非常刺眼以至于让人难以接受，但是在Crysis中，我却完全无法意识到这一点，反复摆弄它的场景后发现，原来是超高密度的植被覆盖使得地形的跳动完全被植被所遮挡，植被同样也在不体地突变，所以一般只能感觉到植被Lod时的突变，而无法感觉到地形Lod时的突变。因为不需要Morph，CryEngine地形的灵活性就体现出来了，它可以切到最小只有2M*2M一个单元，但是越远，绘制的实际单元还是有所合并，具体LOD的细节仍然没有完全搞清楚，如果哪位大牛知道，希望能够不吝地告诉我，不胜感激啊。但是有一点是很清楚，它的LOD是有根据视觉复杂度和距离两个标准来决定网格精度的，我贴一张Wireframe先：

![terrain]({{ site.url }}/assets/CryEngine-profile/terrain.jpg)

可以看到，网格精度并不是均匀地由近到远递减的，而是以地块视觉复杂度为权值的递减，期间有相当多的补面，以保证没有T型接缝。不过由于不需要Morph，跨级LOD之间的补面变得相对容易。

除了LOD外，地表材质也是CryEngine的一个比较出众的特性，Crysis的地表材质细节之丰富，绝对是目前绝无仅有的，但是支持它使用如此多材质的地表细节技术，却是CryEngine的一个特殊做法带来的。CryEngine并没有使用什么四层混合，而是按单层来划分的。当一层材质被刷到地形上的时候，CryEngine会生成一个材质的覆盖范围，也就是说每一个材质绘制时都有独立的IB区段，这样最大的好处是每个层都可以使用独立的渲染技术，而不会影响其它层，比如有parallax的地形层只有很少的一两个，但是混在其它层中却让地形细节有了整体的提升。还有就是因为地形距离有八公里，所以不可能都使用Blend的Tiling细节，在细节的几百米范围之外，就切换成只有一张的预生成大纹理，VirtualTexture技术在这里得到了应用。在同时使用了Tiling和VirtualTexture的情况下，CryEngine的地形远近都有很好的细节表现，同时性能上也能够支撑起八公里的视距。

当然CryEngine的地形技术中还有个神奇的Voxel技术。Voxel其实是个很古老的概念，应用到地形中应该是三角洲的时候。不过那时的Voxel技术不是很适应现在的图形管线，CryEngine只是使用了那Voxel作为一部分特殊地形的数据存储，最后用了某种算法，生成了与高度图Terrain无缝连接的Mesh，使得悬崖，山洞可以在引擎中用Terrain实现。通常要做这些往往都使用模型，而不同的系统使用起来很容易产生接缝。这就是Crysis地貌如此丰富的原因。

不过Voxel的生成算法，我还没有想到，或者看到，很想知道有什么好办法来实现这个。不知道龚敏敏大大是否有这方面的办法，或者有见过关于这个的文献。如果KlayGE能整合一个类似的算法，那就最好了。

![voxel]({{ site.url }}/assets/CryEngine-profile/voxel.jpg)

***

![gonggod]({{ site.url }}/assets/CryEngine-profile/gongminmin.png)


"有一点想问下作者，你的DeferredShading为啥不支持HW的AA呢，而使用一个后处理来进行？DX10开始，MRT不是已经可以AA了吗？难道还有什么障碍？"

MRT支持AA不是问题，问题在于对AA Texture的读取，一直到DX10.1才很好地解决了。我用EdgeAA，虽然效果没有MSAA好，速度上对rendering几乎没有影响。AMD的驱动里面甚至内带了EdgeAA。

至于Voxel，我也没细想过。我想I3D 2010的一篇paper把voxel都存在octree里的方法可能可以借鉴。

预计年底发布的KlayGE 3.11将支持类似Virtual Texture的技术，支持“无穷大”纹理。地形的height map可以直接存进去。同时，KlayGE 3.11也会加入real-time GI，但不是LPV的思路，因为那还是需要预计算的，而且不支持specluar。谢谢关注！

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

很希望看到KlayGE的VirtualTexture和GI。目前我也正在整理一些解决方案，有这样的参考真的很不错哈。我可能考虑把LightMass和一种实时GI解决方案整合到无缝场景中。

接下来继续更新

CryEngine的阴影是我最喜欢的一个CryEngine的解决方案，它的科学性，实用性，让我很震撼。在目前的硬件条件下，CryEngine的阴影系统是一个比较完美的实时阴影解决方案。首先要说明的一点是，CryEngine2虽然不是DeferredShading，但是仍然有深度图可用，所以可以在屏幕空间计算阴影，这样能为PCF节省非常多的采样，与VSM相比，采样次数会更少。这样就有了下面的结论：PCF适用于实时性较高的阴影采样，因为一次采样的次数较少，而VSM更适用于低频更新的阴影，因为VSM可以预采样，采样完之后就能很快地生成阴影，这样的话，只要阴影不更新，VSM就不需要再采样。有了以上的思想，就有了CryEngine的阴影解决方案。首先用四张1024大的D24S8作为PSSM的四层，据我目测，PSSM的距离大概在200-300之间，由于远处的物件通常是在较浓的雾里，所以，其实远处阴影对画面的增益会较弱，近的阴影距离让CryEngine的四层1024图都获得了相当高的阴影清晰度。这四层阴影也并非实时更新，CryEngine对这阴影进行了控频，不过游戏中并未感觉到，只是CryEngine3在GDC2009的那段视频中，很明显看到了控频的痕迹。VSM则被用来渲染地形的阴影，众所周知，WOW的地形没有阴影，一开始我还以为这是对的，后来才发现原来WOW的光不会动，所以在阴影不被拉长的情况下，这样的阴影还是可以接受的，但是如果需要阳光动态变化，甚至拉到很长，没有阴影的地形会让物件的阴影腾空，相当难看，所以渲染地形的阴影还是有必要的。但是地形的阴影一般比较模糊，因为距离比较远，所以PCF对于地形来说，阴影边缘太过生硬，所以CryEngine选择了用VSM，在一张1024的R16G16F上渲染地形，然后进行一次Blur，由于地形的阴影渲染范围很大，再加上地形静止不动，和光线变化较慢的特点，地形的阴影更新频率非常低，这样，地形阴影渲染的开销就会比较小。在渲染地形阴影的同时，CryEngine还加上了云阴影，使得阴影非常完整，逼真。VSM使用半精度其实是非常冒险的，VSM不像PCF，利用统计学算法的VSM如果有完美的实数，将会是无比完美的，可惜，浮点的精度有限，想要VSM的阴影有一个模糊的边缘，高精度很必要。通常都使用32位的浮点，半精度会引起近处非常明显的条状失真。不过CryEngine的这个失真仍然存在，只不过并不是很明显，主要原因有以下两点，第一，地形本身是个低频细节的模型，所以阴影距离通常都比较远，第二，CryEngine使用的阴影距离近，所以，本身深度范围都小，所以精度自然就会比较高，浮点数嘛。所以CryEngine很有效地利用PCF和VSM模拟出了非常高效完美的阴影。这个解决方案目前也被我一直在使用。

关于采样，一般，PCF的采样是不如VSM的，原因是PCF是一个比较出来的二值结果，无法利用MipMap，线性采样和AA之类的预采样，这也正是VSM的优势，但是在DX9中，有个HWShadowMap的解决方案，利用透视校正的采样，可以利用线性过滤，一次采样即可得到4次差值的PCF，而在这个基础上进行多次采样，就可以得到非常柔和的效果。一般的，在动画或光发生改变时，阴影很容易发生抖动，这种抖动是非常刺眼的，这是由于被采样的纹理本身的锯齿抖动造成的。想要解决这个问题，一般就两种，VSM可以利用AA来解决，而PCF则利用随机采样。随机采样之所以不容易感觉抖动是因为高频的抖动不容易被人眼察觉的关系。GPU2上就有一篇文章描述一种随机采样方法，不过GPU上的采样方法比较原始，所以实用性并不强，CryEngine使随机采样到了一个新的境界。首先，GPU上说的方法是连续采样一张随机纹理，来得到随机的信息，但是这显然是种浪费，因为一次采样就可以得到随机了。CryEngine所用的方法是预生成一个圆形分布的采样点集，然后采样一次随机纹理，随机纹理上记录的是这个点集的旋转角度，这样的话，一次随机采样，就能使所有点都随之改变。而且也正是这些图形分布的点的旋转，使得阴影边缘更柔和了。GPU上的方法，随机采样的UV是屏幕空间的，这样的随机采样点抖动很快很频繁，而且容易被察觉到屏幕空间相关。CryEngine则是使用了一个光空间透视较正的办法，得到一个只与远近及角度相关的随机纹理UV，这可以把抖动降低。经过了上面的改进，CryEngine的阴影采样效果非常不错，虽然比不上SAVSM，但是比起以往的PCF阴影效果强了太多，把随机采样推到了一个新的高度。

关于ShadowPass，PSSM需要把阴影存在四个SM中，那么采样就会遇到用几个PASS的问题，像WOW，直接采样四张图，肯定会造成无效采样，性能会比较低，而GB则使用一张大图来存，通过变换来定位，再采样，而CryEngine用了另一种方法，使用模板，模板可以分别提取每层需要的像素，然后进行采样，四个Pass得到最终的阴影，理论上来说，全屏的阴影，应该是这个性能最优，因为采样的图更小，又没有重复采样的浪费。

以下是我使用这个解决方案得到的结果，因为引擎和时间的关系，我没有做阴影距离，所以是全距离阴影，我的视距为400米，我用了2048X4的PSSM，和2048G32R32F的地形阴影。因为我这个距离，地形阴影用半精度会出现严重的精度不足的VSM失真，所以没有办法，CryEngine在这个距离上是掐得相当准的，稍有偏差，就会导致无法接受的失真。

![shadow]({{ site.url }}/assets/CryEngine-profile/shadow01.jpg)

![shadow]({{ site.url }}/assets/CryEngine-profile/shadow02.jpg)

![shadow]({{ site.url }}/assets/CryEngine-profile/shadow03.jpg)

之后我会更新AO，欢迎关注，谢谢

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

在这儿有个问题，想问一下gongminmin老大，不知道DX11中HWShadowMap这个还有没有，我看到CryEngine中DX10的处理是不一样的，而且还利用了抗齿版的ShadowMap来做PCF。为啥Nvidia说DX10的特性能较好解决VSM的问题，你将来是否会在KlayGE中展示下你说的VSM的统一解决方案?

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

CryEngine中的AO是SSAO第一次被应用到游戏中，一才就炒热了实时间接光照。CryEngine的AO技术并不是目前效果最好的，但是它的Blur却是最快的，也就是因为超快的Blur，使得它可以应用FullAO，而不是HalfAO，因为FullAO的使用，叶子或者草之类的高频物体的AO变得比较好，所以总体来说，也算是个不错的AO解决方案。因为CryEngine的渲染体系是没有Normal可用的，所以仅Depth采样所得到的AO或多或少会有一些问题，所以只能说，这个是目前仅深度AO中，较好的效果，因为它的全局性非常强，这得益于算法中会根据距离来缩放射线长度，所以CryEngine的AO非常显眼，不像有些AO的效果离远了就容易被忽略。CryEngine的AO是全精度计算的，这可能会导致比较大的开销，所以CryEngine对采样进行了优化，主采样使用屏幕原大小的图，然后对周围采样时候则用了缩小后的Half图，这节省了一些采样时间，也引起了一个竖条状的失真，这个可以在Crysis中见到，雪天的门口比较明显。随机图方面，CryEngine用了一张4*4的图，一共16种，这使得它的杂点非常难看，但是却给BlurAO带来了便利，因为只需要四次采样，加上线性过滤，就能得到完全平滑的AO效果了，这比起Nvidia的HBAO要快得多。不过这样的AO仍然有比较多的缺点，比如角色的后面容易黑，低多边形镶嵌的痕迹明显，还有就是Blur后会造成象素的偏差，所以Crysis中也容易看到AO中有亮边，但这不影响这个AO全局性好的特点。不过具体如何，还是仁者见仁。我使用这个AO方案是因为草叶之类的高频物件表现相对好的原因。

CryEngine还有个预计算的TerrainAO，我暂时还没有机会去看。

![ao]({{ site.url }}/assets/CryEngine-profile/ao01.jpg)

![ao]({{ site.url }}/assets/CryEngine-profile/ao02.jpg)

![ao]({{ site.url }}/assets/CryEngine-profile/ao03.jpg)

我的结论是远处，以及草叶表现较好，不过如果有Normal的话，还是非常大的改进余地。

KlayGE中的AO是不是和DX中的HDAO差不多啊？感觉那个AO可以钩出非常细节的AO线，但是全局表现不如这个，怪不得叫HDAO，如果能配上一些低频的AO的话，应该会更好看一些。Nvidia的HBAO效果也挺好的，就是Blur老是出现点，但是可以比较好的解决低多边形镶嵌的痕迹，所以还是不错的解决方案。将来我会再尝试做个更好一些的AO，目前这个仍然没有十分让人满意的方案。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

最近我分别使用了下CryEngine和Unreal的编辑器，相比之下，Unreal的编辑器要更专业一些，虽然CryEngine很强大，但是并不一定就是最为成功的商业引擎。Unreal的材质连接器使得Unreal效果的扩展性要好很多，CryEngine相比之下也只是拥有更强大的内建渲染功能，作为一款引擎，与Unreal的积累上仍然存在一些差距。当然说到效果，比起Lightmass所展现出的静态GI，CryEngine也没有办法弄出相应的动态解决方案。目前还是会有很多人觉得虚幻的室内效果更为完美，即使那要以牺牲动态光影为代价。

***

![gonggod]({{ site.url }}/assets/CryEngine-profile/gongminmin.png)

DX11里HWShadowMap仍然存在阿，而且和DX10的一样。Nvidia说得可能是用Texture array来做sm

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

Texture array的好处应该是能一次渲染完四张，其实应该算是PSSM的福音，动态CubeMap和点光源的ShadowCubeMap都会挺Happy。我还以为会有两通道的深度缓冲呢

***

xoyojank

KlayGE的字体缩小并不清晰, 放大才清晰. 像8号字, 还是点阵更清楚

***

![gonggod]({{ site.url }}/assets/CryEngine-profile/gongminmin.png)

缩小是个问题。所以下一个版本我打算用点阵做缩小的部分。

***

hungy

回复 10# Napoleon314 

非常感谢你的分享！写得很好。

关于实时AO，我知道的方法有三类，

一者是在GPU上老实做GI, 包括加速结构（KD-tree)和Photon mapping，例如Kun Zhou的Sig文章。效果很好，能模拟几乎全部光学现象，就是帧率太低，很难用到实际场景中。

二者是Instant Radiosity，类似RSM的做法，Shadow map采样间接光源，然后下一步用Light Volume或者单纯Deffering rendering渲染多光源(gathering, splatting)，通常忽略间接光照的可见性，这种方法目前看来速度最快，但效果不好说，Light Volume的光很容易糊掉。

三者是SSDO，就是在SSAO的基础上计入了样本的颜色信息，GPU Pro释放出了代码和程序（见其出版商主页），在我的电脑上表现不好（当然我显卡很挫，是8400GS，也许没什么发言权。。。）
顺利的话年底我们就能知道龚大的GI方案：）

另外，龚大分享了GameFest 2010的报告（感谢龚大！），上面有对Light Volume和AO的讨论：
Lighting Volumes
Rendering with Conviction: The Graphics of Splinter Cell
都比较有意思。

又另，我自己强烈觉得Light Volumes代替不了光照烘焙。Crysis2的E3影片我也看了，感觉光照效果并不突出，不如《GT5》和之前的《镜之边缘》，这两者都用了烘焙。虽然Light Volume一直标榜全部动态，可是实际中的场景光源不变或者说低频变化的情况更为普遍。而且相比Light Volume，烘焙总能提供更高的视觉精度和更低的渲染代价，后者在PC平台我觉得非常重要。因为现今流行的液晶显示器都是高清的，其只有一个最佳分辨率也就是最高分辨率。在此分辨率下的渲染效果才是对一个游戏软件最好的评估和论断。而1680x1050,1920x1080这个档次对显卡的填充率和带宽要求都非常高，然而不是每个玩家或者网吧老板都喜欢电老虎的显卡。

***

![gonggod]({{ site.url }}/assets/CryEngine-profile/gongminmin.png)

我的GI方案会比较接近RSM的路子

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 15# hungy 

我也非常期待龚大的GI解决方案，应该会有非常的参考价值。近期也可能也要开动一款引擎的研发了，Ogre让我束手束脚的。关于GI解决方案，据Crytek所说，SSDO是LPV的一个非常好的补充，两个一起用，对于室外来说，其实已经足够了，哪怕没有GI，关系也并不是太大。但是室内因为空间小，没有非常强的光源，所以GI会非常明显，需要多次弹射，用实时的GI比较吃力，仍然需要一个带烘培的解决方案。我非常赞成Crytek3.3ms的预算，这个是个理想的结果，希望能找到个好点的解决方案。我的设计是面向无缝场景的，所以有技术都会根据无缝的特点进行改进，烘培也一样，如果室内特别增加静态GI，那就必须慢慢地暗下去，我不希望像一些游戏那样，通过区域来突然改变色调。

***

kuguoxin198

我想知道CryEngine OC做法，希望楼主持续更新。。。。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 20# kuguoxin198 

OC其实关键点并不多，我下回写的时候会记得带上，谢谢关注

***

liumingod

LPV我也实现了一把，但还是感觉比较鸡肋。
速度对实时影响还蛮大的，质量又赶不上烘焙。
没间接遮挡还蛮有问题的，要算间接遮挡会挺费的，而且还很不精确。
目前感觉有点要上不上要下不下的。
DX9实现LPV会需要更多的额外的消耗。。。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 25# liumingod 

回头我会对RSM类的GI进行一定的尝试，只有在巨大的场景下，测出的比例才有参考性。Crytek所说的3.3MS优化得好的话我觉得应该可以达到。效果可能确实会有一点僵，我考虑只用来做阳光的GI。

***

qiaojie

我觉得烘焙在GPU的性能没达到实时高质量GI之前都不会消亡。目前用RSM之类的实时GI渲染出来的画面质量跟预计算出来的GI没法相提并论的，对大部分场景丧失掉一些动态性换来高质量的画面我觉得是值得的。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 27# qiaojie 

牺牲画面来换取动态场景同样也是值得的，所以我想无缝连接虚幻和CryEngine的两种解决方案

***

kuguoxin198

KlayGE结构还是很清晰的啊，唯一的问题的就是要理解这个引擎，先乖乖的把相关的boost知识弄清楚。可惜我对boost库目前还没经验。发现最新版本增加了场景管理，觉得作为一款引擎来说，大牛应该加强下场景方面和常用场景物体（例如地形，建筑，植被，天空等）的内容，而不仅仅是渲染方面。

***

![gonggod]({{ site.url }}/assets/CryEngine-profile/gongminmin.png)

场景管理其实一直都有，这个版本只是略微加强了。地形等feature正在逐步往里加，谢谢关注

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

有Ogre使用经验的话，应该会比较容易看明白，不过我个人最喜欢KlayGE的多API支持，像OGL4.0这种，目前目前只有Klay一个有，所以参考价值比较大，像Ogre和Irrlicht就比较没意思，一切一切都是如此地古老。

***

32220937

楼主有CryEngine 代码？否则 怎么能分析的如此细致

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 33# 32220937 

如果我有代码，我就不会发了。以上全是我分析的结果，所以我不知道CryEngine内部的大部分实现，只能用分析结果加猜测。Shader代码是可以看到的，因为游戏中解出来看就行了。不过我仍然很想弄到代码，因为有太多想做又不知道的内容了。无疑我是目前最想要CryEngine代码的人了

***

wpp

CryEngine对DXAPI的调用是最为简洁的 是不是和他采用super shader这种方式有关？那和ue3的真实两个极端了

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 37# wpp 

我说的不是统计出来的总量，而是看两次渲染之间的切换。比如两个物体用同一的Shader，就绝对不会有切换，如果有一张图共用，那就不会切换，连常量Buffer就只切换必要的部分

***

wpp

那就是他做了大量的冗余判断了，


***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

这个未能得知，我是无法知道C++部分处理的。


***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 39# wpp 

我是不知道他是否做了大量冗余判断，但是我知道我可以做完美无额外开销的材质排序，所以不一定做了大量冗余判断。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

我继续更新，为回应kuguoxin198 之前问的OC，我先在这里面简单说一下CryEngine中OC的理念，其实我也是这么想的，只不过GPU精粹上的说法和这个大相径庭，不过看到CryEngine的做法后，我像是有了靠山一般，很心安理得地也这么做了。当然，我说的OC是指Query的OC，其它使用CPU的OC方法不在这里面讨论。Query和回读差不多，都需要等待显卡指令执行完成，才能得到结果，如果在当帧疯狂查询，那OC会直接给予帧率毁灭性打击。GPU精粹上有一种优化OC的方案，但是这个方案只是减少了帧中查询的数目，并没有解决查询直接导致处理器空闲的问题，所以得到的性能增益非常有限，如果用这种方案来做一个游戏的话，场景规模会有相当大的限制。CryEngine的OC还是在我的意料之中，它在帧头，也就是Begin之后进行查询，这样的好处是，因为Begin已经等待显卡完成所有操作了，所以在Begin之后的查询就会相当迅速，不会因为同步白白浪费宝贵的CPU周期。得到结果后已经晚了一帧了，可以在渲染的时候直接应用，当然也可以等到下一帧场景管理器Cull的时候大幅度减轻CPU负载，不过那样的话，OC结果就需要延迟应用两帧，CryEngine可能是两种中的一种，我个人猜测应该是延两帧的，毕竟宝贵的CPU对CryEngine来说可是要计算几乎全部次世代物理的，在场景规模和视距都如此巨大的情况下，节省下这些资源是很有必要的。这时有人就会发现了，那不是会出现结果错误的情况，两帧的会比一帧的更严重。事实也确实如此，但这就是我之前提到过的瑕疵优化，有如此可观的性能提升，即使有瑕疵，仍然是合算的，因为毕竟OC的错误并不是无法接受的。原因有以下几点：

1.想要在同代硬件上加大场景规模，很多LOD的技术都会引起跳动和突变，有些远处的东西本来就会突然消失，OC导致的消失只不过是增多了会消失的机会罢了。

2.本来就要使用非常粗糙甚至刻意加大的模型代理来OC，这种粗糙会反过来减少OC错误的机会，所以OC错误可以很大程度地被减小，有特别刺眼的消失只要加大代理就行了。

3.相机的高速移动机会并不是很多。

4.事实证明了Crysis疯狂的跳动突变完全被场大规模的场景以及超高规格的细节所掩盖了。

我们对OC的需求是在最低帧率至少不降的前提下，可以较为可观地提高平均帧率，最完美是能同时解放GPU和CPU，而不是完美的无缝。下面发图来大致看下CryEngine的OC结果。

![oc]({{ site.url }}/assets/CryEngine-profile/oc0.jpg)

![oc]({{ site.url }}/assets/CryEngine-profile/oc1.jpg)

![oc]({{ site.url }}/assets/CryEngine-profile/oc2.jpg)

从图中可见，其实OC并不是那么精确的，太精确的OC反而对性能无益。目前好像有个OC引擎叫Umbra，可能这方面会比Crytek做得更精一些

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

接下来说一下HDR，HDR可以说是CryEngine的一个强项，也可以说CryEngine把HDR的效果推到了前所未有的高度，总之CryEngine如此照片的渲染结果和它的HDR关系最为密切。

众所周知，HDR是近年来最热门的技术之一，引擎如果不能很好地支持HDR一定会被划出一流引擎的行列。CryEngine1和Unreal3都相继支持了这个在当时象征着未来的革命性技术。HDR和Bloom有着本质的区别，虽然很多人会觉得差不多，但是HDR比Bloom多出的部分才是HDR的精髓。就算说HDR才是现世代(几年前的次世代)顶级渲染技术的根本也不为过。真实的光照会带来几十几百倍的亮度差距，而用以显示的设备却只能有一个固定有限的亮度范围，用有限的LR（低范围）设备来表现HR（高范围）的世界是一项挑战，最终我们需要一种技术把HR动态地映射到LR的设备上，这就是HDR。照片就是这种效果的典范，照片一样是LR（低范围）这与游戏中的情况一样，为啥我们却能觉得照片的还原度更好，是因为照片有个曝光的过程，通常看到的照片就是曝光较好情况，而如果自己拍摄，经常会遇到曝光不好的情况，曝光其实就是一种HDR的映射过程，人眼也有类似的过程。而对于LR无法表现的超高亮，通常是以光晕的形式出现，这并不是谁定的，而是人眼这样的感光设备在识别世界的时候自然形成的，看到照片的光晕会觉得很亮其实是人脑的潜在暗示。当然人眼的范围比起显示设备要高得多了，所以HDR的目标就是往照片靠近。由此HDR最关键的技术就是ToneMap，没有ToneMap就不叫HDR。ToneMap并不是那么容易，而且经常出现的方法效果都是非常不理想的，这个可以参考DXDemo中HDR相关的内容，这样的ToneMap效果根本就没有办法实用。CryEngine2在这个方面有了个非常大的突破，这使得CryEngine的渲染效果如此接近照片，以致难以辨认。

![HDR]({{ site.url }}/assets/CryEngine-profile/HDR1.jpg)

![HDR]({{ site.url }}/assets/CryEngine-profile/HDR2.jpg)

![HDR]({{ site.url }}/assets/CryEngine-profile/HDR3.jpg)

![HDR]({{ site.url }}/assets/CryEngine-profile/HDR4.jpg)

对比Unreal3中HDR的效果与CryEngine我发现，Unreal3的ToneMap范围很小，从室内出门的时候才稍稍能感觉到，而CryEngine2的ToneMap却有异常夸张的作用，往往台起头，地面就会变成几乎黑色。如此大范围的ToneMap使得CryEngine真正把做到了HDR，并且比起HDRLighting那样灰蒙蒙的还原，CryEngine的还原使得场景异常艳丽，HDR的魅力被CryEngine发挥了出来。对于那些看多了被Bloom搞得饱和度过高的场景而恶心的朋友来说，CryEngine版HDR的舒适自然则尤为明显。饱和度高的艳给人恶心晕菜的感觉，降底饱和度也是HDR的一个非常大的作用，但通常见到的HDR总是把场景变得不那么艳了，CryEngine版的HDR则打破了这个，让人眼前一亮。CryEngine带给我们的惊艳大部分都是因为这样的HDR如此地艳。

经过分析，我了解了CryEngine版HDR的过人之处，再一次深深感叹日尔曼民族的强大。CryEngine的HDR并没有使用ToneMapping文献中描述的ToneMap公式。线性的ToneMap不单还原差，而且会非常明显地削弱对比度，使得场景非常平，而没有生气，其次，即使用了Lwhite作为门限来提升对比度，但过低的门限会造成曝过过度，过高呢，仍然无法解决对比度削弱的问题，这也就是为什么DX的HDRLighting无法实用的原因。不过CryEngine则使用了更好的ToneMap方法，称为ExpToneMap，其实这个也早就出来了，就是没人实实在在地去实现罢了。顾名思义，ExpToneMap会渐渐挤压远离平均亮度的颜色，而且挤压非匀速，这就产生了有很趣的效果，而且这能很好地同时挤压亮与暗的颜色，使得有效果的对比度能够很好地还原，再也不会出现平均颜色的结果了。不过，光有ExpToneMap是不够的，有心的人可以注意到，CryEngine的Bloom也非常真实，比起虚幻巨大的光晕，CryEngine的Bloom层次更多，效果更好。事实确实如此，Bloom在CryEngine中是被强化的，Bloom分成两部分，一是BrightPassBloom，二是ExtraGlow。先说BrightPassBloom，这个就是一般HDR中的Bloom，但是CryEngine中的实现却非常不同，CryEngine中一共用了6个GaussianBlur，加上两次HalfSample，一次BrightPass，一次FinalMerge，总共是十个批次，而且每个Buffer都是64位的，所以CryEngine花了非常多的资源用于Bloom的生成，当然结果是喜人的。把1/2 1/4 1/8的三层GaussianBlur的结果叠加，出现了一个非常有层次的Bloom，而且在ToneMap之前就添加进HR的颜色中，这样有效地防止了Bloom提高过多的饱和度，使Bloom看起来非常有层次。ExtraGlow也是如此，三层叠加，区别是ExtraGlow是LR的，用于处理一些特殊材质的物件的Bloom。这才有了CryEngine最终的效果。

值得一提的是，需要有正常的HDR效果，拥有HDR的天空是非常重要的。如果做静态天空的话，可以去下到HDR的天空盒，动态天空的话，需要计算大气散射。CryEngine的大气散射计算过程是看不见的，所以我自己做了一个，方法就是Siggraph1993的文献中说的。最终才得到了CryEngine般喜人的效果。HDR花了我很久，但是从我修改OGRE的曲线来看，HDR的完成一次性把我的渲染特性提升了一个档次，可见效果有多么显著。

以下是我早期的截屏，很好地说明了HDRToneMap的过程，可以看到，整个过程的色彩还原都是非常好的。

![HDR]({{ site.url }}/assets/CryEngine-profile/hdr_show01.jpg)

![HDR]({{ site.url }}/assets/CryEngine-profile/hdr_show02.jpg)

![HDR]({{ site.url }}/assets/CryEngine-profile/hdr_show03.jpg)

![HDR]({{ site.url }}/assets/CryEngine-profile/hdr_show04.jpg)

***

qiaojie

HDR的难点在于如何正确计算曝光量吧。我采用的方法是统计计算亮度直方图，然后确定一个平移、缩放的范围，不过出来的效果有时候也不是很理想。不知道谁有更好的算法。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 47# qiaojie 

除了我以上说的，CryEngine还把所有可能出现问题的过程都用参数的形式开放给了美术，遮这也是非常重要的。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

接下来说说ColorGrading，ColorGrading其实是一堆的滤镜效果的集合，里面用了许多相当实用的滤镜。在引擎渲染的最后使用一些滤镜可以很好地使画面适合实际需要，使得渲染结果更加容易控制。这样的理念非常值得推荐，所有所有都围绕着ArtistFriend进行。其中有一个效果值得一提，就是ColorSharping，计算方法竟然是把Scene - SceneBlur*d，d为一个很小的值。也不知道谁发明的这方法，减去模糊的部分，突出细节。

![ColorGrading]({{ site.url }}/assets/CryEngine-profile/ColorGrading2.jpg)

![ColorGrading]({{ site.url }}/assets/CryEngine-profile/ColorGrading3.jpg)

![ColorGrading]({{ site.url }}/assets/CryEngine-profile/ColorGrading4.jpg)

经过数次的滤镜处理，渲染结果还是有比较不错的变化

![ColorGrading]({{ site.url }}/assets/CryEngine-profile/ColorGrading0.jpg)

![ColorGrading]({{ site.url }}/assets/CryEngine-profile/ColorGrading1.jpg)

在我的渲染中也添加了这样的特性，效果挺不错，据说CryEngine3加强了这个效果，不知道又加入了什么好玩的东西。

***

qiaojie

Scene - SceneBlur*d就是锐化模板

0  -1  0

-1  4 -1

0  -1  0

***

![gonggod]({{ site.url }}/assets/CryEngine-profile/gongminmin.png)

HDR解析的很好！

Scene - SceneBlur*d的事情，Pixar的新论文里也有提到。不过它的方法不是gaussian blur，而是保边缘的blur，这样细节应该会更自然。

***

kuguoxin198

还是问个cry地形OC的问题，对于地形，它是采用什么作为occluder的？地平线还是低模?

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 55# kuguoxin198 

我好像没有见到，我怀疑被合并在某个过程中了，可能是渲染地形的2.5DAO的时候，顺便开启查询，那样的话用的就是AABB。我自己用的AABB，因为AABB简单，渲染速度快，没有额外数据，而且容易放大，有助于解决OC错误。

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 54# gongminmin 

谢谢认可。CryEngine中锐化Blur的Kernel很奇怪，是不对称的对角四点，有点像你说的保边缘的blur。我回头再去看看，贴出来，希望大家指点指点，我郁闷中啊。

***

kuguoxin198

使用AABB的话，一个Terrain BLOCK 根据外貌使用多个AABB还是使用一个（应该不会是这个吧）?是使用程序自动生成还是手动指定？如果程序自动生成如何生成？？{:4_217:}  谢谢答复

***

![me]({{ site.url }}/assets/CryEngine-profile/napoleon.png)

回复 58# kuguoxin198 

我没有具体看到OC体渲染过程，只是猜测，因为地形有个AO的渲染，用了地形的ABB，所以怀疑之。AABB还有个好处是远处合并方便，AABB只要是地块的整数倍就行了，越远用越大的AABB，这样做很方便。反正OC体肯定要大，要不然就会跳得很严重。

***

![gonggod]({{ site.url }}/assets/CryEngine-profile/gongminmin.png)

贴几张Pixar论文中的图，他们就用保边界的blur做细节加强。

原图

![original]({{ site.url }}/assets/CryEngine-profile/original.png)

保边界的blur

![blur]({{ site.url }}/assets/CryEngine-profile/blur.png)

细节加强后

![sharp]({{ site.url }}/assets/CryEngine-profile/sharp.png)

***

# 结语

好吧，太久之前写的了，那时候资料少，自己也比较水，好多术语都不知道，小白得自己都看不下去了，就当留个纪念吧。还有一些内容其实我当时是分析了的，但是由于菜，没看懂（比如水），或者停更后很久才想起来（比如大气散射），有点点小遗憾。

本来打算做一个填坑系列的专辑，现在感觉时代在进步，以前魔改的OGRE已经过时得不成样子了。由于最近开始在[GitHub][github]上更新自己从零开始挖坑的Venus3D([https://github.com/Napoleon314/Venus3D][venus3d])，计划在未来的POST中以日记的形式发布一些细节一些感悟，做到图形的时候一定会把相关的技术在这几年的变化进行回顾，分析和讲解。

最后还要特别感谢下叛爷([KlayGE][klayge])在我写这篇帖子时给的大力支持，在图形上带我起飞，KAMUI同学给的鼓励，[钱老板][bossqian]安利我用的Jekyll， 以及我并不认识但是由于网页不6而抄了他模板的[RainyAlley][rainyalley]。

[opengpu]:http://www.opengpu.org/
[origin]:http://www.opengpu.org/bbs/forum.php?mod=viewthread&tid=2870&extra=page%3D1
[klayge]:http://www.klayge.org/
[github]:https://github.com/
[venus3d]:https://github.com/Napoleon314/Venus3D
[bossqian]:http://qiankanglai.me/
[rainyalley]:http://blog.rainyalley.com/
