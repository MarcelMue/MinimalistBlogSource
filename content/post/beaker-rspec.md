+++
date = "2017-04-11T19:34:47+01:00"
title = "Testing with Beaker and Serverspec on Windows WIP"
draft = false
+++
In this short post I want to describe my experiences while using Beaker and Serverspec on a Windows target. I aim to provide you with a basic setup to get going as well as some examples and how-tos. I will start with an example for a testing infrastructure, progress through a project structure and finish with some handy Serverspec code.
## Setup
There are three main components to basically any setup with beaker: a host machine which runs beaker, a target machine for our tests and some entity which creates our target machine.

The first priority should be to have a fast and reliable way of creating target machines. One easy way to create machines is [Vagrant](https://www.vagrantup.com/) for a smaller setup or a VMWare ESX to create a large amount of machines. The setup should aim to spin up a machine for a test, run the test and immediatly destroy the machine on success. A machine can be left alive after failed tests in order to give developers time to inspect what exactly lead to the error.

Now that you have atleast some means to create target machines, we need to do atleast some provisioning on that machine. I installed the [Bitvise SSH Server](https://www.bitvise.com/ssh-server) on my target machines in order to connect to them from my host machine. The easiest way to do this for me was to install Bitvise on a machine, snapshotting the machine and then spinning up all further machines in an ESX by creating linked clones from that snapshot. You can also do this with [Packer](https://www.packer.io/) if you want to continuosly rebuild your images and follow a stricter infrastructure as code approach (which I would definetly recommend).

The final step of our setup is the host machine, obviously the main task is to get Beaker running on this machine. The most basic gems you should have installed are:
```ruby
gem 'beaker'
gem 'beaker-rspec'
gem 'beaker-windows'
```
The OS of your host does not have to be windows (as all windows specific tasks will be run on your target). You should also check if a SSH connection from your host to your target is possible, to avoid problems down the line. It is also necessary to create a `.fog` file in your user directory if you are using a VMware ESX or something similar to spin up your target machines (you can read more about this in the beaker docu [here](https://github.com/puppetlabs/beaker/blob/master/docs/how_to/hypervisors/README.md)).
## Project Structure
You should have a functioning setup now, which allows you to create target machines and run beaker tests from a host. The next step is to create a project structure for your tests and run them on your infrastructure. You can use the following file structure:
```
|-- Gemfile
|-- spec
	|-- spec_helper_acceptance.rb
	|-- acceptance
		|-- test_spec.rb
		|-- nodesets
			|-- vsphere.yml
```
**Gemfile:** Contains all gems which have to be installed in order to run your test, a very simple example was given above.

**spec_helper_acceptance.rb:** All the requires you need, a simple example could be:
```ruby
require 'beaker-rspec'
require 'beaker-windows'
require 'beaker-rspec/spec_helper'
require 'beaker-rspec/helpers/serverspec'
```
**test_spec.rb:** The place where your actual tests go, more about them later on.  

**vsphere.yml:** A nodeset to run your target machine, Beaker expects to find a `default.yml` but you can pass another yml to Beaker aswell. It could look something like this in your setup:
```ruby
HOSTS:
  10.10.10:
    platform: windows-server-amd64
    vmname: test-vm
    hypervisor: vsphere
    ssh:
      user: usr
      password: pw
      auth_methods:
        - password
    user: usr
    communicator: bitvise
    is_cygwin: false
CONFIG:
  log_level: verbose
```
Note that the credentials for your hypervisor have to be supplied in the `.fog` file mentioned earlier.
## Running Tests
In a first step you need to make sure, that your Beaker environment variables are set on your host. Some of the important ones are `BEAKER_set=vsphere` for our set and `BEAKER_destroy=onpass` if Beaker has the ability to destroy machines in your setup. Then you will need to run your tests in two commands from your project folder:
```
bundle install --path=somepath/bundle
bundle exec rspec spec/acceptance/
```
The bundle install into a specific path is important to allow your host machine to run multiple beaker tests in parallel.
## Writing Tests for Windows
You should have a setup capable of running tests from your host against your target machine and serverspec should be available in your tests.
Some more declarations have to be made at the beginning of your test file:
```ruby
require 'spec_helper_acceptance'
set :os, :family => 'windows'
```
And now we are off to writing Serverspec code to test the target machine. Most normal Serverspec commands will work on windows without any problems. You can use something like `it { should_not exist }` or `it { should be_file }` without any problems, interacting with the Powershell can be a bit of a hassle though. The `powershell()` caused issues for me by running asynchronly during tests or not being run at all in some cases. Invoking a Powershell instance though `shell()` was more stable for me, for example to check some environment variable on the target system:
```ruby
it "Check if my JAVA_HOME ist set" do
  expect(shell('powershell [System.Environment]::GetEnvironmentVariable(\"JAVA_HOME\", \"Machine\")').stdout).to include("c:\\path\\to\\my\\java")
end
```
Another issue I ran into were `stdout` and `stderr` when running tests. One example would be `java -version` being returned in `stderr`:
```ruby
it "Java should return the correct version" do
  result = shell("c:\\some\\path\\to\\java\\bin\\java -version")
  expect(result.stderr).to include("1.7.0")
end
```
## Putting it together with Jenkins
