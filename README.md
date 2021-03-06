# idm-satellite-openshift-demo
Notes on installing and configuring a OpenShift Container Platform in a disconnected environment with IdM and Satellite 6

If you've ever found yourself working in a disconnected environment
with limited internet access, and wanting to install OpenShift, this
is the document for you!  The following configurations of IdM,
Satellite and OpenShift Container Platform (OCP) assume that we're
working in a lab environment that is disconnected from typical
enterprise services.  We also assume, for the most part, that there is
no access to the internet, with the one exception of the Satellite
server, which is allowed to access the internet via web proxy.

# Step 1: Install Satellite 6

In the disconnected environment, Red Hat Satellite becomes the conduit
through which all content will be provided to the demo environment.
This includes hosting RPM content and all docker container images.
This, of course, assumes that Satellite is able to proxy out to the
internet.

We will not be using Satellite to provision systems.  It will just be
acting as the trusted repository for RPM and container content.

1. Create a VM with 12GB of RAM and 400GB of storage.  Name it
   `sat6.atgreen.org`.

1. Set the hostname in ```/etc/hostname``` and add an entry in
   ```/etc/hosts``` with the fully qualified domain name, then reboot
   and make sure the name is set properly and that you can ping it.

1. Run "`subscription-manager register`" with the appropriate
   --proxy=... and related arguments.

1. Run "`subscription-manager list --all --available`" to find the Pool ID of the Satellite subscription. 

1. Run "`subscription-manager attach --pool=`"... with the appropriate Pool ID.

1. Run "`subscription-manager repos --disable=\*`" to disable all repos.

1. Selectively enable repos with...
<pre><code>for r in rhel-7-server-rpms \
         rhel-server-rhscl-7-rpms \
         rhel-7-server-satellite-6.2-rpms; do
    subscription-manager repos --enable=$r;
done;</code></pre>
  
1. Install bits and reboot with...
<pre><code>yum install -y satellite-installer katello ipa-client \
&& yum update -y \
&& sync && reboot</code></pre>
  
1. Log back into the server, and run the installer like so:
<pre><code>satellite-installer --foreman-initial-organization "OCP PoC" \
                        --foreman-initial-location "Innovation Zone" \
                        --foreman-admin-username=admin \
                        --foreman-admin-password=Redhat1! \
                        --scenario=satellite</code></pre>
   Be sure to provide your own values for initial org, initial
   location, and admin password.  Also, use one or more of the
   following installer options to configure access through the web
   proxy:
<pre><code>--katello-proxy-password Proxy password for authentication (default: nil)
--katello-proxy-port Port the proxy is running on (default: nil)
--katello-proxy-url URL of the proxy server (default: nil)
--katello-proxy-username Proxy username for authentication (default: nil)</code></pre>

1. Configure the satellite firewall like so:
<pre><code>firewall-cmd --add-port="53/udp" --add-port="53/tcp" \
 --add-port="67/udp" \
 --add-port="69/udp" --add-port="80/tcp" \
 --add-port="443/tcp" --add-port="5647/tcp" \
 --add-port="8140/tcp" \
&& firewall-cmd --permanent --add-port="53/udp" --add-port="53/tcp" \
 --add-port="67/udp" \
 --add-port="69/udp" --add-port="80/tcp" \
 --add-port="443/tcp" --add-port="5647/tcp" \
 --add-port="8140/tcp"</code></pre>
 
1. Log into access.redhat.com and generate a manifest for your
   Satellite with all required subscriptions.  You'll need a RHEL
   Server subscription for the IdM server, and some number of
   OpenShift subs for the OCP cluster.  Specific details
   on creating and downloading the manifest file are found here:
   https://access.redhat.com/solutions/118573

1. Log into the Satellite web UI, and navigate to Content>Red Hat
   Subscriptions.  Click on the "Browse..." button to select your
   freshly downloaded manifest zip file, and the "Upload" to send it
   to the Satellite.  Wait until Satellite tells you that it has
   successfully imported the manifest.

1. On the satellite server, create ~/.hammer/cli_config.yml in order
   to provide credentials to the hammer cli:
<code><pre>mkdir ~/.hammer
cat > ~/.hammer/cli_config.yml <<\EOF
    :foreman:
        :enable_module: true
        :host: 'https://localhost/'
        :username: 'admin'
        :password: 'Redhat1!'
EOF</pre></code>

