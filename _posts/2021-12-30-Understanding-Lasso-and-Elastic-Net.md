---
title: "Undestanding Lasso & Elastic Net"
date: 2021-12-30
tags:
excerpt:
categories: [Tutorials]
---

# Understanding Lasso and Elastic Net

Algorithms such as Lasso and Elastic Net enjoy increasing popularity and have long found their way into the toolboxes of economists. Naturally, all standard programming languages feature packages that allow for off the shelf implementation and basic machine learning textbooks offer rich explanations of the theoretical foundations. In contrast, details about the efficient numerical solution algorithms and their implementation are rarely provided. If you - like me - would also like to gain a deeper understanding of the algorithms that are employed to solve for the solution paths of e.g. Lasso and Elastic Net, this tutorial might be interesting to you. It aims at implementing the pathwise coordinate descent algorithm following Friedman et al. (2007). In addition, this tutorial is written in julia, given julia's increasing popularity especially in macroeconomics.

You can also find a jupyter notebook version with interactive charts in the respective repository on my GitHub. As a small disclaimer, I try to write codes for these tutorials that are relatively intuitive and easy to read and understand. I am not a computer scientist and thus there are naturally always things to improve in terms of speed as well as style. In addition, the codes developed in this tutorial might of course contain bugs. As such, these codes are intended to improve the understanding of the theoretical foundations and inner workings of popular librares and packages and are not recommended to be used in actual application. Nonetheless, I hope that this tutorial is useful for some.

## 1 Some Derivations

As will become clear in a second, the algorithm rests on three key features:
* Iterating over the regression coefficients and updating them one at a time conditional on all others
* Partialing out the other coefficients at each step to convert the problem into a univariate one for which solutions are easy to compute
* Normalizing the predictors to be orthonormal

### 1.1 Lasso

Before we start with the actual code, let's derive the algorithm following Friedman et al. (2007) to build some intuition.

Let $N$ denote the number of observations and $K$ denote the number of variables. The Lasso optimization problem is given by:

$$
\text{min}_{\beta_j} \:\:f(\hat{\beta}) = \frac{1}{2} \sum_{i=1}^{N} (y_i - \sum_{k=j}^{K} x_{i,k}\beta_k)^2 %2B \lambda \sum_{k=j}^{K}|\beta_k|
$$

Similarly, the problem can be rewritten as:

$$
\text{min}_{\beta_j} \:\: f(\hat{\beta}) = \frac{1}{2} \sum_{i=1}^{N} (y_i - \sum_{k\neq j}^{K} x_{i,k}\beta_k - x_{i,j} \beta_j)^2 %2B \lambda \sum_{k=j}^{K}|\beta_k| %2B \lambda|\beta_j|
$$


For a single $\beta_j$ (conditional on all others), the solution to the Lasso problem is:

$$
\frac{\partial f(\hat{\beta}) }{\partial \beta_j} =  -\sum_{i=1}^{N} (y_i - \sum_{k\neq j}^{K} x_{i,k}\beta_k - x_{i,j} \beta_j)x_{i,j} \pm \lambda = 0
$$


With $\tilde{y_i}^{(j)} = \sum_{k\neq j}^{K} x_{i,k}\beta_k$, this yields:

$$
\sum_{i=1}^{N} (y_i-\tilde{y_i}^{(j)} - x_{i,j} \beta_j)x_{i,j} \mp \lambda =0
$$

If 𝑋:,𝑗 is normalized, such that it is orthonormal, then this can be rewritten as:

$$
\sum_{i=1}^{N} (y_i-\tilde{y_i}^{(j)})x_{i,j} -  \beta_j \mp \lambda =0
$$

and hence

$$
\hat{\beta}_j -  \beta_j \mp \lambda =0
$$

which gives

$$
\hat{\beta}_{j,Lasso} = \hat{\beta}_j \mp \lambda
$$

Following Donoho and Johnstone (1995) and Friedman et al. (2007), this is equivalent to the soft-thresholding rule:

