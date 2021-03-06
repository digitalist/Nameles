<img width='340px' src='https://i.imgur.com/Y7LOWbg.png'>

# Nameles | Open Source Invalid Traffic Detection

## Background

Nameles provides an easy to deploy, scalable IVT detection and filtering solution that is proven to detect at a high level of accuracy ad fraud and other types of invalid traffic such as web scraping. 

For a high level overview you might want to check out the [website](http://namel.es)

If you have any questions or need support, try the [gitter channel](https://gitter.im/nameles-hub/Lobby)

## Getting Started 

    wget https://raw.githubusercontent.com/Nameles-Org/Nameles/master/setup
    chmod +x setup && ./setup
    
More detailed information related with setup options is provided below. 

## Table of Contents 

[Detection Capability](#method)

[Detection Method](#system)

[System Overview](#before)

1. [Before Deployment](#before)

2. [Install Nameles](#install)

    2.1. [Single Machine Deployment](#single)

    2.2. [Multi Machine Deployment](#multi)

3. [Using Nameles](#using)

## Detection Capability <a name="capability"></a>

While absolute measurement of detection capability is impossible, Nameles is the only detection solution that can be audited by indepedent parties and that is backed by several scientfic papers. 

Nameles can detect invalid traffic on:

- mobile and desktop 
- display, video, and in-app

## Detection Method <a name="method"></a>

Nameles implements a highly scalable entropy measurement using Shannon entropy of the IP addresses a given site is receiving traffic from, and then assigns a normalized score to the site based on its traffic pattern.

<img src='https://raw.githubusercontent.com/Nameles-Org/Nameles-logfile/master/CS_formula.png'>

Entropy have been used widely in finance, intelligence, and other fields where dealing with vast amounts of data and many unknowns characterize the problem. The use of Shannon entropy has been covered in hundreds of scientific papers. Some argue that Shannon received it from Alan Turing himself, and that it was the method Turing used for cracking the Nazi code.

## System Overview <a name="system"></a>

Nameles consist of two separate modules 

- the [scoring-module](https://github.com/Nameles-Org/scoring-module)
- the [data-processing-module](https://github.com/Nameles-Org/data-processing-module)

The scoring-module replies to the query messages sent by DSP with the confidence score of the domain and the category in which the domain falls, based on the statistical thresholds of outlierness. In addition, the scoring-module forwards the messages to the data-processing-module for updating the scores at the end of the day. Modules communicate using [zeromq](http://zeromq.org).

*Figure 1: An example deployment with a DSP*

<img src='https://i.imgur.com/jetJFL3.png'>

Figure 1 presents a high level representation of Nameles functional blocks. Moreover, the figure shows how Nameles could be integrated in the programmatic ad delivery chain as an auxiliary service for the DSPs. The only difference with respect to the current operation of a DSP would be that, as part of the pre-bid phase, the DSP makes a request to Nameles to provide a Confidence Score per bid request. To this end, the DSP sends a scoring request to Nameles (step 2 in Figure 3). The scoring request includes the following fields: bid request id (mapping Nameles result to the corresponding bid request), IP address of the device associated with the bid event and the domain offering the ad space. This information is included in the bid requests as defined in the openRTB protocol standard. The scoring request is delivered to two independent modules of Nameles: the Scoring module and the Filtering module.
 
### Scoring Module

The [scoring-module](https://github.com/Nameles-Org/scoring-module) runs several worker threads that pull the queries from the DSP end and push the reply messages. The workers perform a single lookup in a shared hash table for each message. Therefore, the host running the scoring-module module requires minimal memory and drive. We recommend setting a worker per CPU and running latency tests with your expected throughtput load in order to dimensionate an appropriate number of processors for the host. Note that you can run several scoring modules in your system communicating with the same data processing module.

### Data Processing Module

The data-processing-module performs precomputations with the stream of data received from the scoring module. The data is periodically serialized to a [PostgreSQL](https://www.postgresql.org) database. The scores are computed at the end of each day. The host of this module would benefit from having a high amount of RAM and a certain number of processors in order to reduce the score computation times. We recommend at least 64GB of RAM and 4 cores.

## Scalability 

In the case of a DSP a response to a given bid request has to be received by the Ad Exchange within 100 ms. Hence, the delay introduced by Nameles is limited to few ms in order to minimize the impact in the overall bidding process delay. This ensures that also in Exchange use, the strict requirements for avoiding delays on publisher websites are avoided. 

*Figure 2: Stress-testing results with Nameles using real data*

<img src='https://i.imgur.com/HkhDijN.png'>

Figure 2 the performance of Nameles once deployed. The x-axis shows the different tested scoring request rates. The left y-axis and right y-axis show the 95-percentile filtering delay and 95-percentile memory consumption for the different scoring request rates (QPS). The line in the figure represents the average of 95-percentile values across the 5 experiments whereas the lighter color area shows the max and min 95-percentile values.

## 1. Before Deployment <a name="before"></a>

### 1.1. System Requirements 

You have the option of setting up Nameles on a single machine, or 3 separate machines. For a production system, we recommend: 

#### 1.1.1. Operating System 

Nameles have been built and tested on Ubuntu / Debian systems. 

#### 1.1.2. Single Machine 

- 4 cpu cores
- 64GB of RAM 

#### 1.1.3. Multi Machine

**scoring module**

- 2 cpu cores
- 4GB of RAM

**data processing module**

- 4 cpu cores
- 64GB of RAM 

**dsp emulator module**

- 4 cpu cores
- 8GB of RAM

### 1.2. Depedencies 

Depencies will be taken care by the setup script, so you should not have to worry about anything more than running ./setup as shown in the section 2.1. and 2.2. depending on your system configuration. The main depencies are: 

- docker-ce
- psql

## 2. Install Nameles <a name="install"></a>

You can install Nameless on a single machine or a cluster of multiple machines following the instructions on section 2.1 below. There are two options:

- single configuration deployment
- multiple configuration deployment

If you install Nameles on a multiple machine docker cluster/swarm, then you have two options: a) where you let docker allocate resources per service b) where you allocate reseources yourself.

### 2.1 Installation with Setup Script <a name="single"></a>

For running Nameles on a single server on an Ubuntu or Debian system:

    # download the setup script
    wget https://raw.githubusercontent.com/Nameles-Org/Nameles/master/setup
    
    # change the permissions
    chmod +x setup
    
    # run the setup script
    ./setup


## 2.3. Test Installation

You will have to create another shell, as in the shell where you run the setup now you will have a running docker instance.

  ```bash
  psql -h 127.0.0.1 -p 5430 -U nameles
  ```

NOTE: you need to have installed the postgreSQL client as detailed in section 1.2

## 3. Using Nameles <a name="using"></a>

The [dsp-emulator module](https://github.com/Nameles-Org/dsp-emulator) can be used as an example for interfacing Nameles from your infrastructure, i.e. message formatting and zeromq port bindings. The [latency test](https://github.com/Nameles-Org/dsp-emulator/blob/master/src/dsp_latency_test.cpp) source code is implemented in C++ but a different language for which zeromq is available could be used.

### 3.1. Restarting 

#### 3.1.1. Single Configuration Install

If the machine where Nameles is running reboots or is interrupted for another reason, you can restart with: 

  ```bash
  sudo docker-compose -f ~/Nameles/nameles-docker-compose.yml up
  ```

#### 3.1.2. Multiple Configuration Install

Note that after each command you have to start a new shell, as the current shell has a container running in it. 

  ```bash
    sudo docker-compose -f ~/Nameles/data-docker-compose.yml up
    sudo docker-compose -f ~/Nameles/scoring-docker-compose.yml up
    sudo docker-compose -f ~/Nameles/emulator-docker-compose.yml up
  ```
