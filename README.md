# Kaggle (Mechanisms of Action (MoA) Prediction) ゴールド再現
コンペURL：https://www.kaggle.com/c/lish-moa/overview

なるべくシンプルな方法での再現を目指す。

# 方法
- 1D-CNN のシングルモデル
- Non-Scored target の一部を用いて pre-train
- データオーグメンテーション
    - 出現回数の少ないターゲットを増やす
    - 2つのctl_sample を用いる


# モデル
## base.ipynb

- private: 0.01612
- public: 0.01845

ベースとなる転移学習無しの1DCNN

## augmented_base.ipynb

- private: 0.01612
- public: 0.01845

base.ipynb に、数が少ないターゲットを加えたもの（未提出）


## transfer.ipynb (金メダル)

- private: 0.01603
- public: 0.01830

転移学習+seed値ごとにCVを切る

## transfer_cv.ipynb
drug_id を用いてCVを切ったもの


# 参考URL
- ベースモデル
    - [2nd Place Solution - with 1D-CNN (Private LB: 0.01601)](https://www.kaggle.com/c/lish-moa/discussion/202256)
- その他工夫
    - [Silver to gold? All you need is augmentation](https://www.kaggle.com/c/lish-moa/discussion/200600)
    - [[Update]Augment is all you need](https://www.kaggle.com/c/lish-moa/discussion/200540)
