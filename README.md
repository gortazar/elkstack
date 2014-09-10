# elkstack

**Please note that this cookbook does not restart elasticsearch automatically,
in order to avoid causing an outage of the cluster. It does restart nginx and
logstash, however. You will have to restart elasticsearch after the initial
bootstrap.**

Elasticsearch, Logstash, and Kibana stack. Due to the recommendations of the
community, we are not using the embedded elasticsearch functionality of logstash
at this point. This cookbook provides recipes for all three components, along
with wrapper recipes such as `single` or `cluster` to facilitate different use
cases.

This stack's design is intended for one or many standalone nodes, with a full
stack of elasticsearch, logstash, and kibana. The only difference between one
and many nodes is that elasticsearch is clustered together. Data dispatched to  
Logstash on a particular node will use the local elasticsearch transport
interface to index those logs to the node (and thus, the cluster). HTTP traffic
dispatched to Kibana on port 80 on any node will also use the local
elasticsearch HTTP interface to fetch and manipulate data.

Please read the individual recipe summaries to understand what each recipe does,
as well as what each wrapper recipe is actually wrapping. As much as possible,
upstream attributes have been exposed/overriden for our needs.

## [Changelog](CHANGELOG.md)

See CHANGELOG.md for additional information about changes to this stack over time.

## Supported Platforms

Ubuntu 12.04

Ubuntu 14.04

CentOS 6.5

## Attributes

<table>
  <tr>
    <th>Key</th>
    <th>Type</th>
    <th>Description</th>
    <th>Default</th>
  </tr>
  <tr>
    <td><tt>['elkstack']['config']['cluster']</tt></td>
    <td>Boolean</td>
    <td>whether to search for and connect Elasticsearch to cluster nodes</td>
    <td><tt>false</tt></td>
  </tr>
  <tr>
    <td><tt>['elkstack']['iptables']['enabled']</tt></td>
    <td>Boolean</td>
    <td>Enable/Disable iptables functionality</td>
    <td><tt>true</tt></td>
  </tr>
  <tr>
    <td><tt>['elkstack']['config']['site_name']</tt></td>
    <td>String</td>
    <td>Control the name of the self-signed SSL key and cert in /etc/nginx/ssl</td>
    <td><tt>kibana</tt></td>
  </tr>
  <tr>
    <td><tt>['elkstack']['config']['kibana']['redirect']</tt></td>
    <td>Boolean</td>
    <td>Enable/Disable nginx redirect for kibana from port 80 to port 443</td>
    <td><tt>true</tt></td>
  </tr>
  <tr>
    <td><tt>['elkstack']['config']['data_disk']['disk_config_type']</tt></td>
    <td>String or Boolean</td>
    <td>See customizing the stack section below</td>
    <td><tt>false</tt></td>
  </tr>
  <tr>
    <td><tt>node.run_state['elkstack_kibana_username']</tt> and <tt>['elkstack']['config']['kibana']['username']</tt></td>
    <td>String</td>
    <td>Default username for basic auth for kibana, run_state used first</td>
    <td><tt>kibana</tt></td>
  </tr>
  <tr>
    <td><tt>node.run_state['elkstack_kibana_password']</tt></td>
    <td>String</td>
    <td>Password for basic auth for kibana</td>
    <td>random from <tt>Opscode::OpenSSL::Password</tt></td>
  </tr>
</table>

## Customizing the stack

To override local storage for elasticsearch nodes (the stack will format and mount, as well as configure elasticsearch), set `['elkstack']['config']['data_disk']['disk_config_type']` to `custom` and provide each storage device and mount point in the following way:
```ruby
disk_config = {
  'file_system' => 'ext4',
  'mount_options' => 'rw,user',
  'mount_path' => '/usr/local/var/data/elasticsearch/disk1',
  'format_command' => 'mkfs -t ext4 ',
  'fs_check_command' => 'dumpe2fs'
}

node.override['elasticsearch']['data']['devices']['/dev/xvde1'] = disk_config
node.override['elasticsearch']['path']['data'] = disk_config['mount_path']
```

