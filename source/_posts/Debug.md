---
title: Debug~
date: 2022-08-30 16:40:14
tags: [ Unity]
categories: [ Unity]
about:
description: "Unity踩坑日志。"

---

## PlayableAsset returned a null Playable on Instantiate

> PlayableAsset returned a null Playable on Instantiate

由引擎返回，他表示整个Timeline都没有返回有效的Playable。这意味着整个Timeline都无法创建参数。

```csharp
CSHARP

public static ScriptPlayable<TimelinePlayable> Create(PlayableGraph graph, IEnumerable<TrackAsset> tracks, GameObject go, bool autoRebalance, bool createOutputs)
{
    if (tracks == null)
        throw new ArgumentNullException("Tracks list is null", "tracks");
    if (go == null)
        throw new ArgumentNullException("GameObject parameter is null", "go");
 
    var playable = ScriptPlayable<TimelinePlayable>.Create(graph);
    playable.SetTraversalMode(PlayableTraversalMode.Passthrough);
    var sequence = playable.GetBehaviour();
    sequence.Compile(graph, playable, tracks, go, autoRebalance, createOutputs);
    return playable;
}
```

如果没有看到这些异常，那Playable应该是有效的。

其他的原因

- Timeline 代码已经从构筑中剥离。（这通常是IL2CPP固定去除Timeline，或者在某个场景内只使用一部分Timeline）
- PlayableDirector被加载，但是TimelineAsset没有。

### 参考

- [forums_PlayableAsset returned a null Playable on Instantiate](https://forum.unity.com/threads/playableasset-returned-a-null-playable-on-instantiate.1105789/#:~:text=PlayableAsset returned a null Playable on Instantiate is,means the whole Timeline hasn't been created properly.)