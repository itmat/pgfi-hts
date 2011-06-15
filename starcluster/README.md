Stuff related to running RUM on AWS using [Starcluster](http://web.mit.edu/stardev/cluster/) goes here.  http://web.mit.edu/stardev/cluster/downloads.html tells how to get starcluster.

The `starcluster.cfg.tpl` file is a template for the config file for Starcluster.  Use it as your `~/.starcluster/config` file or use the `-c` option when running starcluster.

You need to make sure the paths in the RUM config file are absolute and not relative paths; the default ones use relative paths.

