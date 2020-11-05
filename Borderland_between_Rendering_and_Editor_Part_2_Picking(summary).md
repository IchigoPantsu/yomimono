

今日はゲームエンジンのオブジェクト選択の実装についての話をします。  
皆さんはゲームエンジン上でマウスを使ったゲームオブジェクト選択機能を実装したことはあるでしょうか？  
マウスを使ったゲームオブジェクト選択は今回の研修で作成した方も多いと思います。  


しかし，技研ブログのアドベントカレンダー2019で青山さんの記事でも述べられているのですが，
このマウスを使ったゲームオブジェクト選択機能について現代的なゲームエンジンでは必須ともいえるにもかかわらず,  
その実装についてを解説している文献はあまりありませんでした．

先日，OurMachineryというゲームエンジンのブログを読んでいたところ，マウスを使ったゲームオブジェクト選択機能の実装についての記事を見つけたので共有したいと思います.




# Borderland between Rendering and Editor — Part 2: Picking

3月初旬に、エディタでのグリッドのレンダリングについての記事を書きました。今日のトピックに飛び込む前に、前回の投稿から追加したグリッドレンダリングの些細な追加を紹介したいと思います。

これは非常に良い結果になったと思いますし、The Machineryのモジュール性の側面が実際にどれだけうまく機能しているかの良い例だと感じました。

とにかく、先に進みましょう。今日の記事では、マウスピッキングについて説明します。

# ピッキング - 背景

3D-editorsのすべてのタイプは、ユーザーがビューポート内でクリックしたときにマウスポインタの下にあるオブジェクトのオブジェクトIDを決定するための何らかの方法が必要になる傾向があります。ここでは、このプロセスを「ピッキング」と呼びます。

ピッキングは、グリッドレンダリングと同じように、グラフィックプログラマーにとってはつまらない、あるいは重要ではないと思われがちな問題の一つです。代わりに、それは、何らかの方法で解決策を考え出すために、いくつかのツールプログラマの膝の上で終わる傾向があります。通常は、ビューポートカメラの位置から、カメラのニアプレーンに投影されたマウスポインタ座標を介して、何らかの形でレイキャストを行います。

私が見てきたピッキングを実装するための2つの最も一般的なアプローチは以下の通りです。

物理学のように、レイキャストを行うための既存のシステムを再利用する。
+ エディタのピッキングにのみ使用される特別なレイキャスティング構造を実装する。
+ レンダリングとツールの人々の間に明確なサイロがある状況全体の状況、レンダリングプログラマの間でのプリマドンナな態度、そして問題に対する2つの最も一般的な解決策は、私は泣きたくなるほど多くの点で悪いです。

例えば物理システムに頼っていると、物理表現を持たないものを選択する必要があるとすぐに壊れてしまいます。一般的なユーザーの回避策は、エディタで選択できるようにするためだけに物理表現を追加することです。理想的ではありませんが...。

一方で、エディタピッキングを行うためだけに使用される特定のレイキャスティングシステムを実装することは、かなりの作業量になる可能性があります。確かに、実装は世界最速である必要はありませんが、一般的にはフレームごとに 1 つのレイをキャストするだけなので（またはそれ以下）、それでも、実際のレンダリングされた三角形に対してピックしたいのは間違いなく、いくつかのプロキシではなく、何らかのアクセラレーション構造（BVH ツリーのような）が必要になります。この構造は、メモリを消費し、調理に時間がかかります。また、多くのオブジェクト（10K以上）を持つやや複雑なシーンを扱うようになったら、すぐに別のレベルのアクセラレーション構造を追加する必要があるかもしれません。

言うまでもなく、これらのアプローチは、デフォルメ可能なオブジェクト（スキンキャラクターなど）に対するピッキングや、アルファマスクされたオブジェクト（木の葉の間など）のピッキング、純粋にボクセルベースのオブジェクトの処理、あるいは「Nanite」ソリューションのような種類の処理を行うことはできないでしょう。

