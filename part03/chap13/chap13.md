## 13. 有限混合分布

<!-- 2.17.1を参照しました -->

結果変数が有限混合分布となるモデルは、複数の分布のうちのひとつから結果変数が抽出され、どの分布から抽出されるかは、混合させるカテゴリカルな分布により制御されると仮定します。混合モデルは通常、多峰の分布となり、その山は混合成分の最頻値に近くなります。混合分布モデルはいくつかの方法でパラメーター化できます。以下の節でそれを記述します。混合モデルは、直接的に多峰分布でデータをモデリングするのにも使えますし、他のパラメーターの事前分布としても使えます。

### 13.1. クラスタリングとの関係

クラスタリングモデル（17章で議論します）は混合モデルの特別な場合になります。混合モデルは、工学や機械学習の文献ではクラスタリングに広く利用されています。この章で議論する正規混合モデルは、$K$-meansアルゴリズムのための統計学的基礎として多変量の形で再び現れます。潜在ディリクレ配分モデルはクラスタリングの問題に普通に利用されていますが、混合メンバーシップ多変量正規混合モデルとしてみることもできます。

### 13.2. 潜在離散値のパラメーター化

混合分布モデルをパラメーター化する方法の1つは、結果変数を負担する混合成分を示す潜在カテゴリカル変数を使うことです。例えば、$K$個の正規分布があり、その位置は$\mu_{k} \in \mathcal{R}$、スケールは$\sigma_{k} \in (0, \infty)$とします。ここで、これらを割合$\lambda$で混合させます。$\lambda_{k} \ge 0$かつ$\sum_{k=1}^{K}\lambda_{k}=1$、すなわち$\lambda$は$K$次元単体です。各結果変数$y_{n}$には潜在変数$z_{n} \in {1,\dots,K}$があり、これは$\lambda$によりパラメーター化されるカテゴリカル分布に従うとします。

$$z_{n} \sim \mathsf{Categorical}(\lambda)$$

変数$y_{n}$は、混合成分$z_{n}$のパラメーターに従って分布します。

$$y_{n} \sim \mathsf{Normal}(\mu_{z[n]},\sigma_{z[n]})$$

離散パラメーター$z_{n}$があるので、Stanではこのモデルは直接扱うことができませんが、次の節で記述するように$z$パラメーターを総和で消去することにより$\mu$と$\sigma$をサンプリングできます。

### 13.3. 負担率パラメーターを総和で消去

前の節であらましを見た混合正規分布モデルは、総和により離散パラメーターをモデルから消去することによりStanで実装できます。$Y$が$K$個の正規分布の混合分布で、正規分布のが位置$\mu_{k}$、スケール$\sigma_{k}$、$K$次元単位単体に従う割合$\lambda$で混合される場合は、次式のように表されます。

$$p_{Y}(y \mid \lambda,\mu,\sigma) = \sum_{k=1}^{K}\lambda_{k}\mathsf{Normal}(\mu_{k},\sigma_{k})$$

### 13.4. 指数の和の対数: 対数軸での線形の和

指数の和の対数関数は、対数軸での混合分布を定義するのに使われます。2つの入力値の場合は以下のように定義されます。

$$\mathrm{log\_sum\_exp}(a,b) = \log(\exp(a)+\exp(b))$$

$a$と$b$が確率を対数軸にしたものなら、$\exp(a)+\exp(b)$は線形軸での両者の和になります。そして外側の対数で、結果を対数軸に戻します。まとめると、log\_sum\_expは対数軸で線形の加算を行なうということになります。Stanの組込み`log_sum_exp`関数を使う理由は、指数計算でのアンダーフローやオーバーフローを防ぐことができるということにあります。結果変数は以下のように計算されます。

$$\log(\exp(a)+\exp(b))=\max(a,b)+\log(\exp(a-c)+\exp(b-c))$$

ここで、$c = \mathrm{max}(a,b)$です。
この式の評価では、$a-c$か$b-c$のいずれかがゼロになり、他方は負になります。これにより、主項でオーバーフローやアンダーフローが発生する可能性をなくし、演算での算術精度を可能な限り確保します。

