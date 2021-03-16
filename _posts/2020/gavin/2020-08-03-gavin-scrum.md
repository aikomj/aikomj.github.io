---
layout: post
title: 架构师敏捷管理思想Scrum框架
category: icoding-gavin
tags: [icoding-gavin]
keywords: architect
excerpt: Scrum敏捷具体落地实现的框架，就好像spring是落地实现java MVC的框架一样，推崇的价值观是自管理
lock: noneed
---

## 前言

技术架构就是技术选型

> Agile Scrum践行

Agile 敏捷的意思，就是我们的开发， 无论你用java还是其他语言，封装继承多态的思想

Scrum就是敏捷具体落地实现的框架，相当于spring落地实现的一种框架

> 两种项目的解决方法

![](/assets/images/2020/icoding/scrum/two-ways-to-resolve-problems.jpg)

- 预定义，好比甲方项目，按照要求实现
- 实验性过程，好比互联网项目

## 1、Agile 的起源

Keywords：20世纪末，互联网，精益生产

敏捷开发不是第一种出现的开发方法，也不会上最后一种，2000年前后是互联网开发的高峰期，现在我们中国的互联网巨头都是那段时间成立的，国外略早一点。互联网未必是最早诞生敏捷的开发行业，实际上敏捷的鼻祖应该算是日本制造业的“精益生产”，但互联网绝对是敏捷开发得以成长、发展和壮大的阵地。

有几个特征在互联网开发中是鲜明的，几乎只有敏捷开发才能与之适应

- 失去客户的概念，需求也因之模糊
- 没有特定的交付期，“越早越好，越快越好”
- 开发人员参与需求和设计，对工作主动性要求提高

## 2、什么是敏捷

它不是一套方法论，不是过程或者框架(工具层面上的东西)，<mark>它是一种核心的价值观、思想</mark>

它主要有4种价值观的体现：

- 与客户个体的互动胜于流程与工具

  面对面的交谈胜于聊天工具，面对面的交谈沟通效率高，能达成共识，能观察到对方的表情，而文字是苍白的

-  可交付的软件胜于详尽的文档

  可以交付的东西 ，每一个程序猿都有一个玻璃心，反玻璃心就是要每天都要review自己交付的东西，提取把问题和风险暴露出来，提前解决

- 客户合作胜于合同谈判

  要站在用户的角度做开发，软件的价值体现在是否对用户有用 ，避免需求蔓延，在沟通好的范围内做好。

- 响应变化胜于遵循计划

  需求在变化，方向错了，你就要变化

![](/assets/images/2020/icoding/scrum/agile-value.jpg)



> 加班是用来通过用工作以外的时间做工作时间内能力之所及而没有完成的事情，而不是能力之外的事情。



### 敏捷原则

<mark>1、尽早地、持续地交付有价值的软件来使客户满意是最高优先级的工作</mark>

2、即使到了开发的后期，也欢迎改变需求，敏捷过程适应变化来为客户创造竞争优势

3、以几周几月的间隔频繁交付可工作的软件，交付间隔越短越好

4、在整个项目开发期间业务人员要和开发人员一起工作

5、激励团队成员建设项目，提高所需的环境与支持并信任他们能够完成工作

6、在团队内部以及团队之间最有效最高效的传递信息的方式是面对面的沟通

7、可工作的软件是首要的进度度量

8、敏捷过程提倡持续性的开发。发起人、开发者、用户应保持长期的、恒定的工作速度

9、持续追求技术卓越和优良设计能提高敏捷性

10、<mark>敏捷的根本在于简单，</mark>

11、最好的架构、需求和设计出自于组织的团队

12、每隔一定的时间，团队反思如何更有效的工作，然后相应的做出调整



### 敏捷是一种状态

agile is a state

- 从产出来看：频繁地交付可见的、可用的成果（软件）
- 从团队来看：主动地、持续地学习和改进



## 3、Agile 敏捷流派

敏捷是一种核心思想，价值观，落地可以有不同的实现框架，如下