明らかに、これはあなたが望んでいるものではありません。CPU上でまともなソリューションを実装することが可能だとしても、あなたが本当に求めているのはGPUベースのピッキングソリューションです。マウスカーソルの下にあるピクセルの正体を知りたいのであって、目に見えないプロキシの正体を知りたいのではありません。あらゆる種類のジオメトリの変形やサーフェスタイプを処理するピクセルパーフェクトソリューションを使用した後は、精度の低いものを使用しても大丈夫ということはありません。

皮肉なことに、GPUベースのピッキングソリューションの基礎を実装することは、CPUベースのピッキングソリューションよりもはるかに簡単です。ここでは、The Machineryの実装を見てみましょう。


# GPUのピッキング - 実装

まず第一に、私たちのソリューションはいくつかの前提条件に依存しています。

+ まず第一に、ビューポートに描画されたすべてのオブジェクトに対して、異なるシェーダバリエーションを効率的に選択できるシェーダシステムに依存しています。これは、この記事で説明した「システム」の概念を使って処理します。

+ 次に、ピクセルシェーダへの無順序アクセスビュー（UAV）のバインドをサポートするハードウェア上で常に動作していることに依存しています。

そのため、アイデアは非常にシンプルです。ユーザーがビューポート内をクリックすると、`picking_system` と呼ばれる特殊なシェーダシステムを起動します。つまり、ピッキングシステムのサポートを実装しているシェーダは、画面上のオブジェクトをレンダリングするために使用されるピクセル（または計算）シェーダの中で、わずかに異なるコードパスをアクティブにします。

有効化されたシェーダのバリエーションは、`picking_system` で定義されている `update_picking_buffer()` の呼び出しを追加します。実際のピッキングバッファは、UAVとしてバインドされた12バイトのバッファのようなバカげたもので、単純な構造体をラップしています。

```c
struct tm_gpu_picking_buffer_t
{
    float depth;
    uint64_t identity;
};
```


`update_picking_buffer()`の素朴な実装は以下のようになります。

```c
// `pos` should be SV_Position.xy (or comparable writing from a CS).
//
// `identity` is a unique identifier of the rendered object. In TM this is
// typically the ID of the entity issuing the GPU work.
// 
// `z` is the Z depth value of the shaded point.
//
// `opacity` is the opacity value of the currenly shaded pixel. 
void update_picking_buffers(uint2 pos, uint2 identity, float z, float opacity) 
{
    // If the opacity of the pixel is less than the "opacity picking threshold"
    // -- return.
    if (opacity < load_opacity_threshold())
        return;
    
    // If this pixel is not under the mouse cursor -- return.
    uint2 cursor_pos = load_cursor_pos();
    if (pos.x != cursor_pos.x || pos.y != cursor_pos.y)
        return;

    // Compare pixel z against the currently stored z-value in the
    // picking_buffer. If it's greater or equal (behind) -- return. 
    uint d = asuint(z);
    if (d >= picking_buffer.Load(0))
        return;
    
    // Update picking buffer. 
    picking_buffer.Store3(0, uint3(d, identity));
}
```

この素朴な実装はほとんどの場合正しいピッキング結果を生成しますが、時折失敗して最も近いサーフェスの後ろにあるピッキング結果を返すことがあります。これは、ピッキングバッファへの同時アクセスから保護する必要があるためです。
アトミックをdの符号ビットを利用してスピンロック機構を実装することで、ピッキングバッファへの同時更新を防ぐことができます。実装は少し毛むくじゃらですが、以下のようになります。

```c
uint d = asuint(z);
uint current_d_or_locked = 0;
do {
    // `z` is behind the stored z value, return immediately.
    if (d >= picking_buffer.Load(0))
            return;
    
    // Perform an atomic min. `current_d_or_locked` holds the currently stored
    // value.
    picking_buffer.InterlockedMin(0, d, current_d_or_locked);
    // We rely on using the sign bit to indicate if the picking buffer is
    // currently locked. This means that this branch will only be entered if the
    // buffer is unlocked AND `d` is the less than the currently stored `d`. 
    if (d < (int)current_d_or_locked) {
            uint last_d = 0;
            // Attempt to acquire write lock by setting the sign bit.
            picking_buffer.InterlockedCompareExchange(0, d, asuint(-(int)d), 
                last_d);
            // This branch will only be taken if taking the write lock succeded.
            if (last_d == d) {
                    // Update the object identity.
                    picking_buffer.Store2(4, identity);
                    uint dummy;
                    // Release write lock. 
                    picking_buffer.InterlockedExchange(0, d, dummy);
            }
    }
// Spin until write lock has been released.
} while((int)current_d_or_locked < 0);
```

