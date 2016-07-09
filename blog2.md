**JRuby port of Mixed-Models**
-
##**Introduction**
For my GSoC 2016 project of JRuby port of NMatrix, we worked on testing NMatrix-JRuby with real-life data. We started with **mixed-models** gem. 
Mixed models are statistical models which predict the value of a response variable as a result of fixed and random effects. All matrix calculations are performed using the gem **nmatrix**, which has a quite intuitive syntax and contributes to the overall code readability as well.




## **Mixed Models**

The real motivation for working with JRuby port of Mixed-Models was to work with real-world data. We ran the code from this [blog](http://sciruby.com/blog/2015/08/19/gsoc-2015-mixed-models/) by Alexej Gossman, the author of 'mixed-models' gem which explains using *mixed-models* with some examples, using JRuby. We then compared the results for Ruby-MRI and JRuby.


####**Example1 : LMM**

I started running example [***LMM.rb***](https://github.com/agisga/mixed_models/blob/master/examples/LMM.rb). Alexej wrote a blog explaining this example and it can be found [here](http://www.alexejgossmann.com/First-linear-mixed-model-fit/). I ran the example using both ruby and jruby and compared output at every stage. Here I found these issues.

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
      res = []
      (0...b.shape[1]).each do |i|
        res << self.solve(b.col(i)).s.toArray.to_a
      end
      index = 0
      (0...b.shape[0]).each do |i|
        (0...b.shape[1]).each do |j|
          result[index] = res[j][i]
          index+=1
        end
      end
      nmatrix.s = ArrayRealVector.new result.to_java :double
      nmatrix.twoDMat =  MatrixUtils.createRealMatrix get_twoDArray(b.shape, result)
      
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
 
When LMM does optimisation line [5] it calls NelderMead.minimize that uses deviation; and autoboxing leads to 0 as element output. Therefore a diagonal matrix gets reduced to a singular matrix and Cholesky solve throws "singular matrix" error [6].
```ruby
...
model_fit = LMM.new(x: x, y: y, zt: z.transpose,
                    start_point: [1,0,1], 
                    lower_bound: Array[0,-Float::INFINITY,0],
                    &parametrization) 
...
```
Here z.transpose is wrong due to boxing. So, we used 
```ruby
zt = (z*5).transpose/5
```
```ruby
model_fit = LMM.new(x: x, y: y, zt: zt,
                    start_point: [1,0,1], 
                    lower_bound: Array[0,-Float::INFINITY,0],
                    &parametrization) 
```
This error is not always replicated.

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
MIxed-models using JRuby initially gave the following result. 
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
There was an error in NMatrix#matrix_solve. After correcting it, we get the correct result.
```ruby
(1) Model fit
Optimal theta:  [4.761283990026765, -0.12007961616262416, 0.5005024020787956]
REML criterion:   162.9075251663791
(2) Fixed effects
Intercept:  
Slope:  
(3) Random effects
Random intercept sd:  3.929341245265402
Random slope sd:  0.4243763791586663
Correlation of random intercept and slope:  -0.2334184232075674
(4) Residuals
Variance:   0.6800937307812492
Standard deviantion:  0.8246779557992618
```
Next, we ran other examples and we got the correct results as we expected.
####**Example2 : Blog_data**

Blog_data example deals with real data. Initially, mixed_models was not supported by latest [daru](https://github.com/agisga/mixed_models/issues/4 .). This issue has been resolved by Alexej. 
Currently NMatrix-JRuby has not been optimized. It is not memory efficient. Running blog_data.rb results in **OutOfMemoryError** even when 12GB of heap-size is alloted to JVM.
This is a **work in progress** now.

###**Test Report**

|Spec file|Total Test|Success|Failure|Pending|
|------------|:------------:|:-----------:|:-------------:|:-------------:|
|Deviance_spec|04|04|0|0|
|LMM_spec|195|195|0|0|
|LMM_categorical_data_spec.rb|48|45|3|0|
|LMMFormula_spec.rb|05|05|0|0|
|LMM_interaction_effects_spec.rb|82|82|0|0|
|LMM_nested_effects_spec.rb|40|40|0|0|
|matrix_methods_spec.rb|52|48|4|0|
|ModelSpecification_spec.rb|07|07|0|0|
|NelderMeadWithConstraints_spec.rb|08|08|0|0|

###**Features not Supported**

1. **Parallel Gem:** Currently Parallel gem is not supported by JRuby. So, we can't use parallel processing to utilize multiple CPU cores. /example/bootstrap.rb can't be run with parallelism.
2. **ArrayStoreException:** We are not exactly sure why this occurs currently. We guess it's due to a lot of memory used by arrays. We believe it can be overcome once we optimize NMatrix-JRuby. This issue was previously reported on JRuby [issues](https://github.com/jruby/jruby/issues/2615) page.
3. **Process.fork not supported:** JRuby currently doesn't support fork. So, we had to run some tests individually which failed while running the entire test file. 

##**Conclusion:**
We have successfully ported mixed_models gem to JRuby. All examples (except blog_data.rb and bootstrap.rb) produce correct results. Now we need to optimize the performance of mixed_models gem which will mostly involve optimizing NMatrix-JRuby as real data as blog_example runs out of memory. 

Next, we will implement ruby-objects as dtype and clear a few tests still remaining for NMatrix-JRuby.
