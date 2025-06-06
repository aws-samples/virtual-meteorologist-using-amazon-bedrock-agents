> [!NOTE]
> The content presented here serves as an example intended solely for educational objectives and should not be implemented in a live production environment without proper modifications and rigorous testing.

# Building a virtual meteorologist using Amazon Bedrock Agents

The integration of [generative AI](https://aws.amazon.com/ai/generative-ai/) capabilities is driving transformative changes across many industries. Although weather information is accessible through multiple channels, businesses that heavily rely on meteorological data require robust and scalable solutions to effectively manage and use these critical insights and reduce manual processes. This solution demonstrates how to create an AI-powered virtual meteorologist that can answer complex weather-related queries in natural language. We use various AWS services to deploy a complete solution that you can use to interact with an API providing real-time weather information. In this solution, we use [Amazon Bedrock Agents](https://aws.amazon.com/bedrock/agents/).

Amazon Bedrock Agents helps to streamline workflows and automate repetitive tasks. Amazon Bedrock Agents can securely connect to your company's data sources and augments the user's request with accurate responses. You can use Amazon Bedrock Agents to architect an action schema tailored to your requirements, granting you control whenever the agent initiates the specified action. This versatile approach equips you to seamlessly integrate and execute business logic within your preferred backend service, fostering a cohesive combination of functionality and flexibility. There is also memory retention across the interaction allowing a more personalized user experience.

In this post, we present a streamlined approach to deploying an AI-powered agent by combining Amazon Bedrock Agents and a [foundation model](https://aws.amazon.com/what-is/foundation-models/) (FM). We guide you through the process of configuring the agent and implementing the specific logic required for the virtual meteorologist to provide accurate weather-related responses. Additionally, we use various AWS services, including [AWS Amplify](https://aws.amazon.com/amplify/) for hosting the front end, [AWS Lambda](https://aws.amazon.com/lambda/) functions for handling request logic, [Amazon Cognito](https://aws.amazon.com/cognito/) for user authentication, and [AWS Identity and Access Management](https://aws.amazon.com/iam/) (IAM) for controlling access to the agent.

## Solution overview

The diagram gives an overview and highlights the key components. The architecture uses Amazon Cognito for user authentication and Amplify as the hosting environment for our front-end application. Amazon Bedrock Agents forwards the details from the user query to the action groups, which further invokes custom Lambda functions. Each action group and Lambda function handles a specific task:

![virtual-meteorologist-figure-1](https://github.com/user-attachments/assets/e39e4976-9038-4866-b2c7-184319f1f751)

The diagram gives an overview and highlights the key components. The architecture uses Amazon Cognito for user authentication and Amplify as the hosting environment for our front-end application. Amazon Bedrock Agents forwards the details from the user query to the action groups, which further invokes custom Lambda functions. Each action group and Lambda function handles a specific task:

1. **geo-coordinates** – Processes geographic coordinates (geo-coordinates) to get details about a specific location
2. **weather** – Gathers weather information for the provided location
3. **date-time** – Obtains the current date and time

## Prerequisites

You must have the following in place to complete the solution in this post:

An [AWS account](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&client_id=signup)
FM [access](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) in Amazon Bedrock for Anthropic's Claude 3.5 Sonnet in the same [AWS Region](https://docs.aws.amazon.com/glossary/latest/reference/glos-chap.html#region) where you'll deploy this solution
The accompanying [AWS CloudFormation](http://aws.amazon.com/cloudformation) template downloaded from the [aws-samples GitHub repo](https://github.com/aws-samples/virtual-meteorologist-using-amazon-bedrock-agents).

## Deploy solution resources using AWS CloudFormation

When you run the AWS CloudFormation template, the following resources are deployed (note that costs will be incurred for the AWS resources used):

* Amazon Cognito resources:
    * [User pool](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html) – `CognitoUserPoolforVirtualMeteorologistApp`
    * [App client](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-client-apps.html) – `VirtualMeteorologistApp`
    * [Identity pools](https://docs.aws.amazon.com/cognito/latest/developerguide/identity-pools.html) – `cognito-identity-pool-vm`

* Lambda resources:
    * Function – `<Stack name>-geo-coordinates-<auto-generated>`
    * Function – `<Stack name>-weather-<auto-generated>`
    * Function – `<Stack name>-date-time-<auto-generated>`

* Amazon Bedrock Agents: virtual-meteorologist
    * Action groups (1) – `obtain-latitude-longitude-from-place-name`
    * Action groups (2) – `obtain-weather-information-with-coordinates`
    * Action groups (3) – `get-current-date-time-from-timezone`

After you deploy the CloudFormation template, copy the following from the **Outputs** tab on the [CloudFormation console](https://console.aws.amazon.com/cloudformation) to be used during the configuration of your application after it's deployed in AWS Amplify.

* `AWSRegion`
* `BedrockAgentAliasId`
* `BedrockAgentId`
* `BedrockAgentName`
* `IdentityPoolId`
* `UserPoolClientId`
* `UserPoolId`

<img width="902" alt="virtual-meteorologist-figure-2" src="https://github.com/user-attachments/assets/61928e18-b768-43d1-8ce5-89c74759605f" />

## Deploy the AWS Amplify application

You need to manually deploy the Amplify application using the front-end code found on GitHub. Complete the following steps:

1. Download the front-end code AWS-Amplify-Frontend.zip from [GitHub](https://github.com/aws-samples/virtual-meteorologist-using-amazon-bedrock-agents).
2. Use the .zip file to manually [deploy](https://docs.aws.amazon.com/amplify/latest/userguide/manual-deploys.html) the application in Amplify.
3. Return to the Amplify page and use the domain it automatically generated to access the application.

#### Use Amazon Cognito for user authentication

Amazon Cognito is an identity service that you can use to authenticate and authorize users. We use Amazon Cognito in our solution to verify the user before they can use the application. We also use identity pool to provide temporary AWS credentials for the user while they interact with Amazon Bedrock API.

#### Use Amazon Bedrock Agents to automate application tasks

With Amazon Bedrock Agents, you can build and configure autonomous agents in your application. An agent helps your end users complete actions based on organization data and user input. Agents orchestrate interactions between FMs, data sources, software applications, and user conversations.

#### Use action group to define actions that Amazon Bedrock agents perform

An action group defines a set of related actions that an Amazon Bedrock agent can perform to assist users. When configuring an action group, you have options for handling user-provided information, including adding user input to the agent's action group, passing data to a Lambda function for custom business logic, or returning control directly through the InvokeAgent response. In our application, we created three action groups to give the Amazon Bedrock agent these essential functionalities: retrieving coordinates for specific locations, obtaining current date and time information, and fetching weather data for given locations. These action groups enable the agent to access and process crucial information, enhancing its ability to respond accurately and comprehensively to user queries related to location-based services and weather conditions.

#### Use Lambda for Amazon Bedrock action group

As part of this solution, three Lambda functions are deployed to support the action groups defined for our Amazon Bedrock agent:

1. **Location coordinates Lambda function** – This function is triggered by the `obtain-latitude-longitude-from-place-name` action group. It takes a place name as input and returns the corresponding latitude and longitude coordinates. The function uses a geocoding service or database to perform this lookup.
2. **Date and time Lambda function** – Invoked by the `get-current-date-time-from-timezone` action group, this function provides the current date and time information.
3. **Weather information Lambda function** – This function is called by the `obtain-weather-information-with-coordinates` action group. It accepts geo-coordinates from the first Lambda function and returns current weather conditions and forecasts for the specified area. This Lambda function used a weather API to fetch up-to-date meteorological data.

Each of these Lambda functions receives an input event containing relevant metadata and populated fields from the Amazon Bedrock agent's API operation or function parameters. The functions process this input, perform their specific tasks, and return a response with the required information. This response is then used by the Amazon Bedrock agent to formulate its reply to the user's query. By using these Lambda functions, our Amazon Bedrock agent gains the ability to access external data sources and perform complex computations, significantly enhancing its capabilities in handling user requests related to location, time, and weather information.

#### Use AWS Amplify for front-end code

Amplify offers a development environment for building secure, scalable mobile and web applications. Developers can focus on their code rather than worrying about the underlying infrastructure. Amplify also integrates with many Git providers. For this solution, we manually upload our front-end code using the method outlined earlier in this post.

## Application walkthrough

Navigate to the URL provided after you created the application in Amplify. Upon accessing the application URL, you'll be prompted to provide information related to Amazon Cognito and Amazon Bedrock Agents. This information is required to securely authenticate users and allow the front end to interact with the Amazon Bedrock agent. It enables the application to manage user sessions and make authorized API calls to AWS services on behalf of the user.

You can enter information with the values you collected from the CloudFormation stack outputs. You'll be required to enter the following fields, as shown in the following screenshot:

* User Pool ID
* User Pool ClientID
* Identity Pool ID
* Region
* Agent Name
* Agent ID
* Agent Alias ID
* Region

![virtual-meteorologist-figure-3](https://github.com/user-attachments/assets/6d60dcf5-3fbe-4d5e-a201-9ee1d22958c0)

You need to sign in with your username and password. A temporary password was automatically generated during deployment and sent to the email address you provided when launching the CloudFormation template. At first sign-in attempt, you’ll be asked to reset your password, as shown in the following video.

![virtual-meteorologist-figure-4](https://github.com/user-attachments/assets/98590635-aab6-41e3-b322-1ae05effa6a1)

Now you can start asking questions in the application, for example, “Can we do barbecue today in Dallas, TX?” In a few seconds, the application will provide you detailed results mentioning if you can do barbecue in Dallas, TX. The following video shows this chat.

![virtual-meteorologist-figure-5](https://github.com/user-attachments/assets/e9b8d61a-4948-432f-988e-fabcb318c3e9)

## Example use cases

Here are a few sample queries to demonstrate the capabilities of your virtual meteorologist:

1. "What's the weather like in New York City today?"
2. "Should I plan an outdoor birthday party in Miami next weekend?"
3. "Will it snow in Denver on Christmas Day?"
4. "Can I go swimming on a beach in Chicago today?"

These queries showcase the agent's ability to provide current weather information, offer advice based on weather forecasts, and predict future weather conditions. You can even ask a question related to an activity such as swimming, and it will answer based on the weather conditions if that activity is okay to do.

## Clean up

If you decide to discontinue using the virtual meteorologist, you can follow these steps to remove it, its associated resources deployed using AWS CloudFormation, and the Amplify deployment:

1. Delete the CloudFormation stack:
    1. On the AWS CloudFormation console, choose **Stacks** in the navigation pane.
    2. Locate the stack you created during the deployment process (you assigned a name to it).
    3. Select the stack and choose **Delete**.

2. Delete the Amplify application and its resources. For instructions, refer to [Clean Up Resources](https://aws.amazon.com/getting-started/hands-on/build-web-app-s3-lambda-api-gateway-dynamodb/module-six/).

## Conclusion

This solution demonstrates the power of combining Amazon Bedrock Agents with other AWS services to create an intelligent, conversational weather assistant. By using AI and cloud technologies, businesses can automate complex queries and provide valuable insights to their users.

## Additional resources

To learn more about Amazon Bedrock, refer to the following resources:

* [GitHub repo: Amazon Bedrock Workshop](https://github.com/aws-samples/amazon-bedrock-workshop)
* [Amazon Bedrock User Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html)
* [Workshop: Using generative AI on AWS for diverse content types](https://catalog.us-east-1.prod.workshops.aws/genai-on-aws/)

To learn more about the Anthropic's Claude 3.5 Sonnet model, refer to the following resources:

* [Anthropic's Claude in Amazon Bedrock](https://aws.amazon.com/bedrock/claude/)


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