スピンロック・アップデートを有効にすると、常にピッキングバッファの中で最も近いサーフェイスを取得しなければなりません。
次にCPU側に移動して、スケジューリングとピッキングバッファの結果をGPUからCPUに読み出すためのC-APIを見てみましょう。これは `tm_gpu_picking_api` と呼ばれ、以下のようになります。

```c
struct tm_gpu_picking_api
{
    struct tm_gpu_picking_o *(*create)(struct tm_allocator_i *allocator, 
        struct tm_renderer_resource_command_buffer_o *res_buf, 
        struct tm_shader_system_o *picking_system, 
        struct tm_shader_o *clear_picking_buffer_shader);

    void (*destroy)(struct tm_gpu_picking_o *inst, 
        struct tm_renderer_resource_command_buffer_o *res_buf);

    bool (*update_cpu)(struct tm_gpu_picking_o *inst, 
        struct tm_renderer_backend_i *rb, bool activate_system, 
        const tm_vec2_t cursor_position, float opacity_threshold,
        uint64_t *result);

    void (*update_gpu)(struct tm_gpu_picking_o *inst, 
        struct tm_shader_system_context_o *context,
        struct tm_renderer_resource_command_buffer_o *res_buf, 
        struct tm_renderer_command_buffer_o *cmd_buf,
        uint32_t device_affinity_mask);
};
```

`create()`と`destroy()`はピッキングオブジェクトの作成と破棄を単純に担当します。ピッキングをサポートする各ビューポートは、それ自身の`tm_gpu_picking_o`を持っています。内部的には`tm_gpu_picking_o`はピッキングバッファといくつかの状態を所有しています。

`update_cpu()`は通常、UI/エディタコードのどこかでフレームごとに呼び出され、2つのことを行います。

+ ピッキングバッファの新しいリードバック結果が利用可能かどうかをチェックします。もしあれば、picking_bufferのidentity部分が結果として返され、関数はtrueを返します。マシナリーでのオブジェクト選択のトラッキングはエンティティIDに基づいていますので、エンティティIDを定数バッファの一部としてすべての描画とディスパッチ呼び出しに渡すだけで、結果の戻り値を追加のルックアップなしで直接使用することができます。

+ ユーザーがビューポート内のどこかでマウスボタンをクリックした場合、カーソル位置と任意の不透明度のしきい値と共にtrueを`activate_system`に渡す必要があります。これは、定数バッファを介して`picking_system`に渡される状態を準備し、アクティブ化のためにキューに入れます。不透明度のしきい値は、ピクセルがクリックされる（つまり、拒否される）表面の不透明度を制御します。エディタで不透明度のしきい値のコントロールを公開する計画がありますが、まだ実現していません。0.5 未満の不透明度のピクセルはクリックされてしまいます。

`update_gpu()` は、ビューポートをレンダリングしようとするときに呼び出されます。これは以下のことを行います。

最後に `update_cpu()` を呼び出したときに picking_system が起動していなかった場合は、すぐに戻ります。

ピッキングバッファをクリアするための計算ディスパッチコールをキューに入れます。

`update_cpu()` に渡されたマウスカーソルの位置と不透明度のしきい値で、 picking_system に関連付けられた定数バッファを更新します。

GPUからCPUへの`picking_buffer`の非同期リードバック操作をキューに入れ、ビューポートでの全てのレンダリングが終了した後に実行するようにスケジュールされています。

ピッキングバッファのリードバックでGPUをストールさせたくないので、結果が利用可能になり、`update_cpu()`からtrueが返されるまでに数フレームかかることがあります。しかし、実際には、この遅延はユーザの視点からは目立ったものではありません。

以下はピッキングシステムの動作を示すgifアニメーションです。

