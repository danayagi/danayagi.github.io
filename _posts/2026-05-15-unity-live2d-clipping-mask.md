---
layout: post
title:  "【Unity×Live2D】複数モデル配置時にクリッピング設定したパーツが消える"
date:   2026-05-15
categories: Live2D RPA
key: 2026-05-15-unity-live2d-clipping-mask
---

### 1. 概要・発生した問題

UnityでLive2D（Cubism SDK for Unity）を使用している際、1つのシーン内に複数のLive2Dモデルを配置すると、**クリッピング設定をしているパーツだけが表示されなくなる現象**に遭遇した。<br>
特定のモデル単体では問題なく表示されるものの、複数のモデルを同時にシーンに存在させた場合にのみ発生する。

- **【動作環境】**
  - Unity 2022.3.62f3
  - Live2D SDKのバージョン: 5-r.4 (2025/5/9)

### 2. 原因：GlobalMaskTextureの「マスク上限数」

原因は、Cubism SDK for Unityの仕様による **「クリッピングマスクの使用上限数」の超過** だった。
<br>
<br>
<br>

- 2-1. Unity版のLive2D SDKでは、デフォルト設定の場合、シーン内の **すべてのモデルのクリッピングマスクを1枚のテクスチャ（GlobalMaskTexture）で共有して処理** している。

{: .caption-text }
▼シーン内に配置したLive2Dモデルを選択した際のインスペクタ。「GlobalMaskTexture」を選択すると、アセットとして存在するGlobalMaskTextureを表示できる。<br>
![fig1-1.png](/assets/img/20260515/fig1-1.png)

{: .caption-text }
▼GlobalMaskTexture（アセット）は「Live2D\Cubism\Rendering\Resources\Live2D\Cubism」内にある。<br>
![fig1-2.png](/assets/img/20260515/fig1-2.png)

{: .caption-text }
▼GlobalMaskTextureのインスペクタ。後述するが「RenderTextureCount」の数値を調整するとマスクテクスチャの枚数が増える（※Cubism SDK for Unity R7以降）。<br>
![fig1-3.png](/assets/img/20260515/fig1-3.png)

<br>
<br>
- 2-2. この共用テクスチャ1枚につき、同時に扱えるマスクの数（≒クリッピング設定したパーツの数）は、デフォルトで「64個」に制限されている。

{: .caption-text }
▼Live2D公式SDKマニュアルより引用（ [SDK機能比較](https://docs.live2d.com/cubism-sdk-manual/cubismsdk-compare/) ）
<br>
![fig1-4.png](/assets/img/20260515/fig1-4.png)

<br>
<br>
- 2-3. つまり、1つのシーン内に存在するLive2Dモデルのクリッピングパーツの合計が64個を超えると、超過した分のパーツに割り当てる領域がなくなり、結果として「クリッピング」設定のあるパーツが表示されなくなる（落ち影が消えるなど）。

> マスクの上限に達した場合は実行時の登録順の遅いほうから、全面にマスクが掛かり表示が消えます。

- 出典: [SDK機能比較 - Live2D Manuals & Tutorials](https://docs.live2d.com/cubism-sdk-manual/cubismsdk-compare/)

<br>
<br>
<br>

#### 注意点：非表示（SetActive(false)）にしても枠は解放されない

注意点として、**モデルを非表示（`SetActive(false)`）にするだけでは、マスクの枠は解放されない**。
デフォルトでは `CubismMaskController` の `Start()` で登録され、`OnDestroy()` で解放される仕様になっている。下記コード内の `AddSource` メソッドと `RemoveSource` メソッドがそれぞれ対応している。

（`CubismMaskController.cs` の該当箇所。`Live2D\Cubism\Rendering\Masking` 内に存在する）

```csharp
/// <summary>
/// Initializes instance.
/// </summary>
private void Start()
{
    // Fail silently.
    if (MaskTexture == null)
    {
        return;
    }

    MaskTexture.AddSource(this);

    // Get cubism update controller.
    HasUpdateController = (GetComponent<CubismUpdateController>() != null);
}

~~~

/// <summary>
/// Finalizes instance.
/// </summary>
private void OnDestroy()
{
    if (MaskTexture == null)
    {
        return;
    }

    MaskTexture.RemoveSource(this);
}
```

そのため、「現在は画面に映っていない待機中のモデル」であっても、シーン内に存在（Start済み）しているだけでマスクの枠を消費し続けてしまう。

### 3. 解決策

#### 3-1. マスク用テクスチャの枚数を増やす（PC向け）

最も手っ取り早く、かつシームレスな遷移を妨げない方法として、**GlobalMaskTextureが内部で保持するテクスチャの枚数を増やす**ことで、マスクの上限を上げる方法がある。

- **手順**
  1. UnityのProjectウィンドウで、以下の場所にある `GlobalMaskTexture` アセットを選択する。
  2. Inspectorウィンドウから、**`RenderTextureCount`** の値をデフォルトの `0`（または `1`）から、**`2` や `3` など任意の数に増やす**。

![fig1-3.png](/assets/img/20260515/fig1-3.png)

これだけで扱えるマスクの上限数は増加する（ベースが64個なので、2枚にすれば128個、3枚なら192個まで対応可能）。

<br>
<br>

※ GlobalMaskTextureのデフォルトサイズは `1024x1024` （ピクセル）。<br>
Live2DのマスクはRGBAの4チャンネルを使用するため、**1枚あたりのVRAM消費量は約4MB**（1024 × 1024 × 4バイト）となる。PC向けのゲームであればVRAM消費は微小なため、基本的に問題にはならない。<br>

<br>
<br>

#### 3-2. 明示的にRemoveSourceメソッドを呼び出す

どのタイミングでどのLive2Dモデルを表示・非表示にするか明確に管理できている場合、明示的に `RemoveSource` メソッドを呼び出してマスク枠を解放することも可能。<br>

`CubismMaskController` の `MaskTexture` は外部からアクセス可能なうえ、シーン内に配置されているLive2Dオブジェクトは `CubismMaskController` をコンポーネントとして保持している。<br>
そのため、「対象のLive2Dオブジェクトの」`CubismMaskController` への参照があれば、

```csharp
maskController.MaskTexture.RemoveSource(maskController);
```

とするだけで、そのオブジェクトの分だけ明示的にマスクを解放できる。

### 4. その他の解決策（公式マニュアルより）

公式マニュアルには以下の手法も紹介されている。

1. 共用のマスクテクスチャを使うのではなく、モデルごとにマスク用テクスチャを設定する手法
2. OnEnable / OnDisable のタイミングでマスク枠を解放するようにスクリプト（`CubismMaskController.cs`）を書き換える手法

- 参考: [SDK for Unityにおけるマスクの使用上限について - Live2D Manuals & Tutorials](https://docs.live2d.com/cubism-sdk-manual/maximum-number-of-masks-used/)

### 5. まとめ

- 1つのシーンに大量のLive2Dモデルを配置した際、**クリッピングパーツだけ**が表示されなくなったら、クリッピングマスクの上限（デフォルト64個）を疑う。
- もっとも簡単な解決策は、`GlobalMaskTexture` の `RenderTextureCount` を増やすこと。
  - PC向けならVRAM消費（1枚約4MB）は気にならないレベルであるため、テクスチャを増やして解決するのが安全かつシームレス。
- 非表示（`SetActive(false)`）にしただけではマスク枠は解放されない点に注意が必要。