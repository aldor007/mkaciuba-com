+++
title = "Homelab - Kubernetes at home"
date = 2022-01-06T15:42:25+01:00
images = []
tags = ["devops", "kubernets", "helm", "ansible"]
categories = ["projects"]
draft = false
type = "posts"
+++


# Introduction

In this post I would like to describe my process of creating Kubernetes servers at home. By reading this post you will learn
* What hardware you can use
* How to setup network
* What CI/CD are useful to home cluster with single human operator

One caveat - this is from my perspective - what was working for me does not always work for everyone.


# Motivation - background

In this paragraph I will describe why someone even would consider using Kubernetes in homelab. If you already know that you want to try K8S you can skip this paragraph.

Before my homelab I was using an OVH dedicated server on which I’ve deployed my [webpage(PHP)](https://mkaciuba.com/blog/posts/mkaciuba-php-webiste/). Daily operations and deploying changes were very painful even with a deploy script. PHP backend started to be slow for me. So in my mind an idea raised to rewrite this to something new. While rewriting, why not change servers? In my home I’ve already got some Raspberry PI and Cubieboards so I wanted to reuse them.
That's how project  for rewriting homepage and creating homelab started

# Hardware

My idea was to have the homelab enclosed in a fairly small package. At this point of time I didn’t want to have an “ikea rack” server. And as this will be in my living room it has to be quiet.
Taking my requirements into account I’ve chosen
ARM64 - 4x  Raspberry PI 4b 8GB RAM
ARM64 - 2x RaspberryPi 4b 4GB RAM
X86_64 - 2x AtomicPi 4GB RAM
I planned to reuse boards that I’ve already have
ARM - 2x Cubieboard 1GB RAM
ARM - 1x Cubietruck 2GB RAM
As a case for everything I’ve used some old PC ATX case, so for power supply the best idea was to use PC power supply (instruction how to use PC power supply without motherboard can be found [here](https://www.overclockersclub.com/guides/atx_psu_startup/))

![case inside](https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy9JTUdfMjQyNF8xMmY2NGU0OTc3LmpwZw/photo_pc-case-inside_big.jpg)

![everything connected](https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy9JTUdfMjQyN18zOGU0ODQ4ZGY4LmpwZw/photo_servers-connected_big.jpg)


## Network setup

For networking I've used one router with BGP capabilities (keep in mind you can use a simpler router with ARP Layer 2) this network configuration is necessary for exposing services via loadbalancer service type by using [Metallb](https://metallb.universe.tf/concepts/) (more about it łater).


## Cooling
When running such a setup 24/7 keeping temperature on a stable level is a key. To be sure that none of my computing power will overheat I've added 2 PC 12V fans. Speed of rotation of the fans is controlled by python script deployed to one of the Raspberry PI.

![cooling system](https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy9JTUdfMjQxN19lNDA2OGY2NzkyLmpwZw/photo_cpu-fans_big.jpg)

Python script controlling fans speed:
```python
import RPi.GPIO as GPIO
import time
import logging
import logging.handlers
from systemd.journal import JournalHandler

log = logging.getLogger('temperature')
log.addHandler(logging.handlers.SysLogHandler(address = '/dev/log'))

log.setLevel(logging.INFO)

GPIO.setmode(GPIO.BCM)
GPIO.setup(12, GPIO.OUT)

p = GPIO.PWM(12, 100)
p.start(0)
time.sleep(5)
p.ChangeDutyCycle(100)
time.sleep(15)

tFile = open('/sys/class/thermal/thermal_zone0/temp')
temp = float(tFile.read())/1000
tFile.close()
log.info("Temp check started, temperature at start %f", temp)
changeCycle = False
prevCycle = 100
newCycle = 0
while True:
  try:
    tFile = open('/sys/class/thermal/thermal_zone0/temp')
    temp = float(tFile.read())/1000
    tFile.close()
    if temp > 59:
       newCycle = 100
    elif temp > 55:
       newCycle = 75
    elif temp > 49:
       newCycle = 54
    elif temp > 40:
       newCycle = 49
    else:
       newCycle = 47
    if newCycle != prevCycle:
        log.info("Change cycle to %d, temp %f", newCycle, temp)
        prevCycle = newCycle
        p.ChangeDutyCycle(newCycle)
        time.sleep(60)
    if newCycle > 74:
       time.sleep(60)
    time.sleep(5)
  except Exception as err:
    log.error("unknown error occured", err)
    GPIO.cleanup()
    exit
```

# Software

Hardware part is finished now let’s talk about software

## Network

Network configuration is done part in Ubiquiti wizard (basic network setup) and via ansible - BGP peers.  But why do we need BGP or Layer 2 LB? If you want to expose service to external client you need to have some kind of load balancer to which you can route traffic. To do so on bare metal you can use MetalLb. It is very easy to install and has two modes of working. More about it [here](https://metallb.universe.tf/concepts/). My ansible playbook with configuration can be found [here](https://github.com/aldor007/homelab/tree/master/ansible/edgerouter)

| ![network diagram](https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy9uZXR3b3JrX2RpYWdyYW1fYTE3ZjBlZDAwZi5wbmc/photo_network-diagram_big.jpg) |
|:--:|
| *Network diagram* |

## Kubernetes

I need to confess something - I’ve lied in the title of this post. I’ve deployed k3s not k8s. Why? As I’m deploying to mostly ARM based servers I needed to have something lighter than full size Kubernetes. K3S is a smaller version of k8s designed to be deployed to less powerful servers. More about it https://k3s.io/
But for me as for now there is no difference in using K3s to k8s (in my work we are using full size kubernetes clusters). Installation of k3s on server is relatively easy - just run ansible playbook https://github.com/k3s-io/k3s-ansible. Example from my homelab https://github.com/aldor007/homelab/tree/master/ansible/k3s-ansible


On this stage we have Kubernetes cluster with network setup and few nodes ready to server traffic. Next we need to create some basic operation setup - monitoring, deploy pipeline, secret management etc. To manage this I’ve used helmfile. It allows user to create mono-repo of infra/seed helm charts that have to be deployed to the bootstrap cluster.

## What next?

Monitoring - prometheus-operator, grafana, loki
Apps deploy Argocd
Secret management vault-operator

Full configuration can be found [here](https://github.com/aldor007/homelab)
