---
title: "Moving EZSchool.com to Google Cloud Platform"
date: 2019-07-15T15:00:08-07:00
draft: false
tags: ["Migration", "Google Cloud Platform", "PHP", "Containers", "Modernization"]
---

A lot of the time, people like myself blog about greenfield apps, best practices, and the latest and greatest cutting edge technology. However, the reality is most real apps are never that easy or that clean.

Many people are not ready to go full-in on cloud-native hype technology, or they just don’t know where to start. The truth is, you can adopt a little at a time where it makes sense for you to do so.

In this blog post, I’ll share the story of how I moved EZSchool.com from a traditional LAMP stack running on a VPS to a containerized system running on GCE and Cloud SQL. I’ll share my thought process and the tradeoffs I had to make in a real world scenario.

**TL;DR:** EZSchool is a monolithic PHP app with a MySQL database. Ops were challenging due to lack of best practices. Moved self hosted MySQL to Google Cloud SQL to benefit from a managed solution. Dockerized the PHP and Nginx layers to simplify operations. Considered Kubernetes, but found the cost of entry too high. Considered serverless, but found the change in developer workflow too high. Settled on running containers on a single GCE VM controlled with Docker Compose and startup scripts.

**Lesson Learned:** You don’t have to use cutting edge tools and best practices to start your modernization journey. You can modernize slowly and only the pieces which make sense at the time.


# A long history lesson:

_Skip to the [Modernization](#modernization) section if you don’t want the long history lesson._

In 1998, my mom launched [EZSchool.com](https://www.ezschool.com/). EZSchool was one of the first “cloud-based” educational platforms, where teachers could manage all aspects of a classroom online. From assigning homework, give tests, and ensuring students were on the right track, the vision for EZSchool was ambitious.

Of course, being a bootstrapped startup in the early 2000s meant that the whole company ran out of two servers in the closet. One server ran Oracle DB, and the other ran Java Web Server 1.1 and has two (2!) physical CPU cores. Dual core baby. Of course, we were in the middle of [Californina’s rolling blackouts](https://en.wikipedia.org/wiki/California_electricity_crisis), so we also had a [UPS](https://en.wikipedia.org/wiki/Uninterruptible_power_supply) that could power both servers for about 30 minutes before falling over. I vividly remember the noise they would make when activated; a power cut to the database server could potentially corrupt everything so when the UPS activated it was a mad scramble to cleanly shut everything down before it lost all power, even at 2am.

When the dot-com crash happened and funding dried up, all hopes of raising money vanished overnight. It looked like the end, but my mom was determined to push forward even if it would be a one-woman operation.

![My Dad in 1999](old-servers.jpg "My Dad in 1999")

## Shared PHP Hosting

At this point, it was clear EZSchool wouldn’t have the money to get space and servers in a [colo facility](https://en.wikipedia.org/wiki/Colocation_centre), and having servers at home that went down all the time was putting massive strain on everyone. So what was the solution?

Around this time, the concept of shared PHP Hosting was becoming more and more popular. For a few dollars a month, you basically got access to a folder on a server that had Apache and CPanel installed. From here, you could FTP HTML and PHP files over to the folder, and bam you have a running website. Of course, there were tons of limitations, the biggest of which was no database support. This meant all the “cloud based” ideas like an online classroom were cut and EZSchool pivoted to a free content platform (games, worksheets, tutorials, etc) that made money from display ads. These were the days that the online advertising market was taking off, and you could make good money from modest amounts of traffic.

So let’s recap:

*   Deployments done with FTPing individual files
*   No source control
*   Shared hosting with limited functionality
*   No database
*   No local or test environment

This also meant throwing away the existing Java codebase and starting from scratch in PHP.


## The VPS Era

![Classic Web 1.0](old-ezschool.png "Classic Web 1.0")

Eventually, EZSchool got enough traffic that even the biggest shared hosting plan couldn’t support it. Remember, there was no way to isolate customers from each other in this model, it was all based on good faith. Our provider gave two choices, move to a “Virtual Private Server” or leave.

So once again, we had to migrate.

A VPS was a fully isolated Virtual Machine that was provisioned just for you. You got full root access to the machine and could do anything you wanted with it. Unlike shared PHP hosting, the resources for your VPS were pre-provisioned and dedicated for you.

Moving to a VPS had it’s pros and cons. More control means more flexibility, but it also means managing the OS updates and running Apache ourselves. Nothing too hard, especially since my mom was already familiar with Linux and bare metal servers.


## Nginx and MySQL

Eventually, the little VPS we were using started to fall over as well. The biggest problem was Apache’s habit of spinning up individual processes for each request, which caused huge amounts of overhead. I started reading about a new web server called “nginx” that claimed to be a lot more performant with a lot less overhead. So along with upgrading to a bigger VPS, we moved from Apache to Nginx.

At the end of the day, we now had more computing power than we needed, and could start to do more. At this point, display ad revenue was getting worse and worse, so we decided to let people pay for memberships that would remove advertisements from the website. Because we had a VPS with full control, my mom spun up MySQL, created a PayPal integration, and implemented a membership program.

So let’s recap:



*   Deployments done with FTPing individual files
*   No source control
*   VPS manually set up by running commands copied from blogs
*   Self managed NGINX webserver, configured by modifying nginx.conf in production
*   Self managed MySQL, manual backups
*   No local or test environment
*   “Ops” meant SSHing into the server, running commands, and crossing your fingers


## Homecoming

Now we had memberships and a fully functional database, it was time to come full circle and go back to the original vision of a “cloud based” learning platform. Of course by this time, this wasn’t a unique proposition anymore, but it still seemed like the right direction to go. I had just finished college, and I had a summer free before starting a full time job.

Over those three months, we designed a database driven platform, where all content was stored in MySQL or dynamically generated and then rendered by PHP and JavaScript. It was a big success, and over the next few years we moved everything to this model.

This let us remove hundreds of folders and thousands of static HTML and PHP files that were mostly all copies of each other and removed a bunch of manual, repetivite, and error prone work.

However, this “centralization” had some negative side effects. I remember  a nasty hours-long outage where we overwrote a critical routing component, forcing us to literally rewrite it from scratch using 'vim' on the server, because we had no backups. So we finally adopted Git, yay! Still just pushed everything to master, but it was better than nothing! Another negative was increased resource usage, with more and more functions going through the database and PHP routers.

A huge issue was MySQL would randomly die, probably due to a spike in traffic causing memory issues. I say probably because monitoring consisted of looking at the default CPU/Memory graphs and our nginx logs and squinting really hard.

We also started running into ops issues. The “server setup” guide was a poorly updated Google Doc with commands copied from random blog posts I had found. While this initially worked, the production server had experienced config drift, and was getting more and more out of sync every day.

So let’s recap:



*   Local development support by running a VM with the same setup as the server locally.
*   Deployments by running “git pull” on the server
*   Self managed NGINX web server, configured by modifying nginx.conf in production
*   Self managed MySQL, manual backups, that sometimes randomly shuts down
*   “Ops” meant SSHing into the server, running commands, and crossing your fingers


# Modernization:

You still with me? Here is where things get interesting.

At this point, I had been working at Google Cloud for some time, and whenever I helped my mom with EZSchool it was like a blast from the ugly past. I knew we could be using a lot of new technology to simplify our lives, but my mom just didn’t have the cycles to learn and implement it.

Lucky for me, our contract with our VPS provider was coming to a close, and we were having some reliability issues with them. So, this was a good opportunity to both modernize and migrate EZSchool to GCP.

There were multiple goals for this project:



1. Remove the guesswork and manual ops toil
2. Move to a managed MySQL service
3. Upgrade to PHP 7
4. Keep costs the same or lower them

I won’t focus much on upgrading to PHP 7, except for saying that the massive improvement in performance over PHP 5 was amazing, and it was mostly backwards compatible with our PHP 5 code.


## To Docker or not to Docker

One decision I made from the start was to move everything into containers. Using containers brings a lot of benefits. Upgrading the version of Nginx or PHP is much easier and safer than using the OS package manager. They also make it easy to add the nginx config into source control, as they can be mounted into the right place in the Nginx container easily. They also make running near identical stacks in prod and dev possible. Restarting everything is just one command, and support for auto-restart makes it even easier.


## Moving MySQL

The first challenge was moving the MySQL server over to Google Cloud SQL. [Creating the instance was easy enough](https://cloud.google.com/sql/docs/mysql/create-instance), but how would weto move the data over?

One way would be to follow the [migration guide in the documentation](https://cloud.google.com/sql/docs/mysql/migrate-data#migrating-to-sql). This involves setting up our current server as an external master for Cloud SQL, waiting for all data to be replicated over, **turning off the app**, promoting Cloud SQL to master, and finally restarting the app to point to Cloud SQL.

So it seems pretty complicated, and you still have to experience downtime. The downtime would be super minimal, but it’s a lot of moving pieces. Can we trade off a little more downtime for a simpler migration?

Because we only had a few GB of data, I came up with a much simpler solution.

→ Run mysql-dump to dump the data into a .sql file

→ Compress .sql file with gzip

→ Upload compressed file to Google Cloud Storage with [gsutil](https://cloud.google.com/storage/docs/gsutil)

→ run ‘[gcloud sql import sql](https://cloud.google.com/sdk/gcloud/reference/sql/import/sql)’ command to load the data into the database

All in all, these steps took about 5 minutes to finish. Not bad! I then put these four commands into a bash script, and we were good to go.


### Cloud SQL Proxy

There are many ways to connect to Cloud SQL from your application. I would highly recommend using the [Private IP](https://cloud.google.com/sql/docs/mysql/private-ip) for better performance and security. On top of that, I would also use the [Cloud SQL Proxy](https://cloud.google.com/sql/docs/mysql/connect-admin-proxy) to set up the connection.

I like the proxy for a few reasons:



1. Emulates a connection on ‘localhost’ so it’s easier to keep Dev/Prod aligned
2. Does all the SSL management for you, so you get better security for free
3. If you are using Public IPs, the proxy lets you connect without whitelisting IPs
4. No need to hardcode IP addresses in config files or things like that, it uses the instance name which is much easier to reason about

## The old architecture

To come up with the new architecture, it was important to understand the current architecture and the issues it had.


![Old VPS Setup](old-vps-setup.png "Old VPS Setup")


Pretty simple design, with some pretty big flaws.

So what were the goals for the new architecture:



1. Ensure the day-to-day dev workflow was as close to the old system as possible
2. Maximize dev/prod parity
3. Automate ops (restarting the server in case of an issue, updating the base OS, etc)

## The new architecture


### Kubernetes?

Because my day job was working with Kubernetes, the first thought I had was to move everything to Kubernetes. It would then look something like this:


![Potential Kubernetes Setup](k8s-setup.png "Potential Kubernetes Setup")


At this point I took a step back and weighed the pros and cons of this new architecture:

*   Good:
    *   Everything is declarative and stored in YAML files
    *   Everything can be checked into source control
*   Bad:
    *   A lot more moving pieces
    *   Harder to debug
*   Ugly:
    *   Cost goes from $20 for a single VPS to $50 - $60 a month (load balancer, f1-micro sql instance, 2 g1-small GKE nodes)
    *   I have to teach my mom Kubernetes.
    *   Completely different dev workflow

It really seemed that Kubernetes was out of the picture for this project. The killer feature of Kubernetes, namely managing multiple microservices at scale, was basically unnecessary. We had a small monolith that really only needed one replica. We would be paying the cost of entry that Kubernetes brings of managing a complex system with none of the benefits.


### Serverless?

My next thought was to use something serverless. These days, I would recommend [Cloud Run](https://cloud.google.com/run/), but that wasn’t an option when I was doing the migration. [Google Cloud Functions](https://cloud.google.com/functions/) doesn’t support PHP, so that was out. That left [App Engine Standard](https://cloud.google.com/appengine/docs/standard/) and [App Engine Flex.](https://cloud.google.com/appengine/docs/flexible/)

App Engine Standard seemed to be an ideal choice. It should dramatically reduce ops work and give us new features like traffic splitting and rolling updates. However, there was a fundamental problem with using App Engine, and that was Nginx. We had thousands of lines of routing rules in the form of HTTP 301 and 302 redirects. built up over years of operation, Unfortunately, App Engine requires rewriting them in a proprietary format which would be very time consuming. We also do some other interesting things in Nginx to get all the routing working the way we want, have a seperate non-public directory for dynamic question generation, and use [CIOPFS](http://www.brain-dump.org/projects/ciopfs/) to make the website case insensitive due to legacy reasons.

App Engine Flexible was out due to the high price, which you can read more about [here](https://medium.com/google-cloud/three-simple-steps-to-save-costs-when-prototyping-with-app-engine-flexible-environment-104fc6736495).

If we were doing the migration these days, [Cloud Run](https://cloud.google.com/run/) would be a great choice. Because it allows us to run arbitrary Docker containers, we have the option to run Nginx and keep all the routing rules and customizations we need. Let’s look at the pros and cons for **Cloud Run**



*   Good:
    *   Zero ops overhead
    *   Everything can be checked into source control
    *   HTTPs for free (currently using CloudFlare and LetsEncrypt)
    *   Costs should be much lower
*   Bad:
    *   Harder to debug in production / perform hotfixes
*   Ugly:
    *   Different dev workflow

Cloud Run seems like a great solution with limited downsides. However, it does require a new dev workflow, which is the biggest hurdle to adoption. It would require a new container to be built and deployed on each change and would remove the ability to SSH into the machine to make hotfixes. Now, both of these things are actually features and good things to have, but it is different and thus harder for my mom to adopt. The biggest thing is it didn’t really exist when I was doing the migration! So I’m going to ignore it for now, but might take another look in a later post.


### The Solution

So is it possible to take the positives from the Kubernetes world and use it without the downsides? We wanted to reduce the ops overhead, but didn’t need scalability. Running on a single VM was fine.

The solution I came up with was to use Docker Compose. Docker Compose gives some of the advantages of using Kubernetes when it comes to using multiple Docker Containers together on a single machine, is super simple to set up, and just requires a simple “docker compose up” to start.


![Setup I ended up with](k8s-setup.png "Setup I ended up with")


*   Good:
    *   Minimal ops overhead
    *   Everything can be checked into source control
    *   Same dev workflow
    *   Dev/Prod parity
*   Bad:
    *   No autoscaling
*   Ugly:
    *   Hacky and not following best practices

The biggest issue with this setup is that it doesn’t follow best practices. Containers are best when they are “immutable”. This would mean adding all the files into the container image when building it, and every time there is a change you should rebuild and retag the container and do a redeploy. Instead, we are just using a Docker Volume Mount to add in the appropriate Nginx config and the source code. However, I’m not trying to win any awards for the cleanest setup, my goal is to decrease ops toil and this hits the sweet spot perfectly.


## The details

Coming up with this exact architecture took a little bit of time. After looking for a combination Nginx/PHP Docker container for quite a while and only finding a bunch of half baked images, I decided to run separate containers for Nginx and PHP and then link them together with Compose.

The PHP container’s Dockerfile is as follows:


```
FROM php:7-fpm-alpine3.7
RUN apk update && docker-php-ext-install pdo pdo_mysql
```


These were the options we needed, but obviously your application will be different. Check out the [official documentation](https://docs.docker.com/samples/library/php/) for more configuration options.

For Nginx and MySQL, the official containers worked without any modification!

Now came the interesting part, using Docker Compose to maintain Dev/Prod parity. I wanted an easy way for the prod service to use Cloud SQL, but dev to use a local MySQL instance.

To do this, I used the [extension](https://docs.docker.com/compose/extends/) feature of Compose to have two different MySQL configurations, but a base nginx and php configuration.

The base configuration looked like this:


```
version: '2'

services:
   web:
       image: nginx:latest
       ports:
           - "80:80"
       volumes:
           - ./www:/home/ezschool/www
           - ./site.conf:/etc/nginx/conf.d/default.conf
           - ./nginx.conf:/etc/nginx/nginx.conf
           - ./nginx-logs:/var/log/nginx/
       networks:
           - code-network
   php:
       image: ezphp:1.0
       volumes:
           - ./www:/home/ezschool/www
           - ./php-fpm.conf:/usr/local/etc/php-fpm.d/custom-fpm.conf
           - ./custom-logs:/home/ezschool/www/Log
       networks:
           - code-network
networks:
   code-network:
       driver: bridge
```


As you can see, Compose runs two containers with a shared bridge network. This is very similar to a Pod in Kubernetes, which is exactly what I want! 

We mount in the config files using volume mounts, and make sure logs are saved outside the container. In an ideal world, we would not be doing these things. Containers work best when they are immutable, so ideally we should be running a build step that adds in all relevant files to the container and tags it with a unique ID. However, this means we need a CI/CD pipeline and more which we don’t have. This is a great topic for a future project. Similarly, logs shouldn’t be written to disk, but rather a logging service like [Stackdriver](https://cloud.google.com/logging/). However, because we only have one host, it was easy to set up [fluentd](https://cloud.google.com/logging/docs/agent/) to send logs to stackdriver.

Now for the MySQL configuration. First, we have the local config stored as a file called “docker-compose.override.yaml”


```
version: '2'

services:
    sql:
      image: mysql
      command: mysqld --user=root --verbose
      volumes:
        - ./mysql_init:/docker-entrypoint-initdb.d
      ports:
        - "3306:3306"
      environment:
        MYSQL_DATABASE: "xxxx"
        MYSQL_USER: "xxxx"
        MYSQL_PASSWORD: "xxxx"
        MYSQL_ROOT_PASSWORD: "xxxx"
        MYSQL_ALLOW_EMPTY_PASSWORD: "yes"
      networks:
          - code-network
```


This file is a pretty standard MySQL setup. The only difference is the volume mount, which let’s us pre-populate the database with a snapshot of the data from prod in the form of a .sql file.

For production, we instead use the CloudSQL Proxy. In a file called “docker-compose.prod.yaml”:


```
version: '2'

services:
   sql:
       image: gcr.io/cloudsql-docker/gce-proxy:1.11
       volumes:
           - /cloudsql:/cloudsql
       command: /cloud_sql_proxy -instances=ezschool-servers:us-central1:ezdatabase=tcp:0.0.0.0:3306
       networks:
           - code-network
```


In this case, we are using the CloudSQL Proxy docker container instead of the standard MySQL one, and use the command section to configure it to connect to the correct database. You might also notice that we are not exposing the MySQL port anymore, which makes sense. In development, you might want to connect directly to the database, but in production you can use the [gcloud cli](https://cloud.google.com/sdk/gcloud/reference/sql/connect) to connect to Cloud SQL.


### Finishing Up

Finally, I created a few scripts to automate basic tasks. I created the following Makefile:


```
restart: stop-server start-server
	@echo "Server Restarted"

start-server:
	docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

start-local-server: stop-server
	docker-compose up -d
	@echo Waiting for MySQL to start...
	@sleep 30

stop-server:
	docker-compose down

sync-db:
	gcloud sql export sql ezdatabase gs://ezschool-database-backups/current.sql
	gsutil cp gs://ezschool-database-backups/current.sql ./mysql_init/data.sql
```


Most of the commands are self-explanatory. The sync-db command is neat as it creates a new sql backup using gcloud, then copies it into the mysql_init folder using gsutil. This means you can get a fresh snapshot for local use by running “make sync-db start-local-server”

We pulled the code onto the new GCE server, and everything worked! Of course, I am glossing over the changes we had to make here and there due to slight differences in configuration and moving to PHP7, but all in all it was very minimal.

The last part was [creating a startup script](https://cloud.google.com/compute/docs/startupscript) that would automatically start the containers on boot.

Now that everything was running, it was time to migrate. Like I said before, we were willing to deal with some downtime. So we SSH’d into the old server and stopped Nginx, ran the db migration script I had previously created, and then switched DNS to point to the new server. All in all, it took just 5 minutes!


## Conclusion

At the end of the day, this was a great lesson in finding low hanging fruit. Did we follow all the best practices? Did we create the most optimized infrastructure? Do we have amazing automation and autoscaling? No, no and no. What we did do is cut our costs, increase our uptime, simplify ops, and establish dev/prod parity. I call that a job done well enough :P

Future Posts:



1. Using Cloud Build to make immutable containers and a CD pipeline
2. Moving to Cloud Run