To add additional logstash configuration to this stack, simply add additional templates in your wrapper cookbook. They should be placed in `"#{@basedir}/#{@instance}/etc/conf.d"` (see the config provider in the logstash cookbook).

To override the nginx configuration, simply supply a new template and specify your cookbook using `['kibana']['nginx']['template_cookbook']` and `['kibana']['nginx']['template']`. You can also override just the password for the reverse proxy using `node.run_state['elkstack_password']`.

To override anything else, set the appropriate node hash (`logstash`, `kibana`, or `elasticsearch`).

## Usage

### elkstack::default

Default recipe, does not do anything.

### elkstack::single

A simple wrapper recipe that sets up Elasticsearch, Logstash, and Kibana. Also
configures an rsyslog sink into logstash on the local box. Everything except
Logstash and Kibana is locked down to listen only on localhost.

### elkstack::cluster

A simple wrapper recipe that sets up Elasticsearch, Logstash, and Kibana. Also
configures an rsyslog sink into logstash on the local box. Sets the cluster flag
so that the elasticsearch recipe builds it in a cluster-friendly way.

### elkstack::elasticsearch

Leans on the upstream `elasticsearch/cookbook-elasticsearch` cookbook for much
of its work. We do override the default set of plugins to be installed, as well
as the amount of JVM heap. See `attributes/default.rb` for those settings.

This recipe also tags the node so that other nodes that run this recipe can
discover it, and configure Elasticsearch appropriately to join their cluster.
It uses a tag, the current chef environment, and the cluster name as the default
search criteria.

Most of this is configurable using the upstream Elasticsearch cookbook's
attributes, including the chef search itself. There is not an easy toggle to
turn off the search, however.
Enables iptables rules if default['elkstack']['iptables']['enabled'] not nil

### elkstack::logstash

Leans on the upstream `lusis/chef-logstash` cookbook for much
of its work. We do override the default set of plugins to be installed, as well
as the amount of JVM heap. See `attributes/default.rb` for those settings.

### elkstack::kibana

Leans on the upstream `lusis/chef-kibana` cookbook for most of its work. Sets up
an nginx site for kibana by default. By default, it also does not pass through
most of the http paths directly to elasticsearch (whitelist).

### elkstack::java

Wrapper for a java recipe. This is not included on the run list normally, so if
you don't already, you must include this recipe or get another JVM installed
before including anything else in this cookbook.

### elkstack::rsyslog

Leans on the upstream `opscode-cookbooks/rsyslog` cookbook for most of its work.
Configures a local rsyslog that sends to the local logstash instance. The stack
has already been configured with attributes for this use case. See
`attributes/default.rb` for those settings.

### elkstack::newrelic

Validates if there is a newrelic license set and based on that, see if the node
is tagged as 'elkstack' or 'elkstack_cluster' and creates a file with
elasticsearch details. Installs python, pip and setuptools packages in order to
support newrelic_meetme_plugin

## elkstack::acl

Adds basic iptables rules and cluster iptables rules if appropriate attributes
are set.

## elkstack::disk_setup

Look for `node['elkstack']['config']['data_disk']['disk_config_type']` to be truthy, and configure the upstream elasticsearch cookbook to format, mount, and use devices appropriately.

## elkstack::\*\_monitoring

These correspond with the recipes above, and just provide a way to pull out the
monitoring work to make the original recipes cleaner.

### Miscellaneous

The wrapper recipes are `single` and `cluster`. These change attributes and then
invoke `elasticsearch`, `logstash`, `kibana`, and `rsyslog`. Finally, there are
utility recipes like `java` and `newrelic` (not invoked otherwise), as well as
`acl` which is called by `_base` if `node['elkstack']['iptables']['enabled']`.

## Contributing

See [CONTRIBUTING](https://github.com/AutomationSupport/elkstack/blob/master/CONTRIBUTING.md).

## Authors

Author:: Rackspace (devops-chef@rackspace.com)

## License
```
# Copyright 2014, Rackspace Hosting
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
```
