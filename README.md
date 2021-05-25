**Abstract**

Anomaly detection techniques vary from supervised, batch processed
algorithms that identify anomalies in historical data to real-time
techniques that allow analysis of streaming data. In most cases, anomaly
detection algorithms utilize a single data source (i.e. one feature of a
data set) to determine the likelihood of anomalous behavior. However, in
many cases, use of more than one features can prove useful and narrow
down the cases that need attention. In this report, we highlight a new
technique of merging results from several passes of a Hierarchical
Temporal Memory (HTM) algorithm that has previously been proposed by
other researchers. Our technique uses an existing prediction algorithm
matched against thresholds of all features of the dataset to identify
individual likelihoods. A weighted average is then calculated and
matched against a lower, weighted average threshold. Weights allow
placing greater level of importance to features that are more relevant
to our use-case. It also opens the possibility to utilize AI techniques
to learn these weights automatically and further optimize the results.

# Introduction

As technology progresses, we see more and more data generating devices
all around us. Being able to understand how these devices are operating
is very important for multiple consumer, commercial and industrial
use-cases, specially in the fields of risk, security, operations,
maintenance and many more.

An anomaly is defined as an instance in time where a system that has
been operating a certain way starts to act differently, so much that it
can be regarded as unusual. Normal changes such as loading up of
additional software, low consumption due to a weekend, peaking usage
during busy workhours are not regarded as anomalies. Where a system
flags these possibly irregular but completely normal events as
anomalies, this is termed as being a false positive.

With the techniques presented in \[1\], it was felt that the algorithm
left the optimization of their algorithm to the user. It can be
difficult to try out various options until a satisfactory result is
obtained. Furthermore, it is possible that a model configured over one
dataset or certain type of events does not fair well in other scenarios.
The cited paper itself identifies multiple short comings in existing
techniques such as the lack of a correlation between errors when
comparing results from various algorithms and the use of a single metric
to evaluate anomalous behavior, whereas in real life, multivariate data
streams are more common.

# Research contributions

-   This paper provides a way to utilize multiple data streams instead
    > of just one to create a model for real-time anomaly detection.

-   Presented technique allows tweaking the model with control over
    > weights and thresholds for each feature, allowing more focused
    > reporting for specific use-cases.

-   Since the model is learning from and reporting anomalies flagged by
    > multiple data streams, the initial model parameters do not have to
    > be optimized as some level of false reporting in one data stream
    > is automatically compensated by reporting from other features.

-   Finally, the paper presents future directions on the possibility of
    > using supervised learning to arrive at industry-specific or
    > generic model optimizations.

# Anomaly detection using HTM

In the paper, the authors presented the use of HTM as an \"online\"
sequence memory algorithm, with a prediction error calculation that
helps determine likelihood of an anomaly. To understand the process, it
is important to note that HTM uses a special scheme of data
representation, known as Sparse Distributed Representation (SDR). SDR is
represented as a bit stream, where overlapping bits indicate some level
of similarity, without requiring any knowledge of the data itself.
Therefore, before any data can be passed to the algorithm, it must be
properly encoded. The authors, therefore, provide four basic encoders:
scalar (encodes data in a sequential bit sequence), random distributed
scalar (encodes data using a random bit sequence instead of a sequential
one but once the representation is derived, it is fixed), date (encodes
date into buckets using a season variable as the bucket width), and
category (encodes categories into unique, sequential bit streams). Any
of these encoders can be used with the HTM algorithm, depending on the
data type.

## Spatial Pooler

With encoded data, we have to know what normal data looks like and by
how much would something have to change for it to be regarded as an
anomaly, both in terms of change in absolute value (spatial change) and
when did the change occur and for how long (temporal change). Clustering
techniques would handle this by batch processing a dataset, finding
centroids and then categorizing new data accordingly. Alternatively, a
Gaussian threshold can be mapped and any value outside of the threshold
is marked as anomalous. However, as the paper mentions, this does not
allow for real-time / online analysis of incoming streaming data.
Therefore, spatial poolers are used that work in real time by reading
through the incoming data, learning from everything that has been read
so far and transforming on the spot to identify the \"new normal\".

The output of a spatial pooler shows connections between synapses
forming in the HTM, represented by the number of columns specified by
the user. At this stage, the synapses are connected to each other
randomly. After the spatial pooler has processed data for a few
iterations, it will be able to assign a specific column within its SDR
records to a certain category, allowing clustering of incoming data into
the relevant category. In some cases, the resultant columns may not
perfectly fit the SDRs of each category, which indicates an imperfect
fit; however, since the idea is to have multiple values represent a
single bucket, and each bucket having largely varied SDRs, the fit would
be much better in real applications.

## Temporal Memory

The algorithm needs to be able to read data and maintain a context, i.e.
store values provided at runtime instead of the source code. To do so, a
sequence memory is used to store the learning (i.e. state of synapses in
the HTM model). The temporal memory algorithm implemented reads an input
vector, stores it and also provides a prediction of next value.

