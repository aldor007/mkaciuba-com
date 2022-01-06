+++
title = "Mort Release"
date = 2017-12-17T12:02:48+01:00
images = ["https://mort.mkaciuba.com/images/transform/ZmlsZXMvc291cmNlcy9nb3BoZXJfODM3MTY0M2Y0Zi5wbmc/photo_mort_big.jpg"]
description = "Mort image processing server written in go"
tags = ["golang", "images", "photos"]
categories = ["projects"]
draft = false
type = "posts"
+++


Mort is image processing service that can on the fly create image transformation (resize, crop, grayscale etc). It can serve as a storage server as well.

The main advantage of it is that it provides full control of what operation is applied to given image and it can store it in any storage.

# Features

List of available features:

*   HTTP server
*   Resize
*   Rotate
*   SmartCrop
*   Convert (JPEG, , PNG, WEBP , .)
*   Multiple storage backends (disk, S3, http)
*   Fully modular
*   S3 API for listing and uploading files
*   Request collapsing
*   Build in rate limiter

And more to come [list](https://github.com/aldor007/mort/issues?q=is%3Aopen+is%3Aissue+label%3Aenhancement)

# URL Format

The approach implemented in mort is similar to Amazon S3 buckets. In the configuration, you have to prepare configuration for bucket storage and optionally configuration for image operations (transforms). So URL that mort process looks like

https://mort.mkaciuba.com/<bucket>/<object>

Decoding what operation should be performed is easy . Out of box there are two decoders that you can use: **presets** and **query**.**Presets** decoder is useful when you have limited range of operations to perform.

Example URL:

https://mort.mkaciuba.com/liip/photocategory_img//media/gallery/0001/01/b5eddb33f85ff9c10046bdd4fbdf281ecc1e0f6c.jpeg

In above example presets has name photocategory_img. The main advantage of this approach is that there are no threats that unkind user will generate millions of image transformation but this approach is not agile.

On other side is **query** decoder. In this kind you provide operations in query string example:

[https://mort.mkaciuba.com/demo/img.jpg?operation=watermark&image=https://i.imgur.com/uomkVIL.png&position=top-left&opacity=0.5&width=500&operation=resize](https://mort.mkaciuba.com/demo/img.jpg?operation=watermark&image=https://i.imgur.com/uomkVIL.png&position=top-left&opacity=0.5&width=500&operation=resize)

Any user can change anything in URL.

Operation on images is described in such abstract way that you besides using built-in requests decoders there is a way to write you own. Documentation - [https://github.com/aldor007/mort/blob/master/doc/UrlParsers.md](https://github.com/aldor007/mort/blob/master/doc/UrlParsers.md)

# Request collapsing

![](https://mort.mkaciuba.com/media/blog/39386/79/thumb_eac1ee73-e307-11e7-922e-0242ac120002_blog_big1000.jpeg)

Generating new image can be expensive and you don’t want to create same image twice.

Because of this mort has built-in request collapse that makes sure only one request for the same image is processed.

After success result of an operation is stored on storage that decreases wait time for the image.

# Rate limiter

![](https://mort.mkaciuba.com/media/blog/9854/49/thumb_3abb3979-e308-11e7-922e-0242ac120002_blog_big1000.jpeg)

Another very useful options it that mort has requests limitter for image transformation.

Like I said in previous paragraph - generation new image may be very heavy operation and it will use many resources. Mort allow you to limit parallel image creation by using bucket throttling. This was important for me because I’m running mort and my others domains on same server.

# Links

More information about configuration and available image operations are provided [here](https://github.com/aldor007/mort/blob/master/doc/Image-Operations.md).

If you like mort don’t hesitate to give star on Github or fork it and add new feature.

[https://github.com/aldor007/mort](https://github.com/aldor007/mort)

PS. This page is using mort for serving static content. 
