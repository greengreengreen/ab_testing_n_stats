# Udacity AB Test Design and Analysis
## Experiment Introduction 
Udacity is planning to test a change called "Free Trial Screener" on the course enrollment. That is if a user clicks “start free trial”, there will a reminder asking how many hours that user is planning to put in the course. If the user answers less than 5 hours, Udacity would suggest enrolling the course later when the user has more time for the course as it requires more time to finish the course. Otherwise, the user can directly enroll. A free trial user will be automatically charged after 14 days of enrollment and transferred to a paid user. Note that a user can choose to unsubscribe at any time including the free trial.</br>
Udacity wants to know if this change could reduce the number of students who unsubscribe the course during a free trial without reducing the number of paid users. 

## Experiment Design
The unit of diversion of the experiment is a cookie. Here the uniqueness of a cookie is determined by day. 

### Metric Choice

The definitions and dmin(practical significance boundary) of metrics are as below.

|Name|Definition|dmin|
|-------------|-------------|-------------|
|Number of cookies | number of unique cookies to view the course overview page. | 3000 |
|Number of user-ids | number of users who enroll in the free trial. |50|
|Number of clicks | number of unique cookies to click the "Start free trial" button (which happens before the free trial screener is trigger). | 240 |
|Click-through-probability | number of unique cookies to click the "Start free trial" button divided by number of unique cookies to view the course overview page. | 0.01 |
|Gross conversion | number of user-ids to complete checkout and enroll in the free trial divided by number of unique cookies to click the "Start free trial" button. | 0.01|
|Retention | number of user-ids to remain enrolled past the 14-day boundary (and thus make at least one payment) divided by number of user-ids to complete checkout. | 0.01|
|Net conversion | number of user-ids to remain enrolled past the 14-day boundary (and thus make at least one payment) divided by the number of unique cookies to click the "Start free trial" button. | 0.0075|

Invariant metrics: Number of cookies, number of clicks, click-through-probability.</br>
Evaluation metrics: Gross conversion, retention, net conversion. </br>

The reasons why I chose those metrics are as below:

Number of cookies: Use this as an invariant metric because the metric should be invariant between experiment group and control group. <br>
Number of user-ids: This metric doesn’t help in the experiment as the unit of analysis is cookie. The number of users could be different in experiment group and control group as the enrolled user will be tracked by user-id while the others don’t.</br>
 
Number of clicks: Use this metric as an invariant metric because it should stay the same between the two groups.</br>
 
Click-through-probability: This metric can be used as an invariant metric since it’s not influenced by the screener. </br>
 
Gross conversion: This metric can be used as an evaluation metric because the numerator changes with the screener. It measures the number of frustrated students who leave the free trial and is expected to decrease after the change. </br>
 
Retention: This metric is used as an evaluation metric as it evaluates the goal that the number of students who continue past the free trial and eventually complete the course is not significantly reduced. </br>
 
Net conversion: This metric is used as an evaluation metric as it evaluates the goal that the number of students who  continue past the free trial and eventually complete the course is not significantly reduced. </br>

The three evaluation metrics quantify the goal: “reducing the number of frustrated students who left the free trial because they didn't have enough time—without significantly reducing the number of students to continue past the free trial and eventually complete the course”. </br>
Gross conversion measures the number of frustrated students and is expected to decrease if the hypothesis holds true.</br>
Retention and net conversion measure the number of who continue past the free trial and is expected to stay the same or not reducing significantly to launch the experiment. </br>

### Measuring Standard Deviation
This is to calculate the standard deviation of the evaluation metrics selected in the last section. </br>
The underlying assumption is that gross conversion, retention and net conversion all follow a binomial distribution. For example, one experiment is that for a cookie, it has p probability to enroll, which is a Bernoulli distribution and a multiple cookies make gross conversion rate a binomial distribution. </br>

The baseline values are as below. 

|Name|Value|
|-------------|:-------------:|
Unique cookies to view course overview page per day| 40000
Unique cookies to click "Start free trial" per day |    3200
Enrollments per day | 660
Click-through-probability on "Start free trial"| 0.08
Probability of enrolling, given click | 0.20625
Probability of payment, given enroll|     0.53
Probability of payment, given click    | 0.1093125

