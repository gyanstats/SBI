---
output:
  pdf_document: default
  html_document: default
---

### SM2 / SC2 Project

# Using ABC/SIR to Model the Spread of Influenza in a Boarding School

For this group project, we investigated the problem of intractable likelihoods using Approximate Bayesian Computation. Such simulation based methods are best applied to this case where we are working with a model such that we are able to simulate results easily from it, yet have no analytical form of the likelihood available. This is exactly the case in epidemiology models, where we are unable to use methods such as maximum likelihood estimation to estimate the infection parameters given data.

## Data-set

We begin by importing the dataset we chose for this project: the `bsflu` dataset from the package `pomp`. This dataset records a 1978 Influenza outbreak in a boy's boarding school.


```r
library(pomp)
library(Rcpp)
library(sitmo)
library(ggplot2)
data(bsflu)
```

The dataset tallies infection information over a period of 14 days, in a boarding school of 763 students.


```r
head(bsflu)
```

```
##         date   B  C day
## 1 1978-01-22   1  0   1
## 2 1978-01-23   6  0   2
## 3 1978-01-24  26  0   3
## 4 1978-01-25  73  1   4
## 5 1978-01-26 222  8   5
## 6 1978-01-27 293 16   6
```

The column `B` contains the number of students who are bedridden with the flu on a given day (i.e. classes as 'infected'). C contains the number of students who are *convalescent* (i.e. not infected but yet unable to return to class).


```r
library(tidyr)
library(ggplot2)


bsflu |>
    gather(variable,value,-date,-day) |>
    ggplot(aes(x=date,y=value,color=variable))+
    geom_line()+
    geom_point()+
    labs(y="Number of Boys",x="Date",title="Boarding school infuenza outbreak (22nd Jan - 4th Feb)")+
  scale_x_date(date_minor_breaks = "1 day",date_labels = "%b %d")+
    theme_light()
```

![](Project_files/figure-latex/unnamed-chunk-3-1.pdf)<!-- --> 


## Model

In order to model disease data, we will use the well-studied SIR model. This model models the number of people in three states: Susceptible, Infected, and Recovered. The model is defined by the following system of differential equations: $$
\begin{align*}
\frac{dS}{dt} &= -\beta S I \\
\frac{dI}{dt} &= \beta S I - \gamma I \\
\frac{dR}{dt} &= \gamma I
\end{align*}
$$

Where $S$ is the proportion of susceptible individuals, $I$ is the proportion of infected individuals, and $R$ is the proportion of recovered individuals. $\beta$ is the rate of infection, and $\gamma$ is the rate of recovery.

With such a definition, we can translate the column `B` in the `bsflu` data directly to the variable $I$ simply by dividing `B` by the total number of students ($N=763$). Unfortunately, the column `C` has no analogy in the model, as it acts as a confusing 'between recovery' state that cannot be grouped in with either $I$ or $R$. Therefore going forward, we will primarily be using the column `B` as the observed data in our SIR model estimate.

Approximate Bayesian computation is a simulation based approach, and will require many individual computations. In the interest of speed therefore, we will implement the SIR model in C++, then use RCPP to call the C++ code from R.


```r
sourceCpp(code = "
  #include <Rcpp.h>
  using namespace Rcpp;

  // [[Rcpp::export]]
  NumericVector SIR(NumericVector s, NumericVector i, NumericVector r, double s0, double i0, double r0, double beta, double gamma) {
    s[0] = s0;
    i[0] = i0;
    r[0] = r0;

    for (int t = 1; t < s.size(); t++) {
      s[t] = s[t-1] - beta * s[t-1] * i[t-1];
      i[t] = i[t-1] + beta * s[t-1] * i[t-1] - gamma * i[t-1];
      r[t] = r[t-1] + gamma * i[t-1];
    }

    return i;
  }
")
```

Below is a demonstration of the SIR model above, with $\beta = 0.8$ and $\gamma = 0.2$.


```r
s <- numeric(20)
i <- numeric(20)
r <- numeric(20)

sir <- SIR(s, i, r, 1 - 1/763, 1/763, 0, 0.8, 0.2)

# convert SIR to a data frame
sir <- data.frame(day = 1:20, sir = sir)
print(sir)
```

```
##    day         sir
## 1    1 0.001310616
## 2    2 0.002095611
## 3    3 0.003349026
## 4    4 0.005347643
## 5    5 0.008527571
## 6    6 0.013569424
## 7    7 0.021518985
## 8    8 0.033942171
## 9    9 0.053083221
## 10  10 0.081917372
## 11  11 0.123828512
## 12  12 0.181407697
## 13  13 0.253810306
## 14  14 0.333041786
## 15  15 0.402372183
## 16  16 0.442376936
## 17  17 0.443721280
## 18  18 0.413185769
## 19  19 0.365510796
## 20  20 0.313113502
```

