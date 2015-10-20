# sddevops-custom-resources

## Create a new cookbook
    chef generate cookbook sddevops-custom-resources

## Make resources directory
    cd sddevops-custom-resources
    mkdir resources

## Define properties and name resource
    #resources/website.rb
    property :instance_name, String, name_property: true
    property :port, Fixnum, required: true

    provides :mysite

## Usage in recipe
    mysite 'foo' do
      port 81
      action :create
    end

## create action (website.rb)
    action :create do
      package 'httpd' do
        action :install
      end
    ...

## create action continued (website.rb)
    ...
      template "/etc/httpd/conf.d/httpd-#{instance_name}.conf" do
        source "httpd.conf.erb"
        variables(
          :instance_name => instance_name,
          :port => port
        )
        owner 'root'
        group 'root'
        mode '0644'
        action :create
        notifies :restart, 'service[httpd]'
      end

      directory "/var/www/vhosts/#{instance_name}" do
        mode '0755'
        recursive true
        owner 'root'
        group 'root'
        action :create
        notifies :restart, 'service[httpd]'
      end
    ...

## finish website.rb
    ...
      service 'httpd' do
        action [:enable, :start]
      end
    end

## complete resource

## create templates directory
    mkdir templates

## templates/httpd.conf.erb
    ServerRoot "/etc/httpd"
    Listen <%= @port %>
    User apache
    Group apache
    <Directory />
      AllowOverride none
      Require all denied
    </Directory>
    DocumentRoot "/var/www/vhosts/<%= @instance_name %>"
    <IfModule mime_module>
      TypesConfig /etc/mime.types
    </IfModule>

## now we have a complete resource
[website.rb](http://github.com/kennonkwok/sddevops-custom-resources/resources/website.rb)

## use resource in recipe - default.rb
   mysite 'foo' do
     port 81
     action :create
   end

## quick look at .kitchen.yml, remove ubuntu
      - name: ubuntu-14.04  <--- take this out
      - name: centos-7.1


## ok... let's see if this runs
    chef install (chefdk 0.9+)
    kitchen converge

## manual verification
    kitchen login
    curl localhost:81

## manual tests aren't cool, let's serverspec this
    # test/integration/default/serverspec/default_spec.rb
    require 'spec_helper'

    describe port(81) do
      it { should be_listening }
    end

    describe port(82) do
      it { should be_listening }
    end

## run tests!
    kitchen verify

## add site on port 82 to make test pass - recipes/default.rb
    mysite 'bar' do
      port 82
      action :create
    end

## will this work in centos 6.7? - .kitchen.yml
      - name: centos-6.7 <--- add this
      - name: centos-7.1

## test everything!
    kitchen test
