# Using-systemd-to-Run-Shiny-Apps

Lots of resources describe how you can host Shiny apps with Docker, Shiny Server, or via other means. But we also know Shiny apps can be launched locally. What makes your local setup different from these other options is that your local machine does not usually have a static internet protocol (IPv4) address. Without a static IPv4, it is really hard to share the app with other people because the address keeps changing unpredictably, and you might sometimes power off your machine.

Shiny uses the httpuv R package under the hood which is an HTTP and websocket server library. Could you just run Shiny directly on a remote server? This post explores this topic using systemd the system and service manager for most modern Linux distributions. All I am trying to do in this post is to make a point that the shiny R package is really self-sufficient and in the simplest case, it does not need any other layer for sharing an app.
Server setup

Spin up a virtual machine on the cloud provider of your choice. I use Ubuntu Linux 20.04 here if you are following along, your root user name might also be different (root on DigitalOcean, ubuntu on AWS, etc.). I also assume you have your ssh keypair configured for passwordless login.

Create a file called setup.sh in your current work directory on the local machine you are working on and copy-paste this into the file:

#!/bin/bash

# add CRAN to apt sources
apt-key adv --keyserver keyserver.ubuntu.com \
    --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9
printf '\ndeb https://cloud.r-project.org/bin/linux/ubuntu focal-cran40/\n' \
    | tee -a /etc/apt/sources.list
add-apt-repository -y ppa:c2d4u.team/c2d4u4.0+

# install system requirements & R
export DEBIAN_FRONTEND=noninteractive
apt-get -y update
apt-get -y upgrade
apt-get -yq install \
    software-properties-common \
    libopenblas-dev \
    libsodium-dev \
    r-base \
    r-base-dev \
    r-cran-shiny \
    r-cran-rmarkdown \
    r-cran-plotly \
    r-cran-ggplot2 \
    r-cran-jsonlite \
    r-cran-forecast

Once you know the IPv4 address, store it in the HOST environment variable and run the setup script via ssh:

export HOST="165.227.39.92"

ssh root@$HOST "bash -s" < setup.sh

After the installation, you should have R installed with some Shiny-related and commonly used packages ready to be used. Now log in with ssh root@$HOST to continue.
Add an application

You will create a shiny user group and a shiny user that will own the home directory /home/shiny. This is where you will use git to pull the COVID-19 app that was introduced in this post:

# add group
addgroup --system shiny

# add new user and create home dir
adduser --system --ingroup shiny shiny

# change dir
cd /home/shiny

# change user to shiny
sudo -u shiny -s

# pull COVID app
git clone https://github.com/analythium/covidapp-shiny.git

You can now launch the app to listen on port 80 (standard HTTP port) as:

R -e "shiny::runApp('/home/shiny/covidapp-shiny/02-shiny-app/app', port = 80, host = '0.0.0.0')"

And immediately see an error: createTcpServer: permission denied. The error message is telling you that the shiny user won't be able to use low port numbers like 80 because ports below 1024 are privileged and only root can open listening sockets on them. One option you have is to run the Shiny app on port 3838:

R -e "shiny::runApp('/home/shiny/covidapp-shiny/02-shiny-app/app', port = 3838, host = '0.0.0.0')"

This command now works fine and is able to execute the expression with runApp() on port 3838 and host address 0.0.0.0 that is a meta-address referring to all IPv4 addresses on the local machine (somewhat different from the loopback address 127.0.0.1 which is the localhost).

Visit http://$HOST:3838 in your browser. You should see the COVID-19 app. But the R process is now running in the foreground. Ctr+C to quit, this kills the process and makes the app grey out in the browser. Can you run the process in the background?
Run Shiny in the background

The nohup command is used to launch long-running scripts in the background, its name stands for "no hang up". nohup makes the command following it to ignore the hang-up (HUP) signal so it can run after the current terminal is closed.

The general usage is nohup command > out.log 2>&1 &. This will run the command in the background, redirect standard output and standard error into the log file (the 2>&1 part). nohup does not send the program to the background, it is done by the & which allows the user to continue using the current shell. Here is the full command to run the Shiny app:

