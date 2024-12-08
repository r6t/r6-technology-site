+++
title = "Deploying Zola static sites on AWS (CloudFront and S3)"
date = 2024-12-06T22:00:00+00:00

[taxonomies]
tags = ["Zola", "AWS"]
+++

This post walks through a solution for deploying maintainable Zola static sites on AWS (CloudFront and S3).
<!--more-->

#### Background
The [Zola](https://www.getzola.org) static site generator came onto my radar in late 2024. A rust-based website engine that generates sites from static files, Zola claims to have speed, scalability, flexibility, and ease of use. Those claims have all proven true for me, and I also find their theme options make the creation of simple + good-looking sites straightforward for engineer-types that typically don't design web pages. Personally, I have found myself needing a few simple websites this year, and while I was able to quickly put something out there with a managed Wordpress account, it was not an ideal solution. It was clunky to work with. Changes were too manual. The solution contained default components (like a database) that ultimately weren't needed by my simple websites.

When I heard about Zola, it immediately piqued my interest and seemed obviously more suitable for what I needed. The same could be said for other static site generators like Hugo. Seeing that I can easily version control the entire site and bundle it with infrastructure-as-code, this would provide the type of management and control that I've grown used to while working in DevOps roles over the past decade. Storing configurations and infrastructure as code helps keep systems adaptable to their evolving needs; there are many other benefits to this more code-centric approach as well. The point of this blog post is running Zola on AWS though, so let's dive into that.

#### Zola
As shown in the [Zola Overview](https://www.getzola.org/documentation/getting-started/overview), once you have it running on your system, getting started is little more than a `zola init` and `zola serve` away. From there, a theme can be added by declaring it in `config.toml` and bringing the theme contents in to the new Zola project. Then, adjust the config and content as you see fit until your static site looks just right. When you're rendering the site locally, it will refresh as you write changes to project files, allowing live changes without restarting the webserver process.

Some Zola themes suggest cloning their theme repo into your project. Instead of that, I recommend using git submodules to bring themes in, even if the theme you are using does not mention it in their setup docs. In each of my own Zola projects, 2 submodules are used: one for the site theme, and a second for the infrastructure part of this solution.

```
# Add a Zola theme
git submodule add git@github.com:en9inerd/zola-hacker.git themes/hacker
```

#### AWS
What is that infrastructure part I just mentioned? While I was initially exploring Wordpress hosting, I kept going back to thinking that a basic S3 website would meet my needs without extra stuff. Zola (or other static site generators) make simple static sites more appealing as they can help with the presentation and graphical design side of things, creating a good looking baseline to put some static content on top of. Zola pages + S3 static hosting seem like a natural fit. Fortunately, AWS provides an example SAM template for static sites in S3 that are fronted by CloudFront (AWS' global CDN) and include domain and certificate resources, as well as the policies and permissions for making this all work. It works almost perfectly for this out of the box.

It's time for that second submodule!  It can be added like this:

```
# Add aws-samples CloudFront static site SAM example
git submodule add git@github.com:aws-samples/amazon-cloudfront-secure-static-site.git amazon-cloudfront-secure-static-site
```

#### Fix Zola link handling for CloudFront
The factor that makes the AWS sample template only almost perfect has to do with link handling. Zola's default structure links to pages by directory name, and creates pages as index.html documents under those directories. This introduces an issue for CloudFront hosted sites that use S3 origins. CloudFront sites allow specification of the default root object, allowing the setting of index.html or another file as your root object. This only applies at the top level though. Unlike standard S3 website hosting, CloudFront sites backed by standard S3 origins do not append /index.html or a specified default object outside of the top level root object. So when the zola-hacker theme links to `posts/blah/`, or the zallery theme links to `artwork/blah` (other themes will have similar patterns), those links go to CloudFront AccessDenied error pages because `/index.html` is omitted from the link path. One way to correct this is introducing CloudFront Functions that run on the viewer-request event and append /index.html when necessary. I've solved for this by adding a patch that will apply to the AWS submodule:

```
# cloudfront-function.patch
diff --git a/templates/cloudfront-site.yaml b/templates/cloudfront-site.yaml
index 3ea3c9a..4c8407b 100644
--- a/templates/cloudfront-site.yaml
+++ b/templates/cloudfront-site.yaml
@@ -52,6 +52,28 @@ Resources:
               StringEquals:
                 'AWS:SourceArn': !Sub 'arn:aws:cloudfront::${AWS::AccountId}:distribution/${CloudFrontDistribution}'

+  CloudFrontFunction:
+    Type: AWS::CloudFront::Function
+    Properties:
+      Name: !Sub "${AWS::StackName}-URLRewriter"
+      FunctionConfig:
+        Comment: "Rewrite URLs to append /index.html"
+        Runtime: "cloudfront-js-1.0"
+      AutoPublish: true
+      FunctionCode: |
+        function handler(event) {
+          var request = event.request;
+          var uri = request.uri;
+
+          // Check if the URI matches Zola posts pattern, including optional trailing slash
+          if (uri.match(/^\/posts\/[^\/]+\/?$/)) {
+            // Remove trailing slash if present, then append index.html
+            request.uri = uri.replace(/\/$/, '') + '/index.html';
+          }
+
+          return request;
+        }
+
   CloudFrontDistribution:
     Type: AWS::CloudFront::Distribution
     Properties:
@@ -68,6 +90,9 @@ Resources:
           TargetOriginId: !Sub 'S3-${AWS::StackName}-root'
           ViewerProtocolPolicy: 'redirect-to-https'
           ResponseHeadersPolicyId: !Ref ResponseHeadersPolicy
+          FunctionAssociations:
+            - EventType: viewer-request
+              FunctionARN: !GetAtt CloudFrontFunction.FunctionARN
         CustomErrorResponses:
           - ErrorCachingMinTTL: 60
             ErrorCode: 404
```
By keeping a file like this in your project, you can copy it into the submodule directory and run `git apply cloudfront-function.patch` to modify the AWS sample template to include a CloudFront Function (that incurs [nominal or no cost](https://aws.amazon.com/cloudfront/pricing/) for low-traffic sites) which will handle Zola links. It is key to run this patch before running the AWS build/package/deploy steps, as those will modify the submodule. There are other ways to address this, and the CloudFront Function shown in this patch file can be applied other ways if you're not using the same submodule approach.

#### Deploy
Once your Zola project is in a good spot, clear out the AWS example site content from the SAM submodule and build your site in its' place.
```
# Remove AWS example content
rm -r <project directory>/amazon-cloudfront-secure-static-site/www/*

# Build Zola site into IaC (force because the directory already exists)
zola build --output-dir <project directory>/amazon-cloudfront-secure-static-site/www/ --force
```
If you've followed along so far, you're ready to deploy the site by changing into infrastructure submodule directory (`amazon-cloudfront-secure-static-site` in my case) and running the SAM packaging and deploying commands detailed in the "Customizing the Solution" section of their [README](https://github.com/aws-samples/amazon-cloudfront-secure-static-site?tab=readme-ov-file#customizing-the-solution). If all goes goes well, your site will be live when the `deploy` command completes. The site can be updated by repeating these steps. This has potential to incur some AWS costs with pages that see a lot of hits, so be aware of your usage. It's not going to be the cheapest way to get a Zola site live on the internet, however I think it strikes a fantastic balance of cost, maintainability, reliability, and scalability. If I decide to run this site somewhere else in the future, Zola sites are also easily portable to other means of hosting.

For a working example, you can check out the [source behind this blog](https://github.com/r6t/r6-technology-site).