- <mark>Scrum: 3355</mark>

- XP: 结对、重构、CI（持续集成，CD 持续部署continue deploy，自动部署）

  XP就是极限编程，

  1、结对就是两个程序员做同一件事情，A在做，B在看，B在看的过程中肯定带有评判、检查的眼光，在这样的情况下，A在做的时候累了困了容易犯错，B可以不断地提醒他避免犯错，在最终的结果上达成共识，检查的都检查了，提升软件的质量和效率

  2、重构就是我们不断的对系统进行自我优化和修改，自我成长

- TDD 测试驱动开发

- FDD 特性驱动开发

- Kanban 看板方法

- 水晶方法

- DSDM  动态软件定义和管理

除了Scrum，后面的所有方法都是有缺陷的，就是只注重工程师文化，就是极客风格，尽量没有管理，而是通过工程师的自管理，自驱动的方式，来做到高效、高质量的产出。个人觉得很少的团队能做到这样子 ，国内的团队都是需要管理的



### Scrum的起源

起源于橄榄球运动

![](/assets/images/2020/icoding/scrum/refence-1.jpg)



> Scrum 是一种灵活的敏捷软件开发管理过程，这个名词起源于英式橄榄球。由 Ken Schwaber 和 Jeff Sutherland 提出，它将软件开发团队比作橄榄球队，全队有明确的最高目标：发布产品的重要性高于一切，团队高度自制，成员们熟悉开发过程中涉及的各种技术，紧密合作，确保每个迭代Sprint都朝着最高目标推进。而且每隔2～4周，每个团队成员都能看到实际工作的软件，并据此决定发布这个版本还是继续开发以加强它的功能。

<mark>scrum作为一种敏捷过程让我们关注于在最短时间内交付最高价值，它是迭代和增量式的</mark>

![](/assets/images/2020/icoding/scrum/scrum-is-a-framework.jpg)



<mark>3355是什么，如下：</mark>

- **3个角色** Scrum roles

  1、产品拥有者 Product owner 

  2、开发团队 Development Team

  3、落地主管 ScrumMaster 相当于教练的角色，维护秩序，确保scrum框架落地的时候遇到问题能够有人通过相应的方法技巧解决它 

- **3个资产** Scrum artifacts

  1、产品待办事项 Product Backlog，相当于概要设计

  2、迭代待办事项 Sprint Backlog ，相当于详细设计，涉及到前端、后端的功能开发，需要的工时、资源等

  3、增量Increment，其实就是系统发布的版本包，版本是增量的

- **5个事件** Scrum events（确保scrum落地执行过程中的动作标准到位，涉及的一些行为规则）

  1、迭代计划会 Sprint Planning 

  2、每日例会 Daily Scrum

  3、迭代评审 Sprint Review，主要针对下一发布版本评审，有没有泄密，有没有不该开放的接口开放了 

  4、迭代回顾 Sprint Retrospective，对整体迭代版本回顾

  5、迭代本身 Sprint，本身也是一个事件在不断的变化

- **5个价值观** Scrum values

  勇气 Courage、开放 Openness、专注 Focus、承诺 Commitment、尊重 Respect

<mark>Scrum 的迭代流转图</mark>

![](/assets/images/2020/icoding/scrum/sprint-cycle.jpg)

1、产品拥有者对产品进行概要设计（功能的描述与优先级）Product Backlog，与开发人员一起规划产品的迭代计划Sprint Planning，进而拆分出迭代待办事项的具体任务Sprint Backlog(给到前端、后端、美工等开发人员的参与)，一个一个水平面上的细项给到对应的开放人员。

2、开发人员执行任务Sprint Executing 生产产品功能，产品拥有者、开发人员、教练参与每天的功能例会 Daily Scrum。

3、版本的增量开发，下一发布版本的评审，产品整体迭代版本的回顾



### 3个角色 Scrum Roles

> 1、开发团队 Development Team 

**特征**

