- [A Practical Guide to Surviving AWS SAM](https://medium.com/claranet-italia/a-practical-guide-to-surviving-aws-sam-d8ab141b3d25)
- Parameters 를 설정하는 방법
  heading:: true
	- ```yaml
	  Parameters:
	    Env:
	      Type: String
	      AllowedValues:
	        - dev
	        - test
	        - prod
	      Description: Environment in which the application will be deployed. Allowed values [dev, test, prod]
	  ```
- Parameters 를 사용하는 방법
  heading:: true
	- Parameter Store 에서 읽어오기
		- ```yaml
		  Parameters:
		    VpcSubnet:
		      Type: AWS::SSM::Parameter::Value<List<AWS::EC2::Subnet::Id>>
		      Description: SSM Parameter store key of type StringList with the list of VPC Subnet to be used by Lambda function
		      Default: /MyVpcSubnet
		    VpcSg:
		      Type: AWS::SSM::Parameter::Value<AWS::EC2::SecurityGroup::Id>
		      Description: SSM Parameter store key of type String with the VPC Security Group to be used by Lambda function
		      Default: /MyVpcSg
		    MySSMParameter:
		      Type: AWS::SSM::Parameter::Value<String>
		      Description: SSM Parameter store key of type String with custom parameter to be passed as env var to Lambda function
		      Default: /MyParameter
		  ```
	- `Pameters`에 선언 없이 읽어오기
		- ```yaml
		  Resources:
		    HelloWorldFunction:
		      Type: AWS::Serverless::Function
		      Properties:
		        Environment:
		          Variables:
		            MY_SSM_PARAMETER_RESOLVE: "{{resolve:ssm:/MyParameter:1}}"
		  ```
	- `FindInMap` 사용하기
	  collapsed:: true
		- ```yaml
		  Parameters:
		    Env:
		      Type: String
		      AllowedValues:
		        - dev
		        - test
		        - prod
		      Description: Environment in which the application will be deployed. Allowed values [dev, test, prod]
		  
		  Mappings:
		    EnvMapping:
		      dev:
		        LogLevelMapping: DEBUG
		      test:
		        LogLevelMapping: DEBUG
		      prod:
		        LogLevelMapping: INFO
		  
		  Resources:
		    HelloWorldFunction:
		      Type: AWS::Serverless::Function
		      Properties:
		        Environment:
		          Variables:
		            LOG_LEVEL_MAPPING: !FindInMap [ EnvMapping, !Ref Env, LogLevelMapping ]
		  
		  ```
	- `Conditions` 사용하기
	  collapsed:: true
		- ```
		  Parameters:
		    Env:
		      Type: String
		      AllowedValues:
		        - dev
		        - test
		        - prod
		      Description: Environment in which the application will be deployed. Allowed values [dev, test, prod]
		  
		  Conditions:
		    HasDebugEnabled: !Not [ !Equals [ !Ref Env, 'prod' ] ]
		  
		  Resources:
		    HelloWorldFunction:
		      Type: AWS::Serverless::Function
		      Properties:
		        Environment:
		          Variables:
		            LOG_LEVEL_CONDITION: !If [ HasDebugEnabled, 'DEBUG', 'INFO' ]
		  ```
	- template 에서 선언하지 않고, `ssm_cache` 또는 `aws_lambda_powertools` 를 이용해서 코드 내에서 해결할 수도 있다.
	  collapsed:: true
		- ```python
		  from ssm_cache import SSMParameter
		  from aws_lambda_powertools.utilities import parameters
		  
		  # ssm cache
		  my_ssm_parameter_ssm_cache = SSMParameter("MyParameter").value
		  # lambda power tools
		  my_ssm_parameter_powertools = parameters.get_parameter("/MyParameter")
		  
		  ```
		- 이때는 권한을 부여해야 함
		- ```yaml
		  Resources:
		    HelloWorldFunction:
		      Type: AWS::Serverless::Function
		      Properties:
		        Policies:
		          - SSMParameterReadPolicy:
		              ParameterName: "MyParameter"
		  ```
	- Parameter Store 로 값을 내보내려면, Export 를 사용한다.
	  collapsed:: true
		- ```yaml
		  Outputs:
		    PublicSubnets:
		      Description: A list of the public subnets
		      Value: !Join [ ",", [ !Ref PublicSubnet1, !Ref PublicSubnet2 ]]
		      Export:
		        Name: 'Network-PublicSubnets'
		    NoIngressSecurityGroup:
		      Description: Security group with no ingress rule
		      Value: !Ref NoIngressSecurityGroup
		      Export:
		        Name: 'Network-NoIngressSecurityGroup'
		  ```
	- ImportValue function 을 이용해서 값을 불러올 수 있다.
	  collapsed:: true
		- ```yaml
		  Resources:
		    HelloWorldFunction:
		      Type: AWS::Serverless::Function
		      Properties:
		        VpcConfig:
		          SubnetIds: !ImportValue Network-PublicSubnets
		          SecurityGroupIds:
		            - !ImportValue Network-NoIngressSecurityGroup
		  ```
	- Secrets Manager 값도 불러올 수 있다.
	  collapsed:: true
		- ```yaml
		  Resources:
		    HelloWorldFunction:
		      Type: AWS::Serverless::Function
		      Properties:
		        Environment:
		          Variables:
		            MY_SECRET: "{{resolve:secretsmanager:MySecret:SecretString:MySecret}}"
		  ```
- Parameters 를 override 하는 방법
  heading:: true
	- `sam deploy` 중 override
	  collapsed:: true
		- ```shell
		  sam deploy --parameter-overrides Env=dev
		  ```
	- `samconfig.toml` 파일에서 값을 override 할 수 있다.
	  collapsed:: true
		- ```toml
		  version = 0.1
		  [default]
		  [default.deploy]
		  [default.deploy.parameters]
		  stack_name = "sam-app"
		  s3_bucket = "my-s3-bucket"
		  s3_prefix = "sam-app"
		  region = "eu-west-1"
		  capabilities = "CAPABILITY_IAM"
		  parameter_overrides = "Env=\"dev\""
		  
		  [test]
		  [test.deploy]
		  [test.deploy.parameters]
		  stack_name = "sam-app"
		  s3_bucket = "my-s3-bucket"
		  s3_prefix = "sam-app"
		  region = "eu-west-1"
		  capabilities = "CAPABILITY_IAM"
		  parameter_overrides = "Env=\"test\""
		  ```