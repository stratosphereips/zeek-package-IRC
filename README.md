# IRC-Zeek-package
Zeek Package that extracts features of IRC communication that is automatically recognized from pcap file. 

## Installation
To install the package do the following in the package directory:
```bash
$ git clone git@github.com:stratosphereips/IRC-Zeek-package.git
$ cd IRC-Zeek-package
$ zkg install .
```
## Run
To extract the IRC features on selected pcap file that contains IRC, the only thing that you need to do is to run following command in terminal:
```bash
$ zeek -r file.pcap irc_feature_extractor
```
output will be redirected `irc_features.log` file in zeek log format. 

### Example output log
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

### Parsing log output in Python

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
To be able to use IRC data, we separated the IRC communication into the connections between source IP, destination IP and destination port (hereinafter IRC connection)(see Figure below). The source port is neglected because we wanted to to merge communication that was splitted to more TCP connections. When the TCP connection is made, the source port is randomly generated from the unregistered port range and thus we needed to neglect the source port to match the IRC connection with the same source IP, destination IP and destination port. 
IRC connection consists of informations that are needed.

![IRC Connection Scheme](figs/irc-connection.png)

## Extracted Features
The feature selection is made manually to provide a good means of characterizing malicious communication. Features were computed for each IRC connection. Here is a final list of features that we used in our models.
### Total Packet Size
Total data packets' size in bytes.
### Session Duration
Duration of IRC connection in milliseconds.
### Number of Messages
Total number of messages in IRC connection.
### Number of Source Ports
Since the source port is neglected in unifying communication into sessions, the source address can use different port per TCP connection when the port is randlomly chosen. We suppose that artificial user could have higher number of source ports than the real user since the number of connections of the artificial user could be higher than the number of connections of the real user.
### Message Periodicity
To compute message periodicity, we firstly compute time differences between every message. On this computed sequence of numbers, we apply a fast Fourier transform (FFT). Fast Fourier transform is an effective algorithm for computing discrete Fourier transform, which we are using to express time sequence as a sum of periodic components and for recovering signal from those components. The output of FFT is a sequence of numbers with the same length as the input. The higher the number on a given position of the output is, the bigger the amplitude on the given position is, and thus it has a more significant influence on the periodicity of the data. The position of the largest element in the FFT's output represents the length of the period which occurrence is the most probable from all other periods.

To compute the quality of the period, we split the data by the length of a period. Then we compute the normalised mean squared error (NMSE) that returns us the resulting number in the interval between 0 and 1 where 1 represents the perfectly periodic messages, and 0 represents not periodic messages at all.

![](figs/formula_per.gif)

### Message Word Entropy
To take into account whether the user is sending the same message multiple times in a row, or whether the message contains limited number of words, we compute word entropy across all messages in IRC connection. For computation of word entropy we are using formula below,

![](figs/formula_entropy.gif)

where n represents number of words and p_i represents the probability that word $i$ is used among all other words.
### Username Special Characters Mean
Average usage of non-alphabetic characters in username.
### Message Special Characters Mean
Average usage of non-alphabetic character across all messages in IRC connection.
