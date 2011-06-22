Stuff related to running RUM on AWS using [Starcluster](http://web.mit.edu/stardev/cluster/) goes here.  http://web.mit.edu/stardev/cluster/downloads.html tells how to get starcluster.

The `starcluster.cfg.tpl` file is a template for the config file for Starcluster.  Use it as your `~/.starcluster/config` file or use the `-c` option when running starcluster.

You need to make sure the paths in the RUM config file are absolute and not relative paths; the default ones use relative paths.

#Running RUM on AWS Using Starcluster#

First, you need to install [Starcluster](http://web.mit.edu/stardev/cluster/) on your local machine.  You can download it at [http://web.mit.edu/stardev/cluster/downloads.html], though for the most part it's easier to use the [PyPI installation instructions](http://web.mit.edu/stardev/cluster/docs/0.92rc1/installation.html) to obtain and install the software package.

You'll need a Starcluster configuration file, for which the `starcluster.cfg.tpl` in this directory acts as a template.  You need to edit it, and fill in the appropriate AWS keys and IDs and whatnot, and also point the `clusterkey` to an appropriate `.pem` file.  Decide how many virtual boxes you want to spin up, and how big they should be, etc.  (I had trouble spinning up more than 19 for a cluster, but maybe that was a fluke).

You don't have to change the NODE_IMAGE_ID; the default one there is fine.  But you'll need to make a volume to do the work on.  You can use starcluster to make a volume if you like, but it's likely easier to instantiate the snapshot we've made.  It contains RUMrunner, its associated programs, and all the 17 sets of indices and genomes listed on <http://cbil.upenn.edu/RUM/userguide.php>.  Log into AWS and go to the Volumes section under the EC2 tab.  Click "Create Volume" and select snapshot snap-a2b9ffcc.  It should be made at least 150GB large, more if you plan to write your results there and you're doing a lot of reads (if there will be a lot of interim files, etc).  Fill the ID of that volume in your `starcluster.cfg` file under the `\[volume RUMrum\]` section.  Be sure you create the volume in the same region that you're making the cluster.

Then you should be able to fire up starcluster:

`$ starcluster -c starconfig.cfg start RUMcluster`

This will take a while as all the instances in the cluster fire up and everything is put together.  Then you should be able to ssh to the master node of the cluster for the run:

`$ starcluster -c starconfig.cfg sshmaster RUMcluster`

In order to make things easier regarding write permissions and all, I just do everything as root on the cluster.

```
$ sudo su -
# cd /mnt/RUM/RUM
```

Edit the config file in the `lib` directory as appropriate: the filenames should be absolute paths if at all possible.  You should probably write the output back to the same volume; otherwise you'll have to make sure by other means that wherever you're writing to is accessible (with NFS) to all the cluster nodes.  

Retrieve the reads file either from s3 or your local machine and upload it to the master node, on the NFS-shared volume.  [S3cmd](http://s3tools.org/s3cmd) may be helpful with this, but I don't think it comes installed.

Then you should be able to run RUM:

```
# perl ./RUM_runner.pl lib/rum.config_platypus reads.fa /mnt/RUM/RUM/data 40 "Cluster run" --qsub
```

For the number of chunks, figure the number of nodes in the cluster times the number of cores per node (2 for a High-Memory XL node, 4 for a High-Memory 2XL node, and 8 for a High-Memory 4XL node.  You should be using High-Memory nodes in most cases; you need at least 6-7GB of memory for each chunk).  And watch it go!  When things are done, either download your results off the volume (onto S3?) or leave them there, since the volume will persist after you shut down the cluster.  Log off the master node (or just go to a command prompt on your local machine again) and shut down the cluster:

```
$ starcluster -c starcluster.cfg stop RUMcluster
```
