A basic demo setting up Localstack's S3.

Read the tutorial [here](https://dev.to/goodidea/how-to-fake-aws-locally-with-localstack-27me).

Or, spin things up here:

1. Install [Docker](https://docs.docker.com/install/) if you haven't already.
2. Install the [AWS CLI](https://aws.amazon.com/cli/). Even though we aren't going to be working with "real" AWS, we'll use this to talk to our local docker containers.
3. Copy the contents of `.env.example` into a new `.env` file. 
4. Initialize Localstack: `npm run localstack:init`
  - *If you used a different BUCKET_NAME in your `.env` file, make sure to change the values in the `localstack:config` script in package.json*.

Upload the test file:

`npm run test-upload`
