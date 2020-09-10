## Trigger

Trigger定义了何时开始使用窗口计算函数计算窗口。每个窗口分配器都会有一个默认的Trigger。如果，默认的Trigger不能满足你的需求，你可以指定一个自定义的trigger().

trigger接口有五个方法允许trigger对不同的事件做出反应：

onElement():进入窗口的每个元素都会调用该方法。

onEventTime():事件时间timer触发的时候被调用。

onProcessingTime():处理时间timer触发的时候会被调用。

onMerge():有状态的触发器相关，并在它们相应的窗口合并时合并两个触发器的状态，例如使用会话窗口。

clear():该方法主要是执行窗口的删除操作。

## TriggerResult

CONTINUE:什么都不做。

FIRE:触发计算。

PURE:清除窗口的元素。

FIRE_AND_PURE:触发计算和清除窗口元素。