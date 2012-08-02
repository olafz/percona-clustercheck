## Percona Clustercheck ##

Script to make a proxy (ie HAProxy) capable of monitoring Percona XtraDB Cluster nodes properly.

## Usage ##
Below is a sample configuration for HAProxy on the client. The point of this is that the application will be able to connect to localhost port 3307, so although we are using Percona XtraDB Cluster with several nodes, the application will see this as a single MySQL server running on localhost.

`/etc/haproxy/haproxy.cfg`

    ...
    listen percona-cluster 0.0.0.0:3307
      balance leastconn
      option httpchk
      mode tcp
        server node1 1.2.3.4:3306 check port 9200 inter 5000 fastinter 2000 rise 2 fall 2
        server node2 1.2.3.5:3306 check port 9200 inter 5000 fastinter 2000 rise 2 fall 2
        server node3 1.2.3.6:3306 check port 9200 inter 5000 fastinter 2000 rise 2 fall 2 backup

MySQL connectivity is checked via HTTP on port 9200. The clustercheck script is a simple shell script which accepts HTTP requests and checks MySQL on an incoming request. If the Percona XtraDB Cluster node is ready to accept requests, it will respond with HTTP code 200 (OK), otherwise a HTTP error 503 (Service Unavailable) is returned.

## Setup ##
This setup will create a process that listens on TCP port 9200 using xinetd. This process uses the clustercheck script from this repository to report the status of the node.

First, create a clustercheckuser that will be doing the checks.

    mysql> GRANT PROCESS ON *.* TO 'clustercheckuser'@'localhost' IDENTIFIED BY 'clustercheckpassword!'

Copy the clustercheck from the repository to a location (`/usr/bin` in the example below) and make it executable. Then add the following service to xinetd (make sure to match your location of the script with the 'server'-entry).

`/etc/xinetd.d/mysqlchk`:

    # default: on
    # description: mysqlchk
    service mysqlchk
    {
            disable = no
            flags = REUSE
            socket_type = stream
            port = 9200
            wait = no
            user = nobody
            server = /usr/bin/clustercheck
            log_on_failure += USERID
            only_from = 0.0.0.0/0
            per_source = UNLIMITED
    }

Also, you should add the mysqlchk service to `/etc/services` before restarting xinetd.

    xinetd      9098/tcp    # ...
    mysqlchk    9200/tcp    # MySQL check  <--- Add this line
    git         9418/tcp    # Git Version Control System
    zope        9673/tcp    # ...

Clustercheck will now listen on port 9200 after xinetd restart, and HAproxy is ready to check MySQL via HTTP poort 9200.