$$
\begin{align}
\hat{\beta}_{j,Lasso} =
\begin{cases}
\hat{\beta}_j - \lambda, &\text{   if $\hat{\beta}_j>0$ and $\lambda < |\hat{\beta}_j|$,}\\
\hat{\beta}_j  + \lambda, &\text{   if $\hat{\beta}_j<0$ and $\lambda < |\hat{\beta}_j|$,}\\
0,                     &\text{    $\lambda > |\hat{\beta}_j|$.}
\end{cases}
\end{align}
$$


Equivalently, the soft-thresholding rule can be expressed as

$$
\hat{\beta}_{j,Lasso} = S(\hat{\beta}_j,\lambda) = \text{sign}(\hat{\beta}_j)(|\hat{\beta}_j|-\lambda)_+
$$

Alternatively, we can again start from

$$
\sum_{i=1}^{N} (y_i-\tilde{y_i}^{(j)})x_{i,j} -  \beta_j \mp \lambda =0,
$$

insert


$$
\sum_{i=1}^{N} (y_i-\tilde{y_i}^{(j)}+\beta_j x_{i,j}-\beta_j x_{i,j})x_{i,j} -  \beta_j \mp \lambda =0,
$$

which simplifies to

$$
\sum_{i=1}^{N} (y_i-\tilde{y_i}+\beta_j x_{i,j})x_{i,j} -  \beta_j \mp \lambda =0,
$$

where $\tilde{y_i} =  \sum_{k= 1}^{K} x_{i,k}\beta_k$. This then yields


$$
 \hat{\beta}_{j,Lasso} = \hat{\beta}_{j,Lasso} + (y_i-\tilde{y_i})x_{i,j} \mp \lambda
$$


This simplifies to the alternative soft-thresholding rule

$$
\hat{\beta}_{j,Lasso} = S(\hat{\beta}_{j,Lasso} + (y_i-\tilde{y_i})x_{i,j},\lambda)
$$

The Lasso solution for a given $\lambda$ can then be computed by iterating on either soft-thresholding rule for each coefficient, $j$, until convergence, i.e.:


$$
\hat{\beta}_{j,Lasso}^{(p)} = S\left((y_i-\tilde{y_i}^{(j)})x_{i,j},\lambda\right)\:\:\:\: \text{$p=1$,$\dots$, until convergence}
$$

or

$$
\hat{\beta}_{j,Lasso}^{(p)} = S\left(\hat{\beta}_{j,Lasso}^{(p-1)} + (y_i-\tilde{y_i})x_{i,j},\lambda\right) \:\:\:\: \text{$p=1$,$\dots$, until convergence}
$$



### 1.2 Elastic Net

The derivation for Elastic Net is generally similar. The optimization problem is given by:

$$
\text{min}_{\beta_j} \:\:f(\hat{\beta}) = \frac{1}{2} \sum_{i=1}^{N} (y_i - \sum_{k=j}^{K} x_{i,k}\beta_k)^2 + \lambda_1 \sum_{k=j}^{K}|\beta_k| +\lambda_2 \frac{1}{2}\sum_{k=j}^{K}\beta_k^2
$$

For ease of implementation, an equivalent formulation reads:

$$
\text{min}_{\beta_j} \:\:f(\hat{\beta}) = \frac{1}{2} \sum_{i=1}^{N} (y_i - \sum_{k=j}^{K} x_{i,k}\beta_k)^2 + \alpha \lambda \sum_{k=j}^{K}|\beta_k| +(1-\alpha)\lambda \frac{1}{2}\sum_{k=j}^{K}\beta_k^2
$$

where $\alpha = \frac{\lambda_1}{\lambda_1+\lambda_2}$ and $\lambda = \lambda_1+\lambda_2$. Following the same steps as above, this leads to

$$
\sum_{i=1}^{N} (y_i-\tilde{y_i}^{(j)})x_{i,j} -  \beta_j \mp \alpha\lambda - (1-\alpha)\lambda \beta_j=0,
$$

and hence the updating equation:

$$
\hat{\beta}_{j,EL}^{(p)} = \frac{S\left((y_i-\tilde{y_i}^{(j)})x_{i,j},\alpha\lambda\right)}{1+(1-\alpha)\lambda}
$$

This is just a rescaled version of the Lasso estimator derived above. This allows for a convenient and parsimonious implementation of the algorithm, where the Lasso estimator is computed as a special case of the Elastic Net estimator. The below code makes convenient use of this feature.

