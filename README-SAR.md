# AWS Lambda Power Tuning

AWS Lambda Power Tuning is an AWS Step Functions state machine that helps you optimize your Lambda functions in a data-driven way.

The state machine is designed to be **quick** and **language agnostic**. You can provide **any Lambda Function as input** and the state machine will **estimate the best power configuration to minimize cost**. The function will be executed in your AWS account (i.e. real HTTP calls, SDK calls, cold starts, etc.) and you can enable parallel execution to generate results in just a few seconds.


## How to deploy the state machine (SAR)

You can find this app in the Serverless Application Repository and deploy it with just a few clicks.

In case you want to deploy it "manually", you can use the commands in `deploy.sh`.

First, install AWS SAM and configure your AWS credentials:


```
$ pip install aws-sam-cli
$ aws configure
```

Now, you can clone this repo as follows:

```
$ git clone https://github.com/alexcasalboni/aws-lambda-power-tuning.git
```

Configure your bucket name and stack name in the deployment script, and then run it:

```
$ bash deploy.sh
```


## How to execute the state machine

Once the state machine and all the Lambda Functions have been deployed, you can execute the state machine and provide an input object.

You will find the new state machine in the [Step Functions Console](https://console.aws.amazon.com/states/) or in your app's `Resources` section.

The state machine name will be prefixed with `powerTuningStateMachine`. Find it and click "**Start execution**". Here you can provide the execution input and an execution id (see section below for the full documentation):

```
{
    "lambdaARN": "your-lambda-function-arn",
    "num": 10
}
```

As soon as you click "**Start Execution**" again, you'll be able to visualize the execution.

Once the execution has completed, you will find the execution results in the "**Output**" tab of the "**Execution Details**" section. The output will contain the optimal power configuration and its corresponding average cost per execution.


## Note about the previous version of this project

This project used to require a generation step to dynamically create the required steps based on which memory/power configurations you wanted to test.

The new version of this project doesn't require any generation step, but you may want to fine-tune the state machine in the `template.yml` file to add additional configurations.

Please note that you can specify a subset of configuration values in the `PowerValues` CloudFormation parameter. Only those configurations will be tested. Even though you will still see them in the stete machine chart, the `executor` steps corresponding to non-tested configurations will not run the input function (they'll be skipped).

## State Machine Input

The AWS Step Functions state machine accepts the following parameters:

* **lambdaARN** (required, string): ARN of the Lambda Function you want to optimize
* **num** (required, integer): the # of invocations for each power configuration (minimum 5, recommended: between 10 and 100)
* **payload** (string or object): the static payload that will be used for every invocation
* **parallelInvocation** (false by default): if true, all the invocations will be executed in parallel (note: depending on the value of `num`, you may experience throttling when setting `parallelInvocation` to true)


## State Machine Output

The AWS Step Functions state machine will return the following outputs:

* **power**: the optimal power configuration
* **cost**: the corresponding average cost (per invocation)
* **duration**: the corresponding average duration (per invocation)


## State Machine Internals

The AWS Step Functions state machine is composed by four Lambda Functions:

* **initializer**: create N versions and aliases corresponding to the power values provided as input (e.g. 128MB, 256MB, etc.)
* **executor**: execute the given Lambda Function `num` times, extract execution time from logs, and compute average cost per invocation
* **cleaner**: delete all the previously generated aliases and versions
* **finalizer**: compute the optimal power value (current logic: lowest average cost per invocation)

Initializer, cleaner and finalizer are executed only once, while the executor is used by N parallel branches of the state machine (one for each configured power value). By default, the executor will execute the given Lambda Function `num` consecutive times, but you can enable parallel invocation by setting `parallelInvocation` to `true`. Please note that the total invocation time should stay below 300 seconds (5 min), which means that the average duration of your functions should stay below 3 seconds with `num=100`, 30 seconds with `num=10`, and so on.

## Contributing
Contributors and PRs are always welcome!

### Tests and coverage

Install dev dependencies with `npm install --dev`. Then run tests with `npm test`, or coverage with `npm run coverage`.