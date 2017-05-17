Figuring out how MLlib works
----------------------------

## Looking at the documentation

[ml-classification-regression](https://spark.apache.org/docs/latest/ml-classification-regression.html)
(mllib-classification-regression)[https://spark.apache.org/docs/latest/mllib-classification-regression.html]

## Through Looking at the code


### [R/pkg/R/mllib_regression.R](https://github.com/apache/spark/blob/master/R/pkg/R/mllib_regression.R)
```
110:     jobj <- callJStatic("org.apache.spark.ml.r.GeneralizedLinearRegressionWrapper",
111:                            "fit", formula, data@sdf, tolower(family$family), family$link,
112:                            tol, as.integer(maxIter), as.character(weightCol), regParam)
113:        new("GeneralizedLinearRegressionModel", jobj = jobj)
```
and the GeneralizedLinearRegressionModel object is a place holder for a java/scala object (defined as such above this, on line 33)
  
*(note the argument tol = "convergence tolerance of iterations")*

### [mllib/src/main/scala/org/apache/spark/ml/r/GeneralizedLinearRegressionWrapper.scala](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/ml/r/GeneralizedLinearRegressionWrapper.scala)

Looking at the following line:

```
77:     val rFormulaModel = rFormula.fit(data)    // This does alot of the work it seems.. 
```

rFormulaModel thus becomes the result of: [mllib/src/main/scala/org/apache/spark/ml/feature/RFormula.scala](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/ml/feature/RFormula.scala)
        
```
198:         val pipelineModel = new Pipeline(uid).setStages(encoderStages.toArray).fit(dataset)
199:         copyValues(new RFormulaModel(uid, resolvedFormula, pipelineModel).setParent(this))
```

So it appears the rFormulaModel contains some stuff which uses the Pipeline approach to transform/assemble some vectors
    
Line 98 and 46 appears to retrieve the model from a pipeline object that has been fitted (.fit() has been called) ```pipeline.stages(1)``` which it then needs to label as a ```GeneralizedLinearRegressionModel``` with ```pipeline.stages(1).asInstanceOf[GeneralizedLinearRegressionModel]```.
    
### [mllib/src/main/scala/org/apache/spark/ml/Pipeline.scala](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/ml/Pipeline.scala)
    
As seen RFormula.fit() and GeneralizedLinearRegressionWrapper.fit(), the Pipeline is loaded with PipelineStage objects. These are transforms and estimators describing the steps to be taken in fitting the model.
    
When the .fit() method is called, the pipeline performs the transformations estimations (.fit()) and (.transform()). So the resulting fitted model can be retrieved later -- as per (from the wrapper.fit()):

```
val pipeline = new Pipeline()
      .setStages(Array(rFormulaModel, glr))
      .fit(data)
val glm: GeneralizedLinearRegressionModel =
      pipeline.stages(1).asInstanceOf[GeneralizedLinearRegressionModel]
      
```

Alternatively, instead of a model being directly extracted from the fit()'ed Pipeline, a resulting **PipelineModel** can be further transformed (as done in RFormulaModel.transform() which uses the already fit()'ed PipelineModel to create a DataFrame or RFormula.transformSchema() to create a scheme structure of the model.
    
### [mllib/src/main/scala/org/apache/spark/ml/regression/GeneralizedLinearRegression.scala](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/ml/regression/GeneralizedLinearRegression.scala)

Contains the definitions for Poisson, Gamma, Binomial, Gaussian and Tweedie (the exponential family?), and the linking functions.. this appears to be the behaviour for the GLM models.

Note, ```setDefault()``` comes from ```spark.ml.para.params.scala```

##### Looking at functions and imports and inheritance..
appears the breeze library is used. But this is limited to using just the breeze.stats.distributions code.. so likely just borrowing [distribution information](https://github.com/scalanlp/breeze/wiki/Quickstart#breezestatsdistributions).
    https://darrenjw.wordpress.com/2013/12/30/brief-introduction-to-scala-and-breeze-for-statistical-computing/
    [breeze quickstart](https://github.com/scalanlp/breeze/wiki/Quickstart)
    
## Some Scala Notes
* **trait:** like Java 'interface' -- a list of functions (that need to be implemented) and provides functions for other code to access through
    http://docs.scala-lang.org/tutorials/tour/traits.html
* **class:** "Classes in Scala are static templates that can be instantiated into many objects at runtime"
    http://docs.scala-lang.org/tutorials/tour/classes
    scala constructor seems 'automatic' -- you give it constructor vars and it stores them as fields
* **object keyword** creates a 'singleton' object or anonymous class with a single available instance (useful for extending classes). Becomes akin to a global static class in Java used to call static methods on. The ```fit()``` method called from in R is called on the *static* wrapper, which on completion returns a proper object from the *class* wrapper
* **Parse r expr**  [scala.util.parsing.combinator.RegexParsers](Uses http://www.scala-lang.org/api/2.10.2/#scala.util.parsing.combinator.RegexParsers)
        ParseAll uses Parser below to parse an R formula
        ```    line 196: (label ~ "~" ~ terms) ^^ { case r ~ "~" ~ t => ParsedRFormula(r, t) }```
        This isn't important, but is interesting and unobvious from reading the code

```
33: private[r] class GeneralizedLinearRegressionWrapper private (
....
44:     val isLoaded: Boolean = false) extends MLWritable {
```
    
looking into this, MLWritable is the same as below, just a common interface for reading/writing ML *instances*. These instances contain data of the spark session, so they seem pretty fundamental..! (See MLReader and MLWriter in ReadWrite.scala if needed)
        [mllib/src/main/scala/org/apache/spark/ml/util/ReadWrite.scala](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/ml/util/ReadWrite.scala)
        
        ```141:     trait MLWritable {  ```
             write, save methods
        The MLReadable trait too
        
        ```201:     trait MLReadable[T] {```
             this trait allows a common interface for reading values of type T
             read, load methods

    