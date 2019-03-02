##Load balancing & pooling with pgBouncer & HAProxy 



So, as part of developing a distributed database systems one way to acheive horizontal scalability is to redirect requests to replicas with the help of proxy. Which can also act as a load balancer, such as HAProxy. 

And pgBouncer one of well-known tools in PostgresSQL based enviroments. It is a lightweight connection pooler for PostgreSQL.


----

### Step 1. Install pgBouncer

Detailed instruction can be found here https://pgbouncer.github.io/install.html 

 > It is pretty common to face this errors `make[1]: pandoc: Command not found when executing the make command` on the installation steps. This is because you didn't Pandoc installed. So you can just download here and  [install](https://github.com/jgm/pandoc/releases/tag/2.6). 

Then configure the `pgbouncer.ini`. Under the databases 

[databases]

`<custom_db_name> = host=<host_which_haproxy_server_is_running> dbname=<the_database_name_PG> user=<the_PG_database_user>`


Under [pgbouncer]

Configuration of pgbouncer for later connection. You can change the auth_type to "md5" then make sure you has your password the put the hash in the users.txt file right after the username.
        
    >$ pg_md5 the_plain_password
    974704840f331ffdcef635c72b494b72

`i.e "<db_user_name>" "<the_password_hash>"`


**Launch pgBouncer**

    $pgbouncer -d pgbouncer.ini

or if pgBouncer is already running run the above command with -R tag which basically means restart.

    $pgbouncer -R -d pgbouncer.ini 

> NB. Whenever you edit the pgBouncer.ini file make sure you restart your pgBouncer service with the new configuration. 




### Step 2. Install HAProxy 

Download and install from [here](https://haproxy.debian.net/).

Then configure the HAProxy to listen to specific port and map to respective replica servers. 

    listen pgsql_pool_bereket 
	    	bind *:5003
            mode tcp
            option pgsql-check user ha
            balance roundrobin
            server master 127.0.0.1:5432 check backup   # here you add your replica servers
            server master2 34.73.182.87:5432 check


Then copy the `haproxy.cfg` file to `/etc/haproxy/` directory.

Restart the haproxy service.

    $ systemctl restart haproxy.service

Now launch connect to pgBouncer

    //psql -p <pgBouncerPort> -U <dbUser>  <database_name_given_in_pgBouncer.ini_file> -h <host_pgBouncer>
    
    $ psql -p 6543 -U postgres  mus_me -h 127.0.0.1 



----

### Step 3 Let's Test

1.run a select query  
2.Turn down one of the server  
3.run again a select query    `# it shows you one of the server are down but it is using another server for that query.` 


## Step 4  GUI statistics report for all connected servers.

add this lines on the `haproxy.cfg` file which is located in /etc/haproxy/haproxy.cfg

    listen  stats
	    	bind   127.0.0.1:1936
            mode            http
            log             global

            maxconn 10

            clitimeout      100s
            srvtimeout      100s
            contimeout      100s
            timeout queue   100s

            stats enable
            stats hide-version
            stats refresh 30s
            stats show-node
            stats auth admin:password      # You need this password and user name to login later.
            stats uri  /haproxy?stats

Restart haproxy service 
        
    $ systemctl restart haproxy.service

Then open browser and go to http://localhost:1936/haproxy?stats enter the credential which is setted up in the listen section of haproxy.cfg which by default is 
>username:admin  
>password :password
