1. Run the following commands to enable the RPM repos that we need:
<pre><code>hammer repository-set enable --name "Red Hat Enterprise Linux 7 Server (Kickstart)" --product "Red Hat Enterprise Linux Server" --organization-id 1 --basearch x86_64 --releasever 7Server
hammer repository-set enable --name "Red Hat Enterprise Linux 7 Server (RPMs)" --product "Red Hat Enterprise Linux Server" --organization-id 1 --basearch x86_64 --releasever 7Server
hammer repository-set enable --name "Red Hat Satellite Tools 6.2 (for RHEL 7 Server) (RPMs)" --product "Red Hat Enterprise Linux Server" --organization-id 1 --basearch x86_64 
hammer repository-set enable --name "Red Hat Enterprise Linux 7 Server - Extras (RPMs)" --product "Red Hat Enterprise Linux Server" --organization-id 1 --basearch x86_64 
hammer repository-set enable --name "Red Hat OpenShift Enterprise 3.2 (RPMs)" --product "Red Hat OpenShift Enterprise" --organization-id 1 --basearch x86_64</code></pre>

2. Save the following script and run it.  It will create all of the
docker image repositories we'll need for OCP.
<pre><code>#!/bin/bash
    
    ORG_ID=1
    PRODUCT_NAME="OCP Docker Images"
    
    upstream_repos=( \
        dotnet/dotnetcore-10-rhel7 \
        jboss-amq-6/amq62-openshift \
        jboss-datagrid-6/datagrid65-openshift \
        jboss-decisionserver-6/decisionserver62-openshift \
        jboss-decisionserver-6/decisionserver63-openshift \
        jboss-eap-6/eap64-openshift \
        jboss-eap-7/eap70-openshift \
        jboss-fuse-6/fis-java-openshift \
        jboss-fuse-6/fis-karaf-openshift \
        jboss-processserver-6/processserver63-openshift \
        jboss-webserver-3/webserver30-tomcat7-openshift \
        jboss-webserver-3/webserver30-tomcat8-openshift \
        openshift3/image-inspector \
        openshift3/jenkins-1-rhel7 \
        openshift3/logging-auth-proxy \
        openshift3/logging-deployment \
        openshift3/logging-elasticsearch \
        openshift3/logging-fluentd \
        openshift3/logging-kibana \
        openshift3/metrics-cassandra \
        openshift3/metrics-deployer \
        openshift3/metrics-hawkular-metrics \
        openshift3/metrics-heapster \
        openshift3/mongodb-24-rhel7 \
        openshift3/mysql-55-rhel7 \
        openshift3/nodejs-010-rhel7 \
        openshift3/ose-deployer \
        openshift3/ose-docker-builder \
        openshift3/ose-docker-registry \
        openshift3/ose-haproxy-router \
        openshift3/ose-pod \
        openshift3/ose-recycler \
        openshift3/ose-sti-builder \
        openshift3/perl-516-rhel7 \
        openshift3/php-55-rhel7 \
        openshift3/postgresql-92-rhel7 \
        openshift3/python-33-rhel7 \
        openshift3/ruby-20-rhel7 \
        rhscl/mariadb-101-rhel7 \
        rhscl/mongodb-26-rhel7 \
        rhscl/mongodb-32-rhel7 \
        rhscl/mysql-56-rhel7 \
        rhscl/nodejs-4-rhel7 \
        rhscl/perl-520-rhel7 \
        rhscl/php-56-rhel7 \
        rhscl/postgresql-94-rhel7 \
        rhscl/postgresql-95-rhel7 \
        rhscl/python-27-rhel7 \
        rhscl/python-34-rhel7 \
        rhscl/python-35-rhel7 \
        rhscl/ruby-22-rhel7 \
        rhscl/ruby-23-rhel7 \
    )
    
    hammer product create --name "$PRODUCT_NAME" --organization-id $ORG_ID
    
    PRODUCT_LIST_FILE=`mktemp`
    hammer repository list --organization-id "$ORG_ID" \
        --product "$PRODUCT_NAME" > $PRODUCT_LIST_FILE
    
    for i in ${upstream_repos[@]}; do
      found=`grep "$i" $PRODUCT_LIST_FILE`
      if [[ -z $found ]]; then
          echo "Creating repository for $i"
          hammer repository create --name "$i" --organization-id $ORG_ID \
    	  --content-type docker --url "https://registry.access.redhat.com" \
    	  --docker-upstream-name "$i" --product "$PRODUCT_NAME";
      else
          echo "Found Satellite mirror for repository $i"
      fi
    done
    
    rm -f $PRODUCT_LIST_FILE > /dev/null</code></pre>
    
1. Synchronize all repositories with the following command:
<pre><code>for i in $(hammer --csv repository list --organization-id=1  | awk -F, {'print $1'} | grep -vi '^ID'); do hammer repository synchronize --id ${i} --organization-id=1 --async; done</code></pre>
   Take a well earned break while all repos sync!

1. Run ```hammer lifecycle-environment create --organization-id 1
--name Dev --description "Development" --prior "Library"``` to create
a developement lifecycle environment.

