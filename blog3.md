**NMatrix**
-
## **Introduction**
NMatrix has been developed as a linear algebra library. NMatrix wraps Apache Commons Maths for its core functionalities. By the end of GSoC, I have been able to implement NMatrix for double dtype and dense stype.



## **Storing n-dimensional matrices as flat arrays**

The major components of a NMatrix is its shape, elements, dtype and stype. Any matrix when initialised, the elements are stored in flat arrays. ArrayRealVector class is used to store the elements.

## **Slicing**
one-d-array
stride of a matrix: lengths; coords
[]
[]=
xslice
slice_copy
slice

## **Rank**


## **Enumerators**
pretty print

## **Two Dimensional Matrices**

Linear algebra is mostly about two-dimensional matrices. When performing calculations in a two-dimensional matrix, a flat array is converted to a two-dimensional matrix. A two-dimensional matrix is stored as a BlockRealMatix or Array2DRowRealMatrix. Each of them has their own advantages.


## **Operators**

univariate
multivariate

trigonometric
error function
log function
gamma function


**Dot and Solve**



##**Other dtypes**
We have tried implementing float dtypes using jblas FloatMatrix. We here used jblas instead of commons math as Commons Math uses Field Elements for Floats and we may have faced issues with Reflection and TypeErasure. However, we had issues with precision. Hence, we shall be using [BigReal](http://commons.apache.org/proper/commons-math/apidocs/org/apache/commons/math3/util/BigReal.html) class. BigReal wraps around BigDecimal class and is strict about precision.


##**Code Organisation and Deployment**


###**Test Report**

|Spec file|Total Test|Success|Failure|Pending|
|------------|:------------:|:-----------:|:-------------:|:-------------:|


##**Conclusion:**