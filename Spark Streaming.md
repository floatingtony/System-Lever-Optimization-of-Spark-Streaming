# System Lever Optimization of Spark Streaming #
## （1）	Background ##

As big data technology will be future trend, there will be huge challenge to streaming data processing technology. Nowadays, the Spark Streaming may generate out of memory error when facing unstable input rate. But this situation should be avoided in production. This article digs into the issue and put up with a method to dynamic changing streaming application parameter, which enable the system running stable with low latency.

## （2）	Streaming Feature Study ##
The streaming processing system could keep stable when input data rate is lower than batch interval. On the contrary, when input data rate processing time is more than batch interval, the system may have latency. Generally, the simple and costly way is to add hardware resource or even drop some data. Figure 1 shows the relation between processing time and batch interval against static input data rate.
As we know from the figure above, data processing time will increase against batch interval when worker number is fixed. From another side, the processing time will decrease while increasing the worker number. This figure shows add hardware enable the streaming system running stable.

 ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure1.png)

Figure 1 Workload under different worker number

Different workload cost different time. We will talk about the streaming processing character between different workload.

**1) Workload analysis**

Figure 2 shows the processing time against batch interval under map workload.
   ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure2.png)
Figure 2 map workload

Map function is to map data one by one. Figure 2 shows the processing time against batch interval under different input data rates. When increase data rate, the processing time will increase accordingly, but the increment is very little. So map workload has little impact on system latency.
Reduce method is to combine different data sources. Figure 3 shows processing time against batch interval under different input data rates. 
  ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure3.png)
Figure 3 reduce workload

Comparing to Figure 2, the input data rate has significant on reduce workload. The relation shows in Figure 3 is linear. So when increase batch interval, the processing time will increase accordingly and the gradient is smaller than one.
Join method is to combine different data sources. Figure 4 shows the streaming processing character under different input data rates.
   ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure4.png)
Figure 4 join workload

From the figure we can get the point that processing time is increasing rapidly according to batch interval. They are nonlinear relation. Assume that if the data input rate keeps increasing, the processing time will increase much faster which will cause the system unstable with high latency. That's because join function is O(M*N) (M and N is data size). 

**2) Workload character analysis**

From Figure 2 and Figure 3 we get the point that the processing time increase along with batch interval. So if the processing time is higher than batch interval, we could try to decrease batch interval in order to decrease processing time.
When processing time is lower than batch interval, the streaming system is stable and low latency.

**3) Shortcoming of static batch interval**

In production, we can know that static batch interval will have some problem if the input data rate is unsteady. Figure 5 shows ideal situation facing static batch interval and dynamic batch interval.
 
   ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure5.jpg)
Figure 5 situation in static batch interval and dynamic batch interval

Generally, it's hard to predict future input data rate. If we set batch interval parameter static and big enough, it can handler a majority of cases. However, this may have high latency whenever the input data rate is low. In order to deal with unstable input data rate situation and still keep a low system latency. We come up with a method of dynamic batch interval parameter. The batch interval will change according to input data rat or more directly processing time in Figure 5.

##（3）	Design of Dynamic Parameter


To design dynamic batch interval algorithm, first we should measure the change of system environment and through the environment rate of change to set new batch interval. We put up with a method of feedback control design shows in Figure 6.
 
 ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure6.jpg)
 
Figure 6 feedback control system
The system consists of three modules, job generating module, job processing module and batch controlling module. Job generating module takes in streaming data and generate new job into job queue per batch interval time. Job processing module gets job from job queue and computer the job, it will output data to outer storage as well as send job statistics to batch controlling module. The last module batch controlling module will compute the next batch interval value by an algorithm with the new job statistics. 
 
 ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure7.jpg)
 
Figure 7 periodically job generator

Figure 7 shows how the job generating module generate job periodically.
DStreamGraph is responsible for setting BatchDuration variable and initialize input and output streaming. RecurringTimer class is a timer which in charge of generate new job. Executor takes in data stream and generate block. Then ReceiverTracker class generator through block id.
  ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure8.png)
  
Figure 8 flow chart of algorithm

Figure 8 is a flow chart which shows how the next batch interval value is computed.

## （4）	Results of Dynamic Parameter Algorithm ##

In this section, we compare the performance of static batch interval and dynamic batch interval. Through the comparison we can know the effect of dynamic parameter method.

Through the test, the test result is listed in Table 1.

Table 1 comparision of static and dynamic batch interval 
    <table>
        <tr>
            <th>item</th>
            <th>static batch interval</th>
            <th>dynamic batch interval</th>
            <th>ratio</th>
        </tr>
        <tr>
            <th>input data rate（events/s）</th>
            <th>83894.76</th>
            <th>133929.28</th>
            <th>59.6% higher</th>
        </tr>
        <tr>
            <th>avg scheduling latency（ms）</th>
            <th>33412</th>
            <th>818</th>
            <th>39.8 times</th>
        </tr>
        <tr>
            <th>avg processing time（ms）</th>
            <th>5029</th>
            <th>5341</th>
            <th>6% higher</th>
        </tr>
		<tr>
            <th>job num in queue</th>
            <th>16</th>
            <th>1</th>
            <th>15 less</th>
        </tr>
    </table>

As we can see from table 1, dynamic batch interval parameter method has significant effect than traditional static batch interval in different aspects. 
Figure 9 will tell you the processing time against batch number in static batch interval parameter.
 
  ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure9.png)
  
Figure 9 processing time against batch number in static parameter

From the figure above, the processing time is higher than batch interval in most of the test time. So streaming system can’t complete processing job before the coming of next job, thus causing the latency of the streaming system. Figure 10 shows the dynamic parameter situation.
   
   ![Alt text](https://github.com/floatingtony/System-Lever-Optimization-of-Spark-Streaming/blob/master/Figure10.png)
   
Figure 10 processing time against batch number in dynamic parameter

When using dynamic batch interval parameter, the value of batch interval is changing. From Figure 10, we can know that the value of batch interval is changing across processing time. So job can be processed completely in batch interval time. This can lead to low system latency and high throughputs.
##（5）	Pros and Cons
From the test above, using dynamic batch interval parameter has better performance than traditional static batch interval parameter, which proved the good effect of this method. This method enable streaming system running stable, low latency and high throughputs which has large value in production. However, there still remains some problem to do in the future.

1. The parameter of dynamic self-adapt algorithm is static and set by experience. We may find other ways to get a more robust parameter for the algorithm.
2. By using dynamic batch interval parameter will lead to a unstable latency to upper application. For application which hardly rely on low latency, this method is not suitable for it.
3. The dynamic parameter method only works for limited data rate. When encountering large data rate, there is no effect for any value of batch interval, the only way is to add hardware resource and enlarge parallelism.