1. Create, publish, and promote the Content View for a regular RHEL server:
<pre><code>hammer content-view create --name "RHEL 7 Server" --organization-id 1 
      
    hammer content-view add-repository --name "RHEL 7 Server" --organization-id 1 \
    --product "Red Hat Enterprise Linux Server" \
    --repository "Red Hat Enterprise Linux 7 Server - Extras RPMs x86_64"
      
    hammer content-view add-repository --name "RHEL 7 Server" \
    --organization-id 1 --product "Red Hat Enterprise Linux Server" \
    --repository "Red Hat Enterprise Linux 7 Server RPMs x86_64 7Server"
      
    hammer content-view add-repository --name "RHEL 7 Server" --organization-id 1 \
    --product "Red Hat Enterprise Linux Server" \
    --repository "Red Hat Satellite Tools 6.2 for RHEL 7 Server RPMs x86_64"
    
    hammer content-view publish --name "RHEL 7 Server" --organization-id 1 \
    --description "Initial publish"

    hammer content-view version promote --organization-id 1 --content-view "RHEL 7 Server" --to-lifecycle-environment "Dev"</code></pre>

1. Run ```hammer activation-key create --name "rhel-7-server-ak" --content-view "RHEL 7 Server" --lifecycle-environment Dev --organization-id 1``` to create the RHEL 7 Server activation key.

1. Run ```hammer subscription list --organization-id 1``` and attach the appropriate subscription to ```rhel-7-server-ak``` like so: ```hammer activation-key add-subscription --organization-id 1 --name "rhel-7-server-ak"  --quantity 1 --subscription-id [SUBSCRIPTION ID HERE]```.

1. Create, publish, and promote the Content View for an OCP server:
<pre><code>hammer content-view create --name "OCP Server" --organization-id 1 
    
    hammer content-view add-repository --name "OCP Server" --organization-id 1 \
    --product "Red Hat Enterprise Linux Server" \
    --repository "Red Hat Enterprise Linux 7 Server - Extras RPMs x86_64"
    
    hammer content-view add-repository --name "OCP Server" \
    --organization-id 1 --product "Red Hat Enterprise Linux Server" \
    --repository "Red Hat Enterprise Linux 7 Server RPMs x86_64 7Server"
    
    hammer content-view add-repository --name "OCP Server" --organization-id 1 \
    --product "Red Hat Enterprise Linux Server" \
    --repository "Red Hat Satellite Tools 6.2 for RHEL 7 Server RPMs x86_64"
    
    hammer content-view add-repository --name "OCP Server" --organization-id 1 \
    --product "Red Hat OpenShift Enterprise" \
    --repository "Red Hat OpenShift Enterprise 3.2 RPMs x86_64"
    
    hammer content-view publish --name "OCP Server" --organization-id 1 \
    --description "Initial publish"

    hammer content-view version promote --organization-id 1 --content-view "OCP Server" --to-lifecycle-environment "Dev"</code></pre>

1. Run ```hammer activation-key create --name "ocp-server-ak" --content-view "OCP Server" --lifecycle-environment Dev --organization-id 1``` to create the OCP Server activation key.

2. Run ```hammer subscription list --organization-id 1``` and attach the appropriate subscription to ```ocp-server-ak``` like so: ```hammer activation-key add-subscription --organization-id 1 --name "ocp-server-ak"  --quantity 1 --subscription-id [SUBSCRIPTION ID HERE]```.

# Step 2: Install IdM

The IdM server will host DNS and user authentication services.

1. Create a VM with 4GB of RAM and 12GB of storage.  Name it
   `idm.atgreen.org`.

1. Add sat6.atgreen.org and ipa.atgreen.org to the ```/etc/hosts```
   file on this new server.

1. Run "`rpm -ihv http://sat6.atgreen.org/pub/katello-ca-consumer-latest.noarch.rpm`"

1. Run "`subscription-manager register --org="OCP_PoC" --activationkey 'rhel-7-server-ak'`"

1. Run "```subscription-manager repos --enable=\*```"

1. Run "`yum install -y ipa-server ipa-server-dns katello-agent && yum -y update && reboot`"

1. Log back into the IdM server and run "```ipa-server-install -r ATGREEN.ORG --setup-dns --forwarder 64.6.64.6 --forwarder 8.8.8.8 --hostname=ipa.atgreen.org -p Redhat1! -a Redhat1! -n atgreen.org --ssh-trust-dns -U```".  Feel free to use some other DNS IPs for the DNS forwarder.  The ones provided here are for Verisign's and Google's
public DNS services.

1. Configure the firewall like so...
<pre><code>firewall-cmd --permanent --add-service=http --add-service=https \
    --add-service=ldap --add-service=ldaps --add-service=kerberos \
    --add-service=kpasswd --add-service=dns --add-service=ntp

    firewall-cmd --reload</code></pre>

