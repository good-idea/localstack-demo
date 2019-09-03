A basic demo setting up Localstack's S3.

Read the tutorial [here](https://dev.to/goodidea/how-to-fake-aws-locally-with-localstack-27me).

Or, spin things up here:

1. Install [Docker](https://docs.docker.com/install/) if you haven't already.
2. Install the [AWS CLI](https://aws.amazon.com/cli/). Even though we aren't going to be working with "real" AWS, we'll use this to talk to our local docker containers.
  - Run `aws configure` to set up some credentials. You can enter dummy credentials here if you'd like.
3. Copy the contents of `.env.example` into a new `.env` file. 
4. Initialize Localstack: `npm run localstack:init`.
  - This will create a new container, then stream the logs as it is setting up. It will start with `Waiting for all LocalStack services to be ready`. After a few moments, you'll see a final `Ready`. When you do, press Ctrl+C to exit the logs.
6. Configure the bucket: run `npm run localstack:config`
  - *Note: If you used a different BUCKET_NAME in your `.env` file, make sure to change the instances of `demo_bucket` in the `localstack:config` script in package.json to match.*.

Upload the test file:

`npm run test-upload`
