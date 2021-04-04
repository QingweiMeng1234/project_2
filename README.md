# Large Scale Data Processing: Project 2
## Jessica Fong Ng & Qingwei Meng
Project 2 report of Large Scale Data Processing class at Boston College. The code had been modify from [project 2 assignment description] (https://github.com/CSCI3390/project_2).

## Resilient distributed datasets in Spark
This project will familiarize you with RDD manipulations by implementing some of the sketching algorithms the course has covered thus far.  

You have been provided with the program's skeleton, which consists of 5 functions for computing either F0 or F2: the BJKST, tidemark, tug-of-war, exact F0, and exact F2 algorithms. The tidemark and exact F0 functions are given for your reference.

## Relevant data

You can find the TAR file containing `2014to2017.csv` [here](https://drive.google.com/file/d/1MtCimcVKN6JrK2sLy4GbjeS7E2a-UMA0/view?usp=sharing). Download and expand the TAR file for local processing. For processing in the cloud, refer to the steps for creating a storage bucket in [Project 1](https://github.com/CSCI3390/project_1) and upload `2014to2017.csv`.

`2014to2017.csv` contains the records of parking tickets issued in New York City from 2014 to 2017. You'll see that the data has been cleaned so that only the license plate information remains. Keep in mind that a single car can receive multiple tickets within that period and therefore appear in multiple records.  

**Hint**: while implementing the functions, it may be helpful to copy 100 records or so to a new file and use that file for faster testing.  

## Calculating and reporting your findings
You'll be submitting a report along with your code that provides commentary on the tasks below.  

1. **(3 points)** Implement the `exact_F2` function. The function accepts an RDD of strings as an input. The output should be exactly `F2 = sum(Fs^2)`, where `Fs` is the number of occurrences of plate `s` and the sum is taken over all plates. This can be achieved in one line using the `map` and `reduceByKey` methods of the RDD class. Run `exact_F2` locally **and** on GCP with 1 driver and 4 machines having 2 x N1 cores. Copy the results to your report. Terminate the program if it runs for longer than 30 minutes.
2. **(3 points)** Implement the `Tug_of_War` function. The function accepts an RDD of strings, a parameter `width`, and a parameter `depth` as inputs. It should run `width * depth` Tug-of-War sketches, group the outcomes into groups of size `width`, compute the means of each group, and then return the median of the `depth` means in approximating F2. A 4-universal hash function class `four_universal_Radamacher_hash_function`, which generates a hash function from a 4-universal family, has been provided for you. The generated function `hash(s: String)` will hash a string to 1 or -1, each with a probability of 50%. Once you've implemented the function, set `width` to 10 and `depth` to 3. Run `Tug_of_War` locally **and** on GCP with 1 driver and 4 machines having 2 x N1 cores. Copy the results to your report. Terminate the program if it runs for longer than 30 minutes. **Please note** that the algorithm won't be significantly faster than `exact_F2` since the number of different cars is not large enough for the memory to become a bottleneck. Additionally, computing `width * depth` hash values of the license plate strings requires considerable overhead. That being said, executing with `width = 1` and `depth = 1` should generally still be faster.
3. **(3 points)** Implement the `BJKST` function. The function accepts an RDD of strings, a parameter `width`, and a parameter `trials` as inputs. `width` denotes the maximum bucket size of each sketch. The function should run `trials` sketches and return the median of the estimates of the sketches. A template of the `BJKSTSketch` class is also included in the sample code. You are welcome to finish its methods and apply that class or write your own class from scratch. A 2-universal hash function class `hash_function(numBuckets_in: Long)` has also been provided and will hash a string to an integer in the range `[0, numBuckets_in - 1]`. Once you've implemented the function, determine the smallest `width` required in order to achieve an error of +/- 20% on your estimate. Keeping `width` at that value, set `depth` to 5. Run `BJKST` locally **and** on GCP with 1 driver and 4 machines having 2 x N1 cores. Copy the results to your report. Terminate the program if it runs for longer than 30 minutes.
4. **(1 point)** Compare the BJKST algorithm to the exact F0 algorithm and the tug-of-war algorithm to the exact F2 algorithm. Summarize your findings.
## Usage
1. type `sbt clean package` to build the project's .jar file.
2. Run the following command
```
// Linux
spark-submit --class project_2.main --master local[*] target/scala-2.12/project_2_2.12-1.0.jar

// Unix
spark-submit --class "project_2.main" --master "local[*]" target/scala-2.12/project_2_2.12-1.0.jar
```
There are 5 functions for computing either F0 or F2: the BJKST, tidemark, tug-of-war, exact F0, and exact F2 algorithms. The input for each are listed as below:
```
filename.csv function inputs
```

## Report Finding
### Compute Exact F2
```
def exact_F2(x: RDD[String]) : Long = {
    return x.map(x => (x, 1)).reduceByKey(_+_).map(a=>a._2*a._2).sum.round
  }
```
### Implement Tug-of-War Algorithm
```
 def Tug_of_War(x: RDD[String], width: Int, depth:Int) : Long = {
  val t_o_w_sketches = Seq.fill(width * depth)(t_o_w(x))
  val avgs = t_o_w_sketches.grouped(width).map(_.sum/width).toArray //average
  val median = avgs.sortWith(_ < _).drop(avgs.length/2).head

  return median
 }


def t_o_w(x: RDD[String]): Long = {
  var n: Long = 0
  val h: four_universal_Radamacher_hash_function = new four_universal_Radamacher_hash_function()
  n = x.map(x => h.hash(x)).reduce(_+_)
  return n*n
}
```
### Implement BJKST

### Result
#### Exact F2 v Tug-of-War Sketch
 algorithm| time |  estimation 
------------|------------|------------
F2 | 41 | 4272998834
Tug-of-War | 276 | 6838827645


The run time is not significantly different becasue the memory bottleneck has not reached. 
