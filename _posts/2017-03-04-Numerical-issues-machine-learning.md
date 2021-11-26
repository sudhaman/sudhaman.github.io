---
layout: post
title: Numerical issues in Machine learning
---

This is another one of the numerical issues posts which changed the final result while being mathematically correct.


My friend was working on the [Hidden Markov Model](https://en.wikipedia.org/wiki/Hidden_Markov_model) problem where he had found a working code from the github repository. While it worked for the sample case of 1000 observations it did not serve the purpose for a larger data set. My friend had a testset which had 18,000 observations and that is when this numerical issue was uncovered. The issue we faced and the way we overcame it is the topic of this blog post.


The glossary for the terms below are as below.

* O - Observation
* $$\pi_{i}$$ - Probability of being in state *i* at time 0
* $$b_{i}(O_{t})$$ - Probability of observing the *t*<sup>th</sup> observation from state i
* $$\alpha(i) = P(O_{1}, O_{2}, ... , O_{t}, q_{t} = S_{i} \vert \lambda) $$ - Probability of having observed $$O_{1}$$ to $$O_{t}$$ and being in state *i* at time t.

In the [forward algorithm](https://en.wikipedia.org/wiki/Forward_algorithm) we need to calculate $$\alpha$$ for each step. The $$\alpha$$s can be calculated by using the formulae below.


For observation 1 $$\alpha$$, we get the formula below.


$$
\alpha_{1}(i) = \pi_{i}b_{i}(O_{1})
$$


For $$\alpha$$ values at other observations we get the recursive formula below.


$$
\alpha_{t+1}(j) = b_{j}(O_{t+1})\sum_{i=1}^{N}\alpha_{t}{i} a_{ij}
$$


Code in python is as below
```python
    def _calcalpha(self,observations):
        '''
        Calculates 'alpha' the forward variable.
    
        The alpha variable is a numpy array indexed by time, then state (TxN).
        alpha[t][i] = the probability of being in state 'i' after observing the 
        first t symbols.
        '''        
        alpha = numpy.zeros((len(observations),self.n),dtype=self.precision)
        
        # init stage - alpha_1(x) = pi(x)b_x(O1)
        for x in xrange(self.n):
            alpha[0][x] = self.pi[x]*self.B_map[x][0]
        
        # induction
        for t in xrange(1,len(observations)):
            for j in xrange(self.n):
                for i in xrange(self.n):
                    alpha[t][j] += alpha[t-1][i]*self.A[i][j]
                alpha[t][j] *= self.B_map[j][t]
                
        return alpha
```


The calculation for $$\alpha$$ has products of previous $$\alpha$$s. Over time we know that this value would become 0 i.e. 0 in the double format while not being 0 mathematically. To understand this, let us say we were raising 0.1 to the power to index every time, then for the 18,000<sup>th</sup> entry it would be 0.1<sup>18000</sup> which is 0 in double format. But in order to avoid this situation we need to change the before mentioned logic. Let us look at another approach of doing this.


Let us say we had to compute a product of *A* and *B* which overflows the double or underflows the double. Then how do I represent this in some other format ?


The best way to make sure it is correctly represented is by taking a logarithm of the number.


$$log(A*B) = log(A) + log(B) $$


A similar logic for division follows.



$$log(\frac{A}{B}) = log(A) - log(B) $$


Hence, by this logic we can take logs on the individual terms and sum them up and yet correctly represent the actual number(we will see that we would normalise later which can be represented in double format).



Let us now look at sum of A and B and see if we can use some similar logic in order to solve the same numerical issues.


$$A + B = X(\frac{A}{X} + \frac{B}{X})$$


By making use of the equation above if we can make sure that the fractions A/X and B/X can be represented in double then we can compute the value of A + B and store it in logarithmic form. Let us look at the math below.


$$x = log(A)
$$


$$
y = log(B)
$$


We need to compute A + B.

$$A + B = e^{x} + e^{y}
$$


$$
A + B = e^{z}(e^{x-z} + e^{y-z})
$$


Now, we are down to choosing a suitable z such that $$e^{x-z}$$ and $$e^{y-z}$$ can be best represented as double. We know that the [format of double](https://en.wikipedia.org/wiki/Double-precision_floating-point_format) can represent numbers from $$e^{-1022}$$ upto $$e^{1023}$$. As this range is almost evenly divided between the positive and negative sides, we can choose z that divides the set of numbers(in this case *x* and *y*) into two equal halves i.e. (max+min)/2. In other words choose the Midrange value as z. Then compute $$e^{z}(e^{x-z} + e^{y-z})$$ as it will now be within range and then use the log of sum rule above to get the new value.


$$
A + B = e^{z}(e^{x-z} + e^{y-z})
$$


$$
log(A + B) = z + log(e^{x-z} + e^{y-z})
$$


And hence, we know the solution for the sum case. A similar argument for the difference case would yield us values within the range of double and we can obtain the log of differences too. 

