+++
title = '自动化测试'
date = 2024-08-22T14:29:40+08:00
draft = false

tags = ["自动化", "爬虫"]

+++

其实叫自动化测试不是很恰当，因为本质需求是操控一台或多台手机，自动化地完成一些固定流程的任务。最正面的广泛应用的场景当然是自动化测试，开发这类工具的厂家的最初的目的往往也都是自动化测试，但其实也可以用它来做爬虫。

列举一些方案或者工具

* [打造无障碍应用  | App quality  | Android Developers](https://developer.android.com/guide/topics/ui/accessibility?hl=zh-cn)

  编写一个自己的安卓APP并开启无障碍服务，然后APP就可以实现去控制手机中的其他APP。很多的群控软件都是基于无障碍开发实现。优点是理论上开发完成后无其它依赖，但是我实测对手机型号似乎有要求，各厂商可能都有一些定制化的安全设置。

* [openatx/uiautomator2: Android Uiautomator2 Python Wrapper (github.com)](https://github.com/openatx/uiautomator2)

  一开始接触手机app爬虫用的就是这个，那个时候作者开发完很久不更新，有些bug也一直没修复，似乎废弃了这个项目。用的不是很爽，但基本功能还是能够实现的。

  最近再看时，惊喜地发现作者又开始更新了，weditor已经被uiauto.dev所替代了，atx-agent也不需要了(原来还要防止它被手机电量优化给干掉)。

  但是有些陈年老bug还在，比如这个 [BUG: container's Child element is taken from next container if child not exists · Issue #903 · openatx/uiautomator2 (github.com)](https://github.com/openatx/uiautomator2/issues/903)，我印象我刚开始用的时候就有，那时还比较菜，各种dirty workaround。现在依据作者提示用xpath可以完美绕过。

* [firerpa/lamda: ⚡️ Android reverse engineering & automation framework | 史上最强安卓抓包/逆向/HOOK & 云手机/远程桌面/自动化取证框架，你的工作从未如此简单快捷。 (github.com)](https://github.com/firerpa/lamda)

  没错又是它，lamda，它也可以实现自动化，但我发现它只提供了基本页面元素定位功能，好像没有展示页面元素的层级结构。这在一些需要遍历某元素下的所有子元素之类的场景不是很友好。但是不是也可以用别的工具来辅助？比如上面的uiauto.dev，应该没问题，还没尝试过。

* [Airtest Project (netease.com)](https://airtest.netease.com/)

  这个网易出的，有个还不赖的IDE。它似乎想做一个大而全的，一个是跨平台，支持Windows、Android和iOS等多种平台，一个是支持多种UI框架的UI控件识别，支持Android原生、iOS原生、Unity3D、cocos2dx、UE4和Egret等平台。我第一次用的时候似乎没遇到什么很大的障碍，体验似乎还不错，可能凑巧手机配置没问题。但第二次用的时候就各种问题，而又是个时间紧任务重的项目，就果断放弃了。可能也不是它的问题，而是国内手机厂商的定制化导致安卓的分裂，使得它就是很难适配，你看看这个长长的问题列表[Android连接常见问题 - Airtest Project Docs (netease.com)](https://airtest.doc.io.netease.com/IDEdocs/3.2device_connection/3_android_faq/#2_1)

* [Welcome - Appium Documentation](https://appium.io/docs/en/latest/)

  似乎也挺有名的，不过还没有用过，只是记录一下，有机会再试。



自动化测试这种爬虫方案，虽然看起来比直接逆向扒接口要low，但其实也有它的优点和使用场景的。首先不需要逆向，门槛一下就低了不少；其次是确定性，就是页面能看到的数据，这种方式可以确定能爬下来。对于一些需求量不大，频次不高的场景，其实很适合。

另外通过自动化测试来爬页面数据也不是只有识别页面元素这一种途径，如果能够抓到包，结合mitmproxy把数据dump下来也是一种思路。即自动化测试只负责操作页面，让它触发请求数据的动作，爬取数据的过程通过mitmproxy来完成。

