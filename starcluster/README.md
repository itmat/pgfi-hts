#Running RUM on AWS Using Starcluster#

## 1. Install Starcluster ##

You need to install [Starcluster](http://web.mit.edu/stardev/cluster/) on your local machine.  You can download it at [http://web.mit.edu/stardev/cluster/downloads.html], though for the most part it's easier to use the [PyPI installation instructions](http://web.mit.edu/stardev/cluster/docs/0.92rc1/installation.html) to obtain and install the software package.

```
$ sudo easy_install StarCluster
```

(You can also install it without root privs; then it will be placed in your ~/.local directory)

## 2. Set Up Configuration File ##

You'll need a Starcluster configuration file, for which the `starcluster.cfg.tpl` in this directory acts as a template.  You need to edit it, and fill in the appropriate AWS keys and IDs and whatnot, and also point the `clusterkey` to an appropriate `.pem` file.  Decide how many virtual boxes you want to spin up, and how big they should be, etc.  Amazon has a limit of 20 on-demand or reserved instances (Starcluster says it can handle spot instances, but that the functionality is in beta.)

## 3. Make a Work Volume ##

You can use starcluster to make a volume if you like, but it's likely easier to instantiate the snapshot we've made.  It contains RUMrunner, its associated programs, and all the 17 sets of indices and genomes listed on <http://cbil.upenn.edu/RUM/userguide.php>.  Log into AWS and go to the Volumes section under the EC2 tab.  Click "Create Volume" and select snapshot snap-a2b9ffcc.  It should be made at least 150GB large, more if you plan to write your results there and you're doing a lot of reads (if there will be a lot of interim files, etc).  Fill the ID of that volume in your `starcluster.cfg` file under the `[volume RUMrum]` section.  Be sure you create the volume in the same region that you're making the cluster.

Or if you're using the command-line tools:

```
$ ec2-create-volume --private-key pk-XXXX.pem --cert cert-XXXX.pem --region us-east-1a --snapshot snap-a2b9ffcc --size 150
```

## 4. Starting Up the Cluster ##

Then you should be able to fire up starcluster:

```
$ starcluster -c starconfig.cfg start RUMcluster
```

This will take a while as all the instances in the cluster fire up and everything is put together.  

## 5. Connect to Master Node ##

Then you should be able to ssh to the master node of the cluster for the run:

```
$ starcluster -c starconfig.cfg sshmaster RUMcluster
```

## 6. Configure RUM Run on Master Node ##

In order to make things easier regarding write permissions and all, I just do everything as root on the cluster.

```
masternode$ sudo su -
masternode# cd /opt/RUM/RUM
```

Edit the config file in the `lib` directory as appropriate: the filenames should be absolute paths if at all possible.  You should probably write the output back to the same volume; otherwise you'll have to make sure by other means that wherever you're writing to is accessible (with NFS) to all the cluster nodes.  

Retrieve the reads file either from s3 or your local machine and upload it to the master node, on the NFS-shared volume.  [S3cmd](http://s3tools.org/s3cmd) is installed on the AMI, so you should be able to retrieve stuff from s3.

### To Use SCP ###

You can copy files to/from the cluster (use its shared directory!) with scp as usual, using the key file specified in the starcluster config file to identify (`clusterkey.pem`).  To find out the names of the machines, use `starcluster -c starcluster.cfg listclusters` (on your local machine) to see the clusters currently running and the names of the machines in each one (we're only dealing with one at the moment).

### To use S3Cmd ###

While logged into the master node, run

    s3cmd --configure
    
to configure s3cmd for accessing your s3 buckets.  Then you can use

    s3cmd get s3://BUCKET/OBJECT LOCALFILE
    
to download things from s3 buckets, and you can use `s3cmd put` to upload stuff into s3, and various other s3 commands (see `s3cmd --help`).

## 7. Run RUM ##

Then you should be able to run RUM:

```
masternode# perl ./RUM_runner.pl lib/rum.config_platypus reads.fa /opt/RUM/RUM/data 40 "Cluster run" --qsub
```

For the number of chunks, figure the number of nodes in the cluster times the number of cores per node (2 for a High-Memory XL node, 4 for a High-Memory 2XL node, and 8 for a High-Memory 4XL node.  You should be using High-Memory nodes in most cases; you need at least 6-7GB of memory for each chunk).  And watch it go! 

## 8. It's Miller Time ##

When things are done, either download your results off the volume (onto S3?) or leave them there, since the volume will persist after you shut down the cluster.  Log off the master node (or just go to a command prompt on your local machine again) and shut down the cluster:

```
$ starcluster -c starcluster.cfg stop RUMcluster
```
