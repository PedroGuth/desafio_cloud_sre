# Desafio Dock Banking as a service

Para construção dessa aplicação foram usados dois serviços da AWS: AWS Lambda e AWS API Gateway. Esses dois serviços então foram automatizados com um terceiro, o AWS CloudFormation. Com eles foi possível fazer chamadas HTTP ao site http://wttr.in/ e retornar informações sobre o tempo.

Para realizar a tarefa foi usado a stack que encontra-se na pasta "cloudformation". Dentro desse YAML podemos definir os nomes das funções e APIs, assim como o metodo HTTP a ser usado.
    
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
