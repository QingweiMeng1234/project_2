# Large Scale Data Processing: Project 2
## Jessica Fong Ng & Qingwei Meng
Project 2 report of Large Scale Data Processing class at Boston College. The code had been modify from [project 2 assignment description](https://github.com/CSCI3390/project_2).

## Report Findings
We use a local test file with 10000 data.
### 1. Implement the `exact_F2` function. Run `exact_F2` locally **and** on GCP with 1 driver and 4 machines having 2 x N1 cores. Copy the results to your report.
```
def exact_F2(x: RDD[String]) : Long = {
    return x.map(x => (x, 1.asInstanceOf[Long])).reduceByKey(_+_).map(a=>a._2*a._2).sum
  }
```
Local Result: Time Elapsed: 0s. Estimate: 16904 <br />
GCP Result: Time Elapsed: 74s, Estimate: 8567966130
### 2. Implement the `Tug_of_War` function. Run `Tug_of_War` locally **and** on GCP with 1 driver and 4 machines having 2 x N1 cores. Copy the results to your report.
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
Local Result: Width: 10. Depth: 3. Time Elapsed: 2s. Estimate: 13946 <br />
GCP Result: Width: 10, Depth: 3, Time Elapsed: 276s. Estimate: 6838827645
### 3. Implement the `BJKST` function. Once you've implemented the function, determine the smallest `width` required in order to achieve an error of +/- 20% on your estimate. Run `BJKST` locally **and** on GCP with 1 driver and 4 machines having 2 x N1 cores. Copy the results to your report.
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
### 4. **(1 point)** Compare the BJKST algorithm to the exact F0 algorithm and the tug-of-war algorithm to the exact F2 algorithm. Summarize your findings.



## Report Finding
### Compute Exact F2
```
def exact_F2(x: RDD[String]) : Long = {
    return x.map(x => (x, 1.asInstanceOf[Long])).reduceByKey(_+_).map(a=>a._2*a._2).sum
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
F2 | 41 | 8567966130
Tug-of-War | 276 | 6838827645


The run time is not significantly different becasue the memory bottleneck has not reached. 