```r
# Unique cookies to view course overview page per day
views <- 40000
# Unique cookies to click "Start free trial" per day
clicks <- 3200
# Enrollments per day
enrolls <- 660
# Click-through-probability on "Start free trial"
ctp <- clicks / views
# Probability of enrolling, given click
p_click <- enrolls / clicks
# Probability of payment, given enroll
p_pays_enroll <- 0.53
# Probability of payment, given click
p_pays_click <- p_pays_enroll * p_click
N <- 5000
# Calculate based on binomial distribution assupmtion
get_bin_std <- function(prob, n) {
  return (sqrt((1-prob)*prob/n))
}
probs <- c(p_click, p_pays_enroll, p_pays_click)
Ns <- c(N * ctp, N * (660 / 40000), N * ctp)
for (i in 1:3) {
  print(get_bin_std(probs[i], Ns[i]))
}

```
```
[1] 0.0202306
[1] 0.05494901
[1] 0.01560154
```

|Metric|Std|
|-------------|-------------|
|Gross Conversion|0.0202|
|Retention|0.0549|
|Net Conversion|0.0156|


For gross conversion and net conversion, the unit of analysis is the same as the unit of diversion. Therefore, the variability of these two metrics tends to be lower and closer to the analytical estimate.</br>
For retention, the unit of analysis is user-id and hence the underlying assumption of analytic estimate that the data is independent is not valid. Hence, the analytic variability is different from empirical variability.</br>


### Sizing

#### Number of Samples vs. Power

Based on alpha = 0.05, beta = 0.2 (typeI error = 0.95, typeII error = 0.2), this is to calculate the size of the experiment. </br>

Since there are three evaluation metrics, one possible way is to use Bonferroni correction. The benefits of Bonferroni correction are: 1) Simple 2) No assumption needed 3)Conservative(always gets the lowest alpha possible). However, Bonferroni could be too conservative when tracking multiple metrics that are moving at the same time and correlated. </br>
In this case, retention, gross conversion and net conversion are very correlated and move at the same time. Bonferroni shouldn’t be used here. </br>
Use the online calculator to get the size. <http://www.evanmiller.org/ab-testing/sample-size.html> </br>
Note that this online calculator is also based on the binomial distribution assumption. 

|Metric|Size|Days(Full Traffic)|
|-------------|-------------|----------|
|Gross Conversion|685325|18|
|Retention|4741213|119|
|Net Conversion|645875|17|


#### Duration vs. Exposure
If all the cookies were involved in the experiment, then 18, 119, 119 days are needed in order to reach the size for each metric. 119 days are too long for an experiment. Also, rentention and net conversion actually both measure the number of students who continue the course after free trial. Hence, we can drop retention and use only the other two metrics.  </br>

Generally, we don’t want the whole traffic to be involved in the experiment. In this experiment, 75% of the traffic will be exposed, which is going to take 23 days to run the experiment.  </br>


## Experiment Analysis
The following analysis will be based on the data provided by Udacity. The data could be found in the directory of this repo. </br>

### Sanity Checks
Based on the data from experiment group and control group, I am going to use invariant metrics to conduct AA tests between the two groups. alpha = 0.05 </br>

Number of cookies and number of clicks are both number metrics. To conduct invariant check on these two, we are actually testing if the probability of a cookie(or click) being assigned to experiment group is equal to control group. The underlying assumption is that the number of cookies(clicks) follows a binomial distribution with p = 0.5. </br>

```r 
# Calculate confidence interval for number of cookies and clicks
Ncookie <- 345543 + 344660 
Nclick <- 28378 + 28325
stdcookie <- get_bin_std(0.5, Ncookie)
stdclick <- get_bin_std(0.5, Nclick)
print(c(0.5 - stdcookie * 1.96, 0.5 + stdcookie*1.96))
print(c(0.5 - stdclick * 1.96, 0.5 + stdclick*1.96))
```

```
[1] 0.4988204 0.5011796
[1] 0.4958845 0.5041155
```

When comparing the click-through-probability between the twp groups, the two groups qualify the three assumptions of pooled-variance t-test, which are 1) Independent simple random samples(or a randomized experiment) 2) Normally distributed population 3) Equal population variances. 
Hence, I am using pooled-variance to get the confidence interval as below. </br>