## 2 The Pathwise Coordinate Descent Algorithm

First, we need to load a couple of packages:

```julia
# Load  the required packages
using DataFrames
using CSV
using LinearAlgebra
using Plots
using Statistics
using StatsBase
```

Below, I construct a Elastic Net (EL) estimator structure. This structure contains the basic parameters that define the optimization problem, such as the outcome variables, the predictors, the number of predictors, the number of observations, and the maximum number of iterations etc.

```julia
abstract type EL_estimator end

mutable struct EL_CD{T} <: EL_estimator

    # initialize data types
    Y::Array{Float64,2}
    X::Array{Float64,2}
    n::T
    p::T
    maxIter::Int
    tol::Float64
    points::T

    function EL_CD(Y::Array{Float64,2}, X::Array{Float64,2},points::T,maxIter=100,tol=1e-5) where T

        # compute the size of the data
        n, p = size(X)

        # fill structure
        new{T}(Y, X, n, p, maxIter, tol, points)
    end

end
```

We will now build a couple of functions that operate on our newly created `EL_CD` structure and do the heavy lifting when computing the EL paths. Generally, it is advisable to split up the code into functions to avoid ambiguity for the compiler which in turn leads to gains in speed.

To make our estimator yet a little faster, we can rely on a clever strategy called `warm-start`. This means, that we use the solution path at the previous grid point as the initial value for the next grid point. The idea is that the solutions should not change too much from one grid point to the next, such that the algorithm then already starts close to the solution. This then reduces the number of iterations required until convergence and hence computational time.

Naturally, the question now is at which grid point to start. As a natural starting point, it makes sense to select the $\lambda$ for which all coefficients are $0$ and to reduce the $\lambda$ in consecutive steps until the OLS solution is reached. This specific $\lambda$ value of course depends on the optimization problem at hand. Let's thus start with writing a function that computes this $\lambda$ value dynamically.


```julia
function findlmax(x::Array{Float64,2},y::Array{Float64,2})

    # find the lambda where all parameters = 0
    return findmax(abs.(X'*Y))[1]
end
```

In addtion, the grid points for $\lambda$ are usually geometrically spaced. The grid itself, however, is still defined by its start and end points, as well as the total number of grid points. Only the spacing between the points is not homogenous. We thus also need a function that takes these three input arguments and returns a corresponding sqeuence of geometrically spaced values.



```julia
function geomprog(a::Float64,b::Float64,points::Int)

    # compute rate of geometric progression
    r = (b/a)^(1/(points-1))

    # return sequence of geometrically spaced values
    return a*r.^[0:1:points-1;]
end
```


To complete the automatic computation of our grid, we now simply need to put the two functions together. `endo_grid` simply calls the `geomprog` function with the $\lambda$ that sets all coefficients to zero. The function then returns a grid or sequence of the desired number of $\lambda$-values that is taylored to the specific problem at hand. This function will be called as soon as the `EL_CD` structure is initialized.


```julia
function endo_grid(x::Array{Float64,2}, y::Array{Float64,2}, points::Int)

    # compute the endogenous grid starting from the Lasso parameter that sets all βicients = 0.
    return geomprog(findlmax(x,y), 0.1, points)

end
```

A core ingredient of the pathwise coordinate descent algorithm is the `soft-thresholding` rule that computes the parameter updates. The below function implements the rule according to the algebra derived above.

```julia
function soft_thresholding(beta::Float64, lambda::Float64)

    t = append!([0.0],abs(beta)-lambda)

    # compute the soft-thresholding rule
    return sign(beta)*findmax(t)[1]
end
```

We now have functions that compute the gird for $\lambda$ as well as the soft-thresholding rule in place. Let's thus finally start to code up the coordinate descent algorihtm. Before we dive into it, however, it makes sense to recall that the final algorithm is essentially composed of two loops. The inner loop iterates over the coefficients to compute the solution at a given $\lambda$. The outer loop iterates over the grid for $\lambda$. For readability and convenience, it again makes sense to split up the code into functions accordingly. First, we will thus write a function that computes the inner loop.