- 5～9个人（7+-2），双披萨团队，一个团队吃两份披萨能吃饱，说明团队已经够大了，团队尽可能不要太大

- 跨职能 Cross-functional

  具备不同领域的必需技能 All skill set in different functional areas，如后端会点前端的知识 

  不同层面的成熟度 Different maturity levels 

  T型人才 T model person

- 全职 Full-time，就是迭代开始了，不允许开发人员换出去，避免打乱影响当前迭代版本的开发计划，直到当前迭代版本完成。所以只允许不同迭代版本间换人，不允许同一个迭代版本开发过程中更换成员 。

  Membership change only between sprint

- 自组织 Self-organized 

  没有管理者的头衔 No management title，就是所有团队成员都是管理者的概念 

  团队决定如何去工作 Team decides how to work，大家都对团队的工作负责，营造富有责任心，积极向上的工作环境，大家都得到成长

  全员责任制 Whole team accountability ，明知故犯比不知道犯下错误更可怕，

  团队承担大部分微观管理工作 Team takes most of the micro management work

**使命** Mission

- 交付产品增量 Deliver Product Increments
- 对“怎样做”和“产品质量”负责 Responsible for HOW & quality of deliver
- 参与Sprint 中的所有会议 Participate in Sprint events
- 管理Sprint Backlog 并跟踪进度 Manage Sprint backlog and track the progress
- 找到团队内部合作的最佳方法 Figure out the best way to work together as a team
- 与其他团队协作 Collaborate with other teams and parties，如果
- 持续自我改进 Make continuous self improvement

> 2、产品拥有者 Product owner

**特征**

- “一”个人 One person playing the role，可以理解为产品经理，“一”表示最终决策的人
- 被搜于产品的决策权 Authorized to make decisions on WHAT
- 驱动产品走向成功 Drives product success
- 提供产品领导力 Provide leadership on product
- 面向干系人（产品相关的利益人）代表团队 Represent project to the stakeholders
- 面向团队代表干系人 Represent stakeholders to the team
- 和所有人合作 Collaborates with everyone，得道者多助，失道者寡助

**使命** Mission

- 建立产品愿景(目标目的) Creates the Product Vision
- **从 “为什么” 开始 Start with WHY** 
- 定义产品功能（做什么） Defines the feature of the Product (the What)
- 负责最大化投资回报 Responsible for Returns of Investment (ROI)
- 为最好实现业务目标将产品Backlog 排定顺序（优先级）Orders/Prioritizes Product Backlog to best achieve goals; 就是规划Sprint backlog，人再多也架不住事多，所以要规划
- 决定版本发布日期和内容 Decides on Release date and content
- 根据反馈调整产品Backlog\优先级 Adjust Product Backlog /priority according to the feedback; 修改 Sprint Backlog
- 参与Sprint事件 Participates in Sprint events
- 愿意投入到合作中并且在需要时被找到 Be committed to collaborate and be available
- 接受或退回工作成果 Accept / reject work results，开放人员找PO验证工作成果

> 3、教练 ScrumMaster

**特征**

- 面向管理层代表团队 Represents project to the management

- 面向团队代理管理层 Represents management to the team

- 不是一个项目经理或者团队经理 Not a project manager or team manager

- **没有管理头衔，不代表团队做出决定** No management title,cannot make decisions on behalf of the team

- 一个仆人式的领导者，授权的牧羊犬 Is a servant leader, authorized to be a sheep-dog 

  维护Scrum秩序，Scrum制度的维护者

- 更像一个教练 Coaching the team rather than being a player

- 团队和组织谋求变化的代理 Change agent of team and organization

- 具备良好的引导等技能 Good techniques on facilitation etc ,沟通技能强

- 听多于说 Listens much more than tell 

- 思路开放 be open-mind

**使命** Mission

