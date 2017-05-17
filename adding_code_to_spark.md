Adding Code to Spark
====================

## General Questions
* [What is spark? How does it work?](https://www.quora.com/What-exactly-is-Apache-Spark-and-how-does-it-work)
* [Why ML and MLlib?](https://www.quora.com/Why-are-there-two-ML-implementations-in-Spark-ML-and-MLlib-and-what-are-their-different-features)

## Connection Between R and Spark

##### R-side
* [mllib.R](https://github.com/apache/spark/blob/master/R/pkg/R/mllib.R)
* [NAMESPACE](https://github.com/apache/spark/blob/master/R/pkg/NAMESPACE)
* [generics.R](https://github.com/apache/spark/blob/master/R/pkg/R/generics.R)
* [Spark](https://github.com/apache/spark/blob/master/R/pkg/R/sparkR.R)
* [mllib_utils.R](https://github.com/apache/spark/blob/master/R/pkg/R/mllib_utils.R)
* [mllib_clustering.R](https://github.com/apache/spark/blob/master/R/pkg/R/mllib_clustering.R)
* [mllib_regression.R](https://github.com/apache/spark/blob/master/R/pkg/R/mllib_regression.R)

##### Spark-side
* [RWrappers.scala](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/ml/r/RWrappers.scala)
* [RWrapperUtils.scala](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/ml/r/RWrapperUtils.scala)

### Data API

* [SQL DataFrame DataSet](http://spark.apache.org/docs/latest/sql-programming-guide.html)

## Building and Running from source

The mission here is to see the chain of execution that occurs from R to spark, to make sure that any new code created will be usable by R.

To do this, I created *just a wrapper* named "notKMeansWrapper" in scala within the ```org.apache.spark.ml.r``` package. 
This is a direct copy of KMeansWrapper, and just uses the existing KMeansModel and Summary. Essentially I copied the KMeansWrapper.scala file and replaced the string "KMeansWrapper" with "notKMeansWrapper".

Below is the process taken to build and use this change, confirming that these are the steps required to access new spark library code from R.

### SBT and Building

I ended up using maven to compile it, instead of sbt. Sbt didn't seem to create the files required for the R package to recognise.. Taken from the Spark [developer tools page](https://spark.apache.org/developer-tools.html).

```
> cd <spark's code directory>
> build/mvn clean package  # do this the first time to compile EVERYTHING

> # some tests fail on ECS system, so use -DskipTests and -Dtest=none to skip these
> build/mvn clean package -DskipTests -Dtest=none

> ... # edit the scala/spark code in mllib
> build/mvn package -DskipTests -Dtest=none -pl mllib  # compile just mllib
>   [Success!]
```

There's also SBT for debugging which returns useful error messages for scala code. ```build/sbt ~compile```

### Running SparkR

#### Changes made to R source files
Needed a couple of additional files to be changed for the notKmeansWrapper to work correctly:

* Add code changes to ```mllib.R``` 
    - copied: ```setMethod("spark.KMeans" ... )``` to ```setMethod("spark.notKMeans" ... )```
    - added case to to read.ml() function if-else chain, ```if notKMeansWrapper ... new(KMeansModel)```
* Add RD (rdocumentation) to ```mllib.R``` for the new setMethod function, otherwise the package won't compile ( [Info about R doc language (RD)](https://cran.r-project.org/web/packages/roxygen2/vignettes/rd.html) ).
* Add notkmeans entry to ```NAMESPACE```
* Add notkmeans entry to ```R/generics.R```

Complete list of changes is saved in an edited git diff output (is edited to remove unrelated changes)

#### If adding more than just a wrapper
When adding a whole new model, classes will also need to be set (```setClass```) and any other methods will need Rdoc. 
I'm not sure exactly what else would need to change in NAMESPACE and generics.R, but use the existing references to KMeansModel etc as a guide. This should be straight forward.

#### Compiling R Package
Compile R package with ```./R/install_dev.sh```, which creates the R/lib directory which can be loaded into R.
(Needs the devtools package: ```install.packages("devtools")```)

#### Test Run of (not)KmeansWrapper

Then finally, I used [the RStudio approach](http://spark.apache.org/docs/latest/sparkr.html#starting-up-from-rstudio), from within regular CLI R to start the sparkR session:

```
Sys.setenv(SPARK_HOME = "/local/scratch/anselljord/spark")
library(SparkR, lib.loc = c(file.path(Sys.getenv("SPARK_HOME"), "R", "lib")))
sparkR.session(master = "local[*]", sparkConfig = list(spark.driver.memory = "2g"))
```

There is another method which will create a package using devtools for simplifying the process of loading spark.

Then followed the example from kmeans Rdoc comment in mllib.R, with minor syntax changes to make it work in spark 2.1.0:

```
data(iris)
df <- as.DataFrame(iris)
model <- spark.notkmeans(df, Sepal_Length ~ Sepal_Width, k = 4, initMode = "random")
summary(model)
```

If the ```spark.notkmeans``` function returns a 'classNotFound' error, make sure you've built spark using maven.

### Working remotely to compile and run the code

Here I describe how to connect to the webpage for spark to see running jobs etc when working on a remote computer. 
Then i talk about the four command lines I used to show the important parts of the process (can all be done in one)

#### Port forwarding to ECS machine

When working from home pc, and running spark at uni. This allows you to visit the 

* pc: ```ssh -N -L 80:<your-spark-machine's-name>:4040 <username@proxy.machine.com>```
* ```<your-spark-machine's-name>```: start the spark server..!

omgoodness.. or just locally: ```ssh -N -L 4040:boat:4040 uni```! this treats boat as the destination server..

#### Command line steps:

* ```git pull``` (Pull changes to code made to git repo)
* ```./build/mvn package -pl mllib``` (compile the code)
* ```./R/install_dev.sh``` (compile the SparkR package)
* R connecting to spark (start and use R, running SparkR)
