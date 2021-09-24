# How to Upload Website Files to a Server
## A Quick Guide to Website Hosting

Ilai Bavati

Tags: Web development

Published on September 17, 2019 / Updated on November 10, 2020

![](images/download-2013201_1280-768x768.png)

### Overview

Skill Level: Beginner

Learn how to upload website files to a server, including how to choose a hosting service and an upload method, how to upload and extract the site archive, how to move your files to the public_html directory, and how to ensure you covered all the steps.

### Ingredients

No prerequisites.

The web is based on a foundation of different files. We are constantly creating and sharing files with one another, whether it is images, videos, Word documents, text documents, or programming files.

If you are a web developer, file uploads are one of the first steps in most web projects. There is an endless number of methods, tools and services to host your files on a web server. This tutorial will provide general steps on how to upload website files using the most common tools.

### Step-by-step

#### 1. Choose a Reliable Hosting Service

Selecting a good hosting service can make your website development easier. Hosting services are responsible for website speed and stability, security and upload time. Choose your host wisely, to avoid issues with load time, capacity, storage, security, and support.

Here are a few important features that you should consider in a web host:

**Hosting that suits your needs**

First, figure out how much resources your website needs. For a personal blog or small website, you will need only the basic hosting capabilities. Shared hosting services can offer small websites a cost-effective solution.

Large businesses or eCommerce stores might need a Virtual Private Server (VPS) hosting or a dedicated server. This kind of web host manages large volumes of traffic and provides extra reliability, including fully managed services.

At the end of the day, each choice has its own features and cost. Make sure to match your website requirements with the options of a hosting service provider.

**Server reliability and uptime**

People from different time zones may visit your site. That is why you need to maintain a fully operational website 24 hours a day. To provide a fully operational website around the clock, a web host must be stable, both in terms of servers and network connection. Uptime score of 99.95% is a standard these days, anything below 99% is unacceptable.

**Hosting server types**

There are different hosting server types available—shared, VPS, dedicated, and cloud hosting. Shared storage is the best choice for websites with less than 5,000 visitors per month. For websites with more visitors, you might want to move to a more powerful server or cloud hosting.

**Site Backup**

A hosting provider should backup your website on a regular basis. Available backup can solve a lot of problems in case of a critical data loss or a cyberattack. Often it does not take more than a few minutes to get that backup file ready.

**Server Responsiveness**

Responsiveness is the time it takes for a server to acknowledge a domain name request. The server response speed is often referred to as Time To First Byte (TTFB). TTFB is a significant factor in having a fast loading website. Visitors of a slow loading website will leave the site before it even finishes loading.

#### 2. Choose your Website Upload Method

**File managers**

File managers are tools that allow users to manage their websites via HTTP instead of the FTP or third party applications. File managers provide the basic functionality required to manage a website. They manage file creation, upload, permissions, and organization. These tools have limited upload size. Usually, files larger than 256MB are not supported.

**File Transfer Protocol (FTP)**

File Transfer Protocol (FTP) is a method to access and share files. The protocol is a way to communicate between computers on a TCP/IP network. Website developers use FTP to make changes on a website. For example, to move a web template, add image files, upload files to the website and more.

**PHP file upload**

There are two main ways to perform [file upload in PHP](https://cloudinary.com/features/file_upload_storage).

*   The hard way, which includes building an HTML form that allows users to submit files, and then creating a PHP file upload script to handle the files.
*   The simple way involves using a PHP file upload service that gives you full control of the process.

#### 3. Upload and Extract the Site Archive

Now it’s time to get your hands dirty and actually upload the site with a tool of your choice. Some web servers like FTP use multiple connections to simultaneously upload 10 files. Other web servers enforce connection limits. When you exceed the connection limit, the firewall automatically bans your IP address for a few minutes.

In some cases like a software update, you will need to upload a few thousand files. It takes a long time to transfer a large number of files if you have a connection limit. This is why dedicated servers allow you to increase the connection limit or to archive files in a single ZIP using your FTP client software and then use a PHP script to extract the file.

#### 4. Move your Files to the public\_html directory

Now make sure that all your files are located in the public\_html directory. The public\_html directory is the root folder for web-readable files.

This means that public\_html is the folder where website files are stored. Files stored in this folder will be visible to anyone who searches for your domain name. Files outside the public\_html folder will not be visible to users.

#### 5. Check if the Website Works

After the file upload, you need to check if the website works. There are a few ways to test whether everything works:

1.  Emulate DNS changes using the hosts file. Usually, websites do not have their own IP address. The hosts file can be used to set domain name on the IP address until the domain is registered worldwide.
2.  Check website availability with online tools. There are a lot of online tools like [Pingdom](https://www.pingdom.com/) or [Uptrends](https://www.uptrends.com/tools/uptime). Paste your domain name into the tool and run the test.
3.  Use browser plugin like [Virtual Hosts](https://chrome.google.com/webstore/detail/virtual-hosts/aiehidpclglccialeifedhajckcpedom) to test DNS changes. All you need is the domain name and your account’s IP address.

#### 6. Conclusion

In this tutorial, you learned how to upload a website file to a server. You can upload multiple files at once using an FTP client. However, each file will be uploaded one-by-one and may take a while. On the other hand, the File Manager tool is extremely useful for importing one or two files quickly, making code tweaks, or uploading a website that does not exceed 256MB in size.

Image Source: [Pixabay](https://pixabay.com/illustrations/download-icon-download-icon-network-2013201/)
