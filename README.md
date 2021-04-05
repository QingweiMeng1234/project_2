# Large Scale Data Processing: Project 2
## Jessica Fong Ng & Qingwei Meng
Project 2 report of Large Scale Data Processing class at Boston College. The code had been modify from [project 2 assignment description](https://github.com/CSCI3390/project_2).

## Project Summary
This project contains 5 functions for computing either F0 or F2: the BJKST, tidemark, tug-of-war, exact F0, and exact F2 algorithms. The testing file `2014to2017.csv` contains the parking tickets issued in New York City from 2014 to 2017. It can be obtain from [project 2 assignment description](https://github.com/CSCI3390/project_2). 

To run the code:
1. ```sbt clean package```
2. You can run any of the function by supplying the `file.csv`, `algorithm`, `inputs`, see example below:

```
// Linux
spark-submit --class project_2.main --master local[*] target/scala-2.12/project_2_2.12-1.0.jar filename.csv ToW 5 2

// Unix
spark-submit --class "project_2.main" --master "local[*]" target/scala-2.12/project_2_2.12-1.0.jar 2014to2017.csv ToW 5 2
```

## Report Finding
### 1. `Exact_F2` Implementation.
```
def exact_F2(x: RDD[String]) : Long = {
    return x.map(x => (x, 1.asInstanceOf[Long])).reduceByKey(_+_).map(a=>a._2*a._2).sum
  }
```
#### Result
Platform | Time |  Estimation 
------------|------------|------------
Local | 95 | 8567966130
GCP | 74 | 8567966130
### 2. `Tug_of_War` Implementation.
This is a serial implementation.
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
Parallelized implementation ( collabroated with Xinyu Yao and Jien Li)
```
 def Tug_of_War(x: RDD[String], width: Int, depth:Int) : Long = {
 	val h = Seq.fill(depth, width){new four_universal_Radamacher_hash_function()} 
	def param0 = (mx: Seq[Seq[Long]], s: String) => Seq.range(0,depth).map( w => Seq.range(0,width).map(d => mx(w)(d) + h(w)(d).hash(s)))
	def param1 = (mx1: Seq[Seq[Long]], mx2: Seq[Seq[Long]]) => Seq.range(0,depth).map( w => Seq.range(0,width).map(d => mx1(w)(d) + mx2(w)(d)))
	var x3 = x.aggregate(Seq.fill(depth)(Seq.fill(width)(0.toLong)))(param0, param1).map(mxs => mxs.map(sum => sum * sum)) 
	val ans = x3.map(sums => sums.reduce(_+_)/width).sortWith(_<_)(depth/2)
	return ans
}
```
#### Result
Algorithm | Platform |Width|Depth| Time |  Estimation 
---------|-----|-----|-----|------|------------
serial| Local| 10 | 3| 275 | 7109545222
serial| GCP | 10 | 3 | 276 | 6838827645
parallelize| Local | 10| 3 | 38| 8551644765 
parallelize| GCP | 10| 3 | x| x 

### 3. 
#### `BJKST` Implementation

```
class BJKSTSketch(bucket_in: Set[(String, Int)] ,  z_in: Int, bucket_size_in: Int) extends Serializable {
/* A constructor that requires initialize the bucket and the z value. The bucket size is the bucket size of the sketch. */

    var bucket: Set[(String, Int)] = bucket_in
    var z: Int = z_in

    val BJKST_bucket_size = bucket_size_in;

    def this(s: String, z_of_s: Int, bucket_size_in: Int){
      /* A constructor that allows you pass in a single string, zeroes of the string, and the bucket size to initialize the sketch */
      this(Set((s, z_of_s )) , z_of_s, bucket_size_in)
    }

    def +(that: BJKSTSketch): BJKSTSketch = {    /* Merging two sketches */
      bucket = this.bucket | that.bucket
      z = math.max(this.z, that.z)
      while(bucket.size >= this.BJKST_bucket_size){
        z += 1
        bucket = bucket.filterNot(s=> s._2 < z)
      }
      return this
    }


    def add_string(s: String, z_of_s: Int): BJKSTSketch = {   /* add a string to the sketch */
      if (z_of_s >= z) {
         var item = Set((s,z_of_s))
	bucket = bucket | item
	
      while(bucket.size >= this.BJKST_bucket_size){
		z += 1
		bucket = bucket.filterNot(s=> s._2 < z)
	}
      }
      return this
  }
}
 

  def BJKST(x: RDD[String], width: Int, trials: Int) : Double = {
       val h = Seq.fill(trials)(new hash_function(2000000000))

    def param0 = (a1: Seq[BJKSTSketch], s: String) => Seq.range(0, trials).map(i => a1(i).add_string(s, h(i).zeroes(h(i).hash(s))  )  ) //parallel computing part
    def param1 = (a1: Seq[BJKSTSketch], a2: Seq[BJKSTSketch]) => Seq.range(0,trials).map(i => (a1(i)) + (a2(i))) //merge bucket
    val x3 = x.aggregate(Seq.fill(trials)(new BJKSTSketch("dim", 0, width)))( param0, param1)
     val ans = x3.map(s => scala.math.pow(2, s.z.toDouble)*s.bucket.size.toDouble).sortWith(_ < _)( trials/2) /* Take the median of the trials */

    return ans

  }
  ``` 
#### Smallest Width Determination 
If the estimate of `BJKST` is between +/- 20% of `exact_F2` , we count it as a success. We set the failure probability equals 5 percent and `depth` equals 5. We want the smallest width that has at least 95 successes out of 100 runs. So, we modified the BJKST algorithm in `main`as following: 
  ```
    if(args(1)=="BJKST") {
      if (args.length != 5) {
        println("Usage: project_2 input_path BJKST #buckets trials number_rounds") /* number_rounds = 100 */
        sys.exit(1)
      }

      var count = 0
      val target = exact_F0(dfrdd)
      for(i <- 1 to args(4).toInt) {
        val ans = BJKST(dfrdd, args(2).toInt, args(3).toInt)
        if (ans < 1.2 * target && ans > 0.8 * target) {
          count += 1
        }
      }

      val endTimeMillis = System.currentTimeMillis()
      val durationSeconds = (endTimeMillis - startTimeMillis) / 1000
      println("BJKST Algorithm. Bucket Size:"+ args(2) + ". Trials:" + args(3) +". Time elapsed:" + durationSeconds + "s. Number of successes: "+ count)
  ```     
We used binary search in command line to estimate width. The smallest width that we can achieve 95 successes is 50.
#### Result
Platform |Width|Depth| Time |  Estimation 
---------|-----|-----|------|------------
Local| 50 |5 | 0 | 9728
GCP | 50 | 5 | 66 | 7406649
### 4. Comparing the Results
#### Exact F2 v Tug-of-War Sketch
 algorithm| time |  estimation 
------------|------------|------------
F2 | 41 | 8567966130
(serial)Tug-of-War | 276 | 6838827645
(parallel)Tug-of-War | 38 | 8551644765 
#### Exact F0 v BJKST Sketch
 algorithm| Time |  Estimation 
------------|------------|------------
F0 | 66 | 7406649
BJKST | 59 | 7340032

#### Summary
1. `BJKST` and `Tug-of-War` algorithms can estimate F0 and F2 well.
2. The running time of `BJKST` is slightly faster than estimating F0 directly, while `Tug-of-War` is much slower than estimating F2 directly if implement serially, the parallelize version has similar run time as estimate F2 directly.
  