```r
ggplot(aes(x=day,y=sir), data = sir)+
  geom_line(color="red")+
  geom_point(color="red")+
  labs(y="Proportion of Infected",x="Day",title="SIR Model with beta=0.8, gamma=0.2")+theme_light()
```

![](Project_files/figure-latex/unnamed-chunk-5-1.pdf)<!-- --> 

## ABC Definition

Consider first the general case of intractable likelihood: having a model $f(.|\theta)$ with intractable likelihood $l(\theta|.)$ and parameter $\theta$.

For ABC, we first repeatedly generate samples of $\theta$ from a prior distribution $\theta \sim \pi(.)$. Then for each generated value of $\theta$, we input this into our effectively 'black box' model to get simulated data $\tilde{y}(\theta) \sim f(.|\theta)$.

We then need to define some distance metric $D$ between the observed data $y$ and simulated data $\tilde{y}(\theta)$, only accepting the proposed $\theta$ if this distance falls below a defined tolerance value $\epsilon$.

Even a simple rejection sampling algorithm such as this can be shown to produce samples $\{\theta_1,...,\theta_M\}$ ($M$ being the number of accepted values) that are samples from the joint distribution:$$
\begin{align*}
    \pi_{\epsilon}(\theta,\tilde{y}|y) = \frac{\pi(\theta)f(\tilde{y}|\theta)\mathbb{I}(\tilde{y}\in A)}{\int_A\int_\Theta\pi(\theta)f(\tilde{y}|\theta) d \tilde{y} d \theta}
\end{align*}$$ Where $\mathbb{I}$ is the indicator function, $\Theta$ is the support of $\theta$, and $A$ is the acceptance region defined by $D$, $y$, and $\epsilon$. Then given a suitable choice of tolerance value, this can produce an approximation to the posterior distribution of $\theta$ [1].$$
\begin{align*}
    \pi_{\epsilon}(\theta|y) = \int_A\pi_{\epsilon}(\theta,\tilde{y}|y)d\tilde{y} \approx \pi(\theta|y)
\end{align*}$$

Clearly here much of the resulting estimate relies on our choice of tolerance parameter $\epsilon$ and distance metric $D$, both of which will be looked at later. Something else to consider is that in practice the distance metric is applied to a *summary statistic* of the data rather than the raw data, in order to reduce dimensionality. This can be anything from the mean $\bar{y}$ and empirical quantiles, to more complex statistics such as kernels or auxiliary parameters. These methods will be looked at near the tail end of our investigation.

## ABC Implementation

For the distance metric within ABC, we will need to compare the simulated data to the observed data. However as noted in the Data-set section, only the column B can be used. Hence we will compare the B column of the dataset to the number of infected individuals I in our SIR model. Similar to the approach of [2], we will use the mean squared error between the proportions of infected individuals in the observed data and the simulated data as our distance metric, and compare it to the other distance metric used; the absolute error between the proportion of infected individuals on the final day of the observed data and the simulated data.

The following code implements the ABC/SIR algorithm in C++.


```r
sourceCpp(code='
#include <Rcpp.h>
#include <RcppParallel.h>
#include <omp.h>
#include <sitmo.h>
#include <cmath>
using namespace Rcpp;

// [[Rcpp::depends(RcppParallel)]]
// [[Rcpp::depends(sitmo)]]
// [[Rcpp::plugins(openmp)]]

// Function to simulate data from SIR model

// [[Rcpp::export]]
double unif_sitmo(int seed) {
  uint32_t coreseed = static_cast<uint32_t>(seed);
  sitmo::prng eng(coreseed);
  double mx = sitmo::prng::max();
  double x = eng() / mx;
  return x;
}

double calc_dist_serial(double* x_sim, NumericVector x) {
  double total;
  for (int i=0; i<20; i++) {
    total += pow(x[i]-x_sim[i], 2);
  }
  return total;
}

// [[Rcpp::export]]
NumericMatrix ABC(int n, double eps, int p, NumericVector x, int ncores, int metric)
{
  
  NumericMatrix accepted_samples(n, p);
  int count = 0;
  double dist;

  #pragma omp parallel num_threads(ncores)
  {
    double theta_sim[2];
    #pragma omp for
    for (int i=0; i<n; i++) {

      #pragma omp critical
      {
      theta_sim[0] = unif_sitmo(i);
      theta_sim[1] = unif_sitmo(i+n);
      }
      
      // NumericVector I = SIR(762.0/763.0, 1.0/763.0, 0.0, theta_sim[1], theta_sim[2]);

      double S[20];
      double I[20];
      double R[20];

      S[0] = 762.0/763.0;
      I[0] = 1.0/763.0;
      R[0] = 0.0;

      for (int t = 1; t < 20; t++) {
        S[t] = S[t-1] - theta_sim[0] * S[t-1] * I[t-1];
        I[t] = I[t-1] + theta_sim[0] * S[t-1] * I[t-1] - theta_sim[1] * I[t-1];
        R[t] = R[t-1] + theta_sim[1] * I[t-1];
      }

      #pragma omp critical
      {
      if (metric == 1) {
        dist = calc_dist_serial(I, x);
      } else if (metric == 2) {
        dist = pow(I[19]-x[19], 2);
      }
      if (dist < eps) {
        accepted_samples(i, 0) = theta_sim[0];
        accepted_samples(i, 1) = theta_sim[1];
        count++;
      }
      }
    }
  }
  
  std::cout << "Acceptance rate: " << (double)count / n << std::endl;
  return accepted_samples;
  
}

')
```

