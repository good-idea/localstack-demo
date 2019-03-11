---
title: How to fake AWS locally with LocalStack
published: true
description: A brief tutorial on setting up LocalStack + Node to simulate Amazon S3 locally
tags: node, tutorial, aws, docker
---

If you're anything like me, you prefer to avoid logging into the AWS console as much as possible. Did you set up your IAM root user with 2FA and correctly configure the CORS and ACL settings on your S3 bucket?

# 🤷‍♂️ nah.

I also prefer to keep my local development environment as close as possible to how it's going to work in production. Additionally, I'm always looking for new ways to fill up my small hard drive. I can't think of a better away to achieve all of the above than putting a bunch of S3 servers inside my computer.

This tutorial will cover setting up [Localstack](https://github.com/localstack/localstack) within a node app. Localstack allows you to emulate a number of AWS services on your computer, but we're just going to use S3 in this example. Also, Localstack isn't specific to Node - so even if you aren't working in Node, a good portion of this tutorial will still be relevant. This also covers a little bit about Docker - if you don't really know what you're doing with Docker or how it works, don't worry. Neither do I.

You can see the [demo repo](https://github.com/good-idea/localstack-demo) for the finished code.

A few benefits of this approach are:

 - You can work offline
 - You don't need a shared 'dev' bucket that everyone on your team uses
 - You can easily wipe & replace your local buckets
 - You don't need to worry about paying for AWS usage
 - You don't need to log into AWS 😛

## Initial Setup

First, we'll need to install a few things.

1. Install [Docker](https://docs.docker.com/install/) if you haven't already.
2. Install the [AWS CLI](https://aws.amazon.com/cli/). Even though we aren't going to be working with "real" AWS, we'll use this to talk to our local docker containers.
3. Make a few files. Create a new directory for your project, and within it: `touch index.js docker-compose.yml .env && mkdir .localstack`
4. Add an image to your project directory and rename it to `test-upload.jpg`
5. `npm init` to set up a package.json, then `npm install aws-sdk dotenv`

## Docker

(disclaimer: I'm not a docker expert. If anyone has any suggestions on how to improve or better explain any of this, please let me know in the comments!)


### Docker Config

You can run Localstack directly from the command line, but I like using Docker because it makes me feel smart. It's also nice because you don't need to worry about installing Localstack on your system. I prefer to use docker-compose to set this up. Here's the config:

`docker-compose.yml` 

```
version: '3.2'
services:
  localstack:
    image: localstack/localstack:latest
    container_name: localstack_demo
    ports:
      - '4563-4584:4563-4584'
      - '8055:8080'
    environment:
      - SERVICES=s3
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - './.localstack:/tmp/localstack'
      - '/var/run/docker.sock:/var/run/docker.sock'
```

Breaking some of these lines down:

#### `image: localstack/localstack:latest`

Use the latest [Localstack image from Dockerhub](https://hub.docker.com/r/localstack/localstack/) 


#### `container_name: localstack_demo`:

This gives our container a specific name that we can refer to later in the CLI.


#### `ports: '4563-4584:4563-4584'` and `'8055:8080'`:

When your docker container starts, it will open up a few ports. The number on the **left** binds the port on your `localhost` to the port within the container, which is the number on the **right**. In most cases, these two numbers can be the same, i.e. `8080:8080`. I often have some other things running on `localhost:8080`, so here, I've changed the default to `8055:8080`. This means that when I connect to `http://localhost:8050` within my app, it's going to talk to port `8080` on the container.

The line `'4563-4584:4563-4584'` does the same thing, but binds a whole range of ports. These particular port numbers are what Localstack uses as endpoints for the various APIs. We'll see more about this in a little bit.

#### `environment`

These are environment variables that are supplied to the container. Localstack will use these to set some things up internally:

- `SERVICES=s3`: You can define a list of AWS services to emulate. In our case, we're just using S3, but you can include additional APIs, i.e. `SERVICES=s3,lambda`. There's more on this in the Localstack docs.
- `DEBUG=1`: 🧻 Show me all of the logs!
- `DATA_DIR=/tmp/localstack/data`: This is the directory where Localstack will save its data *internally*. More in this next:


#### `volumes`

`'./.localstack:/tmp/localstack'`

Remember when set up the `DATA_DIR` to be `/tmp/localstack/data` about 2 seconds ago? Just like the `localhost:container` syntax we used on the ports, this allows your containers to access a portion of your hard drive. Your computer's directory on the left, the container's on the right.

Here, we're telling the container to use our `.localstack` directory for its `/tmp/localstack`. It's like a symlink, or a magical portal, or something.

In our case, this makes sure that any data created by the container will still be present once the container restarts. Note that `/tmp` is cleared frequently and isn't a good place to store. If you want to put it in a more secure place
- `'/var/run/docker.sock:/var/run/docker.sock'`

(edit! I accidentally published an unfinished draft. This post isn't complete. Check back in a little bit!)

### Starting our Container

Now that we have our `docker-compose.yml` in good shape, we can spin up the container: `docker-compose up -d`.

To make sure it's working, we can visit http://localhost:8055 to see Localstack's web UI. Right now it will look pretty empty:

![Empty Localstack UI](https://thepracticaldev.s3.amazonaws.com/i/0ym8w6yneym9nh98xo0e.png)

Similarly, our S3 endpoint http://localhost:4572 will show some basic AWS info:

![Empty S3 bucket](https://thepracticaldev.s3.amazonaws.com/i/mmqvbgjrcgs2bhqguxi4.png)


(If you don't see something similar to these, check the logs for your docker containers)


## Working with Localstack

AWS is now inside our computer. You might already be feeling a little bit like you are [the richest person in the world](https://www.businessinsider.com/amazon-ceo-jeff-bezos-richest-person-net-worth-billions-2018-12). (If not, don't worry, just keep reading 😛)

Before we start uploading files, we need to create and configure a bucket. We'll do this using the AWS CLI that we installed earlier, using the `--endpoint-url` flag to talk to Localstack instead.

1. Create a bucket: `aws --endpoint-url=http://localhost:4572 s3 mb s3://demo-bucket`
2. Attach an [ACL](https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html) to the bucket so it is readable: `aws --endpoint-url=http://localhost:4572 s3api put-bucket-acl --bucket demo-bucket --acl public-read`

Now, when we visit the web UI, we will see our bucket:

![Localstack UI with S3 Bucket](https://thepracticaldev.s3.amazonaws.com/i/7lrpsp1n71xhead0xll1.png)

If you used `volumes` in your docker settings, let's pause for a moment to look at what's going on in `./.localstack/data`.

![Localstack S3 JSON](https://thepracticaldev.s3.amazonaws.com/i/12dwa5s12m1of06w4qcu.png)

Here, we can see that Localstack is recording all API calls in this JSON file. When the container restarts, it will re-apply these calls - this is how we are able to keep our data between restarts. Once we start uploading, we won't see new files appear in this directory. Instead, our uploads will be recorded in this file *as raw data*. (You could include this file in your repo if you wanted to share the state of the container with others - but depending on how much you upload, it's going to become a pretty big file)

If you want to be able to "restore" your bucket later, you can make a backup of this file. When you're ready to restore, just remove the updated `s3_api_calls.json` file, replace it with your backup, and restart your container.

### Uploading from our app

There are a lot of S3 uploading tutorials out there, so this section won't be as in-depth. We'll just make a simple `upload` function and try uploading an image a few times.

Copy these contents into their files:

**.env**, our environment variables

```
AWS_ACCESS_KEY_ID='123'
AWS_SECRET_KEY='xyz'
AWS_BUCKET_NAME='demo-bucket'
```

*Note: it doesn't matter what your AWS key & secret are, as long as they aren't empty.*

**aws.js**, the module for our upload function

```js
const AWS = require('aws-sdk')
require('dotenv').config()

const credentials = {
   accessKeyId: process.env.AWS_ACCESS_KEY_ID,
   secretAccessKey: process.env.AWS_SECRET_KEY,
}

const useLocal = process.env.NODE_ENV !== 'production'

const bucketName = process.env.AWS_BUCKET_NAME

const s3client = new AWS.S3({
   credentials,
   /**
    * When working locally, we'll use the Localstack endpoints. This is the one for S3.
    * A full list of endpoints for each service can be found in the Localstack docs.
    */
   endpoint: useLocal ? 'http://localhost:4572' : undefined,
})


const uploadFile = async (data, name) =>
   new Promise((resolve) => {
      s3client.upload(
         {
            Bucket: bucketName,
            /*
               include the bucket name here. For some reason Localstack needs it.
               see: https://github.com/localstack/localstack/issues/1180
            */
            Key: `${bucketName}/${name}`,
            Body: data,
         },
         (err, response) => {
            if (err) throw err
            resolve(response)
         },
      )
   })

module.exports = uploadFile
```

**test-upload.js**, which implements the upload function

```js
const fs = require('fs')
const path = require('path')
const uploadFile = require('./aws')

const testUpload = () => {
   const filePath = path.resolve(__dirname, 'test-image.jpg')
   const fileStream = fs.createReadStream(filePath)
   const now = new Date()
   const fileName = `test-image-${now.toISOString()}.jpg`
   uploadFile(fileStream, fileName).then((response) => {
      console.log(":)")
      console.log(response)
   }).catch((err) => {
      console.log(":|")
      console.log(err)
   })
}

testUpload()
```

the `testUpload()` function gets the file contents, gives it a unique name based on the current time, and uploads it. Let's give it a shot:

`node test-upload.js`

![testing the upload](https://thepracticaldev.s3.amazonaws.com/i/yx94y5fj0j8rq4y67foy.png)

Copy the URL in the `Location` property of the response and paste it into your browser. The browser will immediately download the image. If you want to see it in your browser, you can use something like JS Bin:

![I love my dog](https://thepracticaldev.s3.amazonaws.com/i/v1vnvbzo7z0wlabm3o43.png)

Then, if you look at `.localstack/data/s3_api_calls.json` again, you'll see it filled up with the binary data of the image:

![Image binary data](https://thepracticaldev.s3.amazonaws.com/i/qcydypsebmmcb8r855jg.png)


**Finally**, let's restart the container to make sure our uploads still work. To do this, run `docker restart localstack_demo`. After it has restarted, run `docker logs -f localstack_demo`. This will show you the logs of the container (the `-f` flag will "follow" them).

After it initializes Localstack, it will re-apply the API calls found in `s3_api_calls.json`:

![Localstack Logs](https://thepracticaldev.s3.amazonaws.com/i/ydjjg14zztr5vvqv9q6s.png)

When you reload your browser, you should see the image appear just as before.

🎉 That's it! Thanks for sticking around. This is my first tutorial and I'd love to know what you think. If you have any questions or suggestions, let me know in the comments!