例として、$\mathsf{Normal}(-1, 2)$と$\mathsf{Normal}(3, 1)$とが確率$\theta=(0.3,0.7)^{\top}$で混ざる混合分布は、Stanでは以下のように実装できます。

```
parameters {
  real y;
}
model {
  target += log_sum_exp(log(0.3) + normal_lpdf(y | -1, 2),
                        log(0.7) + normal_lpdf(y | 3, 1));
}
```

対数確率の項は以下のように導出されます。

$$\begin{array}{ll}\log p_{Y}(y\mid\theta,\mu,\sigma) &= \log(0.3\times\mathsf{Normal}(y \mid -1,2)+0.7\times\mathsf{Normal}(y\mid 3,1))\\ &= \log(\exp(\log(0.3\times\mathsf{Normal}(y\mid -1,2)))\\\ &\qquad+\exp(\log(0.7\times\mathsf{Normal}(y\mid 3,1))))\\ &=\mathrm{log\_sum\_exp}(\log(0.3)+\log\mathsf{Normal}(y\mid -1,2),\\ &\hspace{68pt}\log(0.7)+\log\mathsf{Normal}(y\mid 3,1)) \end{array}$$

##### 一様な混合比の消去

2要素の混合モデルで混合比が0.5ならば、混合比は消去できます。なぜなら、

```
neg_log_half = -log(0.5);
for (n in 1:N)
  target
    += log_sum_exp(neg_log_half + normal_lpdf(y[n] | mu[1], sigma[1]),
                   neg_log_half + normal_lpdf(y[n] | mu[2], sigma[2]));
```

において、$-\log 0.5$の項は比の密度に寄与していませんから、上のコードはもっと効率的なバージョンに置き換えることができます。

```
for (n in 1:N)
  target += log_sum_exp(normal_lpdf(y[n] | mu[1], sigma[1]),
                        normal_lpdf(y[n] | mu[2], sigma[2]));
```

$K$要素で、混合比の単体$\lambda$が以下のように対称であるなら、同じ結果が得られます。

$$\lambda = \left(\frac{1}{K},\dots,\frac{1}{K}\right)$$

この結果は以下の恒等式によるものです。

$$\mathrm{log\_sum\_exp}(c+a,c+b) = c + \mathrm{log\_sum\_exp}(a,b)$$

そして、定数$c$を対数密度アキュムレーターに加えることは何の効果も持ちません。対数密度はそもそも定数を加えずに指定されるからです。これは正規分布に限ったことではありません。定数は常にtargetから消去できます。

#### 混合パラメーターの推定

混合分布を表現する枠組みが与えられたところで、推定の準備に移りましょう。ここでは、位置、スケール、混合成分が未知の量です。さらに、混合成分の数も一般化してデータとして指定したものが以下のモデルになります。

```
data {
  int<lower=1> K;            // 混合成分の数
  int<lower=1> N;            // データ点の数
  real y[N];                 // 観測値
}
parameters {
  simplex[K] theta;          // 混合比
  ordered[K] mu;             // 混合成分の位置
  vector<lower=0>[K] sigma;  // 混合成分のスケール
}
model {
  vector[K] log_theta = log(theta);  // 対数計算のキャッシュ
  sigma ~ lognormal(0, 2);
  mu ~ normal(0, 10);
  for (n in 1:N) {
    vector[K] lps = log_theta;
    for (k in 1:K)
      lps[k] += normal_lpdf(y[n] | mu[k], sigma[k]);
    target += log_sum_exp(lps);
  }
}
```

このモデルには、$K$個の混合成分と$N$個のデータ点があります。混合比のパラメーター`theta`は$K$次元単位単体として宣言されており、成分の位置のパラメーター`mu`とスケールのパラメーター`sigma`はともに`K`次元の`vector`として定義されています。

位置パラメーター`mu`は、モデルを識別する順で`ordered`として宣言されています。このモデルのように、成分`mu[k]`の事前分布が対称であるかぎり（このモデルでは各成分の事前分布は独立に$\mathsf{Normal}(0,10)です$）、成分の順序に依存しない推定にこの宣言は影響しません。成分に階層事前分布を含めることも可能でしょう。