## Combined algorithm

We have deployed an encoder converting all data to the SDR format. A
spatial pooler makes buckets for each of the SDR to be classified into a
temporal memory that keeps record of what values have been read. The
spatial pooler and temporal memory work together to update the synapses
generated, representing the HTM, by a small amount as new data in read.
This slight change allows large shifts to become normal and not every
major change in incoming data is recorded as an anomaly.

The prediction error is calculated using the formula
$s_{t} = 1 - \frac{\pi\left( x_{t - 1} \right)\text{.a}\left( x_{t} \right)}{|a\left( x_{t} \right)|}$,
where $|a\left( x_{t} \right)|$ is the scalar norm, i.e. the number of 1
bits in $|a\left( x_{t} \right)|$. Essentially, this is a bit-wise
comparison of the SDRs of the predicted value and the incoming value in
the data stream. This leads to every change being flagged as a potential
anomaly. While there could be a threshold, which when exceeded
represents an anomaly, the author proposed that a better technique would
be a probabilistic metric that uses a few of the previous prediction
errors to identify how anomalous the current state is.

The anomaly likelihood ($L_{t}$) is, therefore, calculated by applying a
threshold to the Gaussian tail probability (the Q-function, i.e. the
probability that the value is larger than $W$ standard deviations),
where $W$ is the count of values of the historical prediction errors
that are being used in the Q-function (i.e. mean and standard deviation
of $W$ last prediction errors are used in the Q-function). When
$L_{t} > 1 - \epsilon$, the active state is marked as an anomaly, where
$L_{t} = 1 - Q$ and $\epsilon$ is a user-defined threshold.

The following graph shows the results from running the algorithm on the
Hot Gym dataset, which shows electricity consumption:

![](media\image1.png){width="3.0in" height="2.0621358267716534in"}
![](media\image2.png){width="3.0in" height="2.0621358267716534in"}

Figure 1 - difference between input and predicted values increases the
likelihood of a timestamp being marked as anomalous

# Multivariate datasets as a source of improved anomaly detection

As described earlier, using more than just one stream of data allows us
to identify more than one feature to classify changing events as
anomalies. We propose a technique where the HTM and prediction is run
across all of the features (or a subset of them, as chosen by the user)
and individual thresholds are compared to arrive at potential,
individual anomalies in the dataset.

The following charts show the anomalies detected with this technique
using the default set of model parameters:

![](media\image3.png){width="2.6in"
height="1.759847987751531in"}![](media\image4.png){width="2.6in"
height="1.6524245406824147in"}

![](media\image5.png){width="2.6in"
height="1.7734109798775153in"}![](media\image6.png){width="2.6in"
height="1.7300765529308837in"}![](media\image7.png){width="2.6in"
height="1.759847987751531in"}![](media\image8.png){width="2.6in"
height="1.7300765529308837in"}

Figure 2 - charts showing individual anomaly events against multiple
features

The dataset represents usage of a virtual machine from among the 1,594
included in the Grid Workloads Archive (GWA) Trace (T-13) from a
datacenter of Materna \[2\]\[3\]. The dataset presents the following
datastreams:

-   Timestamp: number of milliseconds since 1970-01-01.

-   CPU cores: number of virtual CPU cores provisioned.

-   CPU capacity provisioned (CPU requested): the capacity of the CPUs
    > in terms of MHZ, it equals to number of cores x speed per core.

-   CPU usage: in terms of MHZ.

-   CPU usage: in terms of percentage

-   Memory provisioned (memory requested): the capacity of the memory of
    > the VM in terms of KB.

-   Memory usage: the memory that is actively used in terms of KB.

-   Memory usage: in terms of %

-   Disk write throughput: in terms of KB/s

-   Disk size: In terms of GB (total sum of all virtual HDDs)

-   Network received throughput: in terms of KB/s

-   Network transmitted throughput: in terms of KB/s

Of these, the following 6 features are used for experiments run in our
research:

-   timestamp Timestamp: number of milliseconds since 1970-01-01.

-   cpu CPU usage: in terms of MHZ.

-   ram Memory usage: the memory that is actively used in terms of KB.

-   read Disk read throughput: in terms of KB/s

-   write Disk write throughput: in terms of KB/s

-   download Network received throughput: in terms of KB/s

-   upload Network transmitted throughput: in terms of KB/s

Dataset is read from the provided Comma Separate Values (CSV) files and
preprocessed to the correct format, needed by the HTM algorithm. A
separate model is defined for each of the feature in the dataset, with
separate model parameters. For the purpose of our experimentation, data
is read from a file, whereas in real application, the learning process
shall be real-time using an online data stream. Once the individual
models have been trained and predictions are mapped against thresholds,
obtaining the results shown in the charts above, we proceed with the
second step, i.e. combining all of the data into a single metric.

