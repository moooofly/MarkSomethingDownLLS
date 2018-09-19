# Camo 详解

## 起因

在 github 上经常看到“动图”，例如

![](https://camo.githubusercontent.com/bdc860dbbe237022f883e63f231b7966b59d1a96/68747470733a2f2f63646e2e7261776769742e636f6d2f62617272797a2f676f63692f33373262636363622f64656d6f6e7374726174696f6e2e737667)

查看其链接信息中均有 https://camo.githubusercontent.com/xxx ，那么 camo 到底是什么呢？

## [About anonymized image URLs](https://help.github.com/articles/about-anonymized-image-urls/)

> If you upload an image to GitHub, the URL of the image will be modified so your information is not trackable.

当你上传图片到 GitHub 时，图片的 URL 可能会被修改，进而导致图片无法被正确访问；

> To host your images, GitHub uses the [open-source project Camo](https://github.com/atmos/camo). **Camo generates an anonymous URL proxy for each image** that starts with https://camo.githubusercontent.com/ and hides your browser details and related information from other users.

- GitHub 使用 Camo 提供图片服务；
- Camo 会为每一个图片都生成匿名 URL proxy ，并以 https://camo.githubusercontent.com/ 作为链接的起始部分；
- Camo 能够隐藏你的浏览器等相关信息，防止其他用户获取；

> Anyone who receives your anonymized image URL, directly or indirectly, may view your image. To keep sensitive images private, restrict them to a private network or a server that requires authentication instead of using Camo.

- 收到你 anonymized image URL 的任何人，都能够直接或间接查看你的图片；
- 若想确保图片的私密性，请将其限制在私有网络中，或者使用需要鉴权的服务器作为图片宿主机，而不是 Camo ；

## 使用 Camo 时可能遇到的问题

- An image is not showing up
- An image that changed recently is not updating
- Removing an image from Camo's cache
- Viewing images on private networks

## [Proxying User Images](https://blog.github.com/2014-01-28-proxying-user-images/)

> A while back, we started **[proxying all non-https images](https://blog.github.com/2010-11-13-sidejack-prevention-phase-3-ssl-proxied-assets/)** to avoid mixed-content warnings using a custom **node server** called [camo](https://github.com/atmos/camo). We’re making a small change today and **proxying HTTPS images as well**.

> **Proxying these images will help protect your privacy**: your browser information won’t be leaked to other third party services. Since we’re also routing images through our CDN, you should also see faster overall load times across GitHub, as well as fewer broken images in the future.

## [atmos/camo](https://github.com/atmos/camo) -- Ruby + CoffeeScript

一句话：an http proxy to route images through SSL

## [cactus/go-camo](https://github.com/cactus/go-camo) -- Golang

一句话：Go secure image proxy server

> 值得深入研究一下

## 使用

> TODO
