---
layout: post
title: "Securing a Application Load Balancer with CloudFront, Enabling HTTPS, SSL Termination & Restricting Direct Access to a ALB"
date: 2023-08-11 08:00:00 - 0500
categories: aws
tags: security
image:
  path: /assets/images/
---

Welcome to the comprehensive guide where we are going to take a look at web application security and performance optimization. We'll walk you through a series of crucial enhancements for your Application Load Balancer (ALB):

1. `Securing Your ALB with HTTPS:` Discover how to improve your application's security by implementing HTTPS on your ALB. We'll guide you through the process of integrating a valid SSL/TLS certificate from AWS Certificate Manager (ACM), ensuring robust encryption and data integrity.

2. `Leveraging CloudFront As Your CDN:` Learn how to configure CloudFront as a content delivery network (CDN) to cache content directly from your ALB's origin. This strategic move reduces the load on your ALB, and significantly enhances user experience by decreasing latency.

3. `Enhancing Web Communication Security:` Setup of an Alternate Domain Name (CNAME) and associate a Custom SSL certificate with your CloudFront distribution. This step secures and streamlines your web communications.

4. `Enforcing HTTPS With CloudFront:` Improve your security by configuring CloudFront to exclusively use HTTPS when communicating with your ALB. This approach safeguards your data transfers, ensuring a secure and trustworthy environment for your users.

5. `Exclusive ALB Access Via CloudFront:` Finally, we'll show you how to configure CloudFront and your ALB to restrict direct user access to your ALB. By funneling user access through CloudFront, you'll maximize the security benefits and DDoS protection CloudFront offers. By enforcing HTTPS you prevent potential eavesdropping, keeping your header names and values confidential.

