# r6-technology-site
This project manages static website website content and cloud infrastructure for hosting it. The site is built with [Zola](https://getzola.org) and runs on AWS using CloudFront and S3. For detailed deploy steps, check out [the blog post I've written about hosting Zola sites on AWS](https://r6.technology/posts/quickly-deploying-zola-static-sites-to-aws/).

TL;DR:
| Component               | Path                       | Note                                  |
|--------------------------|----------------------------|---------------------------------------|
| CloudFront Secure Static Site solution | amazon-cloudfront-secure-static-site | AWS IaC submodule                      |
| Git patch                | cloudfront.patch          | Modifies the IaC submodule           |
| Zola theme               | themes/<theme>            | site style/layout submodule           |
| Zola project             | <everything else>         | Zola project content                 |

Basic deploy process:
 - Make changes to Zola content to modify the site
 - `zola build` the site into the AWS submodule's `www/` directory
 - Run AWS SAM [steps](https://github.com/aws-samples/amazon-cloudfront-secure-static-site?tab=readme-ov-file#customizing-the-solution) to generate and deploy templates
