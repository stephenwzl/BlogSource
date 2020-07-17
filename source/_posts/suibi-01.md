---
title: 漫谈前端开发怪力乱象
date: 2019-08-28 19:50:48
tags:
---
> 文题所指的前端为客户端、web前端等 GUI类开发工作的统称

> 本文皆个人观点，如有不妥，请勿辐射误伤他人

以前看过一本不错的小人书，书中有个观点是这么写的：学会不如会学，会学不如会用，会用不如被用。大致可以将程序员儿分为四个阶段：学会、会学、会用、被用。绝大多数人还处于会学和会用的入门阶段，如果你感觉你达到了“被用”的阶段，那么恭喜你，喜提人生第五大错觉。

不过这么嘲讽太过了，作为一个GUI开发工程师，我也只有资格对见到的现象做一点点批判和自省而已。

## 怪象1：固步自封
经常碰到这样的问题：一个业务需求多端合作，比如说要A端给B端传递一个短期内相对固定的文案或者话术。最后给的结论是A给B传递 enum，B根据 enum自行映射决定使用哪个文案。A端没有察觉到可能存在的问题，B端也没有抱怨这种使用是否合理。 
（ 此处 [黑人问号] * 3 ）  
我们从站在不同角度来思考这个问题：
A君觉得：这种 hard code代码也太恶心了，不如定义 enum优雅. 以后有改的需求也不用我这里改，最多加几个 enum就好了。  
B君觉得：A不愿意改有什么办法呢，反正这里也不会经常动，我们产品体量也不大，做那么复杂没意义。万一到哪天要改了再说，都不一定是我改了呢。
产品汪：来来来，这期有个需求，我们把上次的文案做成可运营的，这样每次展示不同的文案，用户一定觉得这个产品很有温度呢。 

如果A端和B端在思考技术方案时上一个台阶，比如说这么考虑：这个文案是我们产品全局性的提示文案，A端直管传递，B端只管展示，那么版本更迭，文案改动，是不是维护成本就很低了呢。这样还能提供文案运营的能力，哪天产品汪想要就有，难不成他还想让文案颜色根据用户手机壳的颜色改变。


## 怪象2：从众如流
各位是不是经常碰到这种事情呢：
- 接手一个项目开发需求，不知道该怎么下手，按部就班 cmd+C/V 
- 接入新的技术栈，动不动就要有人提议：我们看看xxx团队怎么做的吧，他们这么做，我们也这么做！

把这俩case当做反面太矫枉过正了，我只想引出这个话题：从众如流 （对不起我刚发明的词）
人是会从众的，但是团体不具有思考能力，所以才需要领导。但编程不是，编程需要思考，需要想清楚本质。如果编程的人不能思考了，那还不如直接造神，票选出一个程序员界的KOL，他说什么就是什么。事实是这样吗？不是，事实上不少能力不俗的程序员都互相瞧不起，还搞出个鄙视链来，说明蛮会思考的。 

动动脑子吧，不要再一味从众了，做技术第一时间就找案例的方式不可取，这种行为无异于程序员宗教找认同感。我们在做项目、维护项目、接触技术的时候不都应该先思考需求是什么？这个技术怎么学会用？这个技术点的本质是什么？通过思考、比较、观察确定适合自己的最佳的使用方式。

> 划重点：思考清楚本质，最适合的才是最好的。不要一味地照猫画虎


## 怪象3：模式松散
这里指的模式松散并不是编程范式里面的模式，而是一种做项目的行为习惯。比如：
- 为了解决没有电也想照明的问题，发明了太阳能手电筒。为了解决太阳能手电筒没有光的问题，发明了电池供电的手电筒给太阳能手电筒发电。
- 对外提供基础服务时，使用者要一辆摩托车，基础方提供了半辆摩托车零件、半辆自行车零件和一捆鞋带，让使用者攒一辆摩托车。
- 攒起来的摩托骑着不得劲儿，于是问提供方：为啥摩托跑不快啊？是排量不够大啊还是我蹬得不够快啊？ 提供方回复如是：这个问题呢是个空气动力学问题，说了吧你也不懂，现在我建议你换人蹬试试，别人蹬肯定更快。下个双月我们会换钢丝儿代替鞋带给摩托车别零件儿。到时候把您那一万四千多处绑的鞋带儿松开换钢丝儿就行。

用问题去解决问题，得到的一定是问题。那这和我说得模式松散有啥关系呢？没关系，就是批判一下。我们在解决问题的时候，是不是应该更多地想一下，生产一个新工具对整个体系的影响是什么？如果脱离了体系，那就新建一个体系。就好像你写代码一样，你应该对体系负责，做到对内高内聚，对外低耦合。如果你做了若干个工具，没有体系化地解决问题，让问题逃逸到你可控的范围外，那工具带来的麻烦将比便利多得多。

除了要构建可控的体系，还得为体系负责。所有人都可以是用户，并不是说臭写代码的就要比单纯玩手机的用户低一级。写代码的是技术体系产品的用户，服务不好用户，吹出去再🐂🍺的产品也玩完儿。

> 潜移默化的体系习惯比你拙劣的文笔撰写的文档要管用得多

纵观所见的三大现象，还是能反映一些问题的：
- 只学会，不会用。学会了一些乱七八糟的代码洁癖，无异于拿粪勺在粥锅里捞老鼠屎，干净局部却污染全局。
- 学会也会学，就是不会用。搞到最后照猫画虎，没有走出自己的特色化道路来。盲目迷信所谓的权威
- 离被用的阶段走得越来越远。不但不会被用，还搞砸了合作方的心态。PVP游戏里面最重要的就是心态，跟你合作的队友心态被你搞砸了，合作没有共赢，只有双输。服务方始终会觉得使用方在乱叫，使用方也始终只会觉得服务方不靠谱

写代码说到底是一个合作的游戏。为需求认真负责多着想，为项目工程的可维护性多着想，为他人多着想，你好我好大家好，就不会存在互怼单练的问题了。