```r 
getpooledstd <- function(Xcon, Ncon, Xexp, Nexp) {
  ppool <- (Xcon + Xexp)/(Ncon + Nexp)
  se <- sqrt(ppool * (1 - ppool) * (1/Ncon + 1/Nexp))
  return(se)
}
se <- getpooledstd(28378, 345543, 28325, 344660)
#Since H0: Cexp-Ccon = 0, Ha: Cexp-Ccon != 0 therefore the center locates at 0.
print(c(-1.96 * se, 1.96 * se ))

```

```
[1] -0.001295679  0.001295679
```

Since invariant metrics are expected to stay the same between two groups, any observations that are between the confidence interval is considered pass. </br>


|Metric|Lower Bound|Upper Bound|Observed|Passes|
|------|-----------|-----------|--------|------|
|Number of cookies|0.4988|0.5012|0.5006|Yes|
|Number of clicks on "Start free trial"|0.4959|0.5041|0.5005|Yes|
|Click-through-probability on "Start free trial"|-0.0013|0.0013|0.0001|Yes|

Current data has passed three invariant metrics, sanity check passed. </br>

### Result Analysis

#### Effect Size Tests
This is to check the evaluation metrics based on the given data. 
Just like the comparison of click-through-probability, pooled-variance t-test is also used to calculate the confidence interval.</br>

|Metric|Lower Bound|Upper Bound|Statistical Significance|Practical Significance|
|-------------|-------------|----------|---------|
|Gross Conversion|-0.02912|-0.01199|Yes|Yes|
|Net Conversion|-0.01160|0.001857|No|No|

For gross conversion, since 0 and dmin(0.01) are out of the confidence interval, reject the null hypothesis(that gross conversion is the same between two groups), conclude that the experiment group conversion rate is significantly lower than control group conversion rate. </br>
For net conversion, the confidence interval includes both 0 and practical significance boundary. Hence, it’s neither statistically significant nor practically significant. We fail to reject the null hypothesis. We conclude that there's not enough evidence indicates that net conversion is different between the two groups. 


#### Sign Tests
Sign test is used to test if there's a change but how significant is the change or which direction is the change can't be tested. The underlying assumption is that the data follows a binomial distribution. 
For gross conversion rate, among the 23 days, there are 4 days that the experiment group is bigger than the control group. 
For net conversion rate, among the 23 days, there are 10 days that experiment group is bigger than the control group. 

```r 
# sign test for gross conversion rate 
binom.test(4, 23, 0.5)
```

```
    Exact binomial test

data:  4 and 23
number of successes = 4, number of trials = 23, p-value = 0.002599
alternative hypothesis: true probability of success is not equal to 0.5
95 percent confidence interval:
 0.04950765 0.38781189
sample estimates:
probability of success 
              0.173913 
```

```r
# sign test for net conversion rate 
binom.test(10, 23, 0.5) 
```

```
    Exact binomial test

data:  10 and 23
number of successes = 10, number of trials = 23, p-value = 0.6776
alternative hypothesis: true probability of success is not equal to 0.5
95 percent confidence interval:
 0.2319142 0.6550534
sample estimates:
probability of success 
             0.4347826 
```

|Metric|p-value|Statistical Significance|
|-------------|-------------|----------|
|Gross Conversion|0.0026|Yes|
|Net Conversion|0.6776|No|


#### Summary
For gross conversion, both effect size test and sign test rejected the null hypothesis. For net conversion, both effect size and sign test failed to reject the null hypothesis. </br>

### Recommendation
My recommendation would be to launch the experiment. </br>
According to the effect size test results and sign test results, we can conclude that the gross conversion rate of experiment group is less than that of the control group at a confidence level of 95%. And that there's not enough evidence indicating that the net conversion rate is different from the two groups. We reach the goal of "reducing the number of frustrated students who left the free trial because they didn't have enough time—without significantly reducing the number of students to continue past the free trial and eventually complete the course". Hence, we can launch the experiment.</br>
I will run a follow-up experiment to further justify the experiment. Instead of letting the students directly enroll when indicating five hours or more available, maybe 6 hours or more will have lower gross conversion rate without reducing too much the net conversion rate. 

## Reference
https://www.udacity.com/course/ab-testing--ud257
