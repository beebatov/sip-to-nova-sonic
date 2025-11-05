# Nova S2S VoIP Gateway

This project contains an implementation of a SIP endpoint that acts as a gateway to Nova Sonic speech to speech.
In other words, you can call a phone number and talk to Nova Sonic.

## Architecture

![](architecture/nova-sonic-voip-architecture_new.png)

The system consists of three main components:

1. **SIP/RTP Gateway** - Handles VoIP calls and audio streaming
2. **Nova Sonic** - Amazon Bedrock's speech-to-speech AI model
3. **Supporting Services** - Knowledge Base and Work Order Management

### Call Flow

This application acts as a SIP user agent.  When it starts it registers with a SIP server.  Upon receiving a call it will answer, establish the media session (over RTP), start a session with Nova Sonic, and bridge audio between RTP and Nova Sonic.  Audio received via RTP is sent to Nova Sonic and audio received from Nova Sonic is sent to the caller via RTP.

![](flow.png)

## Prerequisites

* Node.js and CDK installed - See https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html
* Docker installed and running
* AWS credentials configured
* SIP account credentials (for VoIP Gateway)

## Deployment Options

This project includes three CDK stacks:

![](architecture.png)

### 1. VoIP Gateway Stack

EC2-backed ECS container for the SIP/RTP gateway running in host mode.

**Deploy:**

1. Build the Maven project:
   ```bash
   mvn package
   ```

2. Copy JAR to docker directory:
   ```bash
   cp target/s2s-voip-gateway-*.jar docker/
   ```

3. Configure SIP credentials in `cdk-ecs/cdk.context.json`:
   ```json
   {
     "config": {
       "sipServer": "your-sip-server.com",
       "sipUsername": "your-username",
       "sipPassword": "your-password",
       "sipRealm": "your-realm"
     }
   }
   ```

4. Deploy:
   ```bash
   cd cdk-ecs
   npm install
   cdk bootstrap  # Only needed once per account/region
   cdk deploy VoipGatewayStack
   ```

5. Get the Network Load Balancer DNS name from the stack outputs:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name VoipGatewayStack \
     --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerDNS`].OutputValue' \
     --output text
   ```

6. Configure your SIP provider (e.g., Vonage) to use the NLB DNS name as the SIP endpoint

7. Call your SIP phone number to test

**Note:** The stack includes a Network Load Balancer that provides a stable DNS endpoint for SIP/RTP traffic. Use this DNS name in your SIP provider configuration instead of the EC2 instance IP address.

**Clean-up:**
```bash
cdk destroy VoipGatewayStack
```

### 2. Bedrock Knowledge Base Stack

Knowledge base for SOPs and operational information.

**Deploy:**
```bash
cd cdk-ecs
npm install
cdk deploy BedrockStack
```

**Clean-up:**
```bash
cdk destroy BedrockStack
```

### 3. WorkOrder AgentCore Stack

Automated work order management with Bedrock AgentCore runtime.

**Deploy:**

1. Configure test user credentials in `cdk-ecs/cdk.json`:
   ```json
   {
     "context": {
       "cognito": {
         "testUsername": "testuser",
         "testPassword": "MyPassword123!"
       }
     }
   }
   ```

2. Deploy:
   ```bash
   cd cdk-ecs
   npm install
   cdk deploy WorkOrderStack
   ```

The CDK will automatically:
- Build and push Docker image to ECR
- Create Cognito User Pool and test user
- Set test user password (from cdk.json)
- Store credentials in Secrets Manager
- Deploy AgentCore Runtime with MCP protocol
- Configure all SSM parameters

**Stack Outputs:**
- `RuntimeArn` - AgentCore runtime ARN
- `RuntimeId` - AgentCore runtime ID
- `UserPoolId` - Cognito user pool ID
- `ClientId` - Cognito client ID
- `WorkOrdersTableName` - DynamoDB table name

**Access Credentials:**
```bash
aws secretsmanager get-secret-value \
  --secret-id work_order_mcp/cognito/credentials \
  --query SecretString --output text | jq .
```

**Update Configuration:**
To change the test user password, update `cdk.json` and redeploy:
```bash
cdk deploy WorkOrderStack
```

**Clean-up:**
```bash
cdk destroy WorkOrderStack
```

## Build

The Nova S2S VoIP Gateway is a Java Maven project.

**Requirements:**
* JDK 9+ (Corretto, OpenJDK, or Oracle)
* Apache Maven

**Build:**
```bash
mvn package
```

Maven will create `s2s-voip-gateway-*.jar` in the `target/` directory.

### Maven settings.xml

mjSIP is distributed from a GitHub Maven repository. Configure your `~/.m2/settings.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <servers>
        <server>
            <id>github</id>
            <username>YOUR_USERNAME</username>
            <password>YOUR_AUTH_TOKEN</password>
        </server>
    </servers>
</settings>
```

Get your GitHub token from: https://github.com/settings/tokens

## Developer Guide

**Entry Points:**
- `NovaSonicVoipGateway.java` - Main application entry point
- `NovaStreamerFactory.java` - Nova Sonic integration and audio streaming

**Extending with Tools:**

Nova Sonic includes tools for date/time retrieval. To add custom tools:

1. Extend `AbstractNovaS2SEventHandler` class
2. Implement your tool functionality
3. Register in `NovaStreamerFactory.createMediaStreamer()`

Example tools: `com.example.s2s.voipgateway.nova.tools`

## Third Party Dependencies

This project uses a fork of mjSIP: https://github.com/haumacher/mjSIP (GPLv2 license)

## License

MIT-0 License. See the LICENSE file for details.
