targets = {
  "centos6.5" => {
    "box" => "chef/centos-6.5"
  },
  "centos7"   => {
    "box" => "chef/centos-7.0"
  },
  "ubuntu14"  => {
    "box" => "ubuntu/trusty64"
  },
  "ubuntu12"  => {
    "box" => "ubuntu/precise64"
  },
  "freebsd10" => {
    "box" => "chef/freebsd-10.0"
  },
  "aws-amazon2015.03" => {
    "box" => "andytson/aws-dummy",
    "regions" => {
      "us-east-1" => "ami-1ecae776",
      "us-west-1" => "ami-d114f295",
      "us-west-2" => "ami-e7527ed7"
    },
    "username" => "ec2-user"
  },
  "aws-rhel7.1" => {
    "box" => "andytson/aws-dummy",
    "regions" => {
      "us-east-1" => "ami-12663b7a",
      "us-west-1" => "ami-a540a5e1",
      "us-west-2" => "ami-4dbf9e7d"
    },
    "username" => "ec2-user"
  },
  "aws-rhel6.5" => {
    "box" => "andytson/aws-dummy",
    "regions" => {
      "us-east-1" => "ami-1643ff7e",
      "us-west-1" => "ami-2b171d6e",
      "us-west-2" => "ami-7df0bd4d"
    },
    "username" => "ec2-user"
  }
}

Vagrant.configure("2") do |config|
  config.vm.provider "virtualbox" do |v|
    if ENV['OSQUERY_BUILD_CPUS']
      v.cpus = ENV['OSQUERY_BUILD_CPUS'].to_i
    else
      v.cpus = 2
    end
    v.memory = 4096
  end

  config.vm.provider :aws do |aws, override|
    # Required. Credentials for AWS API.
    aws.access_key_id = ENV['AWS_ACCESS_KEY_ID']
    aws.secret_access_key = ENV['AWS_SECRET_ACCESS_KEY']
    # Name of AWS keypair for launching and accessing the EC2 instance.
    if [ ENV['AWS_KEYPAIR_NAME'] ]
      aws.keypair_name = ENV['AWS_KEYPAIR_NAME']
    end
    override.ssh.private_key_path = ENV['AWS_SSH_PRIVATE_KEY_PATH']
    # Name of AWS security group that allows TCP/22 from vagrant host.
    if [ ENV['AWS_SECURITY_GROUP'] ]
       aws.security_groups = [ ENV['AWS_SECURITY_GROUP'] ]
    end
    # Set this to the AWS region for EC2 instances.
    if ENV['AWS_DEFAULT_REGION']
      aws.region = ENV['AWS_DEFAULT_REGION']
    else
      aws.region = "us-east-1"
    end
    # Set this to the desired AWS instance type.
    if ENV['AWS_INSTANCE_TYPE']
      aws.instance_type = ENV['AWS_INSTANCE_TYPE']
    else
      aws.instance_type = "m3.large"
    end
    targets["active_region"] = aws.region
    # If using a VPC, optionally set a SUBNET_ID.
    if ENV['AWS_SUBNET_ID']
      aws.subnet_id = ENV['AWS_SUBNET_ID']
    end
  end

  targets.each do |name, target|
    box = target["box"]
    config.vm.define name do |build|
      build.vm.box = box
      if name.start_with?('aws-')
        build.vm.provider :aws do |aws, override|
          if aws.subnet_id != nil
            aws.associate_public_ip = true
          end
          aws.ami = target['regions'][targets["active_region"]]
          aws.user_data = [
            "#!/bin/bash",
            "echo 'Defaults:" + target['username'] +
              " !requiretty' > /etc/sudoers.d/999-vagrant-cloud-init-requiretty",
              "chmod 440 /etc/sudoers.d/999-vagrant-cloud-init-requiretty"
          ].join("\n")
          override.ssh.username = target['username']
          aws.tags = { 'Name' => 'osquery-vagrant-' + name }
        end
        build.vm.synced_folder ".", "/vagrant", type: "rsync",
          rsync__exclude: ["build"]
      end
      if name == 'freebsd10'
        # configure the NICs
        build.vm.provider :virtualbox do |vb|
          vb.customize ["modifyvm", :id, "--nictype1", "virtio"]
          vb.customize ["modifyvm", :id, "--nictype2", "virtio"]
        end
        # Private network for NFS
        build.vm.network :private_network, ip: "192.168.56.101"
        build.vm.synced_folder ".", "/vagrant", type: "nfs"
        build.vm.provision "shell",
          inline: "pkg install -y gmake"
      end
    end
  end
end
