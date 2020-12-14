# Desafio Dock Banking as a service

Para construção dessa aplicação foram usados dois serviços da AWS: AWS Lambda e AWS API Gateway. Esses dois serviços então foram automatizados com um terceiro, o AWS CloudFormation. Com eles foi possível fazer chamadas HTTP ao site http://wttr.in/ e retornar informações sobre o tempo.

Para realizar a tarefa foi usado a stack que encontra-se na pasta "cloudformation". Dentro desse YAML podemos definir os nomes das funções e APIs, assim como o metodo HTTP a ser usado:

AWSTemplateFormatVersion: 2010-09-09
Description: Funcao Lambda e Api Gateway para teste SRE Dock

Parameters:
  apiGatewayName:
    Type: String
    Default: API-SRE-DOCK
  apiGatewayStageName:
    Type: String
    AllowedPattern: "[a-z0-9]+"
    Default: call
  apiGatewayHTTPMethod:
    Type: String
    Default: GET
  lambdaFunctionName:
    Type: String
    AllowedPattern: "[a-zA-Z0-9]+[a-zA-Z0-9-]+[a-zA-Z0-9]+"
    Default: FUNCAO-SRE-DOCK

Resources:
  apiGateway:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Description: Example API Gateway
      EndpointConfiguration:
        Types:
          - REGIONAL
      Name: !Ref apiGatewayName

  apiGatewayRootMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      AuthorizationType: NONE
      HttpMethod: !Ref apiGatewayHTTPMethod
      Integration:
        IntegrationHttpMethod: POST
        Type: AWS_PROXY
        Uri: !Sub
          - arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${lambdaArn}/invocations
          - lambdaArn: !GetAtt lambdaFunction.Arn
      ResourceId: !GetAtt apiGateway.RootResourceId
      RestApiId: !Ref apiGateway

  apiGatewayDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn:
      - apiGatewayRootMethod
    Properties:
      RestApiId: !Ref apiGateway
      StageName: !Ref apiGatewayStageName

  lambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      Code:
        ZipFile: |
            import requests
            import json
            def lambda_handler(event, context):
            response = requests.get("http://wttr.in/sao_paulot?format=j1", timeout=10)
            return {
                'statusCode': 200,
                'body': response.json()
            }
      Description: Example Lambda function
      FunctionName: !Ref lambdaFunctionName
      Handler: index.handler
      MemorySize: 128
      Role: !GetAtt lambdaIAMRole.Arn
      Runtime: python3.8

  lambdaApiGatewayInvoke:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !GetAtt lambdaFunction.Arn
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${apiGateway}/${apiGatewayStageName}/${apiGatewayHTTPMethod}/

  lambdaIAMRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Action:
              - sts:AssumeRole
            Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
      Policies:
        - PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Effect: Allow
                Resource:
                  - !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/lambda/${lambdaFunctionName}:*
          PolicyName: lambda

  lambdaLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub /aws/lambda/${lambdaFunctionName}
      RetentionInDays: 90

Outputs:
  apiGatewayInvokeURL:
    Value: !Sub https://${apiGateway}.execute-api.${AWS::Region}.amazonaws.com/${apiGatewayStageName}

  lambdaArn:
    Value: !GetAtt lambdaFunction.Arn
    
Com isso, serão criados a API no API Gateway e a função Lambda. 

A função importa o módulo request e json e os utiliza para requisitar as informações do site em um formato JSON:

import requests
import json
def lambda_handler(event, context):
response = requests.get("http://wttr.in/sao_paulot?format=j1", timeout=10)
return {
    'statusCode': 200,
    'body': response.json()
}

Por último, com o API Gateway, temos um endpoint para acionar essa função lambda:
https://7gjsghbf4c.execute-api.us-east-1.amazonaws.com/PROD