```julia

function coordinate_descent_EL(init::Array{Float64,2}, lambda::Float64, model::EL_CD, alpha::Float64)

    # initialize an array containing the normed predictors
    X_n = similar(X)

    # orthonormalize each predictor variable
    for i = 1:model.p
        X_n[:,i] = model.X[:,i]./norm(model.X[:,i])
    end

    # set the initial number of iterations to 1
    iter_ = 1

    # initialize coefficients
    coeff = init

    # initialize the supremum of the Elastic Net/Lasso estimates
    sup = 10000000.0

    # initialize the difference of the coefficient estimates between iterations
    diffs = zeros((model.p,1))

    # run the updating algorithm until either the maximum number of iterations is reached or the supremum
    # of the coefficient estimates is smaller than the set tolerance.
    while iter_ < model.maxIter && sup > model.tol

        # iterate over the coefficients/predictors
        for i = 1:model.p

            # compute the predictions
            Y_tilde = X_n*coeff

            # compute the residual
            eps = model.Y - Y_tilde

            # compute the Lasso/Elastic Net estimates
            beta_EL = soft_thresholding((coeff[i].+X_n[:,i]'*eps)[1], alpha*lambda)/(1+(1-alpha)*lambda)

            # compute the difference in coefficient estimates between iterations
            diffs[i] = beta_EL-coeff[i]

            # update the coefficient with the EL/Lasso estimate
            coeff[i] = beta_EL

        end

        # update the number of iterations
        iter_ = iter_ + 1

        # update the supremum of the difference in coefficient estimates
        sup = findmax(abs.(diffs))[1]

    end

    # return the coefficients, supremum, and number of iterations
    return coeff, sup, iter_

end
```

The function works as follows. First, each predictor is normalized (for transparancy, this step is done within the function, but could of course be pre-computed). Then, the initial number of iterations is set to $1$. In addition, the difference of the coefficient estimates between iterations is initialized and its supremum set to a high initial value. For each iteration of the while loop, the function now computes the regression residuals and iteratively updates the parameter estimates according to the derived soft-thresholding rule. This process is repeated until convergence, i.e. until either the maximum number of iterations is achieved or until the supremum of the difference of the coefficient estiamtes between iterations is smaller than the specified tolerance. Finally, the function returns the coefficient estimates, the supremum, and the number of iterations. This yields the solution for a given $\lambda$ and $\alpha$.

Now, we turn to the the function for the outer loop, that computes the entire solution path across grid points.

```julia
function compute_path(model::EL_CD, Type::String="Lasso", alpha::Float64=1.0)

    # initialize the coefficients at 0, because the first grid point sets all coefficients
    # exactly to 0 by construction
    init = zeros((model.p,1))

    # initialize the array containing the solution path
    coeffs = zeros((model.p,model.points))

    # if the Lasso solution is to be computed, set the lambda ratio of Elastic net to 1.
    if Type == "Lasso"
        # Set alpha  to 1.0, if Type is Lasso
        alpha = 1.0
    end

    # compute the endogenous grid
    endogrid = endo_grid(model.X, model.Y, model.points)

    # Compute the Elastic Net/Lasso solution for every gripd point
    for i = 1:model.points

        # compute the solution for grid point i
        c = coordinate_descent_EL(init, endogrid[i], model, alpha)[1]

        # use warm-start to speed up convergence
        init = c

        # save the solution
        coeffs[:,i] = c
    end

    # return the solution path
    return coeffs
end
```

First, a few objects are initialized. Given that we start at the $\lambda$ for which all parameters are zero, it makes sense to set the initial values for the first grid point to zero as well. Next, if Lasso is supposed to be estimated, $\alpha$ is set to $1$. The EL soft-thresholding rule collapses to the one of Lasso in this case. As a next step, the function computes the grid of $\lambda$ values, based on the functions that we set up above. Now comes the actual outer loop. For each of the previously computed grid points, the loop iterates over the grid points and calls the  `coordinate_descent_EL` function, i.e. the function that contains the inner loop which yields the EL solution for a given $\lambda$ and $\alpha$. In addition, the initial value is then updated to the EL solution of the current grid point, which thus serves as a starting value for the next grid point. This is the ` warm-start` strategy outlined at above.

This is all we need!


## 3 The Objective function

Before we jump into an example, it is illustrative to have a look at the behaviour of the EL penalty function for a few theoretical examples. The codes for a variant of these charts with sliders can be found in the GitHub repository.


