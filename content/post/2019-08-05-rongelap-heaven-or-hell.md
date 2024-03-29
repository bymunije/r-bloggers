---
title: 朗格拉普岛是太平洋上的度假胜地还是人间地狱？
author: 黄湘云
date: '2019-08-05'
slug: rongelap-heaven-or-hell
categories:
  - 统计模型
tags:
  - 混合效应模型
  - 空间随机过程
  - 广义线性模型
  - 泊松分布
  - 多元正态分布
  - 朗格拉普岛
thumbnail: https://marshallislands.llnl.gov/images/md_rongelap_resettlement.jpg
description: "朗格拉普岛位于南太平洋上，是马绍尔群岛的一部分，二战后，美国在该岛上进行了多次核武器测试，核爆炸后产生的放射性尘埃笼罩了全岛，目前该岛仍然不适合人类居住，只有经批准的科学研究人员才能登岛。从泊松广义线性模型到泊松空间广义线性混合效应模型，及其在分析朗格拉普岛上核污染物浓度的空间分布的应用"
---

先介绍泊松广义线性模型，包括模拟和计算，并和 Stan 实现的结果比较。第二部分建立空间泊松过程回归模型分析朗格拉普到上的核残留数据

## 1 模拟泊松线性模型

泊松广义线性模型如下：

`$$
\log(\lambda) = \beta_0 + \beta_1*X_1 + \beta_2*X_2, \quad Y \sim \mathrm{Poisson}(\lambda)
$$`

设定参数向量 `$\beta = (\beta_0, \beta_1, \beta_2) = (0.5, 0.3, 0.2)$`，观测变量 `$X_1$` 和 `$X_2$` 的均值都为0，协方差矩阵 `$\Sigma$` 为

`$$
\left[
 \begin{matrix}
   1.0 & 0.8  \\
   0.8 & 1.0 
 \end{matrix}
\right]
$$`

模拟观测到的响应变量值和协变量值

```r
set.seed(2018)
n <- 1000 # 样本量
X <- MASS::mvrnorm(n, mu = rep(0, 2), Sigma = matrix(c(1, 0.8, 0.8, 1), 2))
Beta <- c(0.5, 0.3, 0.2)
log.lambda <- cbind(1, X) %*% Beta
Y <- rpois(n, lambda = exp(log.lambda))
```

制作 Y 关于 X 的关系图

```r
op <- par(mfrow = c(1,3))
plot(Y ~ X[,1])
plot(Y ~ X[,2])
plot(X[,2] ~ X[,1], main = paste("cov(X[,1],X[,2]) =", round(cov(X[,1],X[,2]), digits = 3)))
par(op)
```

