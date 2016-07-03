**JRuby port of Mixed-Models**
-
##**Introduction**
For my GSoC 2016 project of JRuby port of NMatrix, we worked on running other Sciruby gems on JRuby. We started with **mixed-models** gem. 
Mixed models are statistical models which predict the value of a response variable as a result of fixed and random effects. All matrix calculations are performed using the gem **nmatrix**, which has a quite intuitive syntax and contributes to the overall code readability as well.


**[Note]:** *Some features of LMM that are provided by Daru gem, won't be available for now .*


## **Mixed Models**

The real motivation for working with JRuby port of Mixed-Models was to work with real-world data. We ran the code from this [blog](http://sciruby.com/blog/2015/08/19/gsoc-2015-mixed-models/) by Alexej Gossman, the author of 'mixed-models' gem which explains using *mixed-models* with some examples, using JRuby. We then compared the results for Ruby-MRI and JRuby.

####**Example1 : Blog_data**

This is a **work in progress** because of the following issues:
1. mixed_models not supported by latest [daru](https://github.com/agisga/mixed_models/issues/4 .). This issue has been resolved by Alexej.
2. The program gets stuck at [line 31](https://github.com/agisga/mixed_models/blob/master/examples/blog_data.rb#L31). This is the [message](https://gist.github.com/prasunanand/2b2573c3607b5365654e078a6aabbad6) that I get on interrupt. It still needs to be figured out.

####**Example2 : LMM**

Next I started running example [***LMM.rb***](https://github.com/agisga/mixed_models/blob/master/examples/LMM.rb). Alexej wrote a blog explaining this example and it can be found [here](http://www.alexejgossmann.com/First-linear-mixed-model-fit/). I ran the example using both ruby and jruby and compared output at every stage. Here I found these issues.

**Rank of a matrix**

One of the most important part of NMatrix was indexing. NMatrix stores multi-dimensional arrays as flat arrays and the indexing and slicing of elements is done using the shape, dimension, stride and offset of NMatrix. It would not be justice with NMatrix if I don't discuss about slicing and enumerators in NMatrix; so in my next blog I will discuss about slicing and also enumerators.
While working with LMM, there was an issue with getting the rank of a matrix. The rank couldn't be recursively accessed as the new matrix returned was not assigned dimension. This was a small bug but too tough to detect it.
  
**Cholesky/LUD Decomposition to solve a matrix when constants are a n x p matrix**

Given we need to solved a system of linear equations

                        AX = B
where A is an m×n matrix, B and X are n×p matrices, we needed to solve this equation by iterating through B.
Initially, for NMatrix-jruby we considered that X and B are column vectors. So, we got an exception 'dimension error'.
There was a similar issue with NMatrix -MRI : https://github.com/SciRuby/nmatrix/issues/374 .
We solved this issue by implementing NMatrix#matrix_solve that is called by method triangular_solve when using JRuby. 
```ruby
  def matrix_solve b
    if b.shape[1] > 1
      nmatrix = NMatrix.new :copy
      nmatrix.shape = b.shape
      result = []
      (0...b.shape[1]).each do |i|
        result.concat(self.solve(b.col(i)).s.toArray.to_a)
      end
      nmatrix.s = ArrayRealVector.new result.to_java :double
      nmatrix.twoDMat =  MatrixUtils.createRealMatrix 
                  get_twoDArray(b.shape, result)
      return nmatrix
    else
      return self.solve b
    end
  end
```
**Dot product**

I was stuck at another issue in model_fit. Triangular solve method which calls cholesky to solve linear equations threw *singular matrix exception*.  I wasn't unable to figure out what was wrong.  I started comparing the output of LMM.rb using ruby and jruby at each stage. y vector is critical for *model fit*. Apparently, the elements of y were different in the two cases. Looking closely, I found z.dot b to be returning a matrix with all the elements 0.  This is what was happening:
```ruby
...

# Generate the response vector
y = (x.dot beta) + (z.dot b) + epsilon

# Set up the covariance parameters
parametrization = Proc.new do |th| 
  diag_blocks = Array.new(5) { NMatrix.new([2,2], [th[0],th[1],0,th[2]], dtype: :float64) }
  NMatrix.block_diagonal(*diag_blocks, dtype: :float64) 
end

# Fit the model
model_fit = LMM.new(x: x, y: y, zt: z.transpose,
                    start_point: [1,0,1], 
                    lower_bound: Array[0,-Float::INFINITY,0],
                    &parametrization) 
...
```     
When we take dot product of two matrices C = A.dot B. If Aij or Bij are smaller than |1| we get Cij = 0 . So, yeah its autoboxing.
Currently, I solved this by using
```ruby
y = (x.dot beta) + ((z * 5).dot b)/5 + epsilon
```   
Thus, we get the value of y vector same for both cases.


 **Cholesky solve throws "singular matrix exception"**
 
When LMM does optimisation line [5] it calls NelderMead.minimize that uses deviation and autoboxing leads to 0 as element output. Therefore a diagonal matrix gets reduced to a singular matrix and Cholesky solve throws "singular matrix" error [6].
```ruby
...
model_fit = LMM.new(x: x, y: y, zt: z.transpose,
                    start_point: [1,0,1], 
                    lower_bound: Array[0,-Float::INFINITY,0],
                    &parametrization) 
...
```
Here z.transpose is wrong due to boxing
So, we used 
```ruby
zt = (z*5).transpose/5
```
```ruby
model_fit = LMM.new(x: x, y: y, zt: zt,
                    start_point: [1,0,1], 
                    lower_bound: Array[0,-Float::INFINITY,0],
                    &parametrization) 
```

**Result**
MIxed-models using Ruby-MRI
```ruby
(1) Model fit
Optimal theta:  [4.761283990026765, -0.12007961616262416, 0.5005024020787956]
REML criterion:   162.90752516637906
(2) Fixed effects
Intercept:  
Slope:  
(3) Random effects
Random intercept sd:  3.929341245265398
Random slope sd:  0.42437637915866583
Correlation of random intercept and slope:  -0.23341842320756737
(4) Residuals
Variance:   0.6800937307812478
Standard deviantion:  0.824677955799261
```
MIxed-models using JRuby
```ruby
(1) Model fit
Optimal theta:  [0.0056475944592377265, -5.661316609380864e-05, 0.0]
REML criterion:   379.0971583367289
(2) Fixed effects
Intercept:  
Slope:  
(3) Random effects
Random intercept sd:  0.06041274692971716
Random slope sd:  0.0006062864584118943
Correlation of random intercept and slope:  -1.0
(4) Residuals
Variance:   114.11688631756937
Standard deviantion:  10.682550553007898
```


##**Conclusion:**
We are getting different results due to boxing. One thing stands out how important real life testing is. **Unit tests do not cover chaining of results!** The complex computations are determined by boxing of the values.

###Tests

|Spec file|Total Test|Success|Failure|Pending|
|------------|:------------:|:-----------:|:-------------:|:-------------:|
|Deviance_spec|04|02|02|0|
|LMM_spec|195|04|191|0|
|LMM_categorical_data_spec.rb|46|4|42|0|
|LMMFormula_spec.rb|05|05|0|0|
|LMM_interaction_effects_spec.rb|82|06|76|0|
|matrix_methods_spec.rb|52|40|12|0|
|ModelSpecification_spec.rb|07|04|03|0|
|NelderMeadWithConstraints_spec.rb|08|08|0|0|



###Improvements and future work


Now we will focus on clearing all remaining tests for NMatrix and mixed-models. We will implement ruby-objects as dtype and start working on performance.
