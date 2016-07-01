**JRuby port of Mixed-Models**
-
###**Introduction**
For my GSoC 2016 project of JRuby port of NMatrix, we worked on running other Sciruby gems on JRuby. We started with **mixed-models** gem. 
Mixed models are statistical models which predict the value of a response variable as a result of fixed and random effects. All matrix calculations are performed using the gem **nmatrix**, which has a quite intuitive syntax and contributes to the overall code readability as well.
We ran the code from this [blog](http://sciruby.com/blog/2015/08/19/gsoc-2015-mixed-models/) by Alexej Gossman, the author of 'mixed-models' gem which explains using *mixed-models* with some examples, using JRuby. We then benchmarked these results to compare the performance of Ruby-MRI versus JRuby.

**[Note]:** *Some features of LMM that are provided by Daru gem, won't be available for now .*


### **Mixed Models**
####**Example1 : Blog_data**
Challenges
1. https://github.com/agisga/mixed_models/issues/4
2. https://github.com/agisga/mixed_models/blob/master/examples/blog_data.rb#L31

As an example, we use data from the UCI machine learning repository, which originate from blog posts from various sources in 2010-2012, in order to model (the logarithm of) the number of comments that a blog post receives. The linear predictors are the text length, the log-transform of the average number of comments at the hosting website, the average number of trackbacks at the hosting website, and the parent blog posts. We assume a random effect on the number of comments due to the day of the week on which the blog post was published. In mixed_models this model can be fit with

####**Example2 : LMM**

Next I started running example [***LMM.rb***](https://github.com/agisga/mixed_models/blob/master/examples/LMM.rb). I ran the example using both ruby and jruby and compared output at every stage. Here I found these issues.

**Rank of a matrix**

One of the most important part of NMatrix was indexing. NMatrix stores multi-dimensional arrays as flat arrays and the indexing and slicing of elements is done using the shape, dimension, stride and offset of NMatrix. It would not be justice with NMatrix if I don't discuss about slicing and enumerators in NMatrix; so in my next blog I will discuss about slicing and also enumerators.
While working with LMM, there was an issue with getting the rank of a matrix. The rank couldn't be recursively accessed as the new matrix returned was not assigned dimension. This was a small bug but too tough to detect it.
  
**Cholesky/LUD Decomposition to solve a matrix when constants are a n x p matrix**

Given we need to solved a system of linear equations

                        AX = B
where A is an m×n matrix, B and X are n×p matrices, we needed to solve this equation by iterating through B.
Initially, for NMatrix-jruby we considered that X and B are column vectors. So, we got an exception 'dimension error'.
https://github.com/SciRuby/nmatrix/issues/374
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

Solution : I will have a look at jruby tests to figure how it handles dtypes.


###Benchmarks

After the first iteration, we benchmarked NMatrix JRuby versus NMatrix CRuby. 




**Autoboxing**




###Solution






###Tests

|Spec file|Total Test|Success|Failure|Pending|
|------------|:------------:|:-----------:|:-------------:|:-------------:|
|00_nmatrix_spec|188|80|102|6|
|01_enum_spec|12|4|8||
|02_slice_spec|144|20|120||
|03_nmatrix_monkeys_spec|12|4|8||
|blas_spec|12|4|8||
|elementwise_spec|38|4|34||
|math_spec|737|110|598||
|homogeneous_spec|737|110|598||
|io_spec|81|21|60||
|stat_spec|72|28|54||
||||||



###Improvements and future work



