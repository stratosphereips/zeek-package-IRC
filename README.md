# IRC-Zeek-package
IRC Feature Extractor Zeek Package extends the functionality of Zeek network analysis framework. This package automatically recognizes IRC communication in a packet capture (pcap) file and automatically extract features from it.
The goal for the feature extraction is to describe an individual IRC communications that occur in the pcap file as accurately as possible.

## Installation
To install the package, run the following commands in the directory where you want to install the package:
```bash
$ git clone git@github.com:stratosphereips/IRC-Zeek-package.git
$ cd IRC-Zeek-package
$ zkg install .
```
## Run
To extract the IRC features on the selected pcap file that contains IRC, run the following command in a terminal:
```bash
$ zeek -r file.pcap irc_feature_extractor
```
The output will be stored in  `irc_features.log` file in zeek log format. The log will look like this: 

```
#separator \x09
#set_separator	,
#empty_field	(empty)
#unset_field	-
#path	irc_features
#open	2020-01-27-21-54-41
#fields	src	src_ip	src_ports_count	dst	dst_ip	dst_port	start_time	end_time	duration	msg_count	size_total	periodicity	spec_chars_username_mean	spec_chars_msg_mean	msg_word_entropy
#types	string	addr	count	string	addr	port	time	time	double	count	int	double	double	double	double
T!T@null	192.168.100.103	4	#a925d765	111.230.241.23	2407	1532322898.819018	1534860900.996931	2538002.177913	23	48705	0.050294	0.25	0.908075	2.001506
T!T@null	192.168.100.103	33	#a925d765	185.61.149.22	2407	1530166710.153128	1535620500.535362	5453790.382234	231	562890	1.0	0.25	0.908256	2.276218
#close	2020-01-27-21-54-41

```

### Parsing log in Python

Instead of parsing the package manually, you can use the ZAT library  in Python to parse it directly into a Pandas data frame or as a list of dictionaries.

There is an example of how to parse the log as a list of dictionaries:


```python
import zat
from zat.log_to_dataframe import LogToDataFrame
from zat.bro_log_reader import BroLogReader


log_filename = 'irc_features.log'

logs_arr = []
reader = BroLogReader(log_filename)

for log in reader.readrows():
    # log is in dictionary format
    print(log)
    logs_arr.append(log) 
```

## Description
There were some steps to follow to extract the features. First, we separated the whole pcap into communications of individual users. To do that, we separated communication into the connections between the source IP, destination IP, and destination port (hereinafter IRC connection). The source port is randomly chosen from the unregistered port range, and that is why the source port is not the same when a new TCP connection is established between the same IP addresses. That is the reason why we needed to neglect the source port to match the IRC connection with the same source IP, destination IP, and destination port.

![alt](figs/irc-connection.png)

Example of IRC connection - IRC connection that is defined by source IP address 192.168.0.1, destination IP address 192.168.0.2, and destination port 440. Source port is neglected, and therefore one IRC connection can have multiple source ports. The IP addresses and ports are chosen randomly for demonstration purposes.

## Extracted Features
The feature selection is made manually to provide a good means of characterizing malicious communication. Features were computed for each IRC connection. Here is a final list of features that we used in our models.
### Total Packet Size
Size of all packets in bytes that were sent in IRC connection. It reflects how many messages were sent and how long they were.
### Session Duration
Duration of IRC connection in milliseconds - i.e., the difference between the time of the last message and the first message in IRC connection.
### Number of Messages
A total number of messages in IRC connection.
### Number of Source Ports
As we have mentioned before, the source port is neglected in unifying communication into IRC connections because the it is randomly chosen when a TCP connection is established. We suppose that artificial users could have had a higher number of source ports than the real users since the number of connections of the artificial users was higher than the number of connections of the real users.
### Message Periodicity
We suppose that artificial users (e.g., bots that are controlled by botnet master) use IRC for sending commands periodically, so we wanted to obtain that value. To do that, we created a method that would return a number between 0 and 1 - i.e. one if the message sequence is perfectly periodical, zero if the message sequence is not periodical at all.

To compute message periodicity, we firstly compute time differences between every message. On this computed sequence of numbers, we apply a fast Fourier transform (FFT). The output of FFT is a sequence of numbers. The higher the number on the given position of the output, the bigger the amplitude on the given position.Thus it has a more significant influence on the periodicity of the data.
The position of the largest element in the FFT's output represents the length of the period, which is the most significant from all other periods. 

To compute the quality of the most significant period, we split the data by length of that period.. Then we compute the normalised mean squared error (NMSE) that returns us the resulting number in the interval between 0 and 1 where 1 represents the perfectly periodic messages, and 0 represents not periodic messages at all.

![](figs/formula_per.gif)

### Message Word Entropy
To take into account whether the user is sending the same message multiple times in a row, or whether the message contains limited number of words, we compute word entropy across all messages in IRC connection. For computation of word entropy we are using formula below,

![](figs/formula_entropy.gif)

where n represents number of words and pi represents the probability that word i is used among all other words.
### Username Special Characters Mean
Average usage of non-alphabetic characters in username.
### Message Special Characters Mean
Average usage of non-alphabetic character across all messages in IRC connection.
