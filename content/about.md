---
title: "About"
date: 2020-04-01
draft: false
---

This is my BSc thesis Devlog! I'll be detailing problems I run up against during my BSc thesis, as well as the ways in which I have solved them. 

I'll be providing a reference architecture for prefabs in Discrete Event Simulators. The DSE I will be building on is [OpenDC](https://opendc.org).

## How did you make this site?

The content itself is stored on GitHub. I use [Travis-CI](travis-ci.com), which is free with the [Github Student Developer Pack](https://education.github.com/pack), to compile the static HTML files with [Hugo](https://gohugo.io) and then use [`rsync`](https://en.wikipedia.org/wiki/Rsync) to send them to my webserver.

My `.travis.yml` (the configuration file for Travis) looks a little like this:
``` yaml
sudo: required
dist: bionic

addons:
  ssh_known_hosts: 
    - jc0b.computer:$SSH_PORT
    - $HOST_IP:$SSH_PORT

before_install:
- curl -s https://api.github.com/repos/gohugoio/hugo/releases/latest | grep "browser_download_url.*64bit.deb" | cut -d '"' -f 4 | wget -qi -
- rm hugo_extended*.deb
- sudo dpkg -i hugo_*.deb


script:
- hugo

before_deploy:
  - openssl aes-256-cbc -K $encrypted_db2095f63ba3_key -iv $encrypted_db2095f63ba3_iv -in deploy_rsa.enc -out /tmp/deploy_rsa -d
  - eval "$(ssh-agent -s)"
  - chmod 600 /tmp/deploy_rsa
  - ssh-add /tmp/deploy_rsa

deploy:
  provider: script
  skip_cleanup: true
  script: rsync -avz --delete -e 'ssh -p $SSH_PORT' $TRAVIS_BUILD_DIR/public/ $SSH_USER@jc0b.computer:bsc.jc0b.computer/
  on:
    branch: master
```

The private SSH key is stored in encrypted form on the GitHub repo, and Travis has the encryption key in its environment variables. It then decrypts on every run, and uses it to push the files over to my webserver. In this way, I only have to write and test on my laptop: Travis does the rest.