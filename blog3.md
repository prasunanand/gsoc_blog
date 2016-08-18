**NMatrix**
-
## **Introduction**
NMatrix has been developed as a linear algebra library. NMatrix wraps Apache Commons Maths for its core functionalities. By the end of GSoC, I have been able to implement NMatrix for dense matrices with double and object( ruby objects ) data type.
Currently, we haven't introduced double as a new dtype. All the real data types are supported using doubles.


## **Storing n-dimensional matrices as flat arrays**

The major components of a NMatrix is its shape, elements, dtype and stype. Any matrix when initialised, the elements are stored in flat arrays. ArrayRealVector class is used to store the elements.
@s stores the elements, @shape stores the shape of array, while @dtype and @stype store the data type and storage type respectively. currently, we have nmatrix-jruby implemented for only double matrices.
NMatrix-MRI uses @s which is an object containing elements, stride, offset as in C, we need to deal with the memory allocation for the arrays.

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

Linear algebra is mostly about two-dimensional matrices. When performing calculations in a two-dimensional matrix, a flat array is converted to a two-dimensional matrix. A two-dimensional matrix is stored as a BlockRealMatix or Array2DRowRealMatrix. Each of them has its own advantages.

Getting a two-d-matrix

```java
public class MatrixGenerator
{
  public static double[][] getMatrixDouble(double[] array, int row, int col)
    {
      double[][] matrix = new double[row][col];
      for (int index=0, i=0; i < row ; i++){
          for (int j=0; j < col; j++){
              matrix[i][j]= array[index];
              index++;
          }
      }
      return matrix;
    }
  }
```
Flat a two-d matrix
```java
public class ArrayGenerator
{
  public static double[] getArrayDouble(double[][] matrix, int row, int col)
    {
      double[] array = new double[row * col];
      for (int index=0, i=0; i < row ; i++){
          for (int j=0; j < col; j++){
              array[index] = matrix[i][j];
              index++;
          }
      }
      return array;
    }
}
```
Why use java method instead of Ruby method?
1.  Garbage Collection =>
2. Speed =>


## **Operators**

All the operators from NMatrix-MRI have been implemented except moduli. The binary operators were easily implemented through Commons Math Api.
```ruby
def +(other)
   result = create_dummy_nmatrix
   if (other.is_a?(NMatrix))
     #check dimension
     raise(ShapeError, "Cannot add matrices with different dimension")\
      if (@dim != other.dim)
     #check shape
     (0...dim).each do |i|
       raise(ShapeError, "Cannot add matrices with different shapes") \
       if (@shape[i] != other.shape[i])
     end
     result.s = @s.copy.add(other.s)
   else
     result.s = @s.copy.mapAddToSelf(other)
   end
   result
 end
```
Trigonometric, exponentiation and log operators with a singular argument i.e. matrix elements were implemented using mapToSelf method that that takes univariate function as an argument. mapToSelf maps every element of ArrayRealVector to the Univate operator passed to it and returns self object.

```ruby
def sin
  result = create_dummy_nmatrix
  result.s = @s.copy.mapToSelf(Sin.new())
  result
end
```


NMatrix#method(arg) was implemented using Bivariate functions provided by Commons-Maths and Java Maths library.

```ruby
def gamma
  result = create_dummy_nmatrix
  result.s = ArrayRealVector.new MathHelper.gamma(@s.toArray)
  result
end
```

```java
import org.apache.commons.math3.special.Gamma;

public class MathHelper{
  ...
  public static double[] gamma(double[] arr){
    double[] result = new double[arr.length];
    for(int i = 0; i< arr.length; i++){
      result[i] = Gamma.gamma(arr[i]);
    }
    return result;
  }
  ...
}
```
**Dot and Solve**



##**Other dtypes**
We have tried implementing float dtypes using jblas FloatMatrix. We here used jblas instead of commons math as Commons Math uses Field Elements for Floats and we may have faced issues with Reflection and TypeErasure. However, we had issues with precision. Hence, we shall be using [BigReal](http://commons.apache.org/proper/commons-math/apidocs/org/apache/commons/math3/util/BigReal.html) class. BigReal wraps around BigDecimal class and is strict about precision.


##**Code Organisation and Deployment**

To minimise conflict with the MRI codebase all the ruby code has been placed in /lib/nmatrix/jruby directory. /lib/nmatrix/nmatrix.rb decides whether to load nmatrix.so or nmatrix_jruby.rb after detecting the Ruby Platform.

The added advantage of this is at run-time the ruby interpreter must not decide which function to call. The impact on performance can be seen when running programs which intensively use NMatrix for linear algebraic computations(e.g. mixed-models).

###**Test Report**

|Spec file|Total Test|Success|Failure|Pending|
|------------|:------------:|:-----------:|:-------------:|:-------------:|


## **Future work**

##**Conclusion:**