- 负责Scrum 价值观、原则、规则被采纳和彰显 Responsible for enacting Scrum values,principles and rules
- 移除障碍 Remove impediments
- 为产品拥有者和团队服务 Serves the Product Owner and team
- 帮助培养团队 Help to develop the team
- 保护团队 Protects the team 免受一些不正当的要求侵害
- 辅助团队高效协作 Facilitates team collaboration/productivity 
- 想办法提升Scrum 在整个组织中的效果 Work to improve effectiveness of Scrum in the organization

**小结**

产品拥有者：负责产品方向

开放团队： 产品结果的执行与产出

教练：维护产品落地的节奏，正确性，产品开发过程的秩序保障

三个角色的出现，好比我们中国的传统体育运动：赛龙舟

Different Scrum roles in one team

![](/assets/images/2020/icoding/scrum/sailongzhou.jpg)





### 3个资产 Scrum Artifacts 

> 1、产品待办事项 Product Backlog(概要设计)，关注业务流，信息流

- 一份动态的列表（需求会变化），包含了产品可能具备的功能，相当于概要设计

  A dynamic list of functionality the product might include

- 一个健康的Product Backlog应当具备<strong style="color:blue">DEEP</strong>原则

  A healthy product backlog must be DEEP

  - 恰当的详细程度 <font color=blue>D</font>etailed appropriately
  - 已估算 <font color=blue>E</font>stimated，估算需要做多久
  - 涌现式 <font color=blue>E</font>mergent，需求不是一成不变的，跟喷泉一样是会不断变化的 
  - 优先级排序 <font color=blue>P</font>rioritized

- 对所有人开放但最终由产品拥有者维护

  Open to all but ultimately groomed by the Product Owner

- 关注“**什么**”带给用户最大价值

  Focus on "**WHAT**" brings users the biggest value

- 最优秀的产品拥有者从“**为什么**”开始

  The best Product Owner starts with "**WHY**"，

**1. 例子：任务泳道图**

![](/assets/images/2020/icoding/scrum/product-backlog-example.jpg)



开发成员的任务泳道图，每个成员的任务都写成卡片贴出来，任务的拆分原则：

**2. INVEST原则**

- 独立的 <strong style="color:blue">I</strong>ndependent
- 可商议 <strong style="color:blue">N</strong>egotiable
- 有价值 <strong style="color:blue">V</strong>alued，首先任务是独立的，可商议，有价值的，可以估算工作量的
- 可估算 <strong style="color:blue">E</strong>stimable
- 足够小 <strong style="color:blue">S</strong>mall
- 可测试 <strong style="color:blue">T</strong>estable

![](/assets/images/2020/icoding/scrum/horizontal-vertical.jpg)



**3. 用户故事 User Stories（需求）**

就是用户“为什么”要提出这个需求，卡片格式如下：

![](/assets/images/2020/icoding/scrum/user-stories.jpg)



- 使用简单的书面描述，定义一小片用户可以评估、验证的功能

  Concise written description of a piece of functionality valued and verifiable by the user

- 粒度（需求）

  ![](/assets/images/2020/icoding/scrum/granularity.jpg)

- 验收标准

  ![](/assets/images/2020/icoding/scrum/acceptance-criteria.jpg)

**4. DoR 准备就绪**

<strong style="color:blue">D</strong>efinition <strong style="color:blue">o</strong>f <strong style="color:blue">R</strong>eady

为你的当前项目列出至少3项DOR，当我们把这3件事情做完了，才算是项目准备就绪好，要提前把DOR 做声明，规避风险 

> 2、迭代待办事项 Sprint Backlog（详细设计），实施目标的集合

- 产品待办事项 Product Backlog的延伸和子集

  Extension and subset of the product backlog

  - 为实现Sprint目标所要完成的工作集合

    The set of work to achieve the Sprint Goal

  - 涵盖‘恰到好处’的设计

    Just in time design in considered

  - 将大块的工作分解为更小的单元

    Breaks large work down into smaller pieces(PBI -> SBI)

    PBI=Product Backlog Item

    SBI=Sprint Backlog Item

  - 关注“**怎么做**”的问题，如何在一个Sprint内完成工作以交付价值

    Focus on '**HOW**' team is going to get the work done and deliver the value in one sprint

