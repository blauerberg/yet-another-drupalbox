# Yet Another DrupalBox

Yet Another DrupalBox is a simple Drupal environment, built with Vagrant + Ansible.

This project was inspired by [Drupal VM](https://github.com/geerlingguy/drupal-vm).

Currently, work with below environments:
- CentOS 7.x on Virtualbox
- CentOS 7.x on DigitalOcean

## Getting Started

### Windows

currently, Windows not supported...

### OS X

```
$ brew install vagrant
$ vagrant plugin install vagrant-hostmanager # recommended
$ vagrant plugin install vagrant-cachier # if you want to share the cache of packeges
$ vagrant plugin install vagrant-vbguest # required if you installed vagrant-cacher
$ vagrant plugin install vagrant-bindfs
$ vagrant box add centos/7 --provider virtualbox
$ brew install ansible
$ ansible-galaxy install geerlingguy.adminer

$ git clone https://github.com/blauerberg/yet-another-drupalbox.git
$ cd yet-another-drupalbox
$ ln -s default.drupal8.config.yml config.yml (if you want to boot with Drupal 8)
$ ln -s default.drupal7.config.yml config.yml (if you want to boot with Drupal 7)
$ vagrant up
```

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## Supporting Organizations
- https://annai.co.jp
