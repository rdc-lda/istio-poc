# Getting started with Istio on MiniShift

After reading chapter one (very enlightening, recommend this!!), hands-on starts in chapter two.

## Chapter 02 notes

I have installed the Docker machine xhyve as virtualization (using the OSX native virtualization layer) using:

~~~bash
# If you have an older version (pre 0.4.0 installed...)
$ brew unlink docker-machine-driver-xhyve

# Install the latest stable version
$ brew install \
   https://raw.githubusercontent.com/Homebrew/homebrew-core/7310c563d662ddbe094f46f9600cad30ad3551a6/Formula/docker-machine-driver-xhyve.rb

# Execute the output of the brew command above to set permissions
$ sudo chown root:wheel \
   $(brew --prefix)/opt/docker-machine-driver-xhyve/bin/docker-machine-driver-xhyve

$ sudo chmod u+s \
   $(brew --prefix)/opt/docker-machine-driver-xhyve/bin/docker-machine-driver-xhyve

# List the driver -- should be 0.4.0
$ brew info docker-machine-driver-xhyve
docker-machine-driver-xhyve: stable 0.4.0 (bottled), HEAD
~~~

Next, I installed Minishift (version v1.33.0+ba29431) using `brew cask install minishift` and created a startup script to boot MiniShift:

~~~bash
#!/bin/bash

minishift profile set tutorial

minishift config set memory 12GB
minishift config set cpus 4
minishift config set vm-driver xhyve
minishift config set image-caching true
minishift config set network-nameserver 8.8.8.8
minishift config set openshift-version v3.11.0

minishift addon enable admin-user

# Do not apply this setting -- see: https://github.com/minishift/minishift/issues/2809
#minishift addon enable anyuid

minishift start

# after start... enable the anyuid for the projects affected only...
# oc adm policy add-scc-to-group anyuid system:authenticated -n myproject --as system:admin
~~~

As you can see, this is different then the book describes due to an issue with `anyuid`; this should only applied to spaces requiring this feature and not the complete cluster.

Once booted, you can use the commands below to set the environment and open (successfully...) the console:

~~~bash
# Slurp in environment
$ eval $(minishift oc-env)
$ eval $(minishift docker-env)

# Login to OpenShift via API
$ oc login $(minishift ip):8443 -u admin -p admin

# Open the web console
$ minishift dashboard
~~~