1. Log back into the Satellite server and set ```/etc/resolv.conf``` to
   point at the new IdM server.

1. On the Satellite, run "```ipa-client-install --mkhomedir -p admin -w Redhat1! -U```"

1. Set the nameserver on your browser host to point at the IdM server,
   and keep it there.

1. Configure Satellite to authenticate users defined in IdM by running the following command on the satellite server:
"```satellite-installer --foreman-ipa-authentication=true```".

# Step 3: Install OpenShift Container Platform (OCP)

The OCP server will be an all-in-one OpenShift deployment.

1. Create a VM with 8GB of RAM and 30GB of storage.  Name it
   `ocp.atgreen.org`.

1. Make sure `/etc/resolv.conf` is pointing at the new IdM server.

1. Run "`rpm -ihv http://sat6.atgreen.org/pub/katello-ca-consumer-latest.noarch.rpm`"

1. Run "`subscription-manager register --org="OCP_PoC" --activationkey 'ocp-server-ak'`"

1. Run "```subscription-manager repos --enable=\*```"

1. Run "```yum install -y ipa-client katello-agent docker openshift-ansible-playbooks && yum -y update && reboot```"

1. Log back in and run "```ipa-client-install --mkhomedir -p admin -w Redhat1! -U```"

1. Add a second disk for internal docker images (I'm using 400GB), and
determine the device name.  In my example, it is `/dev/vdb`.

1. Edit `/etc/sysconfig/docker-storage-setup` so it looks like this:
<pre><code>DEVS=/dev/vdb
    VG=docker-vg</code></pre>

1. Run `docker-storage-setup && systemctl restart docker`.

1. Make a passwordless SSH key for root by running `ssh-keygen` and hitting `[Enter]` for every question.  Then use `ssh-copy-id ocp.atgreen.org` to enable passwordless ssh.

1. Edit `/etc/ansible/hosts` so it looks like this:
<pre><code># Create an OSEv3 group that contains the masters, nodes, and etcd groups
    [OSEv3:children]
    masters
    nodes
    etcd
    
    # Set variables common for all OSEv3 hosts
    [OSEv3:vars]
    ansible_ssh_user=root
    deployment_type=openshift-enterprise
    openshift_master_default_subdomain=ocp.atgreen.org
    openshift_examples_modify_imagestreams=true
    oreg_url=sat6.atgreen.org:5000/ocp_poc-ocp_docker_images-openshift3_ose-${component}:${version}
    openshift_docker_additional_registries=sat6.atgreen.org:5000
    openshift_docker_insecure_registries=sat6.atgreen.org:5000
    openshift_docker_blocked_registries=registry.access.redhat.com,docker.io
    openshift_docker_disable_push_dockerhub=True
    openshift_master_identity_providers=[{'name': 'allow_all', 'login': 'true', 'challenge': 'true', 'kind': 'AllowAllPasswordIdentityProvider'}]
    
    # host group for masters
    [masters]
    ocp.atgreen.org
    
    # host group for etcd
    [etcd]
    ocp.atgreen.org
    
    # host group for nodes, includes region info
    [nodes]
    ocp.atgreen.org openshift_node_labels="{'region': 'infra', 'zone': 'default'}" openshift_schedulable=true</code></pre>

1. Run "`ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/byo/config.yml`".

1. Create an executable file called `fix.sh` that looks like this:
<pre><code>#!/bin/sh
    perl -p -i -e "s|sat6.atgreen.org:5000/openshift3/|sat6.atgreen.org:5000/ocp_poc-ocp_docker_images-openshift3_|g" $1
    perl -p -i -e "s|sat6.atgreen.org:5000/rhscl/|sat6.atgreen.org:5000/ocp_poc-ocp_docker_images-rhscl_|g" $1</code></pre>
This is required in order to rewrite ImageStream references so they
pull bits out of Satellite correctly.  The strings we're converting to
map to the URLs provided by Satellite's docker registries.  You can
verify these by going logging into the Satellite, going to
Content>Products, selecting the container product ("OCP Docker
Images") and then selecting one of the containers.  You'll see the
Satellite provided URL on that screen.  To execute this script against
every ImageStream in one batch run, do this: `OC_EDITOR=./fix.sh oc
edit is -n openshift`.

# Next Steps

Congratulations!  You now have a functional OpenShift environment
provisioned entirely through Satellite.  Some interesting next steps
are to configure persistent storage, enable metrics and logging, and
to switch user authentication over to LDAP on IdM.

Feedback on this tutorial via the
[github issue tracker](https://github.com/atgreen/idm-satellite-openshift-demo/issues)
is welcome.


Thank you,

[Anthony Green](https://ca.linkedin.com/in/green)