- 被开放团队拥有

  Owned by the Development Team

  - 团队从产品待办事项 Product Backlog 中选取他们可以承诺完成的项目并创建迭代待办事项Sprint backlog

    Team select items from the Product Backlog they can commit to completing and creates the sprint backlog

  - 协作完成，不是由ScrumMaster 负责

    Collaboratively, not done alone by the ScrumMaster

  - 一个可视化的工具让团队在sprint内部自我管理（贴任务卡片）

    A visible tool for the team to manage itself during the sprint

**1. 如何管理Sprint Backlog**

- 任务板是一个常见的用于管理sprint backlog 的可视化工具，也可以使用电子化的东西

  Task board is a common visible tool to manage spring backlog

  ![](/assets/images/2020/icoding/scrum/task-board.jpg)

- 自组织：团队成员或小分队自己领取工作(蜂拥而至的概念)

  Self-organized：Individuals or small groups sign up for work

  - 团队一起将PBI分解为SBI

    Team decomposes PBI to SBI

  - 合适的SBI颗粒度

    Team decides SBI granularity

  - 没有一个人主导任务的分配，任务是团队一起分解的，个人认领任务

    Work is not assigned

  - 完成一项任务才认领另外一项任务，比如一个周的迭代任务

    Sign up for new work after one work is done

  - 按照优先级，努力使一个PBI尽早完全完成

    Based on priority and try to reach fully DONE on a PBI

- 团队每天跟踪Sprint 中剩余的工作，每日例会 Daily Scrum的作用

  Team tracks remaining work of the Sprint,daily

- 任何团队成员可以添加、删除、变更SBI sprint backlog item

  Any team member can add, delete,change the SBI

- Sprint 内的工作有可能动态涌现

  Work for the sprint may emerge 

> 3、增量 Increment

**啥是增量**

就是版本包

- 当前Sprint 以及所有已完成Sprint 内，已完成的Product Backlog 项的总和

  The sum of all the Product Backlog items completed during a Sprint and all previous Sprints

- 潜在可交付，并符合完成的定义

  Potentially shippable and meet the Definition of Done

- 必须的可用的产品，不管PO产品拥有者是否决定对外发布

  Must be in useable condition regardless whether the Product Owner decides to    

  release it 

**1. DoD 完成的定义**

Definition of Done

为你当前的项目列出最少8项DoD，可以覆盖质量、过程、功能和其他方面

List at least 8 DoD for your current project,you may cover quality, process,feature and other perspectives.

做完的定义：达到上线标准，测试已通过，可以发布

开发完成没提测试，就不是“完成”的定义

**2. Sprint 燃尽图 Sprint Burndown Charts**

- 每日更新，通常发生在每日站会之后

  Updated daily, usually after the daily stand-up

- 度量Sprint 剩余工作的总量

  Represent the amount of work remaining

- 剩余工作量估算

  Estimated remaining efforts

- 跟踪已完成项

  Tracking Done

燃尽图一般是向下收敛 的曲线趋势，如果是水平的行线状态，说明我们的任务一直处在没有完成的状态，没有更 新，就会使我们的项目风险越来越大，要集中加班快速的解决任务;如果是向上陡的话，那说明任务规划出现问题，也有可能是临时加的任务

![](/assets/images/2020/icoding/scrum/sprint-burndown-charts.jpg)

如果以周为单位，燃尽图其实没有必要画了，因为迭代周版本是非常细的了，也不会有多复杂的任务。

**3. 任务板 Task Boards**

- 对全世界可见 Visible to the world
- 随时更新 Update in real time
- 直观展示Sprint目标完成的进展 Represents the current progress toward the Sprint Goal
- 团队工作可视化管理的工具 Work visibility management tool for the team

![](/assets/images/2020/icoding/scrum/task-board2.jpg)

**4. 发布计划 Release Planning**

Scurm中版本发布计划是一道简单的算术题 In Scrum release planning is a simple math game.

