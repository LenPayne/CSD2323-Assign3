# CSD-2323 Assignment 3

This set of files includes everything you need to install and configure WordPress
on a basic LAMP server. Follow the steps below to install this in a CentOS 7
system.

For other systems (eg- Ubuntu, Debian, Arch, etc...) refer to the specific
distribution's documentation for default file locations and general steps.


## Setting Up User Directories

One of the first things we want to set up is the ability for users to host their
own web pages. This is set up in the Apache `userdir.conf` file.

In a terminal, execute the command:

```bash
sudo nano /etc/httpd/conf.d/userdir.conf
```

You want to edit the file so that:

1. `UserDir disabled` is commented or deleted, and
2. `#UserDir public_html` is uncommented (remove the #).

Save and quit.

Restart the Apache Web Server to catch these changes:

```bash
sudo systemctl restart httpd
```

At this point, anyone with a properly setup `public_html` folder in their home
folder will be able to serve web content.

### Building the `public_html` Folder

You need to create the `public_html` folder for a regular user. However, security
defaults block access to all user home folders. We need to punch a hole in that
security by giving your home folder execute access. As your user, run the command:

```bash
mkdir ~/public_html
chmod a+x ~/public_html
```

At this point, you should be able to navigate to [http://localhost/~username](http://localhost/~username)
where username is your actual username, and see an empty folder index. If you put
a PHP file as `public_html/index.php` or an HTML file as `public_html/index.html`
you would see those files instead.

### Creating the `public_html` Skeleton for New Users

At this point, if you created a new user, you would have to repeat the above
steps manually for each new user. At one or two, not a big deal. But to set up a
dozen or more would be painful. What we do here is set up the defaults.

Create a `public_html` folder in the skeleton folder that all users start with:

```bash
sudo mkdir /etc/skel/public_html
```

We also need to set the default `umask` for new user folders in order to set
default permissions on the folders correctly. Edit the `login.defs` file:

```bash
sudo nano /etc/login.defs
```

Change the line that says `UMASK            077` to say:

```
UMASK                066
```

This changes the default permission mask so that all new folders created during
user creation will have permissions 711 or `rwx--x--x`. 066 is the opposite of
711, or a mask.

To test this, we'll create a new user and see if their page shows up correctly:

```bash
sudo adduser blog
sudo passwd blog
```

Then you should be able to navigate to [http://localhost/~blog](http://localhost/~blog)
and see the empty folder index.


## Accessing the Server from the Outside World

Right now, the server is humming along well at `localhost`, but we want to be
able to talk to it from any other system: that's the whole point of a server.

We need to open the firewall:

```bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```

To verify this enabled the `http` service, run `sudo firewall-cmd --list-all`.

To simplify some issues later on, we're going to temporarily disable SELinux as
well. SELinux is the Mandatory Access Control scheme for Red Hat based Linux
distros and Android. It controls who gets access to what, and will cause
problems for us later on when we install WordPress. Simply run:

```bash
sudo setenforce 0
```

This only temporarily turns off SELinux enforcement. It will be turned on next
time you reboot the server. That's okay. Assignment 4 covers SELinux config.

To access the server, we also need to know its IP Address. We do this by running:

```bash
ip addr show dev enp0s3
```

The line labelled `inet` will have the IPv4 address. At the College, this will be
something like `10.33.13.65`. It is four numbers separated by decimals. The first
one will be `10`. At home, you are more likely to have something like `192.168.0.6`.

If you have `192.168.56.1` or anything in the `192.168.56.x` space, your
VirtualBox is set to NAT instead of Bridged Adapter. It will be more difficult
to reach. Change the Network settings in VirtualBox to Bridged Adapter.

Whatever your IP address is, keep it handy. We'll use it again and again.

For future reference, I will be using `IP.AD.DR.ESS` as a placeholder for the IP.

If the firewall is open, and you have the IP address, you should be able to navigate
a web browser to `http://IP.AD.DR.ESS/~blog` to see the `blog` user's empty folder.


## Enabling Virtual Hosts

Most small websites exist on a shared server (ie- dozens of sites on a single
physical or virtual server). So how do you get a bunch of different host names
to coexist on the same server? Virtual Hosts.

At the end of this section, we'll be able to access our regular server through
[http://www.server.local](http://www.server.local) and the blog user's path at
[http://blog.server.local](http://blog.server.local).

The first thing we need to do is set up a Virtual Hosts config file:

```bash
sudo nano /etc/httpd/conf.d/virtual.conf
```

This file should not exist yet. You will be putting the following into it:

```aconf
<VirtualHost *:80>
   DocumentRoot /var/www/html
   ServerName   www.server.local
</VirtualHost>

<VirtualHost *:80>
   DocumentRoot /home/blog/public_html
   ServerName   blog.server.local   
</VirtualHost>
```

This configures both Virtual Hosts to listen on port 80 (http) and depending on
the hostname provided with the request, it will either direct to the files in
`/var/www/html`, or in `/home/blog/public_html`. At this point, to catch the new
configuration, you'll also have to restart Apache:

```bash
sudo systemctl restart httpd
```

To make these hostnames actually mean something to your computer, you need to
edit your `hosts` file. This is the first place your computer goes when it wants
to translate a name (eg- www.google.com) into an IP address.

So with your `IP.AD.DR.ESS` handy, let's edit the `hosts` file:

```bash
sudo nano /etc/hosts
```

And add the following two lines to the bottom of the file, where `IP.AD.DR.ESS`
is your actual IP address (eg- 10.33.13.65):

```
IP.AD.DR.ESS    www.server.local
IP.AD.DR.ESS    blog.server.local
```

With that in place, you should be able to use the web browser on your server to
navigate to [http://www.server.local](http://www.server.local) and
[http://blog.server.local](http://blog.server.local) without any problems. If
you see issues talking from the local machine, use the IP address 127.0.0.1.

However, you may have trouble on your host machine (Windows, macOS, or otherwise).

In general, on macOS and Linux, use `sudo` and edit `/etc/hosts`.

On Windows, open a text editor with `Run as administrator` (right-click on the
shortcut to see this option). Then edit `C:\Windows\System32\drivers\etc\hosts`.


## Installing WordPress

[WordPress](https://wordpress.org) is the single most popular blogging and web
content system currently in use today. We're going to be installing it as the
`blog` user that we created earlier.

We're going to set up things as the `blog` user, as if this user was on a shared
hosting system with little access to the administrative tools.

So first, we're going to add `blog` to the `apache` group, and then login as `blog`:

```bash
sudo usermod -aG apache blog
sudo su blog
```

At this point, you should be logged in as `blog`. Next we are going to follow
some steps to download and install WordPress:

```bash
cd ~
wget https://wordpress.org/latest.tar.gz
tar zxvf latest.tar.gz
mv wordpress/* public_html/
chgrp -R apache public_html
chmod g+w public_html
```

In order this does:

1. Navigate to the `blog` user's home directory
2. Retrieve the latest version of WordPress from the web
3. Decompress the latest version of WordPress
4. Move the decompressed files into the `public_html` folder
5. Set all the new WordPress files to be owned by the `apache` group
6. Allow the `apache` group to write in the main `public_html` folder

This will allow us to follow the dirt-simple web installation.

### Set Up a Blank WordPress Database

But first, we need to make a blank database for WordPress to put its content into.

Start off by logging into `mysql`:

```bash
mysql -u root -p
```

And execute the following SQL statements:

```sql
CREATE DATABASE `wordpress`;
CREATE USER "wordpress"@"localhost" IDENTIFIED BY "password";
GRANT ALL PRIVILEGES ON `wordpress`.* TO "wordpress"@"localhost";
```

This creates a user named `wordpress` with the password `password` and gives them
a database named after themselves. Feel free to change the username or password,
but if you do: keep a note.

### Run the Web Installer

Now that you have your database credentials, and you've set the `public_html`
folder to be writable, you can walk through the online installer.

So that WordPress gets installed correctly, make sure to perform this installation
from the `blog.server.local` hostname, and NOT the IP address.

So navigate to: [http://blog.server.local](http://blog.server.local) and follow
the steps online.

When you are done, it will ask you to log in. Do so.

Take a *screenshot* of this page in your web browser, and submit it to D2L.

![Sample Screenshot](https://github.com/LenPayne/CSD2323-Assign3/raw/master/finished-screencap.png)

Note: You may not have exactly the same results, as I took my screenshot after
installing some plugins and themes, but the most important part is having the
WordPress Dashboard up and running.

## Optional Challenges for You to Explore

1. Set up `vsftpd` or another FTP daemon to enable Plugin and Theme installs.
2. Install some Themes and Plugins from the command line with `wget`.
3. Set up some Pages and Posts. Play around with Widgets. Have fun learning WP.
