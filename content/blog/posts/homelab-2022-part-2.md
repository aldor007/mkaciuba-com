+++
title = "Homelab 2022 Part 2"
date = 2022-03-17T18:01:01+01:00
images = []
tags = ["devops", "kubernets", "helm", "ansible"]
categories = ["projects"]
draft = true
type = "posts"
+++

# Introduction

In previous [post](/blog/posts/homelab-2022-part-1) I have described hardware and network setup. Today I will talk about software which I have installed on my cluster


# Monitoring

For monitoring I've decided to use [prometheus-operator](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)