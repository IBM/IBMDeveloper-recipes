# A best Practice in Dockerizing Vue.js Application

## A best Practice in Dockerizing Vue.js or Node.js Application

[Jeya Gandhi Rajan](https://developer.ibm.com/recipes/author/jeyagandhi/)
Published on March 2, 2020 / Updated on March 2, 2020

## Overview

Skill Level: Any Skill Level

This recipe explains a very important Best Practice in Dockerizing the Vue.js application. The same could be applicable for any Node.js applications as well.

## Step-by-step

### About Vue.js

Vue.js is a great JavaScript framework to quickly get started building single page applications in JavaScript. Its simplicity makes it easy to get started, but it’s robust enough to build large production applications.

### Create Vue App

Here are the some steps that we will follow to create a new Vue App.


1. Install Vue Cli
`$ npm install -g @vue/cli`
1. Create new App
`$ vue create vue-app`
2. Build and test the App
```
$ cd vue-app

$ npm run build
```
This will create a production ready minimalistic version of your single page application in /dist folder. You will have index.html in it.

### Dockerize the App

Here is the Normal process for Dockerizing the app

1. Choose the FROM Image. Normally we use node image.
```
FROM node:lts-alpine
```
1. Then we create workdirectory
```
WORKDIR /app
```
1. Install your app dependencies
```
COPY package*.json ./

RUN npm install
```
1. Copy the Vue app files.
```
COPY ./dist .
```
1. Define the command to run your app
```
RUN npm run build
```
1. The final Dockerfile looks like this.
```
FROM node:lts-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY ./dist .

EXPOSE 8080

RUN npm run build
```
1. Create a docker image using this Dockerfile

1. Deploy the image in Kubernetes or in Openshift

1. Expose the app as Service using NodePort.

1. Access the service using NodePort.

### Issues with this Dockerization and Solution

Exposing a service through NodePort is not a good practice. So we need to find other way to expose it.

Openshift uses a concept called Route to expose any service externally. We can use this route to expose the deployed service.

But with the above created image, when you access the application through the route, you will get an error called “Invalid Host Header”.

This is because the image is not wrapped with Webserver. Without a webserver an application can’t be accessed from outside via Proxy (route).

Always bundle the image with a webserver in order to expose your service external to the cluster.

Lets bundle the image with NGINX webserver.

### Best way to Dockerize

1. Use NGINX as FROM Image and copy the app code into the appropriate folder.

The right  Dockerfile looks like this.   
```
# Dockerfile

FROM nginx:1.17

COPY ./nginx.conf /etc/nginx/nginx.conf

WORKDIR /app

COPY ./dist .

    EXPOSE 8080:8080

    CMD [“nginx”, “-g”, “daemon off;”]
```

Now Vue.js app is wrapped with NGINX webserver in the image. So the NGINX Webserver will serve the pages when app is accessed.

1. Build the docker image, deploy in Openshift and create a route for this service

The application would work without any issue.

## Conclusion

If you want to expose any of your service to external to the cluster, it is always wrap your app with NGINX kind of webserver.

## References

Here are some references about this issue

https://blog.openshift.com/deploy-vuejs-applications-on-openshift/