@import "C:\Users\A2010183\Desktop\Projects\MaximilianSchroeder.github.io\assets\images\Lasso1.JPG"

The above plot shows the shape of the penalty function (LHS) as well as objective function (RHS) for $\hat{\beta}=1.1$, $\lambda=0.7$, and $\alpha=1.0$. As we can see - for this parameterization - the EL penalty function is identical to the one of Lasso, because $\alpha=1.0$. In addition, given that $\lambda<\hat{\beta}$, we can directly see from the soft-thresholding rule derived above that $\beta_{Lasso}=0.4$, which is also the result that we see in the plot on the RHS.

In contrast, if we set $\lambda=2.0$, we have $\lambda>\hat{\beta}$ and thus the Lasso solution is $0$, which can again also be seen directly from the soft-thresholding rule.

@import "C:\Users\A2010183\Desktop\Projects\MaximilianSchroeder.github.io\assets\images\Lasso2.JPG"

Finally, if we set $\alpha=0.5$, the penalty function of EL is between the one of Ridge regression and Lasso. In addition, even though $\lambda>\hat{\beta}$, the solution for the EL estimate is now slightly positive. This result can again be directly derived from the soft-thresholding rule for EL.

@import "C:\Users\A2010183\Desktop\Projects\MaximilianSchroeder.github.io\assets\images\Lasso3.JPG"

## 4 Example

As an example I chose the worked "Diabetes" data set that is also included in Python's `sklearn` library. It can also be downloaded [here](https://www4.stat.ncsu.edu/~boos/var.select/diabetes.html). For more information, see Efron et al. (2004).

First, we need to load in the data:

```julia
# read in and extract the data
data   =   CSV.read("data.csv");
target = CSV.read("dtarget.csv");
```

With the data loaded, we can now simply initialize the Elastic Net structure that we created above. Beofre, however, we need to convert the input data to `array` fromat.

```julia
# read out the array of predictors
X = Matrix(data);

# read out the array containing the outcome variable
Y = Matrix(target);

# initlialize the Lasso/Elastic Net problem with 100 grid points.
model = EL_CD(Y,X,100)
```


The `Lasso` solution paths can now be computed using

```julia
# compute the Lasso solution paths
coeffs = compute_path(model, "Lasso");
```

Our Elastic net class will then automatically set $\alpha=1.0$ and the Elastic Net soft-thresholding rule collapses to Lasso.

Alternatively, Elastic Net solution paths can be computed using

```julia
# compute Elastic Net solution paths for an example lambda ratio
coeffs2 = compute_path(model, "Elastic Net", 0.9);
```

In this example code, $\alpha=0.9$. Generally, our pathwise coordinate descent algorithm runs fairly fast. On my laptop, the solution paths for 100 gird points were computed in less than a second. Of course, this depends on the hardware and the dimensions of the input data.

Finally, we can take a look at the computed solution paths. In the GitHub repository, you can find a version with sliders that allows for interactively changing $\alpha$, i.e. the relative weight put on the L1 and L2 regularisation during the EL estimation.

As an example, the below plot depicts the solution paths for the Lasso.

@import "C:\Users\A2010183\Desktop\Projects\MaximilianSchroeder.github.io\assets\images\Lasso4.JPG"

In contrast, the solution paths of EL with $\alpha=0.5$ look like this:

@import "C:\Users\A2010183\Desktop\Projects\MaximilianSchroeder.github.io\assets\images\Lasso5.JPG"

## 5 Conclusion

I hope this small tutorial helped shedding some light onto how pathwise coordinate descent can be used to compute the Lasso/EL solutions efficiently and thus allows for deeper understanding of the inner workings of popular Julia, Matlab, and Python libraries.

## 6 References

Donoho, D. L., & Johnstone, I. M. (1995). Adapting to unknown smoothness via wavelet shrinkage. Journal of the american statistical association, 90(432), 1200-1224.

Efron, B., Hastie, T., Johnstone, I., & Tibshirani, R. (2004). Least angle regression. The Annals of statistics, 32(2), 407-499.

Friedman, J., Hastie, T., Höfling, H., & Tibshirani, R. (2007). Pathwise coordinate optimization. The annals of applied statistics, 1(2), 302-332.
