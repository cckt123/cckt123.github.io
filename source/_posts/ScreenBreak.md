---
title: ScreenBreak
date: 2022-05-12 17:31:01
tags: [ Unity]
categories: [ Unity]
about:
description: "屏幕破碎效果，翻译文献。"
---

## 前言

+ [这里是原文章地址](https://qiita.com/KTN44295080/items/e9f8106943ebee15aa2a#%E3%81%AF%E3%81%98%E3%82%81%E3%81%AB)
+ [这里是原文章GitHub地址，这里下载模型不会被墙](https://github.com/KTN44295080/ScreenBreak)
+ 原作者有写模型的制作方法，但是这里并没有翻译
+ PS：翻译完之后发现好像看图就可以领悟大概了...

## 开始

文章意图制作类似于DMC4战斗结束，FFX战斗场景切换中使用的破碎屏幕特效。

![DMC4](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F6276585c-d168-9852-ac4d-b8c7958fc0c7.gif?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=fb2055d4be8794607152c7374b7806aa)

![FFX](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F51955f83-1180-99ed-5eaa-527ae511e772.gif?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=aa17418c94e7c859b4ff01d63577bcbc)

然后这里是原作者的制作图。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F627e3270-03bb-95ea-9da9-abbe5be57deb.gif?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=0cdaa28cc2bbcb79f4484dce892897fe)

## 程序制作内容

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F9138092f-925a-6018-d89b-6cbe91b7f04b.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=ea5e668fdce091eaa477203ee44057f0)

将这个模型导入UnityAsset，并置入Scence中。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F681f4de7-7a75-69a5-56cd-dce4f1403d3e.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=5861e100c1c716d3dac6194197e656cc)

在Project中右键菜单->Create->RenderTexture

调整你的渲染纹理大小与屏幕大小一致。

将相机画面投影到这个纹理上，然后将这个纹理贴到模型上，就可以表现出破碎的画面效果。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2Fb0f7981e-038b-c4e5-251c-fdcd12158fd4.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=278c4fa52debd2702ed8fd09ede562ac)

然后创建一个材质球，将Shader从Unlit调整为Texture。

这样可以不受光线影响，保持纹理颜色不变。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2Fa223d7b7-3fd2-4d7e-74e3-fca7cce20857.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=22d7688fa7fe5078c2322d0ee56e24c6)

将现在制作的素材应用于模型当中，选择刚刚的材质Material，将其置于材质球的NoName当中。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F4d180d4b-3200-58cb-a061-af3cc228063c.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=f5338262c4d7558a1f22307baecab605)

调整相机的角度与位置，如果我们将Z轴设置为-2，发现刚刚好。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F25ca7c6f-9854-1130-4c55-73edb494cfa8.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=39c348c835a76a2d27a1b2541d544133)

如上图，创建一个脚本，命名为ScreenBreak。

``` csharp
using UnityEngine;
using System.Collections;

public class ScreenBreak : MonoBehaviour
{
    [SerializeField] private bool useGravity = true;                            // 重力を有効にするかどうか
    [SerializeField] private Vector3 explodeVel = new Vector3(0, 0, 0.1f);      // 爆発の中心地
    [SerializeField] private float explodeForce = 200f;                         // 爆発の威力
    [SerializeField] private float explodeRange = 10f;                          // 爆発の範囲
    private Rigidbody[] rigidBodies;

    void Start()
    {
        rigidBodies = GetComponentsInChildren<Rigidbody>();                     // 子(破片)のRigidbodyを取得しておく
        StartCoroutine("BreakStart");                                           // 動作にディレイを掛けるためコルーチンを使用
    }

    IEnumerator BreakStart()
    {
        foreach (Rigidbody rb in rigidBodies)
        {
            rb.isKinematic = false;
            rb.useGravity = useGravity;
            rb.AddExplosionForce(explodeForce / 5, transform.position + explodeVel, explodeRange);
        }
        yield return new WaitForSeconds(0.02f);                                 // 一瞬動かすことでひび割れを演出

        foreach (Rigidbody rb in rigidBodies)
        {
            rb.isKinematic = true;
        }
        yield return new WaitForSeconds(0.8f);

        foreach (Rigidbody rb in rigidBodies)
        {
            rb.isKinematic = false;
            rb.AddExplosionForce(explodeForce, transform.position + explodeVel, explodeRange);
        }
    }
}
```

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F23b786e4-3d02-1a58-5c15-d37c6e08d8a6.gif?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=0416addd349b3e936b1623be09b5fa43)

将ScreenBreak脚本附加到场景中的模型上，我们确实看到了破碎效果。

但是现在爆炸开的只是一个黑色的方块。

所以我们需要把摄像机的内容投影到方块上。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2Fdbc977d9-181e-9e5e-a2ff-09803a7e9de1.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=c94565b35fa1e9b2c9e055c009b74658)

这次要制作新的场景，当条件满足时转换成新场景。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F87dd926f-b3d0-42d4-586b-b170b8299751.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=f907494e834de27736a27ba32f765d2f)

在新场景中适当的调整对象。

![](https://qiita-user-contents.imgix.net/https%3A%2F%2Fqiita-image-store.s3.ap-northeast-1.amazonaws.com%2F0%2F918507%2F1de25984-17e5-8693-96d2-5feac283c9ba.jpeg?ixlib=rb-4.0.0&auto=format&gif-q=60&q=75&w=1400&fit=max&s=d93d205bfc382783615760158da8ef15)

当相机的图像渲染到纹理上的时候，相机便不再输出图像，因此我们将不必要使用的相机在常态下关闭，减少渲染压力。

然后再创建一个GameManager脚本，使用如下代码

``` csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.SceneManagement;

public class GameManager : MonoBehaviour
{
    [SerializeField] private RenderTexture renderTexture;
    private Camera renderCamera;

    // Start is called before the first frame update
    void Start()
    {
        renderCamera = Camera.main.transform.GetChild(0).GetComponent<Camera>();    //映像投影用のカメラを取得
    }

    // Update is called once per frame
    void Update()
    {
        if (Input.GetKeyDown(KeyCode.F))
            StartCoroutine("TraceOn");                                              // 画面割れを始めたいタイミングに置く
    }

    IEnumerator TraceOn()
    {
        renderCamera.enabled = true;
        renderCamera.targetTexture = renderTexture;                                 // 投影、開始
        yield return null;                                                          // 1f待ってもらって映像をレンダーテクスチャに投影する
        renderCamera.targetTexture = null;
        renderCamera.enable = false;                                                // 全行程、完了
        SceneManager.LoadSceneAsync("ScreenBreak");                                 // シーンを遷移する
    }
}
```

