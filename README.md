Installing Nectar-OMERO on AWS, Centos 6
=========================================

The repository above is a full chef-solo cookbook repository for installing OMERO on Centos 6. The instructions for how I built this on a clean AWS Centos 6 image are below. You could either follow those instructions in order to recreate the whole thing, or you could clone this repo, navigate into the chef-repo folder and simply run chef-solo on the solo.rb and omero.json files therein, like so:

```
sudo chef-solo -c solo.rb -j omero.json
```
I would have made the fully installed AWS image available, but seemingly because I used the official Centos 6 image from the AWS marketplace, I am not allowed to redistribute it. AWS is great, but sometimes the most basic things aren't allowed. Bollocks.

For those that want to know my process in building the repo available here, the instructions are below:

1. install basic tools
======================

```
yum update -y
yum install -y vim git wget
yum groupinstall -y "Development tools"
# installing yum may require the EPEL repo. This may be installed by Chef below, so run "yum install -y Django" after

```

2. install Chef solo
====================

```
#install to home directory
cd ~
curl -L https://www.opscode.com/chef/install.sh  | bash -s -- -v 12.4.1

```

3. setup a Chef repository 
==========================
```
cd ~
wget http://github.com/opscode/chef-repo/tarball/master
tar -zxf master
mv chef-chef-repo* chef-repo
rm master
cd chef-repo
mkdir .chef
echo "cookbook_path ['/root/chef-repo/cookbooks']" > .chef/knife.rb
``` 

4. clone nectar-cookbooks/omero
===============================
```
#download to the cookbooks directory within the Chef repository
cd ~/chef-repo/cookbooks
git clone https://github.com/nectar-cookbooks/omero.git
``` 

5. Set solo.rb
==============
```
vim ~/chef-repo/solo.rb
``` 
add the following lines
```
file_cache_path “/root/chef-solo”
cookbook_path “/root/chef-repo/cookbooks”
```

6. Set ‘omero.json’ 
===================

```
vim ~/chef-repo/omero.json
```

In chef-solo, you must specify the postgresql root password so the file should look like:
```
{
  "postgresql": {
    "password": {
      "postgres": "omeropgpass"
    }
  },
  "run_list": ["recipe[omero::server]", "recipe[omero::web]"]
}

```


7. add dependency cookbooks
===========================
depends "java", ">= 1.13.0"
depends "python"
depends "postgresql"
depends "apache"
depends "nginx"

```
cd ~/chef-repo/cookbooks

knife cookbook site download java
tar zxf java*
rm -f java*.tar.gz

knife cookbook site download python
tar zxf python*
rm -f python*.tar.gz

knife cookbook site download postgresql
tar zxf postgresql*
rm -f postgresql*.tar.gz

knife cookbook site download apache
tar zxf apache*
rm -f apache*.tar.gz

knife cookbook site download nginx
tar zxf nginx*
rm -f nginx*.tar.gz

```

Also add the dependencies of these dependencies as they are asked for. First is apt:

```
cd ~/chef-repo/cookbooks
knife cookbook site download apt
tar zxf apt*
rm -f apt*.tar.gz
```

Next, bluepill

```
cd ~/chef-repo/cookbooks
knife cookbook site download bluepill
tar zxf bluepill*
rm -f bluepill*.tar.gz
```

Rsyslog:
```
cd ~/chef-repo/cookbooks
knife cookbook site download rsyslog
tar zxf rsyslog*
rm -f rsyslog*.tar.gz


```

build-essential:

```
cd ~/chef-repo/cookbooks
knife cookbook site download build-essential
tar zxf build-essential*
rm -f build-essential*.tar.gz
```

ohai:
```
cd ~/chef-repo/cookbooks
knife cookbook site download ohai
tar zxf ohai*
rm -f ohai*.tar.gz

```

runit:

```
cd ~/chef-repo/cookbooks
knife cookbook site download runit
tar zxf runit*
rm -f runit*.tar.gz

```