![泊松分布图](https://wp-contents.netlify.com/2019/07/poisson-sim-01.png)

根据观察值 `$(X,Y)$` 和对应的模型，使用优化算法，求解 `$\beta_0$`, `$\beta_1$` 和 `$\lambda$`

```r
glm.poisson <- glm.fit(y = Y, x = cbind(1,X), family = poisson(link = "log"))
# IWLS 迭代加权最小二乘估计
summary(glm.poisson)
```
```
                  Length Class  Mode   
coefficients         3   -none- numeric # 回归系数
residuals         1000   -none- numeric # 残差向量
fitted.values     1000   -none- numeric # Y 的拟合值/预测值
effects           1000   -none- numeric # 
R                    9   -none- numeric
rank                 1   -none- numeric #
qr                   5   qr     list   
family              12   family list    # 联系函数的形式 
linear.predictors 1000   -none- numeric # 线性预测 \beta_0 + \beta_1*X_1 + \beta_2*X_2
deviance             1   -none- numeric
aic                  1   -none- numeric # AIC
null.deviance        1   -none- numeric
iter                 1   -none- numeric # IWLS 算法迭代次数
weights           1000   -none- numeric
prior.weights     1000   -none- numeric
df.residual          1   -none- numeric # 残差自由度
df.null              1   -none- numeric
y                 1000   -none- numeric # Y 观测值
converged            1   -none- logical # 迭代是否收敛
boundary             1   -none- logical 
```
```r
# 回归系数
glm.poisson$coefficients
```
```
[1] 0.4508777 0.2877587 0.2574147
```
```r
# AIC
glm.poisson$aic
```
```
[1] 3139.414
```
```r
# 均方误差 MSE 观测值 - 预测值
var(glm.poisson$y - glm.poisson$fitted.values)
```
```
[1] 1.820285
```
```r
# 是否收敛
glm.poisson$converged
```
```
[1] TRUE
```
```r
# 拟合值是否在可到达的边界上
glm.poisson$boundary
```
```
[1] FALSE
```

将模型拟合结果绘制在图上

```r
op <- par(mfrow = c(1,3), family = "GB1")
plot(Y, main = "观察值（黑圈）和拟合值（红圈）", xlab = "", ylab = "")
points(glm.poisson$fitted.values, col = "red")
plot(glm.poisson$residuals, ylab = "残差值", xlab = "")
plot(glm.poisson$residuals, glm.poisson$fitted.values, xlab = "残差值", ylab = "拟合值")
par(op)
```

![泊松模型](https://wp-contents.netlify.com/2019/07/poisson-sim-02.png)

下面用brms提供的`brm`命令求解泊松模型，结果和 R 内置的 `glm` 差不多，但是由于有编译和抽样的过程，速度不及 R。

```r
dat <- cbind(Y,X)
colnames(dat) <- c("y", "x1", "x2")
library(brms)
glm.brm.poisson <- brm(y ~ x1 + x2, data = dat, family = poisson())
```
```
glm.brm.poisson
 Family: poisson 
  Links: mu = log 
Formula: y ~ x1 + x2 
   Data: dat (Number of observations: 1000) 
Samples: 4 chains, each with iter = 2000; warmup = 1000; thin = 1;
         total post-warmup samples = 4000

Population-Level Effects: 
          Estimate Est.Error l-95% CI u-95% CI Eff.Sample Rhat
Intercept     0.45      0.03     0.40     0.50       2447 1.00
x1            0.29      0.04     0.21     0.37       1458 1.00
x2            0.26      0.04     0.18     0.34       1474 1.00

Samples were drawn using sampling(NUTS). For each parameter, Eff.Sample 
is a crude measure of effective sample size, and Rhat is the potential 
scale reduction factor on split chains (at convergence, Rhat = 1).
```

```r
png(filename = "rats.png", res = 200, width = 480*200/72, height = 480*200/72,
    type = "cairo", bg = "transparent")
plot(glm.brm.poisson, pars = c("Intercept", "x1", "x2"))
dev.off()
```

![密度图和轨迹图](https://wp-contents.netlify.com/2019/07/glm-brm-possion-dens-trace.png)

```r
loo(glm.brm.poisson)
```
```
Computed from 4000 by 1000 log-likelihood matrix

         Estimate   SE
elpd_loo  -1569.7 24.6
p_loo         2.9  0.2
looic      3139.4 49.1
------
Monte Carlo SE of elpd_loo is 0.0.

All Pareto k estimates are good (k < 0.5).
See help('pareto-k-diagnostic') for details.
```

## 2. 朗格拉普岛上的核污染物浓度的空间分布

### 2.1 背景介绍

广义线性混合效应模型在地质统计具有重要的地位，除了采矿业，还可应用于卫生统计、 气象统计和空间经济统计等领域，如分析地区范围内的疟疾分布，有限气象站点条件下，预测地区 PM2.5 污染物浓度分布等。

1998 年 Peter J. Diggle 等提出蒙特卡罗极大似然方法估计不带块金效应的响应变量服从泊松分布的空间广义混合效应模型的参数，分析了朗格拉普岛上核污染浓度的空间分布，2004 年 Ole F Christensen 在 Peter J. Diggle 等人的基础上添加了块金效应，同样使用蒙特卡罗极大似然方法估计了模型中的参数。

```r
# 数据采集的日期
rongelap <- read.table(
  file =
    paste("http://www.leg.ufpr.br/lib/exe/fetch.php",
      "pessoais:paulojus:mbgbook:datasets:rongelap.txt",
      sep = "/"
    ), header = TRUE
)
# 准备数据
N = 157
x = as.matrix(dist(rongelap[, c("cX", "cY")]))/100 # 除以100为了防止大数吃掉小数
y = rongelap$counts
units = rongelap$time
# 保存 R 对象到磁盘文件
dump(c("N", "x", "y", "units"), file = 'rongelap.data.R')

# 朗格拉普岛海岸线
rongelap_coastline <- read.table(
  file =
    paste("http://www.leg.ufpr.br/lib/exe/fetch.php",
      "pessoais:paulojus:mbgbook:datasets:rongelap-coastline.txt",
      sep = "/"
    ), header = TRUE
)
head(rongelap)
```
```
     cX    cY counts time
1 -6050 -3270     75  300
2 -6050 -3165    371  300
3 -5925 -3320   1931  300
4 -5925 -3165   4357  300
5 -5800 -3350   2114  300
6 -5800 -3165   2318  300
```

wireframe 图

```r
library(lattice)
wireframe(counts/time ~ cX*cY, data = rongelap,
   scales = list(arrows = FALSE), 
   xlab = "X", ylab = "Y", zlab = "Z",
   drape = TRUE)
# https://www.stat.auckland.ac.nz/~ihaka/787/lectures-trellis.pdf
cloud(counts/time ~ cX*cY, data = rongelap,
   scales = list(arrows = FALSE), 
   xlab = "X", ylab = "Y", zlab = "Z",
   type = "h", pch = 16)

library(scatterplot3d)
scatterplot3d(x = rongelap$cX, y = rongelap$cY, z = rongelap$counts/rongelap$time, 
  xlab = "X", ylab = "Y", zlab = "Z",
  type = "h", color = "black", angle = 30, pch = 16)

library(plot3D)
png(filename = "rongelap.png", width = 10*80, height = 8*80, res = 150, type = "cairo")
par(mar = c(0.1, 2.5, 0.1, 0.1))
scatter3D(x = rongelap$cX, y = rongelap$cY, z = rongelap$counts/rongelap$time,
  xlab = "\nX", ylab = "\nY", zlab = "\nZ",
  pch = 16, type = "h", ticktype = "detailed",
  phi = 40, theta = 30, r = 50, d = 0.1, expand = 0.5, 
  bty = "f", box = TRUE, 
  col = "black", asp = 1, col.axis = "gray")
dev.off()
```

在朗格拉普岛上采样点的位置及观测的辐射强度，见下图

![rongelap gamma-ray](https://wp-contents.netlify.com/2019/07/rongelap-scatter3d.png)

数据集 rongelap 记录了157个测量点的伽马射线强度，即在时间间隔 time 内放射的粒子数目 counts，测量站点的横纵坐标分别为 cX 和 cY。

```r
library(grid)
library(ggplot2)

gg <- ggplot(data = rongelap_coastline, aes(x = cX, y = cY)) +
  geom_path() +
  geom_point(data = rongelap, aes(x = cX, y = cY))
  
gg2 <- ggplot(data = rongelap_coastline, aes(x = cX, y = cY)) +
  geom_path() +
  geom_point(data = rongelap, aes(x = cX, y = cY)) +
  coord_cartesian(xlim = c(-5700, -5540), ylim = c(-3260, -3100))

png(filename = "rongelap.png", width = 20*80, height = 10*80, res = 150, type = "cairo")
gg
print(gg2, vp = viewport(x = .20, y = .70, width = .2, height = .4))
dev.off()
```  

![朗格拉普岛上测量核污染浓度的站点分布图](https://wp-contents.netlify.com/2019/07/rongelap-ggplot2.png)

### 2.2 模型结构

`$$
\begin{align}
\log\{\lambda(x_i)\} & =  \beta + S(x_{i}) \\
\log\{\lambda(x_i)\} & =  \beta + S(x_{i}) + Z_{i}
\end{align}
$$`

根据 `${}^{137}\mathrm{Cs}$` 放出的伽马射线在 `$N=157$` 站点不同时间间隔`$t(x_{i})$`的放射量，建立泊松广义线性混合效应模型。模型中，截距 `$\beta$` 相当于平均水平，放射粒子数作为响应变量服从均值为 `$t(x_{i})\lambda(x_i)$` 的泊松分布，即 `$Y_{i} \sim \mathrm{Poisson}(t(x_{i})\lambda(x_i))$`，`$\lambda(x_i)$` 可理解为辐射强度，`$S(x_{i})$` 表示位置 `$x_i$` 处的空间效应，服从指数型自协方差函数为 

`$$
\mathrm{Cov}( S(x_i), S(x_j) ) = \sigma^2 \exp( -\|x_i -x_j\|_{2} / \phi )
$$`

的平稳空间高斯过程 `$S(x),x \in \mathbb{R}^2$`，且 `$Z_i$` 之间相互独立同正态分布`$\mathcal{N}(0,\tau^2)$`，`$Z_i$`表示与空间效应无关的块金效应，即非空间的随机效应，可以理解为测量误差或空间变差，这里 `$i = 1,\ldots, 157$`。

```bash
# 使用 cmdstan 
make examples/rongelap/rongelap

examples/rongelap/rongelap \
  sample random seed=1506552817 \
  data file=examples/rongelap/rongelap.data.R

bin/stansummary output.csv
```
```
Inference for Stan model: rongelap_model
1 chains: each with iter=(1000); warmup=(0); thin=(1); 1000 iterations saved.

Warmup took (705) seconds, 12 minutes total
Sampling took (760) seconds, 13 minutes total

                    Mean     MCSE   StdDev        5%       50%       95%  N_Eff  N_Eff/s    R_hat
lp__             3.4e+06  2.1e+00  1.3e+01   3.4e+06   3.4e+06   3.4e+06     36  4.8e-02  1.0e+00
accept_stat__    9.4e-01  2.1e-03  7.0e-02   8.0e-01   9.7e-01   1.0e+00   1093  1.4e+00  1.0e+00
stepsize__       1.2e-02  2.3e-17  2.3e-17   1.2e-02   1.2e-02   1.2e-02    1.0  1.3e-03  1.0e+00
treedepth__      8.0e+00  7.2e-03  2.2e-01   8.0e+00   8.0e+00   8.0e+00    930  1.2e+00  1.0e+00
n_leapfrog__     2.9e+02  4.4e+00  1.2e+02   2.6e+02   2.6e+02   5.1e+02    797  1.0e+00  1.0e+00
divergent__      0.0e+00  0.0e+00  0.0e+00   0.0e+00   0.0e+00   0.0e+00    500  6.6e-01     -nan
energy__        -3.4e+06  2.1e+00  1.5e+01  -3.4e+06  -3.4e+06  -3.4e+06     52  6.8e-02  1.0e+00
phi              1.2e+00  4.7e-02  2.9e-01   7.8e-01   1.1e+00   1.7e+00     38  4.9e-02  1.0e+00
sigmasq          1.0e-02  3.1e-04  2.0e-03   7.3e-03   1.0e-02   1.4e-02     43  5.6e-02  1.0e+00
alpha            1.3e+00  2.9e-03  1.6e-02   1.3e+00   1.3e+00   1.3e+00     31  4.0e-02  1.0e+00
eta[1]          -5.4e+00  7.2e-02  5.2e-01  -6.3e+00  -5.4e+00  -4.6e+00     52  6.8e-02  1.0e+00
eta[2]          -7.4e-01  8.1e-02  5.4e-01  -1.7e+00  -7.3e-01   1.4e-01     44  5.8e-02  1.0e+00
eta[3]           1.9e+00  6.6e-02  3.9e-01   1.2e+00   1.8e+00   2.5e+00     36  4.7e-02  1.0e+00
eta[4]           2.9e+00  4.1e-02  2.1e-01   2.5e+00   2.9e+00   3.2e+00     27  3.5e-02  1.0e+00
eta[5]           6.6e-02  2.6e-02  1.4e-01  -1.8e-01   7.2e-02   2.9e-01     30  3.9e-02  1.0e+00
eta[6]          -1.5e-01  3.3e-02  1.8e-01  -4.4e-01  -1.5e-01   1.6e-01     30  4.0e-02  1.0e+00
eta[7]           5.5e-02  2.3e-02  1.3e-01  -1.6e-01   6.1e-02   2.6e-01     31  4.1e-02  1.0e+00
eta[8]          -1.3e-01  1.3e-02  8.4e-02  -2.7e-01  -1.4e-01   4.1e-03     41  5.4e-02  1.0e+00
eta[9]          -8.1e-02  7.7e-03  8.0e-02  -2.1e-01  -8.0e-02   4.8e-02    109  1.4e-01  1.0e+00
eta[10]         -9.0e-02  8.3e-03  8.1e-02  -2.3e-01  -8.5e-02   3.8e-02     96  1.3e-01  1.0e+00
eta[11]          1.1e+00  1.7e-02  1.0e-01   9.5e-01   1.1e+00   1.3e+00     37  4.9e-02  1.0e+00
eta[12]         -1.0e-01  1.8e-02  1.0e-01  -2.8e-01  -9.9e-02   8.1e-02     33  4.4e-02  1.0e+00
eta[13]          8.2e-01  9.7e-03  8.1e-02   6.9e-01   8.2e-01   9.5e-01     71  9.3e-02  1.0e+00
eta[14]          9.6e-02  5.5e-03  7.7e-02  -2.4e-02   9.5e-02   2.2e-01    198  2.6e-01  1.0e+00
eta[15]          2.6e-01  6.7e-03  7.4e-02   1.3e-01   2.6e-01   3.8e-01    122  1.6e-01  1.0e+00
eta[16]          2.1e-01  8.0e-03  7.4e-02   8.9e-02   2.1e-01   3.3e-01     84  1.1e-01  1.0e+00
eta[17]          8.5e-01  1.4e-02  9.0e-02   7.0e-01   8.5e-01   1.0e+00     44  5.8e-02  1.0e+00
eta[18]         -1.8e+00  2.4e-02  1.4e-01  -2.0e+00  -1.8e+00  -1.6e+00     33  4.3e-02  1.0e+00
eta[19]          4.8e-01  7.8e-03  8.5e-02   3.4e-01   4.8e-01   6.2e-01    117  1.5e-01  1.0e+00
eta[20]         -1.0e-01  5.6e-03  7.6e-02  -2.3e-01  -1.0e-01   1.8e-02    184  2.4e-01  1.0e+00
eta[21]          5.4e-01  9.8e-03  7.5e-02   4.2e-01   5.4e-01   6.6e-01     59  7.7e-02  1.0e+00
eta[22]         -1.4e+00  2.3e-02  1.4e-01  -1.6e+00  -1.4e+00  -1.2e+00     37  4.9e-02  1.0e+00
eta[23]          8.6e-01  1.5e-02  9.6e-02   7.1e-01   8.6e-01   1.0e+00     43  5.7e-02  1.0e+00
eta[24]          2.0e+00  2.4e-02  1.3e-01   1.8e+00   2.0e+00   2.2e+00     29  3.9e-02  1.0e+00
eta[25]         -2.0e+00  2.9e-02  1.6e-01  -2.2e+00  -2.0e+00  -1.7e+00     31  4.0e-02  1.0e+00
eta[26]          2.5e+00  3.1e-02  1.7e-01   2.2e+00   2.5e+00   2.8e+00     28  3.6e-02  1.0e+00
eta[27]         -8.6e-01  1.7e-02  1.1e-01  -1.0e+00  -8.6e-01  -6.7e-01     42  5.5e-02  1.0e+00
eta[28]         -6.8e-01  1.4e-02  1.0e-01  -8.3e-01  -6.8e-01  -5.1e-01     51  6.7e-02  1.0e+00
eta[29]          4.5e-01  8.3e-03  7.8e-02   3.2e-01   4.5e-01   5.8e-01     90  1.2e-01  1.0e+00
eta[30]          3.0e-01  8.2e-03  7.6e-02   1.7e-01   3.0e-01   4.1e-01     87  1.1e-01  1.0e+00
eta[31]          1.1e+00  1.5e-02  9.1e-02   9.3e-01   1.1e+00   1.2e+00     37  4.9e-02  1.0e+00
eta[32]         -2.5e+00  3.3e-02  1.8e-01  -2.7e+00  -2.5e+00  -2.2e+00     30  3.9e-02  1.0e+00
eta[33]          8.5e-01  2.4e-02  1.3e-01   6.5e-01   8.5e-01   1.1e+00     29  3.8e-02  1.0e+00
eta[34]          5.0e-01  1.7e-02  9.7e-02   3.4e-01   5.0e-01   6.6e-01     33  4.3e-02  1.0e+00
eta[35]          3.5e-01  2.9e-02  1.5e-01   8.1e-02   3.5e-01   6.0e-01     28  3.7e-02  1.0e+00
eta[36]          9.8e-01  1.6e-02  1.0e-01   8.1e-01   9.7e-01   1.2e+00     39  5.2e-02  1.0e+00
eta[37]         -7.8e-02  1.9e-02  1.2e-01  -2.6e-01  -8.7e-02   1.3e-01     37  4.8e-02  1.0e+00
eta[38]          3.8e-01  1.6e-02  1.0e-01   2.2e-01   3.7e-01   5.6e-01     41  5.4e-02  1.0e+00
eta[39]          6.9e-01  1.7e-02  1.0e-01   5.3e-01   6.9e-01   8.6e-01     37  4.8e-02  1.0e+00
eta[40]          6.1e-01  1.5e-02  1.0e-01   4.4e-01   6.1e-01   7.9e-01     49  6.4e-02  1.0e+00
eta[41]         -6.1e-01  1.5e-02  1.0e-01  -7.8e-01  -6.1e-01  -4.5e-01     48  6.4e-02  1.0e+00
eta[42]          9.0e-01  1.2e-02  8.1e-02   7.7e-01   9.0e-01   1.0e+00     47  6.1e-02  1.0e+00
eta[43]         -1.1e+00  2.0e-02  1.2e-01  -1.3e+00  -1.1e+00  -8.5e-01     37  4.9e-02  1.0e+00
eta[44]          7.2e-01  1.3e-02  8.5e-02   5.8e-01   7.1e-01   8.6e-01     45  5.9e-02  1.0e+00
eta[45]         -7.7e-02  1.6e-02  1.0e-01  -2.5e-01  -8.1e-02   9.7e-02     40  5.3e-02  1.0e+00
eta[46]          1.1e+00  1.4e-02  9.8e-02   9.5e-01   1.1e+00   1.3e+00     47  6.2e-02  1.0e+00
eta[47]         -1.1e+00  1.9e-02  1.1e-01  -1.3e+00  -1.1e+00  -9.2e-01     33  4.4e-02  1.0e+00
eta[48]          7.6e-01  1.1e-02  8.0e-02   6.3e-01   7.6e-01   8.9e-01     56  7.3e-02  1.0e+00
eta[49]          3.1e-01  1.2e-02  9.0e-02   1.6e-01   3.1e-01   4.7e-01     53  6.9e-02  1.0e+00
eta[50]          4.7e-02  1.4e-02  9.5e-02  -1.1e-01   5.2e-02   2.0e-01     45  5.9e-02  1.0e+00
eta[51]          1.1e+00  1.3e-02  8.4e-02   9.5e-01   1.1e+00   1.2e+00     43  5.7e-02  1.0e+00
eta[52]         -4.7e-01  1.3e-02  9.6e-02  -6.3e-01  -4.7e-01  -3.1e-01     58  7.6e-02  1.0e+00
eta[53]          3.5e-01  9.1e-03  7.9e-02   2.1e-01   3.5e-01   4.7e-01     75  9.9e-02  1.0e+00
eta[54]         -5.8e-01  1.6e-02  1.1e-01  -7.7e-01  -5.8e-01  -4.1e-01     44  5.8e-02  1.0e+00
eta[55]          1.7e+00  2.5e-02  1.3e-01   1.5e+00   1.7e+00   1.9e+00     30  4.0e-02  1.0e+00
eta[56]         -6.8e-01  1.8e-02  1.1e-01  -8.6e-01  -6.8e-01  -4.8e-01     41  5.4e-02  1.0e+00
eta[57]         -6.1e-01  1.2e-02  9.6e-02  -7.8e-01  -6.1e-01  -4.6e-01     61  8.1e-02  1.0e+00
eta[58]          1.1e+00  1.4e-02  9.1e-02   9.2e-01   1.1e+00   1.2e+00     45  5.9e-02  1.0e+00
eta[59]         -1.3e+00  2.1e-02  1.3e-01  -1.5e+00  -1.3e+00  -1.1e+00     39  5.1e-02  1.0e+00
eta[60]          9.8e-01  2.8e-02  1.5e-01   7.3e-01   9.9e-01   1.2e+00     28  3.7e-02  1.0e+00
eta[61]         -1.2e+00  1.9e-02  1.2e-01  -1.4e+00  -1.2e+00  -1.0e+00     41  5.4e-02  1.0e+00
eta[62]          1.1e+00  2.7e-02  1.4e-01   8.7e-01   1.1e+00   1.3e+00     27  3.6e-02  1.0e+00
eta[63]          7.6e-01  2.2e-02  1.2e-01   5.7e-01   7.6e-01   9.7e-01     29  3.9e-02  1.0e+00
eta[64]          2.9e-01  3.2e-02  1.7e-01  -1.6e-02   2.9e-01   5.4e-01     28  3.6e-02  1.0e+00
eta[65]          7.4e-01  3.2e-02  1.7e-01   4.6e-01   7.5e-01   1.0e+00     27  3.6e-02  1.0e+00
eta[66]         -1.0e+00  2.0e-02  1.2e-01  -1.2e+00  -1.0e+00  -8.1e-01     36  4.7e-02  1.0e+00
eta[67]         -1.3e+00  2.3e-02  1.7e-01  -1.5e+00  -1.3e+00  -9.6e-01     53  7.0e-02  1.0e+00
eta[68]          1.9e-01  2.5e-02  1.5e-01  -4.1e-02   1.7e-01   4.6e-01     37  4.9e-02  1.0e+00
eta[69]          6.3e-01  2.9e-02  1.5e-01   3.7e-01   6.3e-01   8.8e-01     28  3.6e-02  1.0e+00
eta[70]          1.1e+00  3.7e-02  1.9e-01   7.5e-01   1.1e+00   1.4e+00     28  3.6e-02  1.0e+00
eta[71]          4.7e-01  3.4e-02  1.8e-01   1.6e-01   4.8e-01   7.6e-01     28  3.7e-02  1.0e+00
eta[72]          5.3e-01  3.1e-02  1.6e-01   2.6e-01   5.4e-01   7.9e-01     28  3.6e-02  1.0e+00
eta[73]          3.1e-01  2.8e-02  1.5e-01   4.5e-02   3.2e-01   5.5e-01     29  3.8e-02  1.0e+00
eta[74]          2.2e-01  2.6e-02  1.4e-01  -2.5e-02   2.2e-01   4.3e-01     29  3.9e-02  1.0e+00
eta[75]          5.6e-01  2.9e-02  1.5e-01   2.9e-01   5.6e-01   8.1e-01     28  3.7e-02  1.0e+00
eta[76]          6.8e-01  3.2e-02  1.7e-01   3.9e-01   6.9e-01   9.7e-01     28  3.6e-02  1.0e+00
eta[77]          6.9e-01  3.4e-02  1.7e-01   3.8e-01   6.9e-01   9.6e-01     27  3.6e-02  1.0e+00
eta[78]          1.2e-02  2.7e-02  1.5e-01  -2.4e-01   2.2e-02   2.4e-01     29  3.8e-02  1.0e+00
eta[79]          8.6e-02  2.4e-02  1.4e-01  -1.4e-01   9.1e-02   3.1e-01     32  4.2e-02  1.0e+00
eta[80]         -6.9e-01  2.1e-02  1.4e-01  -9.0e-01  -7.0e-01  -4.4e-01     44  5.8e-02  1.0e+00
eta[81]          1.2e+00  3.0e-02  1.6e-01   9.0e-01   1.2e+00   1.4e+00     27  3.6e-02  1.0e+00
eta[82]         -4.7e-01  2.5e-02  1.4e-01  -7.1e-01  -4.7e-01  -2.6e-01     32  4.2e-02  1.0e+00
eta[83]          6.7e-01  2.4e-02  1.3e-01   4.6e-01   6.7e-01   8.9e-01     29  3.8e-02  1.0e+00
eta[84]         -1.8e-01  2.3e-02  1.4e-01  -4.0e-01  -1.8e-01   2.5e-02     34  4.5e-02  1.0e+00
eta[85]          3.6e-01  1.9e-02  1.1e-01   1.9e-01   3.6e-01   5.3e-01     32  4.3e-02  1.0e+00
eta[86]         -1.7e+00  2.5e-02  1.9e-01  -2.0e+00  -1.7e+00  -1.4e+00     54  7.0e-02  1.0e+00
eta[87]          3.3e-01  2.0e-02  1.3e-01   1.1e-01   3.2e-01   5.6e-01     43  5.6e-02  1.0e+00
eta[88]         -2.8e-01  2.1e-02  1.3e-01  -4.8e-01  -2.8e-01  -8.6e-02     38  4.9e-02  1.0e+00
eta[89]          7.5e-01  2.0e-02  1.1e-01   5.8e-01   7.5e-01   9.4e-01     32  4.2e-02  1.0e+00
eta[90]          7.9e-01  1.4e-02  9.0e-02   6.4e-01   7.9e-01   9.4e-01     41  5.3e-02  1.0e+00
eta[91]         -7.0e-01  1.5e-02  9.1e-02  -8.4e-01  -7.1e-01  -5.5e-01     36  4.7e-02  1.0e+00
eta[92]          9.4e-01  1.6e-02  8.8e-02   8.0e-01   9.4e-01   1.1e+00     30  3.9e-02  1.0e+00
eta[93]         -8.9e-01  1.7e-02  9.4e-02  -1.0e+00  -9.0e-01  -7.3e-01     31  4.1e-02  1.0e+00
eta[94]         -1.4e-01  1.2e-02  8.5e-02  -2.8e-01  -1.4e-01   5.0e-03     49  6.4e-02  1.0e+00
eta[95]          1.9e-01  6.2e-03  7.4e-02   7.6e-02   1.9e-01   3.2e-01    140  1.8e-01  1.0e+00
eta[96]         -1.0e+00  1.6e-02  9.1e-02  -1.2e+00  -1.0e+00  -9.0e-01     34  4.5e-02  1.0e+00
eta[97]         -6.0e-02  4.7e-03  5.1e-02  -1.4e-01  -5.9e-02   2.5e-02    121  1.6e-01  1.0e+00
eta[98]         -5.9e-01  7.5e-03  6.2e-02  -6.9e-01  -5.9e-01  -4.8e-01     70  9.2e-02  1.0e+00
eta[99]         -9.6e-01  2.7e-02  2.0e-01  -1.3e+00  -9.7e-01  -6.1e-01     51  6.7e-02  1.0e+00
eta[100]         9.5e-01  1.8e-02  1.0e-01   7.9e-01   9.5e-01   1.1e+00     32  4.2e-02  1.0e+00
eta[101]        -1.1e+00  1.4e-02  1.0e-01  -1.3e+00  -1.1e+00  -9.0e-01     53  6.9e-02  1.0e+00
eta[102]         9.9e-01  1.4e-02  9.9e-02   8.4e-01   9.9e-01   1.2e+00     48  6.4e-02  1.0e+00
eta[103]        -8.5e-01  1.0e-02  6.6e-02  -9.5e-01  -8.5e-01  -7.4e-01     42  5.6e-02  1.0e+00
eta[104]        -7.7e-02  6.0e-03  6.1e-02  -1.8e-01  -7.6e-02   2.1e-02    103  1.4e-01  1.0e+00
eta[105]         1.5e+00  2.3e-02  1.2e-01   1.3e+00   1.5e+00   1.7e+00     29  3.9e-02  1.0e+00
eta[106]        -6.1e-02  7.0e-03  8.2e-02  -1.9e-01  -6.2e-02   7.4e-02    137  1.8e-01  1.0e+00
eta[107]         9.1e-02  4.4e-03  7.7e-02  -3.2e-02   8.8e-02   2.2e-01    306  4.0e-01  1.0e+00
eta[108]        -2.5e-01  5.2e-03  5.8e-02  -3.4e-01  -2.4e-01  -1.5e-01    128  1.7e-01  1.0e+00
eta[109]        -6.5e-01  8.8e-03  6.8e-02  -7.7e-01  -6.4e-01  -5.4e-01     59  7.8e-02  1.0e+00
eta[110]        -1.2e+00  1.4e-02  8.9e-02  -1.3e+00  -1.2e+00  -1.0e+00     38  5.0e-02  1.0e+00
eta[111]        -1.7e+00  2.4e-02  1.8e-01  -2.0e+00  -1.7e+00  -1.4e+00     59  7.8e-02  1.1e+00
eta[112]         8.1e-01  1.4e-02  9.4e-02   6.6e-01   8.1e-01   9.8e-01     45  5.9e-02  1.0e+00
eta[113]         2.2e-01  6.0e-03  7.7e-02   9.9e-02   2.3e-01   3.5e-01    162  2.1e-01  1.0e+00
eta[114]        -4.1e-01  4.6e-03  6.4e-02  -5.1e-01  -4.1e-01  -3.1e-01    196  2.6e-01  1.0e+00
eta[115]         4.0e-01  1.1e-02  7.7e-02   2.7e-01   4.0e-01   5.3e-01     52  6.9e-02  1.0e+00
eta[116]         3.0e-01  1.1e-02  8.3e-02   1.6e-01   3.1e-01   4.3e-01     58  7.6e-02  1.0e+00
eta[117]         7.0e-01  2.7e-02  1.7e-01   4.4e-01   6.9e-01   1.0e+00     37  4.9e-02  1.0e+00
eta[118]        -3.9e-01  1.2e-02  8.5e-02  -5.3e-01  -3.9e-01  -2.4e-01     54  7.1e-02  1.0e+00
eta[119]        -1.0e-01  1.1e-02  8.2e-02  -2.3e-01  -1.1e-01   3.8e-02     57  7.5e-02  1.0e+00
eta[120]         7.2e-02  7.2e-03  6.9e-02  -4.4e-02   7.2e-02   1.8e-01     92  1.2e-01  1.0e+00
eta[121]         4.2e-01  7.9e-03  7.3e-02   3.0e-01   4.2e-01   5.4e-01     85  1.1e-01  1.0e+00
eta[122]        -6.4e-01  1.0e-02  7.4e-02  -7.6e-01  -6.4e-01  -5.2e-01     53  6.9e-02  1.0e+00
eta[123]        -6.1e-01  9.4e-03  7.8e-02  -7.3e-01  -6.2e-01  -4.8e-01     69  9.1e-02  1.0e+00
eta[124]        -2.0e+00  2.2e-02  1.4e-01  -2.3e+00  -2.0e+00  -1.8e+00     43  5.7e-02  1.1e+00
eta[125]        -6.2e-01  2.1e-02  1.5e-01  -8.5e-01  -6.3e-01  -3.5e-01     47  6.2e-02  1.0e+00
eta[126]         1.7e-01  1.7e-02  9.5e-02   2.0e-02   1.7e-01   3.3e-01     33  4.3e-02  1.0e+00
eta[127]         6.0e-01  1.6e-02  9.1e-02   4.5e-01   5.9e-01   7.6e-01     34  4.5e-02  1.0e+00
eta[128]         1.2e+00  1.9e-02  1.1e-01   1.0e+00   1.2e+00   1.4e+00     31  4.0e-02  1.0e+00
eta[129]         5.2e-01  9.6e-03  7.0e-02   4.0e-01   5.2e-01   6.4e-01     52  6.8e-02  1.0e+00
eta[130]         1.1e+00  1.5e-02  8.8e-02   9.9e-01   1.1e+00   1.3e+00     36  4.7e-02  1.0e+00
eta[131]        -1.8e+00  2.2e-02  1.2e-01  -1.9e+00  -1.8e+00  -1.6e+00     28  3.7e-02  1.0e+00
eta[132]         1.5e+00  2.5e-02  1.4e-01   1.2e+00   1.5e+00   1.7e+00     31  4.1e-02  1.0e+00
eta[133]         1.4e+00  2.4e-02  1.3e-01   1.2e+00   1.4e+00   1.6e+00     32  4.2e-02  1.0e+00
eta[134]        -5.2e-01  1.5e-02  9.3e-02  -6.7e-01  -5.2e-01  -3.6e-01     38  4.9e-02  1.0e+00
eta[135]        -1.5e+00  1.9e-02  1.2e-01  -1.7e+00  -1.5e+00  -1.3e+00     41  5.4e-02  1.0e+00
eta[136]         4.9e-01  9.9e-03  7.5e-02   3.7e-01   4.9e-01   6.2e-01     58  7.7e-02  1.0e+00
eta[137]        -7.8e-01  1.0e-02  7.1e-02  -8.9e-01  -7.8e-01  -6.5e-01     46  6.1e-02  1.0e+00
eta[138]         2.0e-01  7.9e-03  6.6e-02   9.7e-02   2.0e-01   3.1e-01     69  9.1e-02  1.0e+00
eta[139]         5.8e-01  1.1e-02  7.5e-02   4.6e-01   5.8e-01   7.0e-01     45  6.0e-02  1.0e+00
eta[140]         9.8e-01  1.5e-02  9.2e-02   8.3e-01   9.8e-01   1.1e+00     39  5.2e-02  1.0e+00
eta[141]        -9.9e-01  1.4e-02  8.6e-02  -1.1e+00  -9.9e-01  -8.5e-01     39  5.1e-02  1.0e+00
eta[142]        -2.4e-01  4.3e-03  6.1e-02  -3.4e-01  -2.3e-01  -1.4e-01    208  2.7e-01  1.0e+00
eta[143]        -2.5e-02  7.0e-03  6.6e-02  -1.3e-01  -2.9e-02   8.7e-02     90  1.2e-01  1.0e+00
eta[144]        -1.2e+00  1.7e-02  1.3e-01  -1.4e+00  -1.2e+00  -1.0e+00     59  7.8e-02  1.0e+00
eta[145]        -5.9e-01  1.5e-02  1.1e-01  -7.7e-01  -6.0e-01  -4.1e-01     52  6.9e-02  1.0e+00
eta[146]        -1.7e+00  1.9e-02  1.3e-01  -1.9e+00  -1.7e+00  -1.5e+00     47  6.2e-02  1.1e+00
eta[147]        -3.7e-01  1.6e-02  1.2e-01  -5.7e-01  -3.8e-01  -1.7e-01     55  7.2e-02  1.0e+00
eta[148]         2.7e-01  2.0e-02  1.2e-01   8.2e-02   2.7e-01   4.8e-01     37  4.8e-02  1.0e+00
eta[149]         5.2e-01  2.6e-02  1.4e-01   2.8e-01   5.2e-01   7.4e-01     30  3.9e-02  1.0e+00
eta[150]         4.6e-01  2.9e-02  1.5e-01   1.9e-01   4.6e-01   6.9e-01     28  3.7e-02  1.0e+00
eta[151]         4.2e-02  2.6e-02  1.4e-01  -1.8e-01   4.8e-02   2.4e-01     28  3.7e-02  1.0e+00
eta[152]        -1.9e-01  2.2e-02  1.3e-01  -3.8e-01  -1.9e-01   1.5e-02     32  4.2e-02  1.0e+00
eta[153]        -9.7e-01  2.2e-02  1.5e-01  -1.2e+00  -9.8e-01  -7.0e-01     44  5.7e-02  1.0e+00
eta[154]        -4.5e-01  2.3e-02  1.5e-01  -6.8e-01  -4.6e-01  -1.6e-01     43  5.6e-02  1.0e+00
eta[155]        -9.2e-01  1.7e-02  1.1e-01  -1.1e+00  -9.2e-01  -7.4e-01     42  5.5e-02  1.0e+00
eta[156]         6.6e-02  1.5e-02  1.1e-01  -1.1e-01   6.5e-02   2.5e-01     50  6.6e-02  1.0e+00
eta[157]         3.4e-01  1.6e-02  1.1e-01   1.7e-01   3.3e-01   5.1e-01     47  6.1e-02  1.0e+00

Samples were drawn using hmc with nuts.
For each parameter, N_Eff is a crude measure of effective sample size,
and R_hat is the potential scale reduction factor on split chains (at
convergence, R_hat=1).
```

R 语言方案

```r
# 无空间效应
fit <- glm(rongelap$data ~ 1, family = poisson(link = "log"), offset = log(rongelap$units.m))
fit
```
```

Call:  glm(formula = rongelap$data ~ 1, family = poisson(link = "log"), 
    offset = log(rongelap$units.m))

Coefficients:
(Intercept)  
      2.014  

Degrees of Freedom: 156 Total (i.e. Null);  156 Residual
Null Deviance:      61570 
Residual Deviance: 61570        AIC: 63090
```

> 注意
> 
> 不能将响应变量设为 `rongelap$data/log(rongelap$units.m)` 因为 `family = poisson(link = "log")` 的时候，响应变量只能放正整数！


```r
# 不收敛
library(MASS)
rongelap$dummary <- 1:157
glmmPQL.rongelap <- glmmPQL(counts ~ 1, random = ~ 1 | dummary,
  data = rongelap, weight = time, family = poisson(link = "log"),
  corr = corExp(3.5, form = ~ cY + cX, nugget = FALSE)
)
```

rstan 方案

```r
library(rstan)
options(mc.cores = 2)
rstan_options(auto_write = TRUE)
data(rongelap, package = "geoRglm")
rongelap_pois_stan <- stan_model("rongelap.stan")
# 指数型相关函数
# 坐标 scale 防止大数吃小数
rongelap_data <- list(
  N = 157, x = as.matrix(dist(rongelap$`coords`))/100,
  y = rongelap$data, units = rongelap$units.m
)
# 预计 37.5 个小时
# 20000 留下来的采样点，后验分布的样本量
rongelap_fit <- sampling(rongelap_pois_stan,
  data = rongelap_pois_data,
  chains = 2, thin = 5,
  iter = 50000, control = list(adapt_delta = 0.98, max_treedepth = 15)
)
rongelap_fit
```


# 3 结论

1. 计算方面，如果能用 R 自带的程序实现就用自带的，它比 Stan 要快；尽量用 CmdStan 来计算，其它接口如 rstan/pystan 效率不及它。而且具有复杂的依赖！自从在 Docker 内配置 CmdStan 后，计算方便多了，配置过程见 <https://www.xiangyunhuang.com.cn/post/2019/05/02/intro-cmdstan/>

2. 朗格拉普岛居民生活现状、岛的生态环境现状、周边海域生态状况，岛上和海洋里的活体，包括植物和珊瑚都变异了，惨不忍睹，画面有点吓人就不贴图了，自行放狗搜！目前，辐射水平依然很高，不适合人类居住！！！

3. 有待改进的地方包括 Stan 代码中选择更加合适的先验分布，以加快迭代速度和迭代效率（每秒产生更多的有效样本），通过手动编码后验分布，增加更多的编码技巧，详见 <https://www.xiangyunhuang.com.cn/post/2019/08/11/exact-sparse-car-stan/>，针对 模型输出文件 `output.csv`，提供更多关于模型结果的分析，

# 4 代码

`rongelap.stan` 文件内容

```
functions {
  matrix matern_covariance( int N, matrix dist, real phi, real sigma_sq) {
    matrix[N,N] S;
    real dist_phi; 

    // exponential == Matern nu=1/2 , (p=0; nu=p+1/2)
      for(i in 1:(N-1)){
        for(j in (i+1):N){
          dist_phi = fabs(dist[i,j])/phi;
          S[i,j] = sigma_sq * exp(- dist_phi ); 
        }
      }

    for(i in 1:(N-1)){
      for(j in (i+1):N){
        S[j,i] = S[i,j];  // fill upper triangle
      }
    }

    for(i in 1:N) S[i,i] = sigma_sq;            
    return(S);
  }
}               

data {
  int<lower=1> N;
  matrix[N, N] x;
  int<lower=0> y[N];
  real<lower=0> units[N];
}
parameters {
  real<lower=0> phi;
  real<lower=0> sigmasq;
  real alpha;
  vector[N] eta;
}
model {
  vector[N] f;
  vector[N] lambda;
  {
    matrix[N, N] L_K;
    matrix[N, N] K = matern_covariance(N, x, phi, sigmasq);
    // diagonal elements
    for (n in 1:N)
      K[n, n] = K[n, n] + 1e-12;

    L_K = cholesky_decompose(K);
    f = L_K * eta;
  }

  phi ~ std_normal();
  sigmasq ~ std_normal();
  alpha ~ std_normal();
  eta ~ std_normal();
  
  // sigmasq is not nugget
  for (i in 1:N)
    lambda[i] =  log(units[i]) * (alpha + f)[i];
  
  y ~ poisson_log(lambda);
}
```

# 参考文献

1. Diggle, P. J., Harper, L. and Simon, S. L. (1997). Geostatistical analysis of residual contamination from nuclea testing. In: Statistics for the environment 3: pollution assesment and control (eds. V. Barnet and K. F. Turkmann), Wiley, Chichester, 89-107.<https://www.wiley.com/en-us/exportProduct/pdf/9780471964353>

1. Diggle, P. J. and Tawn, J. A. and Moyeed, R. A. 1998. Model-based Geostatistics. Journal of the Royal Statistical Society: Series C (Applied Statistics). 47(3): 299--350. <http://dx.doi.org/10.1111/1467-9876.00113>

1. Ole F Christensen. 2004. Monte Carlo Maximum Likelihood in Model-Based Geostatistics. Journal of Computational and Graphical Statistics. 13(3): 702--718. <https://doi.org/10.1198/106186004X2525>

1. Bordner, Autumn S. and Crosswell, Danielle A. and Katz, Ainsley O. and Shah, Jill T. and Zhang, Catherine R. and Nikolic-Hughes, Ivana and Hughes, Emlyn W. and Ruderman, Malvin A. 2016. Measurement of background gamma radiation in the northern Marshall Islands. Proceedings of the National Academy of Sciences. 113(25): 6833--6838. <https://doi.org/10.1073/pnas.1605535113>

1. Chandra, Noirrit and Bhattacharya, Sourabh. 2019. Non-marginal decisions: A novel Bayesian multiple testing procedure. Electronic Journal of Statistics. 3(1): 489--535. <https://doi.org/10.1214/19-EJS1535>

1. 美国在朗格拉普岛进行核武器测试导致的核污染 <https://en.wikipedia.org/wiki/Rongelap_Atoll>

1. Guofeng Cao, Phaedon C. Kyriakidis and Michael F. Goodchild (2011) A multinomial logistic mixed model for the prediction of categorical spatial data, International Journal of Geographical Information Science, 25:12, 2071-2086, <https://doi.org/10.1080/13658816.2011.600253>

1. Fatemeh Hosseini and Omid Karimi (2019) Approximate composite marginal likelihood inference in spatial generalized linear mixed models, Journal of Applied Statistics, 46(3):542-558, <https://doi.org/10.1080/02664763.2018.1506020>

1. Evangelos Evangelou and Vivekananda Roy. 2019. Estimation and prediction for spatial generalized linear mixed models with parametric links via reparameterized importance sampling. Spatial Statistics. 29:289--315. <https://doi.org/10.1016/j.spasta.2019.01.005>

1. Erlis Ruli, Nicola Sartori, and Laura Ventura. 2016. Improved Laplace approximation for marginal likelihoods. Electronic Journal of Statistics. 10(2):3986--4009. <https://doi.org/10.1214/16-EJS1218>

1. Diggle, P. J. and {Ribeiro Jr.}, Paulo Justiniano. 2000. Model-based Geostatistics,  <http://leg.ufpr.br/~paulojus/pub/MBGsinape.pdf>

1. Christensen, O., Roberts, G., and Sköld, M. (2006). Robust Markov Chain Monte Carlo Methods for Spatial Generalized Linear Mixed Models. Journal of Computational and Graphical Statistics, 15(1), 1-17. <https://www.jstor.org/stable/27594162>

1. Estimation and prediction for spatial generalized linear mixed models with parametric links via reparameterized importance sampling. <https://arxiv.org/abs/1803.04739v4>
