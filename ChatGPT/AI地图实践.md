
大家好，我是坤哥



AI 时代已至！在工作中我大量使用 ChatGPT 来提升工作效率，得到了很好的效果

今天我就给大家分享一个案例，来看一下我在工作中是利用 AI 把原本半天的工作量压缩到不到半小时的。希望能对职场人士尤其是程序员群体有所启发



最近我司要做一个 AI 工具地图，效果如下



![](https://img-blog.csdnimg.cn/img_convert/10ee97b29f8905b9f8c66ac88cdd39a8.png)



需求是把每个 AI 产品的图标分门别类地先合成一个小图，再把所有小图整合成一个 AI 大地图，注意每个图标下面的文字都是其对应的产品名哦,放大看某一类AI产品的效果如下:



![](https://img-blog.csdnimg.cn/img_convert/9853da7598ac80b5184edc4e6c4de7d4.png)







如果把所有的工作如包括下载图标，合成每个类别的 AI 产品，再到合成最后的大图都交给 UI，可想而知这样的工作量是非常巨大的，所以我们就想能不能尽可能地减轻 UI 的工作量，用技术的手段至少能先做到分门别类地合成小图，这样 UI 所要做的事就比较简单了，只要把小图拼成大图就行了







首先我们需要找到这样 AI 图标，毫无疑问 AI 导航网站最合适不过了，这些导航网站基本上分门别类地给你整理好了这些AI产品的图标，我们决定使用 https://ai-bot.cn/ 这个导航网站里的图标，它的首页截图如下



![](https://img-blog.csdnimg.cn/img_convert/634bcbf1fbc81efd71ee4760f07c5d3e.png)



好了再来明确我们的需求，首先需要获取每一类 AI 产品下的图标，并将图标命名为此 AI 对应的产品名，然后将这属于同一类AI产品的图标置于同一个文件夹下，如下



![](https://img-blog.csdnimg.cn/img_convert/567b2c18cb2f3355533b5c6950df981b.png)







图标有几百个，如果人工一个个下载图标并命名，工作量巨大不说，还很容易出错，最容易想到的当然是用脚本如 Js 或 Python 来爬取网页中的图标和文案，但是如果人工去写脚本，也挺费时的，而且很难一次性写对所有的代码， 需要花很多时间 来 debug，所以写脚本这样的重活最好让 ChatGPT 来帮我们写，又快又好，只要我们把需求写清楚，ChatGPT 基本一次性就能把脚本给我们写好。







我们首先观察网站的结构，注意到网站的结构很相似，基本都是 「AI 类别标题 +  AI 类别图标集合」这样的组合结构，我们就取一个来观察



下图中绿框为AI 类别标题对应相应的 div，红框为标题下的图标集合对应相应的 div



[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-oYadyg7z-1691679560090)(/Users/ronaldo/Library/Application Support/typora-user-images/image-20230720103226947.png)]







继续观察，每一个类别的标题对应着 class 为 d-flex 的  `div > h4 > i` 中的文字







![](https://img-blog.csdnimg.cn/img_convert/570b18be89410322111bc917cfdda408.png)







而每个 AI 图标的 url 和名称在 html 中的元素如下



![](https://img-blog.csdnimg.cn/img_convert/030e7312ed4a4cf55bc7f07b3321262c.png)



`注意`：有一些AI产品分类如热门工具等不是这样的结构，不过结构都非常类似，用  Visual Studio 来将它们调整成以上的结构即可 







了解了我们要提取的 html 结构后，我们就可以给 ChatGPT 下指令来让其为我们生成脚本来提取每一类 AI 产品下的图标并在下载后将其命名为相应的 AI 产品名了，指令如下（test.html 即网页的 html）:



![](https://img-blog.csdnimg.cn/img_convert/a2e3dda910dac30bde7da89089115718.png)



最终它会给我生成类似以下的 Python 代码:



![](https://img-blog.csdnimg.cn/img_convert/cc307e7cfce74b309d69121c07b06363.png)



放在本地，下载对应的依赖后一键执行即可按要求提取每个分类下的AI图标，如下 



![](https://img-blog.csdnimg.cn/img_convert/567b2c18cb2f3355533b5c6950df981b.png)



做到这一步还不够， 为了进一步减轻 UI 的工作量我们希望帮 UI 把每一个 AI 类别的图标合成一张大图，如下：



![热门工具](https://img-blog.csdnimg.cn/img_convert/20890e864227540251e550c50fe705ec.png)



合成图片这种工作 Python 也能做，指令如下：







![](https://img-blog.csdnimg.cn/img_convert/6b194cf6078a013e657603cf7fb54bd2.png)







执行代码后效果如下:



![](https://img-blog.csdnimg.cn/img_convert/6c09d9eba8ffc48600abaae1bc7a1928.png)



可以看到，每一类的图标都合成了一个大图



这样的话交付给UI的就是一些合成好的每个 AI 类别的小图了，他们拿到后再将其拼成一张大图就相对容易多了





\### 总结



本文给大家展示了一个典型地利用 AI 来提效的案例，这只是其中之一，实际上在工作中我大量使用了 AI 来编码，以网站的后端代码为例上，90%的代码都是 AI 写的，当然指令还是我的下的，有人说 ChatGPT 的出现可能会替代程序员，但就我的大量实践经验来看，暂时还没达到这种程度，比如我一开始让它写后端，但它没给我处理跨域这种情况，这就需要程序员本人有相关的经验指引它补全，再比如我希望后端的很多接口都需要有用户身份的验证后才能进入具体的代码逻辑，但又不想每个接口都写重复的校验代码，那就需要指示 GPT 帮我封装校验逻辑，使用类似`export default withMiddleware(handler, cors, authenticate)`这样的责任链的方式来重用代码以提高代码的可扩展性，所以事在人为，对于程序员来说，你越资深，掌握的知识越多，就越能指示 ChatGPT 最大程序地发挥其功效， AI 能够帮我们处理那些重复的，不需要怎么动脑的工作，但更高层次的抽象还是需要程序员来指示它来完成。未来 AI 可能会进化，但至少当下我们能做的还是提升我们的功力以进一步利用释放 AI 的潜能



我建了一个 [AI 导航网站](https://ainavtech.com/)，欢迎大家体验