| Total of Story points Estimated 用户故事点估算 | 300  |
| ---------------------------------------------- | ---- |
| Low Velocity 低速率产能                        | 30   |
| High Velocity 高速率产能                       | 50   |

需要几个Sprint 来交付所有的用户故事?

How many Sprints will be needed to deliver all User Stories?

根据上面的表格，我们就可以计算出按低速率产能有300/30=10个Sprint 迭代版本

按高速率产能就有300/50=6个 Sprint 迭代版本



### 5个事件 Scrum Events

![](/assets/images/2020/icoding/scrum/scrum-events.jpg)

> 1、Sprint 迭代本身

- Scrum 项目由一系列“Sprint” 组成 Scrum projects make progress in a series of sprints

  借鉴了极限编程中的“迭代” Analogous to Extreme Programming iterations

- 一个迭代通常2-4周，最多一个月，或者更短 Typical duration is 2-4 weeks or a calendar month at most,or even shorter

  Sprint 原来的意思是短距离快速奔跑(或游泳); 冲刺。所以我们的迭代是以周为单位，短距离冲刺

- 通常是固定时长的，有利于产生更好的交付节奏

  A constant duration leads to a better rhythm

  就是固定一周或者两周发布一个版本（迭代），

- 只有当时间盒到期时，Sprint 才结束

  Sprint ends only when the time-box expires

- 根据DoD定义，全部相关工作在sprint 内完成

  Product is developed according to DoD within a Sprint



**1、变更 Change**

- 不去改变Sprint的目标 Not to change Sprint Goal;

- 不改变当前运行中的Sprint的长度，你可以改变下个Sprint的长度，例如一周改为2周，但是要经过Sprint Review 会议评审，团队决策同意，避免频繁变更影响交付节奏

  Not to change Sprint length during a Sprint;

**2、中止Sprint**

- Sprint 可以被中止吗？可以

- 出于业务需要，Product Owner 可以取消Sprint

  Product Owner can cancel the Sprint if business circumstances require

- 如果无法完成任何东西，团队可以和PO协商应对

  Team can discuss with Product Owner to see how to handle,if they are unable to accomplish anything

- 重新做Sprint计划，所有还未完成的工作放回产品Backlog
- 罕有发生 Very rarely done

**3、第一个Sprint 开始前Before first Sprint**

讨论约定，沟通，明确到位

- 团队的工作约定协议是什么

- 团队将会使用的工具和工作过程

- 整体的项目计划，进行一次Backlog的梳理，使用DoR/验收标准/怎样演示

  Perform Backlog Refinement,Using DoR/AC/How to Demo

- 你的Sprint完成的定义是什么 What is your Sprint's DoD

> 2、迭代计划会 Sprint Planning 

**1、它包括两部分**

- Part One ：选择 Selection

  定义Sprint目标 Define the Sprint Goal

  选择团队可以承诺完成的Product Backlog项

- Part Two：计划 Planning

  决定如何实现Sprint目标-Decide how to achieve the Sprint Goal

  创建Sprint Backlog -Create the Sprint Backlog

  估算Sprint Backlog项(工时估算)-Estimate the Sprint Backlog Items

迭代计划会的时长，Scrum官方建议：1个月的Sprint 最长8小时，因此一周的Sprint计划会最长2小时

Timebox: max 8 hours for 1 month Sprint

**2、参与者**

- Part One 选择 Selection 

  参与者：Product Owner/Development Team/[ScrumMaster 可以不参与，他的核心作用维护秩序确保Scrum健康的执行，并不是管理团队，团队是自管理的]

  输入：健康的产品Backlog Healthy Product Backlog

  输入：最新版本的增量 Latest Increment

  输入：团队这个Sprint的产能/速率

  输出：代表这个Sprint目标，所一起选择的产品Backlog事项 Selected Product Backlog items representing the Sprint Goal