The function `ABC` takes in the number of samples `n`, the tolerance `eps`, the number of parameters `p`, the observed data `x`, and the number of cores to use `ncores`. It then returns a matrix of accepted samples. It is important to note that we should normalise the `x` vector before inputting it into the ABC function, as the SIR model uses proportions rather than raw numbers.

As an example, we can run the ABC algorithm with $10^7$ samples, a tolerance of 0.8, and 2 parameters, using the mean squared error as the distance metric.


```r
x <- bsflu$B/763
accepted_samples <- ABC(1e7, 0.8, 2, x, 4, 1)

# remove zeros
accepted_samples <- accepted_samples[!rowSums(accepted_samples==0),]
```

We can then plot the marginal posterior distributions of the parameters $\beta$ and $\gamma$.


```r
# plot the density histograms
ggplot(aes(x=accepted_samples[,1]), data = as.data.frame(accepted_samples))+
  geom_density(fill="blue",color="black")+
  labs(x="Beta",y="Frequency",title=expression(paste("Marginal Posterior Distribution of ", beta)))+theme_light()
```

![](Project_files/figure-latex/unnamed-chunk-8-1.pdf)<!-- --> 

```r
ggplot(aes(x=accepted_samples[,2]), data = as.data.frame(accepted_samples))+
  geom_density(fill="blue",color="black")+
  labs(x="Gamma",y="Density",title=expression(paste("Marginal Posterior Distribution of ", gamma)))+theme_light()
```

![](Project_files/figure-latex/unnamed-chunk-9-1.pdf)<!-- --> 

Now we can compare these posterior distributions to the distributions obtained when we use the absolute error between the proportion of infected individuals on the final day of the observed data and the simulated data as the distance metric.


```r
accepted_samples2 <- ABC(1e7, 0.01, 2, x, 4, 2)
accepted_samples2 <- accepted_samples2[!rowSums(accepted_samples2==0),]
```


```r
ggplot(aes(x=accepted_samples2[,1]), data = as.data.frame(accepted_samples2))+
  geom_density(fill="blue",color="black")+
  labs(x="Beta",y="Frequency",title=expression(paste("Marginal Posterior Distribution of ", beta)))+theme_light()
```

![](Project_files/figure-latex/unnamed-chunk-11-1.pdf)<!-- --> 


```r
ggplot(aes(x=accepted_samples2[,2]), data = as.data.frame(accepted_samples2))+
  geom_density(fill="blue",color="black")+
  labs(x="Gamma",y="Density",title=expression(paste("Marginal Posterior Distribution of ", gamma)))+theme_light()
```

![](Project_files/figure-latex/unnamed-chunk-12-1.pdf)<!-- --> 

## Acceptance Rates

We can plot the acceptance rates for the two distance metrics as a function of the tolerance value.


```r
eps <- seq(0.1, 3, 0.1)
acceptance_rates <- numeric(length(eps))
for (i in 1:length(eps)) {
  accepted_samples <- ABC(1e7, eps[i], 2, x, 4, 1)
  acceptance_rates[i] <- sum(accepted_samples[,1] != 0) / nrow(accepted_samples)
}
```


```r
eps2 <- seq(0, 0.1, 0.005)
acceptance_rates2 <- numeric(length(eps2))
for (i in 1:length(eps2)) {
  accepted_samples2 <- ABC(1e7, eps2[i], 2, x, 4, 2)
    acceptance_rates2[i] <- sum(accepted_samples2[,1] != 0) / nrow(accepted_samples)
}
```


```r
# plot the acceptance rates on two adjacent plots
par(mfrow=c(1,2))
plot(eps, acceptance_rates, type="l", xlab="Tolerance", ylab="Acceptance Rate", main="Mean Squared Error")
plot(eps2, acceptance_rates2, type="l", xlab="Tolerance", ylab="Acceptance Rate", main="Absolute Error")
```

![](Project_files/figure-latex/unnamed-chunk-15-1.pdf)<!-- --> 


## References

[1] J.-M. Marin, P. Pudlo, C. P. Robert, and R. J. Ryder, "Approximate bayesian computational methods" Stat Comput, vol. 22, pp. 1167--1180, Nov. 2012.

[2] A. Minter, and R. Retkute, "Approximate Bayesian Computation for infectious disease modelling". Epidemics, vol. 29, p.100368.