While the goal is to use multiple data streams, reporting anomalies on
each of them separately defeats the purpose. We must consolidate data,
or at least the anomaly scores to identify a single value of whether and
when an anomaly occurs.

While combining the various features of the raw data, it is possible
that each feature has a different size of domain and, therefore, any
visualization is not comparable. Since our purpose is served by the
trends in the data rather than absolute values, data is normalized and
then scaled between 0 and 1 to allow easier comparison in trends. At the
same time, anomalies from all features are consolidated into a single
chart as shown below:

![](media\image9.png){width="3.0in" height="2.046242344706912in"}

Figure 3 - chart showing all input values and anomaly events in a single
visualization

As shown, events that had large fluctuations are marked as anomalies.
However, a fluctuation is any single feature is marked as an anomaly. It
is possible for us to model in more sensible results by taking an
average of the anomaly scores for each feature. To allow further
flexibility, we propose the use of weighted average where the
modification of weights can provide improved results for different
use-cases; for example, a higher weight to CPU and RAM can be used to
identify software related anomalies while a higher weight to network
bandwidth can be used to identify possible DDOS attacks.

The weighted average is calculated using a set of weights spread across
the features of the dataset, adding up to a sum of 1. Each anomaly score
from the input value is multiplied with the respective weight and added
to the weighted average anomaly score (WAAS). If the WAAS is higher than
$\epsilon_{w}$ (the weighted average threshold), the timestamp is marked
as being anomalous.

The algorithm for calculating $\epsilon_{w}$ is as follows:

![](media\image10.png){width="3.1381944444444443in"
height="2.3819444444444446in"}

# Results and discussion

The following graph shows the result of identifying anomalies using the
proposed approach:

![](media\image11.png){width="3.0in" height="2.046242344706912in"}

Figure 4 - chart showing the individual anomaly scores and a weighted
average (with equal weights)

Since all features combined are pointing towards a single event being
anomalous, it can be said with a higher level of confidence that this is
an actual anomaly and not a false positive. By varying the weights and
$\epsilon_{w}$, the model can be further optimized.

The addition of multiple feature calculations does not impact the speed
too much for a small number of features. From datasets researched in
popular corpus, anomaly detection has usually been for a single stream
of data; therefore, comparable results could not be drawn against other
techniques implementing a similar approach.

![](media\image12.png){width="3.1666666666666665in"
height="2.7836023622047246in"}

Figure 5 - training over 500 rows of data with 6 features takes 37.3
seconds, resulting in 12.43 ms per data point, i.e. upto 80 features can
be used for data streaming at 1 data point per second.[^1]

# Future directions

The model uses weights and thresholds to optimize the model. It may be
possible to use a supervised learning approach to automatically learn
the best weights and thresholds for a specific domain or any combination
that works in a generic use-case.

Additionally, since weights and thresholds can be modified as required,
it may be possible to identify specific types of anomalies, such as a
DDOS attack by placing higher weight on network usage and ignore other
anomalies. If implemented, having this choice would mean more focused
reporting, improving the effectiveness of the responding system.

Furthermore, experiments performed use a small number of features, it is
unclear yet whether the algorithm will remain robust and performant for
a large number of features. Further experiments can be performed to test
this.

Finally, while we have used HTM for prediction and a weighted average
for consolidation, it is possible to test out other techniques in both
positions and assess performance, accuracy and resourcefulness of each
in various scenarios.

# Conclusion

It is seen that the proposed technique allows using multiple features,
as compared to a single data stream, providing higher level of
flexibility in the model and adjusting the model to various use-cases,
as and when necessary. Since the speed of the algorithm is not affected,
it holds true to the real-time nature.

# References

\[1\] S. Ahmad, A. Lavin, S. Purdy, and Z. Agha, "Unsupervised real-time
anomaly detection for streaming data," *Neurocomputing*, vol. 262, pp.
134--147, Nov. 2017, doi: 10.1016/j.neucom.2017.04.070.\[2\]
"FederatedCloudSim \| Proceedings of the 2nd International Workshop on
CrossCloud Systems." https://dl.acm.org/doi/10.1145/2676662.2676674
(accessed Jan. 13, 2021).\[3\] A. Kohne, D. Pasternak, L. Nagel, and O.
Spinczyk, "Evaluation of SLA-based decision strategies for VM scheduling
in cloud data centers," in *Proceedings of the 3rd Workshop on
CrossCloud Infrastructures & Platforms*, New York, NY, USA, Apr. 2016,
pp. 1--5, doi: 10.1145/2904111.2904113.

[^1]: ***Specifications of device used for test:***

    **Processor:** Intel Core i7-9750H @ 4.1 GHz

    **RAM:** 32 GB

    ***Specifications of software used for the test:***

    **Operating System:** Ubuntu 20.08 running on Windows Subsystem for
    Linux on Windows 10 Home

    **Python:** 2.7
