---
title: 更新unity编辑器到6000.0.9f1后Player input出现警告的解释和解决
date: 2025-11-03 17:39:00
tags: unity
categories: 游戏开发学习
keywords: Input system 
---

# 问题出现：更新unity编辑器

更新unity hub后，提示我使用的旧版编辑器6000.1.4f1版本存在漏洞已不推荐使用，遂升级为最新支持版，然后发现player input组件出现以下警告（也可能是我之前一直没注意到？）

![image-20251103174316611](https://raw.githubusercontent.com/bbnostars/imgrepo/main/20251103174316646.png?token=AZMU6N3QT3PKY76OMAAVKE3JBB4XG)

大意是player input组件不推荐使用项目范围的actions资源，因为这是一个单例引用并且所有的actions maps都是默认开启的。你应该在Start()里禁用所有的actions maps后单独启用你设置的default action map。

# 一些疑问

初读这段警告我实在无法通顺理解其含义，什么是`Project-wide actions`？如果他是单例引用，那存在多个player input时是否存在相同引用的冲突呢？

# 查询文档

## 关于project-wide actions asset（以下称为全局actions asset）

顾名思义，是整个项目范围内都可用的Action Asset。可以在**Edit** > **Project Settings** > **Input System Package**中指定全局actions asset。

好处是可以把所有的输入系统配置全放在这个asset里面，需要访问actions的时候直接通过`InputSystem.actions`来访问。

## 为什么会出现这段警告

当前的action maps：

![image-20251103204518660](https://raw.githubusercontent.com/bbnostars/imgrepo/main/20251103204518694.png?token=AZMU6NZOWULGXNZHG4ZTILDJBCSB2)



使用一段代码来解释：

```c#
// 以下代码分别放入update和start中运行，结果相同
Debug.Log(UnityEngine.Object.ReferenceEquals(Player.Instance.GetComponent<PlayerInput>().actions, InputSystem.actions));  // 结果是 true → 两份相同的 asset

var asset = Player.Instance.GetComponent<PlayerInput>().actions;  //InputActionAsset
foreach (var map in asset.actionMaps)
{
    Debug.Log($"{map.name} is enabled: {map.enabled}");
}

// 当前激活的 map
if (Player.Instance.GetComponent<PlayerInput>().currentActionMap != null)
{
    Debug.Log($"Current active map: {Player.Instance.GetComponent<PlayerInput>().currentActionMap.name}");
}
```

运行结果：

![image-20251103203816524](https://raw.githubusercontent.com/bbnostars/imgrepo/main/20251103203816607.png?token=AZMU6N2YIIPSAUYJXFFB76DJBCRHQ)

可以发现Player身上挂载的Player input组件引用的actions和全局actions asset引用的是同一个对象。并且，不论是放在start中还是update中，player和ui的action maps始终都是激活的。

再看文档：

>The asset may contain an arbitrary number of action maps. By setting default ActionMap, one of them can be selected to enabled automatically when PlayerInput is enabled. If no default action map is selected, none of the action maps will be enabled by PlayerInput itself. 
>
>Notifications will be sent for all actions in the asset, not just for those in the first action map. This means that if additional maps are manually enabled and disabled, notifications will be sent for their actions as they receive input.

player input组件最开始只会激活组件上设置的default action，至于asset中其他的actions maps，需要我们自己手动管理是否激活。

问题出现在这里：警告中告知，全局actions asset里所有的actions maps都直接默认开启，并且player input收到输入时通知是会发送给其他所有的asset里的actions maps，此时如果不同actions maps中有**冲突键位**，就会导致同时触发相应的输入事件，这可能不是我们想要的结果。这也解释了为什么上面的运行结果中ui的action map是被激活的，即使player input的default map设置的是player。

如果单独为player input设置一个actions asset，通知就只会影响自己的asset不会发送到全局的，这样更好定位问题，也更符合逻辑。

所以如果选择为player input挂载全局actions asset，最好听从警告建议在start函数中手动控制不同actions maps的激活状态。

# 对于多个player input同时存在的情况

文档给出的解释是，如果多个player input组件同时挂载同一个actions asset，第一个player input会引用这个原始的asset，其余的则会创建一份私有拷贝并引用。此时每个player input其实拥有的是不同的actions asset引用。所以请不要在这种时候使用`InputSystem.actions`获取当前player input的actions，这个是全局的”单例“asset，并非该player input引用的。

为什么单例会有拷贝一说？

因为InputActionAsset本身继承的是scriptable object，它是一个资产文件，并不是严格意义上的单例模式。

# 总结

尽量避免设置player input的actions为project-wide actions asset。

如果项目只需要一个actions asset且想用player input，可是不为项目配置全局actions asset，或者自己在start函数中手动管理哪些actions maps需要默认启用或禁用。

实现本地多人游戏并使用player input时，不要使用`InputSystem.actions`控制player input挂载的actions。





