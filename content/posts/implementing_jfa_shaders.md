---
title: "Jump Flooding Algorithmを実装する (Implementing Jump Flooding Algorithm)"
date: 2021-12-02T13:44:02+09:00
description: "Jump flooding algorithmというSDF生成アルゴリズムについて"
draft: true
---

高速にSDFを生成するアルゴリズムであるJump Flooding Algorithmを実装したメモ。
<!--more-->

## 前置き
使ったコードは全てGLSL及びCで、githubに設置してある。また、コードを参照せずともこの記事を読むだけで自力で実装できるようには書いたつもり。

コードをそのまま実行するにはGLFWとGLEW、stb_imageが必要。

## モチベーション
2Dゲーム等であまり見ない「スプライト同士が溶け合うような重なり合い」を実現しようと思ったのが始まり。
「Sprite merge shader」とかで検索しても特に自分が思ったようなエフェクトを作っている人がおらず、自力で作る為のアプローチを調べ回った末、以下の理由からJump Floodアルゴリズムを実装してみる事にした。

 - 自分でも実装できそうなほど簡単そうだったから。
 - 符号付き距離フィールド（SDF）を使えば最も自分の印象に近い見た目が実現できそうだったから。
 - リアルタイムにスクリーン全体のSDFを生成したかったから。
 - 本格的なシェーダーに興味があり、いい勉強にもなるだろうと思ったから。

完成と動作までは行ったものの、2Dのスプライトを2つ置いた800x600のスクリーンですら90FPSまで落ちる[^スペック]など実用には厳しい事がわかった為結局は他のアプローチを探ることにしたが、せっかく試行錯誤して作った経験が水の泡になるのももったいないなと思ったので記事にすることにした。

[^スペック]:（Arch Linux, Intel Core i3-7100U, Intel Corporation HD Graphics 620）

## Jump Flood Algorithm（以下: JFA）とは
フラッディングアルゴリズム（与えられたグラフ領域の全体に有用なデータを書き込むアルゴリズム？）の一つ。2006年に公開されたものらしい。ボロノイ図を高速で生成するために考案されたもので、GPUが並列演算に長けているのを有効活用し、各ピクセルがどれだけ原点から離れているかを計算する。

活用方法次第ではメッシュのアウトラインを生成したりもできる他、ソフトシャドウの生成などにも使えるとのこと（詳しく調べてはいない）。[^Wikipedia]

プロセスは以下の通り:
 1. 計算の起点として扱うスプライトだけを描画したバッファーを用意し、スプライトが描画された部分の色をそのピクセルのUV位置に置き換える。このUV位置を**シード**と呼ぶ。
 2. 各ピクセルに対して指定の**ジャンプ幅**だけ離れた8方向の近隣ピクセルを参照し、以下の条件に従って比較・現在のピクセルの色を更新する:
    1. 両方にシードなし -> 無視。
    2. 現在のピクセルにシードあり、参照ピクセルにシードなし -> 無視。
    3. 現在のピクセルにシードなし、参照ピクセルにシードあり -> 現在のピクセルに新しいシード値を格納
    4. 両方にシードあり -> 両方のシード値と現在のピクセルの距離を計算。距離が近かった方を新しいシード値として格納

 3. これを複数回繰り返す。`バッファーのサイズ / 2`を最初のジャンプ幅とし、各ステップ毎にこのジャンプ幅を半減する。幅が１になったら終了。

## 実装
まず基本的なシェーダーを試す上で最小限のOpenGLのセットアップを行う。使用するのはglfwとglew。

### ステップ1
SDFを生成する上で専用のバッファー（フレームバッファ）を3つ用意する。今回は以下の3つにした。
 1. メインバッファ - 最終結果を出力するバッファ。
 2. ピンポンバッファ１ - SDFを計算する・UVを生成するエフェクト用のバッファ。
 3. ピンポンバッファ２ - SDFを計算する・UVを生成するエフェクト用のバッファ。

アルゴリズムの処理は全てピンポン１−２を行ったり来たりして行われる。このプロセスはピンポンバッファの解像度によって処理速度が大きく変わるので、今回は通常の1/4（幅1/2, 高さ1/2）の大きさのバッファにする。

```cpp
// =========================
// 構造体。
// スプライト（テクスチャ）。
struct Sprite {
    unsigned int tex_id;
    int w; // 幅
    int h; // 高さ
};

// フレームバッファ。
// バッファのサイズはSpriteの中に格納する。
struct Frame_Buffer { 
    unsigned int fbo_id;

    Sprite color; // 出力先のカラーテクスチャ
    unsigned int depth_id; // デプスバッファ（RBO）のID

    void activate();   // バッファをバインドしてViewportを合わせるだけ
    void deactivate(); // バッファのバインドを解除するだけ
};

// =========================
// グローバル変数。
static Frame_Buffer main_buffer; // 出力先
static Frame_Buffer pingpong_buffer0; // ピンポンバッファその１
static Frame_Buffer pingpong_buffer1; // ピンポンバッファその２

// =========================
// 関数。
// 画像をロードしスプライトに格納する。
Sprite load_texture_as_sprite(const char *file_name);
// OpenGL用のスプライトを生成する。
Sprite make_sprite(int width, int height);
// 新しいフレームバッファーの生成。
Frame_Buffer make_frame_buffer(int width, int height);

```

次に、それぞれの処理を行うフラグメントシェーダーを3つ用意する（本当はもっと統一化出来るのかもしれない）
 1. UV生成シェーダー    - 画素にピクセルが描画されていれば、現在のX-Y座標（テクスチャ座標でなくFragCoordの方）に置き換える。
 2. JumpFloodシェーダー - 上記のUVが生成されたものを連続で参照しSDFを生成する。
 3. 通常描画シェーダー  - 普通に描画するだけのシェーダー。

頂点シェーダーは単に投影を適応してから必要なデータをフラグメントシェーダーに送るだけなので、今回は割愛。
```cpp
// =========================
// グローバル変数。
static unsigned int normal_render_shader;
static unsigned int uv_generation_shader;
static unsigned int jump_flood_shader;

// Uniform変数名 / Attributeの位置。
const char *resolution_uniform_name = "v2_resolution";
const char *mvp_uniform_name = "m4_mvp";
const char *step_uniform_name = "i_steptime";

const unsigned int position_attribute = 0;
const unsigned int texcoord_attribute = 1;

// =========================
// 関数。
// シェーダーのコンパイル・リンクを行い生成物を返す。
unsigned int make_shader(const char *v_name, const char *f_name);
```

上記の３つのバッファを以下の順番で有効化し、求める結果を得る。
 1. ピンポンバッファ１ - UV生成シェーダーを使ってスプライトをレンダリングし、JumpFloodに与えるデータを用意する。
 2. ピンポンバッファ２ - JumpFloodシェーダーを使ってピンポンバッファ１をレンダリングする。
 3. ピンポンバッファ１ - JumpFloodシェーダーを使ってピンポンバッファ２をレンダリングする。ループが終わっていなければ２に戻る。
 4. メインバッファ     - 通常描画シェーダーにJumpfloodが生成したテクスチャを与えてから、それをエフェクトに使いつつ再度スプライトをレンダリングする。

[^Wikipedia]: https://en.wikipedia.org/wiki/Jump_flooding_algorithm

