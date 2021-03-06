VAGRANTFILE_API_VERSION = '2'

class Hash
  def symbolize_keys!
    keys.each do |key|
      self[(key.to_sym rescue key) || key] = delete(key)
    end
    self
  end
end

def walk(obj, &fn)
  if obj.is_a?(Array)
    obj.map { |value| walk(value, &fn) }
  elsif obj.is_a?(Hash)
    obj.each_pair { |key, value| obj[key] = walk(value, &fn) }
  else
    obj = yield(obj)
  end
end

require 'yaml'

config_path = "#{File.dirname(__FILE__)}/config.yml"
unless File.exist?(config_path)
  raise 'Configuration file not found! Please copy default.drupal*.config.yml to config.yml and try again.'
end

# prepare configuration
vconfig = YAML.load_file(config_path)
vconfig = walk(vconfig) do |value|
  while value.is_a?(String) && value.match(/{{ .* }}/)
    value = value.gsub(/{{ (.*?) }}/) { vconfig[Regexp.last_match(1)] }
  end
  value
end
vconfig.symbolize_keys!

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = vconfig[:vagrant_box]
  config.vm.hostname = vconfig[:vagrant_hostname]
  config.ssh.username = vconfig[:vagrant_username]

  if File.exist? vconfig[:vagrant_ssh_private_key_path]
    config.ssh.private_key_path = [
      vconfig[:vagrant_ssh_private_key_path],
      "~/.vagrant.d/insecure_private_key"
    ]
  end

  config.ssh.insert_key = false
  config.ssh.forward_agent = true
  config.vm.network "private_network", ip: vconfig[:vagrant_private_ip]
  unless vconfig[:vagrant_public_ip].empty?
    config.vm.network "public_network", ip: vconfig[:vagrant_public_ip]
  end

  config.nfs.map_uid = Process.uid
  config.nfs.map_gid = Process.gid

  config.vm.provider "virtualbox" do |vm, override|
    vm.cpus = vconfig[:vagrant_vm_cpus]
    vm.memory = vconfig[:vagrant_vm_memory]

    vconfig[:vagrant_vm_synced_folder].each do |synced_folder|
      override.vm.synced_folder(
        synced_folder['local_path'],
        synced_folder['vm_path'],
        {
          type: synced_folder['type'],
          create: synced_folder['create']
        }
      )

      if synced_folder['type'] == 'nfs'
        override.vm.synced_folders[synced_folder['vm_path']][:guestpath] = synced_folder['bindfs']['path']
        override.bindfs.bind_folder synced_folder['bindfs']['path'], synced_folder['vm_path'],
          u: synced_folder['bindfs']['user'],
          g: synced_folder['bindfs']['group'],
          perms: synced_folder['bindfs']['perms'],
          o: synced_folder['bindfs']['options']
      end

      if Vagrant.has_plugin?('vagrant-cachier')
        override.cache.scope = :box
      end
    end

    if Vagrant.has_plugin?('vagrant-hostmanager')
      aliases = []
      aliases.concat(config.vm.hostname.split)
      vconfig[:nginx_hosts].each do |host|
        aliases.concat(host['server_name'].split)
      end
      if vconfig[:extra_features].include? 'mautic'
        aliases.push(vconfig[:mautic_site_name])
      end

      aliases = aliases.uniq - [config.vm.hostname]

      config.hostmanager.enabled = true
      config.hostmanager.manage_host = true
      config.hostmanager.aliases = aliases
    end
 end

  config.vm.define vconfig[:vagrant_machine_name] do |droplet|
    droplet.vm.provider :digital_ocean do |provider, override|
      override.nfs.functional = false
      override.vm.box_url = vconfig[:digitalocean_box_url]
      override.ssh.private_key_path = vconfig[:vagrant_ssh_private_key_path]
      provider.ssh_key_name = ENV['DIGITALOCEAN_SSH_KEY_NAME']
      provider.token = ENV['DIGITALOCEAN_TOKEN']
      provider.image = vconfig[:digitalocean_image]
      provider.region = vconfig[:digitalocean_region]
      provider.size = vconfig[:digitalocean_size]
      provider.backups_enabled = vconfig[:digitalocean_backup_enabled]
      if !vconfig[:user_data_path].nil? and File.exist? vconfig[:user_data_path]
        provider.user_data = File.read vconfig[:user_data_path]
      end
    end
  end

  config.vm.provider :aws do |aws, override|
    override.vm.box = "aws"
    aws.access_key_id = ENV['VAGRANT_AWS_ACCESS_KEY_ID']
    aws.secret_access_key = ENV['VAGRANT_AWS_SECRECT_ACCEESS_KEY']
    aws_config = vconfig[:aws]
    aws_config.each do |k,v|
      aws.send("#{k}=", v)
    end

    override.ssh.username = "ec2-user"
    override.ssh.private_key_path = vconfig[:vagrant_ssh_private_key_path]
    if !vconfig[:user_data_path].nil? and File.exist? vconfig[:user_data_path]
      aws.user_data = File.read vconfig[:user_data_path]
    end
  end

  config.vm.provider :azure do |azure, override|
    override.vm.box = 'azure'
    azure_config = vconfig[:azure]
    # use Azure Active Directory Application / Service Principal to connect to Azure
    # see: https://azure.microsoft.com/en-us/documentation/articles/resource-group-create-service-principal-portal/

    # each of the below values will default to use the env vars named as below if not specified explicitly
    azure.tenant_id = ENV['VAGRANT_AZURE_TENANT_ID']
    azure.client_id = ENV['VAGRANT_AZURE_CLIENT_ID']
    azure.client_secret = ENV['VAGRANT_AZURE_CLIENT_SECRET']
    azure.subscription_id = ENV['VAGRANT_AZURE_SUBSCRIPTION_ID']
    azure_config.each do |k,v|
      azure.send("#{k}=", v)
    end
    override.ssh.pty = true
    override.ssh.private_key_path = vconfig[:vagrant_ssh_private_key_path]
  end

  config.vm.provider :google do |google, override|
    override.vm.box = "google/gce"

    google.google_project_id = ENV['GOOGLE_PROJECT']
    google.google_client_email = ENV['GOOGLE_CLIENT_EMAIL']
    google.google_json_key_location = ENV['GOOGLE_APPLICATION_CREDENTIALS']
    override.ssh.username = ENV['GOOGLE_USERNAME']
    override.ssh.private_key_path = vconfig[:vagrant_ssh_private_key_path]

    google_config = vconfig[:google]
    google_config.each do |k,v|
      google.send("#{k}=", v)
    end
  end

  if File.exist? vconfig[:vagrant_ssh_public_key_path]
    config.vm.provision "file",
      source: vconfig[:vagrant_ssh_public_key_path],
      destination: "~/.ssh/authorized_keys"
  end

  config.vm.provision 'ansible' do |ansible|
    ansible.playbook = "#{File.dirname(__FILE__)}/provisioning/site.yml"
    ansible.extra_vars = "#{File.dirname(__FILE__)}/config.yml"
  end

  # Allow an untracked Vagrantfile to modify the configurations
  eval File.read 'Vagrantfile.local' if File.exist?('Vagrantfile.local')
end