- Part Two 迭代目标的工作计划，授权团队本身落地实现这件事了

  参与者：Development Team/[Product Owner]/[ScrumMaster]

  输出：如何实现Sprint目标的工作计划Sprint Backlog

  输出：大家对Sprint目标形成共识 Mutual agreement on the Sprint Goal

Scrum敏捷推崇的价值观是全世界可见，我做的所有工作都不怕你来检查验证



> 3、每日站会 Daily Scrum

它实际上是对我们日常工作的一个检查，检查Sprint是否能按预期完成，完成任务板的进度更新 

- 参数 Parameter

  1) 每日 Daily

  2)  同一时间同一地点 Same time same place

  3) 15分钟 15 minutes

  4）站立 Stand-up

  5) 参与者 Development Team ,[Product Owner],[ScrumMaster]

- 为达Sprint 目标检视进展和调整计划

  1) 不讨论和解决具体问题，只陈述事实和问题本身 

  2）其他人可以受邀来旁听，全世界可见

  3）只有团队成员，ScrumMaster和Product Owner可以说话

- 帮助避免其他不必要的会议 Helps avoid other unnecessary meetings
- 分享日常工作中的一些经验，方式，方法，确保开发人员能够在具体的事情上面 得到一定的帮助，或者一定的引导。

**3个问题**

- 昨天我完成了什么？**为昨天的工作付出了多少努力**

  What did I get <strong style="color:red">DONE</strong> yesterday

- 今天我要完成什么? **今天的规划是什么**

- 有什么障碍影响我的进度吗，有什么可以和大家分享的

  Are there any impediments slowing/blocking my progress?

  不是向ScrumMaster汇报状态，而是向所有组员的广播，属于**自管理**的一部分

  This is not status for the ScrumMaster，it is broadcast in front of peers for <strong style="color:red">self-management</strong>

> 4、Sprint 评审 Sprint Review

- 检视和调整产品和产品Backlog 

  Inspect & adapt on the Product and Product Backlog

- **团队展示Sprint的成果** Team presents what it accomplished during the sprint

- 产品负责人接受“完成”的工作及退回“未完成”的工作

  PO accepts "Done" work and rejected "un-done" work

- 经常已Demo新功能（及其依赖的架构）的形式，获取反馈和讨论要做的调整

- 非正式，1个月的Sprint时间盒最长4小时，所有1周的Sprint时间盒最长1小时

  不要用幻灯片

- 全Scrum团队参与 Whole Scrum team participates
- 邀请所有干系人及感兴趣的人士 All stakeholders and interested parties are invited，全世界可见

> 5、Sprint Retrospective Sprint 回顾

**Retrospective 是一个Sprint结束后的整体回顾会议**，目的对整个团队在上个Sprint出现的问题，犯下的错误进行一个总结回顾，及时纠正，确保下个迭代能够做得更好

- 对我们如何工作进行检视和调整 Inspect & adapt on how we work

- 对过程的持续改进 Continuous improvement on process

- 时间盒：1个月的Sprint 最长3小时， Timebox:  以周为单位的话45分钟

- 通常30-60分钟 Normally 30-60 分钟，不适合太长

- 全Scrum团队参与 Whole Scrum team participates

- 三个问题：回顾是在一个Sprint结束后进行的会议，团队要决定下个Sprint的3个问题内容：

  开始做什么？Start doing

  停止做什么？Stop doing，就是某些功能没必要做了

  继续做什么？Keep doing

- 输出：下一个Sprint的改进行动计划

  Improvement Action Plan for next Sprint





### 5个价值观 Scrum Values





## 4、Agile Estimation 敏捷估算

就是估算任务完成的工作量

### 绝对值估算 Absolute Estimation

- 带有“单位”的数字，例如人天/小时，代码行数

  number with a 'unit',like MD/Hours,line of code

  适用已知的功能环境：这个需求任务你是做过的，或者很明确多少天可以完成

### 相对值估算 Relative Estimation

- 不带“单位”的数字，一个数字与另一个数字对比

  适用未知的事物功能环境：比如你做一个功能花了3天时间，做另外一个功能更复杂一些，那就比对一下，相对值估算花5天的时间，这就是所谓的相对值估算的概念