スケールのベクトル`sigma`の値は非負に制約されています。また、ゼロ値になって成分がなくなることを防ぐために、このモデルでは弱情報事前分布を与えています。

モデル中で、長さ`K`の局所配列変数`lps`が宣言されており、混合成分からの寄与を積算するのに使われています。作業の中心は、データ点`n`についてのループ中にあります。各点について、$\theta_{k}\times\mathsf{Normal}(y_{n}\mid\mu_{k},\sigma_{k})$の対数が計算され、配列`lps`に加算されます。それから、それらの値の指数の和の対数だけ対数確率を増加させます。

### 13.5. 混合分布のベクトル化

Stanでは（今のところ）混合分布モデルを観測値のレベルでベクトル化する方法はありません。この節は読者に注意をうながすものです。単純にベクトル化しようとしてはいけません。そのようにすると、結果として別のモデルとなってしまいます。観測値のレベルでの正しい混合は以下のように定義されます。
観測値のレベルで正則な混合分布を以下のように定義します。すなわち、`lambda`、`y[n]`、`mu[1]`、`mu[2]`および`sigma[1]`、`sigma[2]`がすべてスカラーで、`lambda`は0から1の間の値をとるとします。

```
for (n in 1:N) {
  target += log_sum_exp(log(lambda)
                          + normal_lpdf(y[n] | mu[1], sigma[1]),
                        log1m(lambda)
                          + normal_lpdf(y[n] | mu[2], sigma[2]));
```

下も等価です。

```
for (n in 1:N)
  target += log_mix(lambda,
                    normal_lpdf(y[n] | mu[1], sigma[1]),
                    normal_lpdf(y[n] | mu[2], sigma[2]));
```

この定義では、各観測値$y_{n}$が混合成分のいずれかから得られると仮定しています。以下のように定義されます。

