Setting up Swingbench for Oracle Autonomous Transaction Processing (ATP)
14/08/18 15:18
The following blog details how to install swingbench against ATP and use it to run load tests. The example below used the Simple Order Entry (SOE) benchmark but its possible to follow the methodology laid out below to run any of the other supplied benchmarks (SH, TPC-DS etc). To do this you’ll need to run through the following steps (Step 8 is optional). 

Step 1/ Make Sure you have a SSH Public key

You are likely to already have a ssh key but it is possible that you want to create another purely for this exercise. You’ll need this key to create your application server. You can find details on how to do this here

https://git-scm.com/book/en/v2/Git-on-the-Server-Generating-Your-SSH-Public-Key

It’s the .pub file or more precisely its contents that you’ll need. The public key file is typically created in the hidden .ssh directory in your home directory. The public key will look something like this (modified)

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDfO/80wleUCYxY7Ws8c67PmqL2qRUfpdPGOduHmy9xT9HkCzjoZHHIk1Zx1VpFtQQM+RwJzArZQHXrMnvefleH20AvtbT9bo2cIIZ8446DX0hHPGaGYaJNn6mCeLi/wXW0+mJmKc2xIdasnH8Q686zmv72IZ9UzD12o+nns2FgCwfleQfyVIacjfi+dy4DB8znpb4KU5rKJi5Zl004pd1uSrRtlDKR9OGILvakyf87CnAP/T8ITSMy0HWpqc8dPHJq74S5jeQn/TxrZ6TGVA+xGLzLHN4fLCOGY20gH7w3rqNTqFuUIWuIf4OFdyZoFBQyh1GWMOaKjplUonBmeZlV

You’ll need this in step 3.

Step 2/ Create the ATP Instance

You’ll have to have gone through the process of acquiring an Oracle Cloud account but that’s beyond the scope of this walkthrough. Once you have the account and have logged into Oracle Cloud Infrastructure, click on the menu button in the top left of the screen and select “Autonomous Transaction Processing”. Then simply follow these steps.

ATP 11-08-18, 9.48.53 am


Step 3/ Create a compute resource for the application server

Whilst the ATP instance is creating we can create our application to run swingbench. For any reasonable load to be run against the application server you’ll need a minimum of two cores for larger workloads you may need a bigger application or potentially a small cluster of them. 

In this walkthrough we’ll create a small 2 core Linux Server VM.

Iaas Creation 11-08-18, 9.48.29 am

This should only take a couple of minutes. On completion we’ll need to use the public IP address of the application server we created in the previous step.

Step 4/ Log onto application server and setup the environment

In this step we’ll use ssh to log onto the application server and setup the environment to run swingbench. Ssh is natively available on MacOS and Linux. On platforms like Windows you can use Putty. You’ll need the IP address of the application server you created in the previous step.

First bring up a terminal on Linux/Mac. On Putty launch a new ssh session. The username will be “opc”
1
ssh opc@< IP Address of Appserver >

You should see something similar to 
1
2
3
4
5
6
$> ssh opc@129.146.65.101
ECDSA key fingerprint is SHA256:kNbpKWL3M1wB6PUFy2GOl+JmaTIxLQiggMzn6vl2qK1tM.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '129.146.65.101' (ECDSA) to the list of known hosts.
Enter passphrase for key '/Users/dgiles/.ssh/id_rsa':
[opc@swingbench-client ~]$



By default java isn’t installed on this VM so we’ll need to install it via yum. We’ll need to update yum first

$> sudo yum makecache fast

Then we can install java and its dependencies

$> sudo yum install java-1.8.0-openjdk-headless.x86_64

We should now make sure that java works correctly

$> java -version
openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)

We can now pull the swingbench code from the website

$> curl http://www.dominicgiles.com/swingbench/swingbench261082.zip -o swingbench.zip

and unzip it

$> unzip swingbench.zip

Step 5/ Download the credentials file

The next step is to get the credentials file for ATP. You can do this by following these steps.

Credentials 11-08-18, 4.09.07 pm

You’ll need to upload this to our application server with a command similar to 

$> scp wallet_SBATP.zip opc@129.146.65.101:

This will place our credentials file in the home directory of the application server. You don’t need to unzip it to use it.

Step 6/ Install a workload schema into the ATP instance

We can now install a schema to run our transactions against. We do this by first changing in to the swingbench bin directory

$> cd swingbench/bin

And then running the following command replacing your passwords with those that you specified during the creation of the ATP instance.
./oewizard -cf ~/wallet_SBATP.zip \
           -cs sbatp_medium \
           -ts DATA \
           -dbap <your admin password> \
           -dba admin \
           -u soe \
           -p <your soe password> \
           -async_off \
           -scale 5 \
           -hashpart \
           -create \
           -cl \
           -v
view rawoewizardcommand.zip hosted with ❤ by GitHub

A quick explanation of the parameters we are using

