+++
title = "Quickly deploying Zola static sites on AWS (CloudFront and S3)"
date = 2024-12-06T22:00:00+00:00

[taxonomies]
tags = ["Zola", "AWS"]
+++

This post walks through a solution for quickly deploying Zola static sites on AWS (CloudFront and S3)
<!--more-->

#### Background
The [Zola](https://www.getzola.org) static site generator came onto my radar in late 2024. A rust-based website engine that generates sites from static files, Zola claims to have speed, scalability, flexibility, and ease of use. Those claims have all proven true for me, and I also find their theme options make the creation of simple + good-looking sites straightforward for engineer-types that typically don't design web pages. Personally, I have found myself needing a few simple websites this year, and while I was able to quickly put something out there with a managed Wordpress account, it was not an ideal solution. It was clunky to work with. Changes were too manual. The solution contained default components (like a database) that ultimately weren't needed by my simple websites.

When I heard about Zola, it immediately peaked my interest and seemed obviously more suitable for what I needed. (The same could be said for other static site generators like Hugo.) Seeing that I can easily version control the entire site and bundle it with infrastructure-as-code, this would provide the type of management and control that I've grown used to while working in DevOps roles over the past decade. Storing configurations and infrastructure as code helps keep systems adaptable to their evolving needs; there are many other benefits to this more code-centric approach as well. The point of this blog post is running Zola on AWS though, so let's dive into that.

#### Zola
As we can see in the [Zola Overview](https://www.getzola.org/documentation/getting-started/overview), once you have it running on your system, getting started is little more than a `zola init` and `zola serve` to get it running locally. From there, we can add a theme by declaring it in `config.toml` and bringing the theme contents in to the new Zola project. Then, adjust the config and content as you see fit until your static site looks just right. When you're rendering the site locally, it will refresh as you write changes to project files, allowing live changes without restarting the webserver process.

Some Zola themes suggest cloning their theme repo into your project. Instead of that, I recommend using git submodules to bring themes in, even if the theme you are using does not mention it in their setup docs. In each of my own Zola projects, 2 submodules are used: one for the site theme, and a second for the infrastructure part of this solution.

```
# Add a Zola theme
git submodule add git@github.com:en9inerd/zola-hacker.git themes/hacker
```

#### AWS
What is that infrastructure part I just mentioned? While I was initially exploring Wordpress hosting, I kept going back to thinking that a basic S3 website would meet my needs without extra stuff. Zola (or other static site generators) make simple static sites more appealing as they can help with the presentation and graphical design side of things, creating a good looking baseline to put some static content on top of. Zola pages + S3 static hosting seem like a natural fit. Fortunately, AWS provides an example SAM template for static sites in S3 that are fronted by CloudFront (AWS' global CDN) and include domain and certificate resources, as well as the policies and permissions for making this all work. It works perfectly for this.

It's time for that second submodule!  It can be added like this:

```
# Add aws-samples CloudFront static site SAM example
git submodule add git@github.com:aws-samples/amazon-cloudfront-secure-static-site.git amazon-cloudfront-secure-static-site
```

Once your Zola project is in a good spot, clear out the AWS example site content from the SAM submodule and build your site in its' place.

```
# Remove AWS example content
rm -r <project directory>/amazon-cloudfront-secure-static-site/www/*

# Build Zola site into IaC (force because the directory already exists)
zola build --output-dir amazon-cloudfront-secure-static-site/www/ --force
```

If you've followed along so far, you're ready to deploy the site by changing into infrastructure submodule directory (`amazon-cloudfront-secure-static-site` in my case) and running the SAM packaging and deploying commands detailed in the "Customizing the Solution" section of their [README](https://github.com/aws-samples/amazon-cloudfront-secure-static-site?tab=readme-ov-file#customizing-the-solution). If all goes goes well, your site will be live when the `deploy` command completes. The site can be updated by repeating these steps. This will incur nominal cost for low-traffic sites. It's not going to be the cheapest way to get a Zola site live on the internet, however I think it strikes a fantastic balance of cost, maintainability, reliability, and scalability. If I decide to run this site somewhere else in the future, Zola sites are also easily portable to other means of hosting.

For a working example, you can check out the [source behind this blog](https://github.com/r6t/r6-technology-site).
