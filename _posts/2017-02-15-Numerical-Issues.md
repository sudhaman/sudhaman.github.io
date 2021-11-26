---
layout: post
title: Numerical Issues
---

Numerical issues that go undetected lead to unexpected issues. This is one of those tales of a bug that was lurking in the code and went undetected for long.

I was trying to fit a histogram to a set of data points. The bins of the [histogram](https://en.wikipedia.org/wiki/Histogram) were defined as below.

S - Smallest  
L - Largest  
  
$$ G = \frac{L-S}{N-3} $$

* Bin 1, covering interval (-infinity, S-G/2).
* Bin 2, covering interval [S-G/2, S+G/2).
* Bin 3, covering interval [S+G/2, S+G+G/2).
* Bin 4, covering interval [S+G+G/2, S+2G+G/2).
...
* Bin N-1, covering interval [S+(N-4)G+G/2, S+(N-3)G+G/2). This interval is the same as [L-G/2, L+G/2).
* Bin N, covering interval [S+(N-3)G+G/2, +infinity). This interval is the same as [L+G/2, +infinity).

#### My first approach
If we were to separate the first and the last bins then there exists a pattern to it. All these bins are  separated by a width of G.  
  
So, I could write forumlae for the bin as below,  

* Bin 1 : value < S - G/2
* Bin N : value > S + (N-3)G + G/2
* Bin (x > 1 and x < N) : $$ 2 + \lfloor\frac{x - Min + \frac{G}{2}}{G}\rfloor $$

The above expression can be validated by taking a few end points of the bins.

The code below in [MATLAB](https://en.wikipedia.org/wiki/MATLAB) represents the above mapping.
``` matlab
function [ b ] = mapToBucket( Min, Max, num, x )
    G = max(( Max - Min ) / (num-3), 0.0001);
    if( x < Min - (G/2) )
    	b = 1;
    % elseif( x >= Max + (G/2))
    % The above expression does not work when Max = Min in which case the
    % last bucket would be 0.0001 added from first bucket end point onwards
    elseif( x >= Min + G*(num-3) + (G/2))
    	b = num;
    else
    	b = 2 + floor(( x - Min + (G/2)) / G);
    end
end
```

This looks right and for many different values this looks to be genrating correct results. So, I moved on implementing the rest of the histogram code. When I completed my code, I could find results to be different than what was expected. Not suspecting this piece of code, I looked around through all other peices not find anything wrong. When I finally saw the bins that had excess and lower values than expected, I started to probe into this function and found the culprit - Numerical issues.  

Can you guess where the error lies in the above code ?
  
In a math world numbers can have infinte precission but in the world of computers this is not true. There are no REAL $$\mathbb{R}$$ numbers, instead we have **float**s and **double**s, which have fixed precission.

The line where we have the floor function is where the bug lies. The division within the floor function would ideally work but in the world of computers this is not really true.

Let us look at an example,  

Min = 0.2800  
Max = 0.6000  
num = 7  
x   = 0.4800  

``` matlab 
K>> floor(( x - Min + (G/2)) / G)

ans =

     2

K>> ( x - Min + (G/2)) / G

ans =

    3.0000

K>> ( x - Min + (G/2)) / G < 3

ans =

  logical

   1

K>>
```

The above evaluation clarifies whats happening inside. But what is we use higher precission of **MATLAB** ?  

Do we still face issues ?

```matlab
K>> floor(vpa( x - Min + (G/2)) / G)
 
ans =
 
3
 
K>>
```

And I tried to use this higher precission of numbers in my code. But this time I wanted to make sure I have no bugs lurking in my code. So, I did cross verify. And it was correct while being pretty slow.
``` matlab
function [ b ] = mapToBucket( Min, Max, num, x )
    G = max(( Max - Min ) / (num-3), 0.0001);
    if( x < Min - (G/2) )
    	b = 1;
    % elseif( x >= Max + (G/2))
    % The above expression does not work when Max = Min in which case the
    % last bucket would be 0.0001 added from first bucket end point onwards
    elseif( x >= Min + G*(num-3) + (G/2))
    	b = num;
    else
    	b = 2 + floor(vpa(( x - Min + (G/2)) / G));
    end
end
```

To increase the spead of execution and also after taking into consideration that data in reality will only bre repsented by certain decimal precession, I decided to write a different version of the same code but which deals with integers. The equation that we have derived before can be simplified further as below.

For bins 1 to *N*-1 we have the equation : $$ x < Min + (Bin -1)G - \frac{G}{2} $$  
This equation can be further simplified as below.  
Multiplying by 2 we get  
$$
2x < 2*Min + 2*(Bin-1)*G - G
$$  
$$
2x < 2*Min + (2*Bin -3)*G
$$  
  
We can substitute the value of G in the above equation.  
$$
2*x < 2*Min + (2*Bin -3)*\frac{Max-Min}{Num-3}
$$  
  
Multiplying by Num -3 we obtain  
$$
2*x*(Num-3) < 2*Min*(Num-3) + (2*Bin -3)*(Max-Min)
$$  
  
If we can run a loop through the bins then the smallest bin satisfying this equation, would be the bin we need.  
  
The final version of my code below.
``` matlab
function [ b ] = mapToBucket( Min, Max, num, x )
    % To take care of the Max = Min case
    if( Max == Min && x == Min)
        b = 2;
        return;
    end
    for b = 1:num-1
        if(2*x*(num-3) < (2*Min*(num-3)+(2*b-3)*(Max - Min)))
            return
        end
    end
    b = num;
end
```
  
The code above does not work directly on the fractional input. We need to convert the values of Min, Max and x into uint64 by multiplying them with the inverse of the least significant decimal unit(for eg, in Min was 0.41, Max was 0.833 and x was 0.59 then we would divide them by 0.001 making them Min - 441, Max - 833 and x - 590) . Dealing with integers reduces the uncertainty involved with the numbers as they are precise while being smaller in range. A uint64 was large enough for my data I was dealing with, you will need to make the choice based on the data you are dealing with.
