# Kaggle (Mechanisms of Action (MoA) Prediction) ゴールド再現
コンペURL：https://www.kaggle.com/c/lish-moa/overview

なるべくシンプルな方法での再現を目指す。

# 方法
- 1D-CNN のシングルモデル
- Non-Scored target の一部を用いて pre-train
- データオーグメンテーション
    - 出現回数の少ないターゲットを増やす
    - 2つのctl_sample を用いる

# 参考URL
- ベースモデル
    - [2nd Place Solution - with 1D-CNN (Private LB: 0.01601)](https://www.kaggle.com/c/lish-moa/discussion/202256)
- その他工夫
    - [Silver to gold? All you need is augmentation](https://www.kaggle.com/c/lish-moa/discussion/200600)
    - [[Update]Augment is all you need](https://www.kaggle.com/c/lish-moa/discussion/200540)