nohup R -e "shiny::runApp('/home/shiny/covidapp-shiny/02-shiny-app/app', port = 3838, host = '0.0.0.0')" > /home/shiny/covidapp.log 2>&1 &

After running this command, you get back the prompt and a printout of the process identifier (PID). Use kill $PID or kill -9 $PID if you need to force kill. The following command finds the R process and kills it: kill $(ps aux | grep '[R] -e' | awk '{print $2}') (this SO post explains how it is working).

Visit http://$HOST:3838 in your browser: the sight should be the familiar Shiny app. Open up the /home/shiny/covidapp.log file to see the logs.

You might stop here, but the process can exit due to an error, or not restart after a reboot. The solution: systemd. exit the current shell for the shiny user and return to the root shell (the prompt will change from $ to #).
Configure and start a service

Create the shinyapp.service file in the /etc/systemd/system folder, shinyapp is the name of the service after the file name:

touch /etc/systemd/system/shinyapp.service

Add this as content:

[Unit]
Description=COVID-19 Shiny App
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=shiny
ExecStart=nohup R -e "shiny::runApp('/home/shiny/covidapp-shiny/02-shiny-app/app', port = 3838, host = '0.0.0.0')" > /home/shiny/covidapp.log 2>&1 &

[Install]
WantedBy=multi-user.target

What do we have here? The After directive means that the service must be started after the network is ready, Restart instructs the service to always restart on exit irrespective of exit status, ExecStart contains the command to run, User is the owner of the app. See this post for an explanation of the other directives.

Reload the systemctl daemon to update with your changes using systemctl daemon-reload. Enable (to restart after reboot) then start the service:

systemctl enable shinyapp

systemctl start shinyapp

Visit the http://$HOST:3838 address to see the app running. Check the status with systemctl status shinyapp to see something like this:

systemctl status shinyapp
● shinyapp.service - COVID-19 Shiny App
     Loaded: loaded (/etc/systemd/system/shinyapp.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2021-06-02 06:09:13 UTC; 17s ago
   Main PID: 710 (R)
      Tasks: 2 (limit: 2344)
     Memory: 167.1M
     CGroup: /system.slice/shinyapp.service
             └─710 /usr/lib/R/bin/exec/R -e shiny::runApp('/root/covidapp-shiny/02-shiny-app/app',~+~port~+~=~+~80,~+~>

Jun 02 06:09:22 ubuntu-s-1vcpu-2gb-intel-tor1-01 nohup[710]:   method            from
Jun 02 06:09:22 ubuntu-s-1vcpu-2gb-intel-tor1-01 nohup[710]:   as.zoo.data.frame zoo
Jun 02 06:09:24 ubuntu-s-1vcpu-2gb-intel-tor1-01 nohup[710]: Attaching package: ‘plotly’
Jun 02 06:09:24 ubuntu-s-1vcpu-2gb-intel-tor1-01 nohup[710]: The following object is masked from ‘package:ggplot2’:
Jun 02 06:09:24 ubuntu-s-1vcpu-2gb-intel-tor1-01 nohup[710]:     last_plot
Jun 02 06:09:24 ubuntu-s-1vcpu-2gb-intel-tor1-01 nohup[710]: The following object is masked from ‘package:stats’:
Jun 02 06:09:24 ubuntu-s-1vcpu-2gb-intel-tor1-01 nohup[710]:     filter

Use systemctl restart shinyapp to restart the service after you make changes to the Shiny app and pull a new version via git. Because the service is enabled, it will restart after rebooting the server.

One last thing that you can do is to redirect port 80 to 3838 using iptables:

iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3838

Now visit http://$HOST to see the Shiny app. Disabling port 3838 can be done with the uncomplicated firewall (ufw) tool. Don't forget to destroy your server if you don't need it any more.
Summary

The setup presented here is generally applicable to all kinds of web services. You can add multiple apps by creating multiple services for them. Managing R package versions is possible by adding multiple users and install R packages into their user libraries, just like you would do with Shiny Server. Serving systemd based apps on URL paths requires a reverse proxy like Nginx or Caddy, similarly to how we proxied ports using Docker.