-cf tells oewizard the location of the credentials file
-cs is the connecting for the service of the ATP instance. It is based on the name of the instance and is of the form followed by one of the following _low, _medium,_high,_parallel
-ts is the name of the table space to install swingbench into. It is currently always “data”
-dba is the admin user, currently this is always admin
-dbap is the password you specified at the creation of the ATP instance
-u is the name you want to give to the user you are installing swingbench into (I’d recommend soe)
-p is the password for the user. It needs to follow the password complexity rules of ATP
-async_off you need to disable the wizards default behavior of using async commits. This is currently prohibited on ATP
-scale indicates the size of the schema you want to create where a scale of 1 will generate 1GB of data. The indexes will take an additional amount of space roughly half the size of the data. A scale of 10 will generate a 10GB of data and roughly of 5GB of indexes
-hashpart tells the wizard to use hash partitioning
-create tells swingbench to create the schema (-drop will delete the schema)
-cl tells swingbech to run in character mode
-v tells swingbench to output whats going on (verbose mode)

You should see the following output. A scale of 1 should take just over 5 mins to create. If you specified more CPUs for the application server of ATP instance you should see some improvements in performance, but this is unlikely to truly linear because of the nature of the code.
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
SwingBench Wizard
Author  :    Dominic Giles
Version :    2.6.0.1082
 
Running in Lights Out Mode using config file : ../wizardconfigs/oewizard.xml
Connecting to : jdbc : oracle : thin : @sbatp_medium                            
Connected                                                                 
Starting run                                                              
Starting script ../sql/soedgdrop2.sql                                     
Script completed in 0 hour(s) 0 minute(s) 2 second(s) 691 millisecond(s)  
Starting script ../sql/soedgcreatetableshash2.sql                         
Script completed in 0 hour(s) 0 minute(s) 1 second(s) 433 millisecond(s)  
Starting script ../sql/soedgviews.sql                                     
Script completed in 0 hour(s) 0 minute(s) 0 second(s) 31 millisecond(s)   
Starting script ../sql/soedgsqlset.sql                                    
Script completed in 0 hour(s) 0 minute(s) 0 second(s) 196 millisecond(s)  
Inserting data into table ADDRESSES_1124999                               
Inserting data into table ADDRESSES_2                                     
Inserting data into table ADDRESSES_375001                                
Inserting data into table ADDRESSES_750000                                
Inserting data into table CUSTOMERS_749999                                
Inserting data into table CUSTOMERS_250001                                
Inserting data into table CUSTOMERS_500000                                
Inserting data into table CUSTOMERS_2                                     
Run time 0:00:19 : Running threads (8/8) : Percentage completed : 5.36

You can then validate the schema created correctly using the following command

1
2
3
4
5
6
7
8
9
10
11
$> ./sbutil -soe -cf ~/wallet_SBATP.zip -cs sbatp_medium -u soe -p < a password for the soe user > -val
The Order Entry Schema appears to be valid.
--------------------------------------------------
|Object Type    |     Valid|   Invalid|   Missing|
--------------------------------------------------
|Table          |        10|         0|         0|
|Index          |        26|         0|         0|
|Sequence       |         5|         0|         0|
|View           |         2|         0|         0|
|Code           |         1|         0|         0|
--------------------------------------------------

You may have noticed that the stats failed to collect in the creation of the schema (known problem) so you’ll need to collect stats using the following command

$>./sbutil -soe -cf ~/wallet_SBATP.zip -cs sbatp_medium -u soe -p -stats

And see the row counts for the tables with 

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
$ ./sbutil -soe -cf ~/wallet_SBATP.zip -cs sbatp_medium -u soe -p < your soe password > -tables
Order Entry Schemas Tables
----------------------------------------------------------------------------------------------------------------------
|Table Name                  |                Rows|              Blocks|           Size|   Compressed?|  Partitioned?|
----------------------------------------------------------------------------------------------------------------------
|ORDER_ITEMS                 |           4,271,594|              64,448|        512.0MB|              |           Yes|
|ADDRESSES                   |           1,500,000|              32,192|        256.0MB|              |           Yes|
|LOGON                       |           2,382,984|              32,192|        256.0MB|              |           Yes|
|CARD_DETAILS                |           1,500,000|              32,192|        256.0MB|              |           Yes|
|ORDERS                      |           1,429,790|              32,192|        256.0MB|              |           Yes|
|CUSTOMERS                   |           1,000,000|              32,192|        256.0MB|              |           Yes|
|INVENTORIES                 |             897,672|               2,386|         19.0MB|      Disabled|            No|
|PRODUCT_DESCRIPTIONS        |               1,000|                  35|          320KB|      Disabled|            No|
|PRODUCT_INFORMATION         |               1,000|                  28|          256KB|      Disabled|            No|
|ORDERENTRY_METADATA         |                   4|                   5|           64KB|      Disabled|            No|
|WAREHOUSES                  |               1,000|                   5|           64KB|      Disabled|            No|
----------------------------------------------------------------------------------------------------------------------
                                                            Total Space           1.8GB

Step 7/ Run a workload
