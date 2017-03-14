# AutomateScript
In configuration management what ever you want install anything go to  https://supermarket.chef.io
First we need to chef hosted server and chef client and chef Node this is the basic things of chef configuration management .
When you install chef client i.e chefDK we need to crate directory name chef-repo. 
After we need to generate cookbook . what ever you upload based on this cookbook. Any download Tomcat, Apache, Mysql what ever….. after you can upload chef server..
1.	Apache2.2  version set to autstart on  linux
In Automate_script.rb script file the initially lines helps in executing the apache server on server startup.
###### Install Apache and start the service.#########
httpd_service 'customers' do
mpm 'prefork'
  action [:create, :start]
end

# Add the site configuration.

httpd_config 'customers' do
  instance 'customers'
  source 'customers.conf.erb'
  notifies :restart, 'httpd_service[customers]'
end

####
2 Tomcat 7 Instances serving the java application (2 nodes on same server) set to autostart on server start 
I create a recipe in chef, which you just define the role for which server you put this role, and it will install many tomcat instances in different port, for you.  “In Automate_script.rb”
Recipe : tomcat
attributes
default['tomcat']['default']['port']  = "8081"				

files/default/tomcat-users.xml
<tomcat-users><role rolename="manager-gui"/><role rolename="admin-gui"/><user username="admin" password="tucanoNAj4n3l4" roles="manager-gui,admin-gui"/></tomcat-users>
recipes/default.rb
apache_tomcat 'tomcat' do
  url 'http://archive.apache.org/dist/tomcat/...'
  # Note: Checksum is SHA-256, not MD5 or SHA1. Generate using `shasum -a 256 /path/to/tomcat.tar.gz`
  checksum 'sha256_checksum'
  version '7

  # ... apache_tomcat_instance definitions
end

# Install Tomcat 7 and create 2 independent instances called 'instance1'
# and 'instance2'.
apache_tomcat 'my_tomcat' do

  # Instance will install in `/opt/tomcat/instance1/`
  apache_tomcat_instance 'instance1' do
    apache_tomcat_service 'instance1'
  end

  # Instance will install in `/opt/tomcat/instance2/`
  apache_tomcat_instance 'instance2' do
    apache_tomcat_service 'instance2'
  end
end


user "tomcat" do
    system true
end

bash "download_tomcat_7" do
  user "root"
  code <<-EOH
  cd /company/inst-files/
  if [ ! -f /company/inst-files/tomcat-7.tar.gz ]; then
    wget https://s3.amazonaws.com/company-backup/infra/linux/software/tomcat-7.tar.gz
    tar xzvf tomcat-6.tar.gz -C /company/inst-files/
  fi
  EOH
end

