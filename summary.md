# Kaggle (Mechanisms of Action (MoA) Prediction)
コンペURL：https://www.kaggle.com/c/lish-moa/overview

# コンペの概要について
薬の作用機序(Mechanisms of Action)についてのコンペでした。
薬を使った実験1つに対する、800程度の遺伝子発現と100程度の細胞生存の反応のデータに基づいて、ターゲットとなるタンパク質を予測します。

# 問題設定とモデルの選択について
ターゲットとなるタンパク質は、薬に対して1つというわけではなく、複数のターゲットが反応する可能性があります。
（つまり multi-label classification であり、multi-class classification ではない）
kaggle.com/c/lish-moa/discussion/180500

通常の分類ならLightGBM などで上手くいくことが多いようですが、multi-labeling の問題は上手く行かないことが多いようです。
（今回は206種類のターゲットがあったので、206列分の予測値を出力する必要がありました）

そのためテーブルデータにも関わらず、比較的 multi-labeling が得意なニューラルネットを用いた解法が多いコンペとなりました。


# 提出方法について
house prices のような結果だけを提出するコンペとは違い、ノートブックを提出してスコアリングの直前に実行させる形でした。
GPUを使えるのが2時間以内だったので、提出時に学習を行う場合はそこがネックとなりました。
しかし、上位の解法を見ると、学習済みのモデルを用いている人もいたようです。場合によりけりだと思います。
https://www.kaggle.com/c/lish-moa/discussion/200614

※ RankGauss や PCA を行う場合は、実務においては未来のデータとなる test データを含めることはできないはずです。しかし、kaggleでは private LB にオーバフィッティングさせれば順位が上がるので、testデータを含めて前処理を行うことがあります。この場合、提出時に学習を行う必要がありました。

# 特徴量について
遺伝子発現や細胞生存のデータについては、すでに何らかのスケーリングが施されていたようです。遺伝子の種類もわかりませんし、値が何を指しているのかもわかりません。

特別な特徴量というものはあまりなく、どの解法も大体似たような感じになっているように見受けられました。

特徴量は以下のものをよく見ました
- QuantileTransformer (or RankGauss)
    - train と test を合わせてデータの分布が正規分布になるように変換する。private test とともに変換を施す場合は、提出時にtrainしなくてはならない。
- PCA features for genes and cells
    - 主成分分析によって新しくできた特徴量を加える。通常は次元削減のために用いるが、今回は加えることでスコアが良くなる模様
    - PCAの他に SVD, ICA　なども
    - これも private test を含める場合は提出時に train しなくてはならない。
- VarianceThreshold
    - 分散が一定以下の特徴量を削除
- Simple statistics
    - mean, min, max, skew, kurt, (X_train>9).sum(axis=1)

PolynominalFeaturesとかもあったみたいです。

# モデルについて
モデルは以下のものがありました
- Multi layer パーセプトロン（中間層が2~4層）
- GrowNet
- TabNet
    - テーブルデータに対するニューラルネット
    - 特徴量の重要度なども分かる
- DAE (denoising autoencoder)
- EffNet (deepinsight images)
  - DeepInsight: 非画像データにCNNを適用するためのテクニック
- transfer learning
    - スコアに関係ないタンパク質のターゲットデータを与えられていたので、それを用いて事前に学習し、あとで fine tuning を行う。
    - 層が深いニューラルネットについて、層の初期値を事前学習によってうまく決めることができるのではないかとのこと
- ResNet-like shallow NN
- Multi Input ResNet
    - 入力層を分割した（ヘッドが別れている）ニューラルネット


# 上位入賞のポイント
どのようにすれば上位入賞できたかのワンポイントのまとめです。
- 様々なモデルをブレンドしている
- Data Augmentation：数が少ないターゲットについてのデータを水増しする
    - https://www.kaggle.com/c/lish-moa/discussion/200600
- Data Augmentation: ランダムに選んだcntrol2つの差を値として加えデータ量を増やす
    - https://www.kaggle.com/c/lish-moa/discussion/200540
    - This step should be completed before feature engineering. You can augment data each epoch.
- Layer Normalization
    - https://www.kaggle.com/c/lish-moa/discussion/201051
- From multilabel to multiclass
    - 206のターゲットの組み合わせは328通りしか無いのでmulti-classの問題に変換できる
    - 単体では精度はあまり出ないが、ブレンドした時に効果を発揮する
    - https://www.kaggle.com/c/lish-moa/discussion/200992

## その他ポイント
- 時間・薬の量(D1/D2)で6種類に分割。それぞれの control の平均値を引いてあげるとスコアが良くなる
- [Some tips to avoid overfitting](https://www.kaggle.com/c/lish-moa/discussion/196913)
    - 同じ検証データを頻繁に使用しないこと
    - ブレンドの重みを過度にPublic LBを基に決めないこと
    - モデルの更新は、CVが十分に改善されている場合にのみ行うこと
- metric learning
    - train に出現しない薬が test に出現するのでそれを予測する。類似度に応じてCVの切り方を変える
    - https://www.kaggle.com/c/lish-moa/discussion/200533

# shake や CV について
今回はpublic のLB のスコアが、手元のCVのスコアと大きくずれているのが特徴でした。
よってクロスバリデーションをどのように行うのかというのも議論になりました。

一番用いられたのはターゲットの分布が均等になるように分割する方法です。206列あったので通常のstratified k-fold は使えずに、MultilabelStratifiedKFold というものが使われました。

もう一つは MultilabelStratifiedGroupKFold で、drug id に基づいて分割します。こちらは手元のCVとLBの差が小さくなるので喜ばれていました。

public が25%, private が75% という割合で分けられていたのですが、
最終的には手元のCVを信じた人が強かったようです。
（public 2位だった人は 500位近くまでprivate でダウンしてました。3位の人は118位までダウンしたそうです。）
https://www.kaggle.com/c/lish-moa/discussion/200614

CVとpublic LB に差が生じた原因としては、
ターゲットとなるタンパク質が、train と public や private の全てのデータにおいて、必ず出現するように分割しているため、分布に偏りが生じていたせいのようです。
とりわけ、trainデータに数回しか出現しないようなターゲットが、public test にも同程度出現していたのが大きいとの分析がありました。
（public test の方が総数が少ないので、割合としては大きくなってしまいます）
https://www.kaggle.com/c/lish-moa/discussion/200832

また、drugs がtrain に出現していないものがtest に含まれているのも、信頼できる validation が難しかった理由のようです。


# その他資料
- [日本語の解説](https://imokuri123.com/blog/2020/12/kaggle-lish-moa/?utm_campaign=Weekly%20Kaggle%20News&utm_medium=email&utm_source=Revue%20newsletter#node-neural-oblivious-decision-ensembles)
- [TabNetの解説動画](https://www.youtube.com/watch?v=ysBaZO8YmX8&feature=youtu.be)
- [3rd Place Public - We Should Have Trusted CV - 118th Private](https://www.kaggle.com/c/lish-moa/discussion/200614)
- [Public 2nd/Private 560th (Private 0.01609 Silver Solution)](https://www.kaggle.com/c/lish-moa/discussion/200338)
- [1st place solution-Summary](https://www.kaggle.com/c/lish-moa/discussion/200736)
- [4th Place Solution](https://www.kaggle.com/c/lish-moa/discussion/200808)
- [5th place solution [Updated]](https://www.kaggle.com/c/lish-moa/discussion/200533)
- [8th place solution | From multilabel to multiclass](https://www.kaggle.com/c/lish-moa/discussion/200992)
- [Private 14th / Public 24th place solution!](https://www.kaggle.com/c/lish-moa/discussion/200585)