packagecloud:

```
cd ~/chef-repo/cookbooks
knife cookbook site download packagecloud
tar zxf packagecloud*
rm -f packagecloud*.tar.gz

```
yum-epel:

```
cd ~/chef-repo/cookbooks
knife cookbook site download yum-epel
tar zxf yum-epel*
rm -f yum-epel*.tar.gz
```

yum (due to the installation of yum-epel earlier, I'm specifying the exact files for tar and rm commands below... yum-* would get confused. You should check which version of yum installs on your system then set the file name accordingly):
```
cd ~/chef-repo/cookbooks
knife cookbook site download yum
tar zxf yum-3.6.0.tar.gz
rm -f yum-3.6.0.tar.gz
```

openssl:
```
cd ~/chef-repo/cookbooks
knife cookbook site download openssl
tar zxf openssl*
rm -f openssl*.tar.gz

```

gimme some... chef-sugar, honey!:

```
cd ~/chef-repo/cookbooks
knife cookbook site download chef-sugar
tar zxf chef-sugar*
rm -f chef-sugar*.tar.gz

```


8. Fix fedora version issue
===========================
Running at this stage produces an error, claiming that '16' does not match 'x.y.z' or 'x.y'. This appears to come from the line --supports "fedora", ">= 16" -- in the omero cookbook's metadata.rb. Try adding .0 so the line is now:
supports "fedora", ">= 16.0"



9. Fix apache to work with chef-solo 12
=============

Running at this stage produces this error: "Cookbook loaded at path(s) [/root/chef-repo/cookbooks/apache] has invalid metadata: The `name' attribute is required in cookbook metadata"

The problem is that Chef 12 requires a name attribute in the metadata.rb for each cookbook, so alter the file to look like:

```
name             "apache"
maintainer       "YOUR_COMPANY_NAME"
maintainer_email "YOUR_EMAIL"
license          "All rights reserved"
description      "various apache server related resource provides (LWRP)"
long_description IO.read(File.join(File.dirname(__FILE__), 'README.md'))
version          "0.0.5"
depends          "apache2"

%w{ gentoo ubuntu }.each do |os|
  supports os
end
```


10. Fix apache's name in Omero metadata.rb
==========================================
Omero 'depends apache' but seemingly it is actually called apache2 so alter omero's metadata.rb file:

```
depends "apache2"
```
NB: this may also require that the 'name' previously added to apache's metadata.rb ought to be 'apache2' also, but the error happened when it was 'apache' so I'm assuming that 'name' is not important for this error. 



11. Fix omero/recipes/server.rb for pip installs and dependencies (not correct in nectar file):
===============================================================================================
The nectar recipe creates a variable called 'pips' that contains a list of python packages to be pip installed. This variable has room for 'dependencies' ('deps') that are installed using the yum package manager (on Centos anyway). The Pillow package is installed first, but fails to include the 'gcc' dependency and so it throws an error unless you've manually installed 'Development tools' earlier (step 1). Thus it may be worth adding 'gcc' to the 'deps' of that first 'pip':

```
pips = [ { 'module' => 'pillow',
             'deps' => [ 'python-devel', 'libjpeg-devel', 'libtiff-devel',
                         'zlib-devel','gcc' ]
           },

```

There is also a problem with Cython which is named as a dependency for the 'tables' module, but the recommended way for installation of Cython is via Pip... so perhaps try putting that as a pip install prior to the tables pip:

```

           {
             'module' => 'cython',
             'version' => '0.17.2',
             'deps' => [],
           },

           { 'module' => 'tables',
             'version' => '2.4.0',
             'deps' => [ 'gcc', 'hdf5-devel' ]
           }

```

There is also a problem with the hd5-devel instruction too, because it's not included in the basic yum repository and they don't make a call to add a new repo prior to this. Suggestion is to attempt to add the epel repo at the same location they add the ZeroC repo (just below this pip declaration)

```

yum_repository 'epel' do
  description 'Extra Packages for Enterprise Linux'
  mirrorlist 'http://mirrors.fedoraproject.org/mirrorlist?repo=epel-6&arch=$basearch'
  gpgkey 'http://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-6'
  action :create
end

```


12. Fix etc/init.d/omero and omero-web
======================================
The omero cookbook comes with an .erb file that is used to create an init script for omero. Sadly, this doesn't work, but there is a template available at http://www.openmicroscopy.org/site/support/omero5.1/_downloads/omero-init.d

Trying to work with the cookbook system, we can replace the templates/default/omero-init.erb with the template but, we need to alter it from using environment variables (not set by the cookbook) to chef variables. Three variables are required, the base location of OMERO.server, the omero user and the /bin/omero location. The current cookbook .erb template makes use of the first two of these and the third is really just an extension of the first one, so it's fairly easy to copy from the .erb template to the new template. Replace the following code:

```
OMERO_SERVER=${OMERO_SERVER:-/home/omero/OMERO.server}
OMERO_USER=${OMERO_USER:-omero}
OMERO=${OMERO_SERVER}/bin/omero
```

With the following code:
```
OMERO_SERVER=<%= @omero_home %>
OMERO_USER=<%= @omero_user %>
OMERO=${OMERO_SERVER}/bin/omero

```
I have saved this new file in templates/default/omero-init2.erb so I'll alter the recipe file to point to it, with the altered code looking like:

```
template '/etc/init.d/omero' do
  source "omero-init2#{rc_flavour}.erb"
  mode 0755
  variables({
     :omero_user => omero_user,
     :omero_home => "#{omero_install}/OMERO.server"
  })
end

```
NB: I've only added a 2 to the end of "omero-init... in this block of code. The code is also setting the filename with 'rc_flavour'. This only changes the .erb filename if you're using a Debian system, and it adds '-debian' to the filename, so you'd have to rename your file to omero-init2-debian.erb if you were using that system. NB2: I do not know what changes are necessary to the file, on a debian system - this has only been tested on Centos6.

 
This also needs to be done to omero-web-init.erb. You can get the file from online using:

```
wget http://www.openmicroscopy.org/site/support/omero5.1/_downloads/omero-web-init.d
```
Replace exactly the same three lines as above (the changing of environment variables, OMERO_USER etc... and it's only really 2 lines that are being changed)

Then fix the reference to that template in the recipe (this time in recipes/web.rb)

```
template '/etc/init.d/omero-web' do
  source "omero-web-init2#{rc_flavour}.erb"
  mode 0755
  variables({
     :omero_user => omero_user,
     :omero_home => "#{omero_install}/OMERO.server"
  })
end
```



13 Run Chef
===========

```
cd ~/chef-repo
sudo chef-solo -c solo.rb -j omero.json
```


14. Make sure iptables are open
===============================
The Centos template on AWS has an annoying firewall setting that rejects all traffic. If you're using that system, you can reopen your internet traffic using:
```
iptables -D INPUT 5 
```

15. Set PYTHON PATH
===================
export OMERO_PREFIX=/opt/omero/OMERO.server-5.0.2-ice35-b26
export PYTHONPATH=$PYTHONPATH:$OMERO_PREFIX/lib/python

16. Disable SELinux
===================
Edit /etc/sysconfig/selinux so that the line 
```
SELINUX=permissive
```
now reads:

```
SELINUX=disabled
```
Other options you may notice are SELINUX=enforcing - if so, still change to =disabled.

17. Log into OMERO web!
=======================
Hopefully you should be set up and OMERO ought to be findable at http://localhost:4080 (or the IP address of your server:4080)



Troubleshooting
===============
Let's face it - nothing works. Here are some observed errors AFTER making all the changes described above

a) Cython crashes with gcc exit error 1

pip installs can take a while via Chef. Cython seems to take an absolute age. And it may produce this error. But, it seems that simply running Chef again will get rid of this error. yeah, awesome. 