node['tomcat']['port'].each do |port|
  directory "/company/tomcat-7-port-#{port}" do
    owner "tomcat"
    group "tomcat"
    mode 00755
    action :create
  end

  directory "/company-logs/tomcat-7-port-#{port}/old" do
    owner "tomcat"
    group "tomcat"
    mode 00755
    action :create
    recursive true
  end

  directory "/company/tomcat-7-port-#{port}/webapps" do
    owner "tomcat"
    group "tomcat_admin"
    mode 00755
    action :nothing
    recursive true
  end

  bash "install_tomcat_7" do
    user "root"
    code <<-EOH
    if [ ! -d "/company/tomcat-6-port-#{port}/conf" ]; then
        cd /company/inst-files
        cp -rv tomcat-7/* /company/tomcat-7-port-#{port}/
        rm -rf /company/tomcat-7-port-#{port}/logs
        ln -s /company-logs/tomcat-7-port-#{port}/ /company/tomcat-7-port-#{port}/logs

        port="#{port}"
        echo "$port" > /tmp/port.log
        short_port="${port:2:2}";
        sed 's:<Server port="8005" shutdown="SHUTDOWN">:<Server port="'"$short_port"05'" shutdown="SHUTDOWN">:' /company/tomcat-7-port-#{port}/conf/server.xml > /company/temp_server_1.xml;
        sed 's:<Connector port="8080" protocol="HTTP/1\.1:<Connector port="'"$port"'" protocol="HTTP/1\.1:' /company/temp_server_1.xml > /company/temp_server_2.xml;
        sed 's:<Connector port="8009" protocol="AJP/1\.3" redirectPort="8443" />:<Connector port="'"$short_port"09'" protocol="AJP/1.3" redirectPort="'"$short_port"43'" />":' /company/temp_server_2.xml > /company/temp_server_3.xml;
        sed 's:<Engine name="Catalina" defaultHost="localhost">:<Engine name="Catalina" defaultHost="localhost" jvmRoute="'tomcat"$port"'">":' /company/temp_server_3.xml > /company/temp_server_4.xml;
        mv /company/tomcat-7-port-#{port}/conf/server.xml /company/tomcat-7-port-#{port}/conf/server.xml.original;
        mv /company/temp_server_4.xml /company/tomcat-7-port-#{port}/conf/server.xml;
        rm /company/temp_server*;

        /bin/chown tomcat:tomcat /company/tomcat-7-port-#{port}/ -R
        /bin/chown tomcat:tomcat /company-logs/tomcat-7-port-#{port}/ -R
        /bin/chown tomcat:tomcat /company/tomcat-7-port-#{port}/webapps -R
    fi        
    EOH
  end

  execute "set permissions tomcat" do
    command "/bin/chown tomcat:tomcat /company/tomcat-7-port-#{port}/ -R"
    action :run
    only_if do File.exists?("/company/tomcat-7-port-#{port}") end
  end

  execute "set permissions tomcat" do
    command "/bin/chown tomcat:tomcat /company-logs/tomcat-7-port-#{port}/ -R"
    action :run
    only_if do File.exists?("/company-logs/tomcat-7-port-#{port}") end
  end

  execute "set permissions tomcat" do
    command "/bin/chown tomcat:tomcat_admin /company/tomcat-7-port-#{port}/webapps -R"
    action :run
    only_if do File.exists?("/company/tomcat-7-port-#{port}/webapps") end
  end    

  template "/etc/init.d/tomcat-7-port-#{port}" do
    owner "root"
    group "root"
    mode "0755"
    source "tomcat-service.erb"
    variables(
    :port => "#{port}"
    )
  end

  cookbook_file "/company/tomcat-7-port-#{port}/conf/tomcat-users.xml" do
    owner "tomcat"
    group "tomcat"
    mode "0755"
    source "tomcat-users.xml"
  end

  cookbook_file "/company/tomcat-7-port-#{port}/bin/catalina.sh" do
    owner "tomcat"
    group "tomcat"
    mode "0755"
    source "catalina.sh"
  end

  service "tomcat-7-port-#{port}" do
   supports :status => true,
            :start => true,
            :stop => true,
            :restart => true
   action [ :enable, :start ]
  end
end

templates/default/tomcat-service.erb

# This is the init script for starting up the
#  Jakarta Tomcat server
#
# chkconfig: 345 91 10
# description: Starts and stops the Tomcat daemon.
#

# Source function library.
. /etc/rc.d/init.d/functions

# Get config.
. /etc/sysconfig/network

# Check that networking is up.
[ "${NETWORKING}" = "no" ] && exit 0

tomcat=/company/tomcat-6-port-<%= @port %>
startup=$tomcat/bin/startup.sh
shutdown=$tomcat/bin/shutdown.sh
export JAVA_HOME=/company/jdk6

start(){

  #move os arquivos de log para os arquivos antigos
  /bin/ls /company-logs/tomcat-6-port-<%= @port %>/*.log > /dev/null  2>&1
  if [ $? -eq 0 ]; then
    mv -f /company-logs/tomcat-6-port-<%= @port %>/*.log /company-logs/tomcat-7-port-<%= @port %>/old/
  fi

  #move o catalina.out para os arquivos antigos
  if [ -e /company-logs/tomcat-6-port-<%= @port %>/catalina.out ]; then
    mv -f /company-logs/tomcat-6-port-<%= @port %>/catalina.out /company-logs/tomcat-7-port-<%= @port %>/old/
  fi

  numproc=`ps -ef | grep "/company/tomcat-7-port-<%= @port %>/bin/bootstrap.jar" | grep -v grep |awk -F' ' '{ print $2 }'`;

  if [ $numproc ]; then
    echo "Tomcat <%= @port %> is running!"
    echo "Stop then first!"
  else
    action $"Starting Tomcat <%= @port %> service: " su - tomcat -c $startup
    RETVAL=$?
  fi

}

stop(){
  numproc=`ps -ef | grep "/company/tomcat-7-port-<%= @port %>/bin/bootstrap.jar" | grep -v grep |awk -F' ' '{ print $2 }'`;

  if [ $numproc ]; then
    action $"Stopping Tomcat <%= @port %> service: " $shutdown
    RETVAL=$?
  else
    echo "Tomcat <%= @port %> is not running..."
  fi

  numproc=`ps -ef | grep "/company/tomcat-7-port-<%= @port %>/bin/bootstrap.jar" | grep -v grep |awk -F' ' '{ print $2 }'`;
  if [ $numproc ]; then
    kill -9 $numproc
  fi
}

restart(){
  stop
  start
}

status(){
  numproc=`ps -ef | grep "/company/tomcat-7-port-<%= @port %>/bin/bootstrap.jar" | grep -v grep | wc -l`
  if [ $numproc -gt 0 ]; then
    echo "Tomcat is running..."
  else
    echo "Tomcat is stopped..."
  fi
}

# See how we were called.
case "$1" in
start)
 start
 ;;
stop)
 stop
 ;;
status)
 status
 ;;
restart)
 restart
 ;;
*)


    echo $"Usage: $0 {start|stop|status|restart}"
     exit 1
    esac

exit 0
3 Mysql Database on the server 5.6
Install database  as per your requirement 
######## Install My SQL #######

Install mysql 5.6 version

  version '5.6'
  bind_address '0.0.0.0'
  port '3306'
  data_dir '/data'
  initial_root_password 'Ch4ng3me'
  action [:create, :start]
end

depends "apache2"
depends "mysql", "5.6"

include_recipe "apache2"
include_recipe "mysql::client"
include_recipe "mysql::server"

apache_site "default" do
  enable true
end

mysql_database 'apache2' do
  connection ({:host => 'localhost', :username => 'root', :password => node['mysql']['server_root_password']})
  action :create
end

mysql_database node['apache2']['database'] do
  connection ({:host => 'localhost', :username => 'root', :password => node['mysql']['server_root_password']})
  action :create
end