$$p(y \mid \lambda,\mu,\sigma)=\prod_{n=1}^{N}(\lambda\times\mathsf{Normal}(y_{n}\mid\mu_{1},\sigma_{1})+(1-\lambda)\times\mathsf{Normal}(y_{n}\mid\mu_{2},\sigma_{2})$$

上のモデルと、下の（間違った）モデルのベクトル化の試みを対比してみてください。

```
target += log_sum_exp(log(lambda)
                        + normal_lpdf(y | mu[1], sigma[1]),
                      log1m(lambda)
                        + normal_lpdf(y | mu[2], sigma[2]));
```

下も等価です。

```
target += log_mix(lambda,
                  normal_lpdf(y | mu[1], sigma[1]),
                  normal_lpdf(y | mu[2], sigma[2]));
```

この2番目の定義は、観測値の系列$y_{1},\dots,y_{n}$全体が一方の成分から、あるいは他方の成分から得られていることを意味します。別の密度を定義しているのです。

$$p(y\mid\lambda,\mu,\sigma)=\lambda\times\prod_{n=1}^{N}\mathsf{Normal}(y_{n}\mid\mu_{1},\sigma_{1})+(1-\lambda)\times\prod_{n=1}^{N}\mathsf{Normal}(y_{n}\mid\mu_{2},\sigma_{2})$$

### 13.6. 混合分布でサポートされる推定

多くの混合モデルでは、モデル中で混合成分は基本的に交換可能であり、したがって識別可能ではありません。混合成分のパラメーターが交換可能な事前分布を持ち、混合比が一様事前分布を持っていて、混合成分のパラメーターも尤度中で交換可能となる場合にこの問題が発生します。

この基本的な問題は、パラメーターに順序を付けることでうまく処理できました。場合によっては、当てはめに先立っても、その後でも、混合成分を取り出すことができます（例えば、男と女や、民主党と共和党）。

そうでない場合には、混合成分が実際にどのように割り当てられているには興味はなく、インデックスに依存しない推定を考えたいということになります。例えば、新しい観測値についての事後予測にのみ興味があることもあるでしょう。

#### 識別不可能な成分の混合分布

例題として、前の節の正規混合分布を考えます。パラメーターの組$(\mu_1,\sigma_1)$と$\mu_2,\sigma_2$に交換可能な事前分布を与えます。

$$ \mu_1,\mu_2 \sim \mathsf{Normal}(0,10) $$
$$ \sigma_1,\sigma_2 \sim \mathsf{HalfNormal}(0,10) $$

混合比の事前分布は一様分布とします。

$$ \lambda \sim \mathsf{Uniform}(0,1) $$

すると、尤度は次式になります。

$$ p(y_{n}\mid\mu,\sigma) = \lambda\mathsf{Normal}(y_{n}\mid\mu_{1},\sigma_{1}) + (1 - \lambda)\mathsf{Normal}(y_{n}\mid\mu_{2},\sigma_{2}) $$

そして、同時分布$p(y,\mu,\sigma,\lambda)$では、$\lambda$と$1-\lambda$が入れ替わって、パラメーター$(\mu_1,\sigma_1)$と$(\mu_2,\sigma_2)$が交換可能となります。^[ $\theta<0.5$のような制約を課すことでこの対称性は解決できますが、モデルと事後推定値を根本的に変えてしまいます。]

#### ラベルスイッチングのもとでの推定

混合成分が識別可能でない場合には、サンプリングや最適化アルゴリズムの収束を診断することが難しくなることがあります。MCMCの連鎖や最適化の実行ごとにラベルが入れ替わったり、順番が変わったりするためです。幸い、特定の成分のラベルを参照しないような事後推定値はラベルスイッチングのもとでも不変で、そのまま使うことができます。この小節では、2つの例を考えます。

##### 予測尤度

完全なパラメーターのベクトル$\theta$が与えられたときの新しい観測値$\tilde{y}$の予測尤度は次式になります。

$p(\tilde{y}\mid y) = \int_{\theta} p(\tilde{y}\mid\theta)p(\theta\mid y)d\theta$

前の節の正規混合分布の例では$\theta = (\mu,\sigma,\lambda)$で、ラベルスイッチングのもとでは尤度は同じ密度を返し、したがって予測推定値は正しいことがわかります。Stanでは、予測推定値は$p(\tilde{y}\mid y)$を計算するか、$\tilde{y}$の抽出をシミュレートするかして得られます。前者は、有効サンプルサイズの点で統計的により効率的です。後者は、ほかの推定に組み込むのがより簡単です。どちらの手法も、プログラムの`generated quantities`ブロックに直接コーディングできます。以下は直接的な（サンプリングではない）方法の例です。

```
data {
  int<lower = 0> N_tilde;
  vector[N_tilde] y_tilde;
  ...
generated quantities {
  vector[N_tilde] log_p_y_tilde;
  for (n in 1:N_tilde)
    log_p_y_tilde[n]
      = log_mix(lambda,
                normal_lpdf(y_tilde[n] | mu[1], sigma[1]),
                normal_lpdf(y_tilde[n] | mu[2], sigma[2]));
}
```

これは後で少し面倒になります。というのは、対数関数は線形ではなく、したがって平均全体には分布しないからです（Jensenの不等式がこの不均衡がどうなるかを示しています）。正しくは、`log_p_y_tilde`の事後抽出の`log_sum_exp`を使用します。`log(N_new)`を引くと、平均の対数予測密度が得られます。

##### クラスタリングと近似度

クラスタリングの問題に混合モデルはよく使われます。二つのデータ項目$y_i$と$y_j$があるときに、それらが同じ混合成分に由来するかどうかが問題になることがあるでしょう。$z_i$と$z_j$を、要素の負担を示す離散変数とすると、関心のある量は$z_i = z_j$であり、生起確率として要約できます。

$$ \Pr[z_i = z_j \mid y] = \int_{\theta}\frac{\sum_{k=0}^1 p(z_i = k, k_j = k, y_i, y_j \mid \theta)}{\sum_{k=0}^1 \sum_{m=0}^1 p(z_i = k, z_j = m, y_i, y_j \mid \theta)}p(\theta \mid y) d\theta $$

他の生起確率と同様に、これは`generated quantities`ブロックで計算できます。$z_i$と$z_j$をサンプリングして、等しいかどうかを示す関数を使うか、あるいは積分の内側の項を生成量として計算するかで、できます。予測尤度と同様に、期待値で計算する方がサンプリングよりも統計的に効率的です。

### 13.7. ゼロ過剰モデルとハードルモデル

ゼロ過剰モデルとハードルモデルはともに、ポアソン分布とベルヌーイ分布との確率質量関数の混合分布で、結果変数がゼロになる確率をより柔軟にモデリングできます。ゼロ過剰モデルは、Lambert (1992)により定義されたもので、零値の結果変数に対して追加の確率質量を与えるものです。一方、ハードルモデルは零値と非零値の結果変数の純粋な混合分布として定式化されます。

ゼロ過剰モデルとハードルモデルは、ポアソン分布以外の離散分布についても定式化できます。連続分布でのゼロ過剰は、導関数の問題のためにStanでは扱えません。特に、回帰係数の事前分布として正規分布をゼロ過剰にするようなときには、連続分布に対して点の質量を加算する方法がありません。

#### ゼロ過剰

以下のゼロ過剰ポアソン分布の例について考えます。ここではパラメーター`theta`を使って、確率$\theta$でゼロから抽出が行なわれ、確率$1-\theta$で$\mathsf{Poisson}(\lambda)$から抽出が行なわれるとしています。確率関数は以下のとおりです。

$$p(y_{n}\mid\theta,\lambda)=\begin{cases}\theta+(1-\theta)\times\mathsf{Poisson}(0\mid\lambda) & y_{n}=0のとき \\(1-\theta)\times\mathsf{Poisson}(y_{n}\mid\lambda)   & y_{n} > 0のとき\end{cases}$$

対数確率関数はStanでは以下のようにそのまま実装できます。

```
data {
  int<lower=0> N;
  int<lower=0> y[N];
}
parameters {
  real<lower=0, upper=1> theta;
  real<lower=0> lambda;
}
model {
  for (n in 1:N) {
    if (y[n] == 0)
      target += log_sum_exp(bernoulli_lpmf(1 | theta),
                            bernoulli_lpmf(0 | theta)
                              + poisson_lpmf(y[n] | lambda));
    else
      target += bernoulli_lpmf(0 | theta)
                  + poisson_lpmf(y[n] | lambda);
  }
}
```

`log_sum_exp(lp1,lp2)`関数は対数確率を線形軸で加算します。`log(exp(lp1) + exp(lp2))`と同じですが、算術的に より安定で高速です。これは条件演算子を使って書くこともできます。4.6節を参照してください。

#### ハードルモデル

ハードルモデルはゼロ過剰モデルに似ていますが、もっと柔軟で、零値の結果変数が過剰のときのみならず少なすぎるときも扱うことができます。ハードルモデルの確率質量関数は以下のように定義されます。

$$p(y_{n}\mid\theta,\lambda)=\begin{cases}\theta & y=0のとき \\(1-\theta)\frac{\mathsf{Poisson}(y\mid\lambda)}{1-\mathsf{PoissonCDF}(0\mid\lambda)} & y > 0のとき\end{cases}$$

ここで$\mathsf{PoissonCDF}$は、ポアソン分布の累積分布関数です。ハードルモデルはStanではさらに直接的にプログラミングできます。明示的な混合分布とする必要はありません。

```
if (y[n] == 0)
  1 ~ bernoulli(theta);
else {
  0 ~ bernoulli(theta);
  y[n] ~ poisson(lambda) T[1, ];
}
```

ベルヌーイ分布の文は、$\log\theta$と$\log(1-\theta)$を対数密度に足すようにするとちょっと短くなります。
ポアソン分布のあとの`T[1,]`は1未満の値が切断されることを示しています。これについては12.1節を、$\mathsf{PoissonCDF}$の詳細については52.5節を参照してください。正味の効果は、対数尤度の直接的な定義と等価です。

```
if (y[n] == 0)
  target += log(theta);
else
   target += log1m(theta) + poisson_lpmf(y[n] | lambda)
             - poisson_lccdf(0 | lambda));
```

以下はJulian Kingによる指摘です。

$$\log\mathsf{PoissonCDF}(0\mid\lambda)=\log(1-\exp(-\lambda))$$

なので、`else`節のCCDFはもっと簡潔な式に書き直すことができます。

```
target += log1m(theta) + poisson_lpmf(y[n] | lambda)
          - log1m_exp(-lambda));
```

これにより、CCDFを使ったコードよりも15%ほど高速になります。

前もって計数値をまとめておくことにより、密度を変えることなく実行速度をおおいに高速化することもできるという例を示します。データサイズが$N=200$で、パラメーターが$\theta=3$と$\lambda=8$のときは、10倍高速です。$N$が小さいときにはその程度は小さくなりますが、$N$が大きいときには大きくなります。$\theta$が大きいときにも大きくなります。

この高速化のためには、整数配列中にある非零値の要素の数を数える関数を定義しておくと役に立ちます。

```
functions {
  int num_zero(int[] y) {
    int nz = 0;
    for (n in 1:size(y))
      if (y[n] == 0)
        nz += 1;
    return nz;
  }
}
```

すると、`transformed data`ブロックで十分統計量を格納しておくことができます。

```
transformed data {
  int<lower=0, upper=N> N0 = num_zero(y);
  int<lower=0, upper=N> Ngt0 = N - N0;
  int<lower=1> y_nz[N - num_zero(y)];
  {
    int pos = 1;
    for (n in 1:N) {
      if (y[n] != 0) {
        y_nz[pos] = y[n];
        pos += 1;
      }
    }
  }
}
```

`model`ブロックは3文にまで減らすことができます。

```
model {
  N0 ~ binomial(N, theta);
  y_nz ~ poisson(lambda);
  target += -Ngt0 * log1m_exp(-lambda);
}
```

1番目の文は、零値と非零値の数についてのベルヌーイ分布の寄与に相当します。2行目は、非零値の数からのポアソン分布の寄与です。これはベクトル化されています。最後に、切断の正規化を1行で書いています。こうすると、0のところでの対数CCDFの式を繰り返さずにすみます。定数`Ngt0`を負にしていることにも注意しましょう。可能であれば、部分式を定数にします。そうすると、非定数項が出てくるまで、勾配を伝播させる必要がないからです。

### 13.8. 混合モデルでの事前分布と有効データサイズ

混合比$\lambda \in (0,1)$の2成分の混合モデルを考えます。混合成分の尤度は、混合成分の重みに比例して重みづけられますから、混合成分のそれぞれを推定するために使われる有効データサイズも、全体のデータサイズに対する割合として重みづけられます。したがって$N$個の観測値があったとしても、2成分について、なんらかの$\theta \in (0,1)$についての$\theta N$と$(1-\theta)N$という有効データサイズによって混合成分が推定されます。有効重みづけサイズは、単純に混合比$\lambda$によって決定されるのではなく、事後の負荷量によって決定されます。

#### モデルアベレージングとの比較

混合モデルは観測のレベルで混合を生成します。これと比較すると、モデルアベレージングは、完全なデータセットについて別々に当てはめたモデルの事後分布から混合を生成します。この状況で、事後分布が思うようになるのは、モデルを独立に当てはめ、事後分布が完全な観測データ$y$に基づいているときです。

異なるモデルで異なる観測値を説明したいときには、そのまま混合モデルを構築することをおすすめします。混合するモデルが似ているときには、1個の拡張モデルが両方の特徴を捕捉してしまって、推測（推定や意思決定、予測など）にはそちらだけが使われるということがよくあります。例を挙げると、切片だけの回帰と傾きだけの回帰とに当てはめて、その予測をアベレージングしたり、さらには混合モデルとするよりも、傾きと切片のある1個の回帰モデルを構築することをおすすめします。データ点よりも予測変数が多いといった複雑なモデルでも、適切に調整した事前分布を使うことでうまく使えるようになる場合があります。計算がボトルネックになる場合には、唯一の頼みがモデルアベレージングということはありえます。これは、各モデルを独立に当てはめたあとで、計算することができます。理論的・計算機的な詳細はHoeting et al. (1999)とGelman et al.(2013)を参照してください。
