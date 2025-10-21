# Explorando Workflows Automatizados com AWS Step Functions

Este repositório contém orquestração de workflows executado no AWS Step Functions


O AWS Step Functions é um orquestrador de serviços (workflows), ele cria fluxos de trabalho visual que pode ser utilizado crindo um fluxo com Lambda, S3, EC2 e etc.


Dentro desse projeto foi realizado Workflows de serviços conforme foi solicitado no deafio de projeto, através de um exemplo dentro do modelo que utiliza o S3 para armazenamento e o Lambda prório AWS Step Functions. 

Foram criados os seguntes passo a passo desse woorkflow.

1- Ele inicia e vai para o ChecarBucket que verifica o conteúdo que está no Bucket;

2- Depois de checar ele vai executar o bucket pegando todo o arquivo que está como .year;

3- Depois ele vai para o próximo passo que é ValidaArquivo, ele escolhe (Choice) qual o próximo passo a ser feito; 

4- Caso o bucket tiver algum objeto, ele lista esse objeto e faz avalidação e indo para o próximo passo que é ListarObjetos. Este passo serve para validar os arquivos;

5- Caso ele não encontre o arquivo, ele vai chamar o Lambda e encerra o processo;

6- Se ele encontrar o arquivo, ele Lista os objetos, salva no bucket, faz a validação cria uma cópia e disponibiliza essa cópia no buckete por último utiliza o Lambda e encerra o processo.


Aqui está o arquivo em json que foi gerado desse workflow:

```{
  "Comment": "Distributed map to find temperature stats by month",
  "StartAt": "ChecarBucket",
  "QueryLanguage": "JSONata",
  "States": {
    "ChecarBucket": {
      "Type": "Task",
      "Next": "ValidaArquivo",
      "Resource": "arn:aws:states:::aws-sdk:s3:listObjectsV2",
      "Arguments": {
        "Bucket": "stepfunctionssample-distributedmapn-noaadatabucket-b30b2pm78gnc",
        "Prefix": "{% $string($states.input.year) %}",
        "MaxKeys": 1
      },
      "Output": "{% $states.result.KeyCount %}",
      "Assign": {
        "year": "{% $string($states.input.year) %}"
      },
      "Comment": "Ele checa o conteúdo do bucket."
    },
    "ValidaArquivo": {
      "Type": "Choice",
      "Default": "ListarObjetos",
      "Choices": [
        {
          "Next": "ProcessNOAAData",
          "Condition": "{% $states.input >= 1%}"
        }
      ],
      "Comment": "Este Choice serve para validar os arquivos."
    },
    "ListarObjetos": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:s3:listObjectsV2",
      "Next": "Distributed S3 copy NOA Data",
      "Arguments": {
        "Bucket": "noaa-gsod-pds",
        "Prefix": "{% $year %}"
      },
      "Comment": "Lista objetos do bucket."
    },
    "Distributed S3 copy NOA Data": {
      "Type": "Map",
      "ItemProcessor": {
        "ProcessorConfig": {
          "Mode": "DISTRIBUTED",
          "ExecutionType": "EXPRESS"
        },
        "StartAt": "Map",
        "States": {
          "Map": {
            "Type": "Map",
            "ItemProcessor": {
              "ProcessorConfig": {
                "Mode": "INLINE"
              },
              "StartAt": "CopiaObjeto",
              "States": {
                "CopiaObjeto": {
                  "Type": "Task",
                  "Resource": "arn:aws:states:::aws-sdk:s3:copyObject",
                  "End": true,
                  "Retry": [
                    {
                      "ErrorEquals": [
                        "S3.SdkClientException",
                        "S3.S3Exception"
                      ],
                      "BackoffRate": 2,
                      "IntervalSeconds": 1,
                      "MaxAttempts": 3,
                      "Comment": "Retry on failed connections"
                    }
                  ],
                  "Arguments": {
                    "Bucket": "stepfunctionssample-distributedmapn-noaadatabucket-b30b2pm78gnc",
                    "Key": "{% $states.input.Key %}",
                    "CopySource": "{% 'noaa-gsod-pds/' & $states.input.Key %}"
                  }
                }
              }
            },
            "End": true,
            "Items": "{% $states.input.Items %}"
          }
        }
      },
      "Label": "DistributedS3copyNOAAData",
      "Next": "Has next page?",
      "ItemBatcher": {
        "MaxItemsPerBatch": 100
      },
      "Items": "{% $states.input.Contents %}",
      "MaxConcurrency": 1000
    },
    "Has next page?": {
      "Type": "Choice",
      "Default": "ProcessNOAAData",
      "Choices": [
        {
          "Next": "ListObjectsInNoaBucket with pagination",
          "Condition": "{% $exists($states.input.NextContinuationToken) %}"
        }
      ]
    },
    "ListObjectsInNoaBucket with pagination": {
      "Type": "Task",
      "Arguments": {
        "Bucket": "noaa-gsod-pds",
        "Prefix": "{% $year %}",
        "ContinuationToken": "{% $states.input.NextContinuationToken %}"
      },
      "Resource": "arn:aws:states:::aws-sdk:s3:listObjectsV2",
      "Next": "Distributed S3 copy NOA Data"
    },
    "ProcessNOAAData": {
      "Type": "Map",
      "ItemReader": {
        "Resource": "arn:aws:states:::s3:listObjectsV2",
        "Arguments": {
          "Bucket": "stepfunctionssample-distributedmapn-noaadatabucket-b30b2pm78gnc"
        }
      },
      "ItemProcessor": {
        "ProcessorConfig": {
          "Mode": "DISTRIBUTED",
          "ExecutionType": "EXPRESS"
        },
        "States": {
          "Lambda Invoke": {
            "Type": "Task",
            "Resource": "arn:aws:states:::lambda:invoke",
            "Retry": [
              {
                "ErrorEquals": [
                  "Lambda.TooManyRequestsException",
                  "Lambda.ServiceException",
                  "Lambda.AWSLambdaException",
                  "Lambda.SdkClientException"
                ],
                "IntervalSeconds": 2,
                "MaxAttempts": 6,
                "BackoffRate": 2,
                "JitterStrategy": "FULL"
              }
            ],
            "End": true,
            "Arguments": {
              "Payload": "{% $states.input %}",
              "FunctionName": "arn:aws:lambda:us-east-1:099597654193:function:StepFunctionsSample-Distribute-TemperatureFunction-lDf7T5EsWlRA"
            },
            "Output": "{% $states.result.Payload %}"
          }
        },
        "StartAt": "Lambda Invoke"
      },
      "Label": "ProcessNOAAData",
      "MaxConcurrency": 3000,
      "ToleratedFailurePercentage": 5,
      "ResultWriter": {
        "Resource": "arn:aws:states:::s3:putObject",
        "Arguments": {
          "Bucket": "stepfunctionssample-distributedmapno-resultsbucket-mclmdrrfc3rv",
          "Prefix": "results"
        }
      },
      "ItemBatcher": {
        "BatchInput": {},
        "MaxItemsPerBatch": 500
      },
      "Next": "Reducer"
    },
    "Reducer": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.TooManyRequestsException",
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 6,
          "BackoffRate": 2,
          "JitterStrategy": "FULL"
        }
      ],
      "End": true,
      "Arguments": {
        "Payload": "{% $states.input %}",
        "FunctionName": "arn:aws:lambda:us-east-1:099597654193:function:StepFunctionsSample-DistributedMap-ReducerFunction-eoBkB4IUuscw"
      }
    }
  }
}
```
