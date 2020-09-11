# Lab 2: Docker

## Exploring Docker container registries

Every docker image is build upon a base image which is defined in the Dockerfile.

Now, let's try to find the base image from the following Dockerfile:

```Dockerfile
FROM registry.access.redhat.com/ubi8/nodejs-10

# Change working directory
WORKDIR /opt/app-root/src/

# Install npm production packages
COPY package*.json /opt/app-root/src/
RUN cd /opt/app-root/src; npm ci

COPY . /opt/app-root/src/

ENV NODE_ENV production
ENV PORT 3000

EXPOSE 3000

CMD ["npm", "start"]
```

Have a look at the Dockerfile of the base image!

Hint: There are two well known public container registries:

- [Official redhat registry](https://catalog.redhat.com/software/containers/search)
- [Docker Hub](https://hub.docker.com/)

Be aware, that everybody can upload images to Docker Hub, so use them with caution! All the images from the redhat registry are certified and should be used whenever possible.

Now, explore the container registry. Do you find an image that you might use for one of your own applications?