{% include embed/youtube.html id='pdwGsM2MmDU' %}
📺 [Watch Demo](https://www.youtube.com/watch?v=pdwGsM2MmDU)

## Objectives

- Set up a HTTPS Listener with a ACM Certificate on your Application Load Balancer
- Create a CloudFront Distribution using your Application Load Balancer as the Origin with HTTPS enabled
- Add an Alternate Domain Name (CNAME) and Custom SSL certificate to the CloudFront Distribution
- Configure CloudFront to include a custom HTTP header for Application Load Balance requests and configure the Application Load Balancer to only forward requests that contain the custom HTTP header
- Integrate a custom domain with your CloudFront Distribution

## Task 1: Setting up a HTTPS Listener with ACM Certificate on ALB

Begin by enhancing the security of your application by configuring a HTTPS Listener for your ALB, attaching a SSL/TLS Certificate, and setting up a redirect from HTTP to HTTPS.

- Navigate to the `EC2 Dashboard Console`
- In the left-hand navigation pane select Load Balancers and select your `Application Load Balancer`
- In the Listeners and rules tab, choose `Add listener`
- For Protocol : Port, choose `HTTPS and keep the default port`
- For Default actions, Choose `Forward to target group` and choose your `target group`

![img-description](/assets/images/task1.png)

For Security policy, AWS recommend that you keep the console recommended security policy.

- For Default SSL/TLS certificate, choose From `ACM` and choose `Request new ACM certificate`

A new page opens and you are redirected to the AWS Certificate Manager Console.

ACM is a regional service, when you request a public SSL/TLS certificate using ACM, the certificate is associated with the specific AWS region where you made the request. This means that the certificate is only usable within the region where it was issued.

- Under Certificates choose `Request`
- Under Certificate type choose `Request a public certificate`
- Choose `Next`
- Under Domain names for Fully qualified domain name enter `your domain name`
- For Validation method choose `DNS validation - recommended`
- Choose `Request`
- In Certificates choose the `refresh button`

![img-description](/assets/images/task1.1.png)

Your certificate should now be listed with a status of pending validation.

- Choose `View Certificate`
- Under Domains choose `Create Records in Route 53`
- Ensure the records are selected and choose `Create Records`

A green banner confirms that you Successfully created DNS records. To verify that the CNAME records were created and properly propagated in your DNS configuration navigate to the `Route 53 Dashboard`

- Under DNS management, choose `Hosted Zones` and choose your `Hosted zone`

![img-description](/assets/images/task1.2.png)

Verify that the CNAME records required for ACM's DNS validation are present and correct.

Keep in mind that DNS records might take some time to propagate across the DNS infrastructure. Usually, CNAME records propagate relatively quickly, but it's not uncommon for the process to take up to an hour or so. Be patient and recheck the records after some time if needed.

- Switch back to the Load Balancer page, and choose the `refresh button`
- From the dropdown panel choose your `certificate`
- Choose `Add`

![img-description](/assets/images/task1.3.png)

Your ALB should now be properly configured with HTTPS listeners, and associated with the SSL/TLS certificate from ACM.

Next, you will configure an HTTP listener rule that redirects HTTP requests to HTTPS.

- In Listeners and rules select the `HTTP:80 listener`
- Select `Manage listener` and in the drop down choose `Edit listener`
- Under Default actions, choose `Redirect to URL`
- For Protocol : Port, choose `HTTPS` and for port number enter: `443`
- Choose `Save changes`

## Task 2: Creating A CloudFront Distribution

In the next steps, you'll set up a CloudFront Distribution using your Application Load Balancer as the origin, enabling HTTPS for secure communication. You'll also add an Alternate Domain Name, associate a Custom SSL certificate, and configure CloudFront to include a custom HTTP header in requests sent to the Application Load Balancer to to prevent users from directly accessing your Application Load Balancer.

- Navigate to the `CloudFront console`
- Choose Create a `CloudFront distribution`

`Note:` If a setting is not specified in the following steps, use the default.

1. Configure the `Origin settings` for the distribution.

- Origin domain: Search for and choose your `Application Load Balancer` under Elastic Load Balancer
- In Protocol choose `HTTPS`

2. Choose Add custom header and configure the following:

- For Header name: enter: `X-Custom-Header`
- For Value enter: `random-value-1234567890`

3. Configure the Default cache behavior settings.

- Path pattern: Keep the Default `(*)` setting. This means that all requests will go to the origin
- Viewer protocol policy: Choose `Redirect HTTP to HTTPS.` This will help users find the website, even if they load the HTTP URL.
- Allowed HTTP methods: Keep the default `GET`, `HEAD` setting.
- Restrict viewer access: Keep the default `No setting.` This will be a public website.

4. Configure Cache key and origin requests

- Keep the default, `Cache policy and origin request policy (recommended)`
- For Cache policy, choose `Cache Optimized`
- For Origin request policy - optional, choose `Create origin request policy`

The Create origin request policy page opens.
Including the Host header in the cache key when creating a custom Cache policy in CloudFront is essential to enable HTTPS between CloudFront and your Application Load Balancer using your Application Load Balancers DNS name as the origin. This ensures the correct SSL certificate is used based on the requested domain name and allows proper isolation and caching for different origins served by the same Application Load Balancer.

- In the details section for name enter: `HTTPS-ALB-Cache-Policy`
- Under headers choose, `include the following headers`
- For add headers, choose `Host`
- Choose `Create`

- Return to the Cloudfront page, and under Origin request policy - optional, choose the `refresh button`
- Choose `HTTPS-ALB-Cache-Policy`

5. Configure Web Application Firewall (WAF) settings

- Choose `Do not enable security protection`

6. Configure the Settings section of the distribution

- Price class: Keep the default `Use all edge locations setting`
- Alternative domain name (CNAME): `Specify your unique domain name here`
- Custom SSL certificate: Choose, `Request certificate`

The AWS Certificate Manager console opens.

To use an ACM certificate with Amazon CloudFront, you must request a public certificate in the US East (N. Virginia) region. ACM certificates in this region that are associated with a CloudFront distribution are distributed to all the geographic locations configured for that distribution.

- Under Certificates choose `Request`
- Under Certificate type choose `Request a public certificate`
- Choose `Next`
- Under Domain names for Fully qualified domain name enter `your domain name`
- For Validation method choose `DNS validation - recommended`
- Choose `Request`
- In Certificates choose the `refresh button`

Your certificate should now be listed with a status of pending validation.

- Choose `View Certificate`
- Under Domains choose `Create Records in Route 53`
- Ensure the records are selected and choose `Create Records`

A green banner confirms that you Successfully created DNS records.

- Return to the Cloudfront page, and under Custom SSL certificate - optional, choose the `refresh button`
- Choose the `Certificate` you just created.
- Scroll to the bottom and, choose `Create distribution`

While the distribution is being created, Last modified at the top of the page displays Deploying.

`Note:` It might take 5 to 15 minutes for the deployment to complete.

## Task 3: Configuring An Application Load Balancer to Forward Requests Containing A Custom HTTP Header

After configuring CloudFront to add a custom HTTP header to the requests that it sends to your Application Load Balancer, you can configure your load balancer to only forward requests that contain the custom header to prevent users from directly accessing your Application Load Balancer. You do this by adding a new rule and modifying the default rule in your load balancer's listener.

- To update the rules in an Application Load Balancer listener open the `Load Balancers page` in the `Amazon EC2 console`
- Choose the `load balancer` that is the origin for your CloudFront distribution
- Choose`Listeners and rules tab`
- Choose `listener HTTPS 443`
- Choose `rules` and in the drop down choose `add rule`
- Under Names and tags enter: `Forward-Custom-Header-Rule`
- Choose `Next`

In the `Define rule conditions` section:

- Choose `Add condition` and then in Rule condition types, choose `HTTP header`
- For HTTP header name, enter `X-Custom-Header`
- For HTTP header value, enter `random-value-1234567890`

These values should be the same you entered in your CloudFront Distributuon.

- Choose `Confirm`
- Choose `Next`

In the `Define rule actions section`, under action types:

- choose `Forward to target groups` and, choose your `target group`
- Choose `Next`

In `Set rule priority:`

- In priority enter `1`
- Choose `Next`

In `Review and create:`

- Choose `Create`

Next, you will edit the default rule.

- Scroll down and under `Listener rules` select the `default rule` and choose `Actions` and `edit rule`
- Under `Listener details`, for `Default actions`, choose `Return fixed response`
- For Response code, enter `403`
- For Response body, enter `Access Denied`
- Choose `Save changes`

![img-description](/assets/images/task1.4.png)

After you complete these steps, your ALB should have two rules, as shown above.

1. The first rule forwards requests that contain the HTTP header (requests that come from CloudFront)
2. The second rule sends a fixed response to all other requests (requests that don't come from CloudFront)

## Task 4: Integrating Your Custom Domain With Your CloudFront Distribution

In the final step you will integrate your custom domain with a CloudFront Distribution using Route 53's Alias records.

- Navigate to `Route 53`
- Choose `Hosted zones`
- Choose the hosted zone for your domain
- Under Records, choose `Create record`
- For Routing policy, choose `Simple`
- Choose `Next`
- Under Configure records, choose `Define simple record`
- For Record name, enter the `domain` or `subdomain` that you want to use to route traffic to your ALB.

`Note:` The default value is the name of the hosted zone

- For Value/Route traffic to, Choose `Alias to CloudFront distibution`

Choosing an Alias record in Amazon Route 53 allows you to create a DNS record that points to specific AWS resources, such as your ALB.

- For choose region, choose the `region` where your ALB was configured
- In the dropdown box, choose your `ALB`
- Choose `Define simple record`
- In configure records, choose `Create records`

## Task 5: Verifying Solution

You can verify that the solution works by testing that traffic your application can flow only through CloudFront. This can be done using your standard Internet browser. The request to CloudFront returns your web application or content, and the one sent directly to your Application Load Balancer returns a 403 response with the plain text message Access Denied.

What you see now is that by making Application Load Balancer to forward only requests from CloudFront to your web servers behind it you can be confident that nobody can bypass your Edge perimeter. Surface of any attack is greatly reduced and you can focus on building up the security of the perimeter.