### 速率 Velocity

估算的最终目的是为了得到速率 Velocity，可以说是**团队的产能**，概念如下：

速率=团队可以在一个Sprint 中交付多少个用户故事点

Velocity = how many Story Points team can deliver in one sprint

- 速率是历史数据 Velocity is historical data
- 在第一个Sprint开始时估算的速率仅是最佳猜测 Best Guess before Sprint 1

只估算‘已完成’项 Measuring only 'Done'

> 例子：5个人的开发团队，一周就是5\*5=25个人天，25\*8=200个人时

### 计划扑克 Planning Poker

![](/assets/images/2020/icoding/scrum/planning-poker.jpg)

是一种评估完成开发任务需要工作量的方式。

团队是自组织的，所有东西都是自己管理自己。计划扑克，可以拉平所有成员对系统功能需求的认知，比如说我们每个人到目前为止都还没有领任务的，但是我作为PO 产品拥有者，已经提前把任务拆解为比较小的单元，团队一起开会，PO说明讲解一个个任务是做什么的，然后问大家：理解了吧。那么这个时候大家说理解了，这个时候需求功能已经描述完了，大家也都听明白了，大家需要对一个任务功能的工作量进行评估，每个人手上都有一副扑克，如需要5个人天（工时），我就出一张5，扣着反面放在桌子上，因为这样子可以表现出他们对需求最真实的一个理解（每个程序员都有一个玻璃心）。

然后我们一个一个的翻开，一般会有3个结局：

- 大家基本上一致，都是个3到4天，这个时候可以选择谁来完成这个任务需求

- 这个任务需求可能一部分人听懂，一部分人没听懂，听懂的人需要1天就可以完成了，没听懂的可能需求4天完成，这个时候差异就非常大了，原因有2点：

  第一：你(PO)的需求没讲清楚

  第二：需求讲清楚了，但是要4天完成任务的人员本身对需要的技术栈不了解或者不会，对未知的恐惧，那么这个时候你通过这件事情，就会发现有些人对某些技术栈确实是不理解的，是否可以采用一些措施帮助他理解和提升， 那么下次再做同样事情的时候他就有经验了。

- 如果确实需求讲清楚了，大家就需求技术栈都了解了，但还是一部分人出1天，一部分人出4天，那这个时候你就需要让他们说出各自的理由相互碰撞，沟通，快的人是否有什么技巧工具提升开发效率，慢的人是否考虑了工作中的其他问题。

最终做决策用多少天估算工作量，团队成员都参加计划会议，拉平了所有成员对系统功能需求的认知

**计划**

其实是团队估算完工时后得出的一个结论，它一定是要排定优先级的，计划就像洋葱圈一样一层一层的剥， 最后是一个刚刚好的局面

-  团队估算用户故事 Stories are estimated by team

- 计划做到什么程度 How much planning?

  Just in time 刚刚好

  ![](/assets/images/2020/icoding/scrum/planning-onion.jpg)

## 5、敏捷质量Quality

### 文化

- Agile Principle #1 敏捷原则的的第一条：

  我们的最高优先任务是通过持续的交付有价值的软件来使客户满意

- Agile Principle #9 敏捷原则的的第九条：

  持续的注重技术的优秀性及好的设计增强敏捷能力

- 破窗原理 

  如果我们自己都不坚持质量的原则， 有一天完好无损的窗户被人砸碎后，别人就会在别的玻璃上继续砸碎，这就是一个模仿的过程，换句话说就是有人破坏了规则，你不去维护秩序，就会有人模仿破坏规则，后果就会造成损失

- 《黄帝内经-素问》论医

  在问题出现之前，就把它解决掉，防患于未燃，将问题处理于无形，这才是最高明的 。这就是我们的质量想要追求的点，不是等到问题出现之后，通过一大堆分析管理方法来解决，为什么不去做从一开始就规避它尼。

### 测试

