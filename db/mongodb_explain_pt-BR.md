## Seu índice no MongoDB está sendo bem utilizado?

Recentemente no trabalho me deparei com uma collection no MongoDB que apresentou uma certa lentidão em suas queries. Essa lentidão não estava evidente, pois embora a collection seja razoavelmente grande (4 milhões de documentos) as consultas a ela eram realizadas dentro de um processo maior, no qual tinham diversos procedimentos que somavam um tempo de retorno aceitável.

O problema só foi identificado com a ajuda do [Sysdig](https://sysdig.com/), que apresentou as collection mais lentas do DB, e que para minha surpresa aquela era a collection com mais lentidão. [Você pode ler mais sobre como configurar um agente do sysdig no mongoDB clicando aqui](https://docs.sysdig.com/en/mongodb.html).

Para ilustrar melhor o problema vamos supor que temos uma collection de 4 milhões de pessoas com o seguinte modelo:
```
{
	"_id": ObjectId("")
	"name": "Daniel",
	"cpf": "12345678900",
	"age": 28,
	"occupation": "programmer",
	"phone": "123456789"
}
```
[Usei esse script para gerar dados aleatórios no MongoDB e simular a situação.](https://gist.github.com/danidr7/f4e87ff2548db48001584e4b2466a913)

A primeira coisa que fiz foi identificar quais campos estavam sendo consultados naquela collection:
```
1. db.person.find({name: "Daniel", cpf: "12345678900"})
2. db.person.find({occupation: "programmer", cpf: "12345678900"})
```

Após isso verifiquei quais índices existiam para essa collection:
```
$ db.person.getIndexes()
[
	{
		"v" : 2,
		"key" : {
			"_id" : 1
		},
		"name" : "_id_",
		"ns" : "test.person"
	},
	{
		"v" : 2,
		"unique" : true,
		"key" : {
			"name" : 1,
			"cpf" : 1
		},
		"name" : "name_1_cpf_1_idx",
		"ns" : "test.person"
	},
	{
		"v" : 2,
		"key" : {
			"occupation" : 1
		},
		"name" : "occupation",
		"ns" : "test.person"
	}
]
```

Beleza, a gente tem um índice composto de 'name' e 'cpf' e outro índice individual de 'occupation', dá pra melhorar esses índices! Mas o que está acontecendo quando o MongoDB executa essas queries? O MongoDB está fazendo uso dos índices?
Aí que entra o [explain()](https://docs.mongodb.com/manual/reference/method/cursor.explain/)!
O explain vai ajudar a gente a investigar o plano da consulta:
```
$ db.person.find({name: "Daniel", cpf: "12345678900"}).explain("executionStats")
{
	"queryPlanner" : {
		"plannerVersion" : 1,
		"namespace" : "test.person",
		"indexFilterSet" : false,
		"parsedQuery" : {
			"$and" : [
				{
					"cpf" : {
						"$eq" : "12345678900"
					}
				},
				{
					"name" : {
						"$eq" : "Daniel"
					}
				}
			]
		},
		"winningPlan" : {
			"stage" : "FETCH",
			"inputStage" : {
				"stage" : "IXSCAN",
				"keyPattern" : {
					"name" : 1,
					"cpf" : 1
				},
				"indexName" : "name_1_cpf_1",
				"isMultiKey" : false,
				"multiKeyPaths" : {
					"name" : [ ],
					"cpf" : [ ]
				},
				"isUnique" : false,
				"isSparse" : false,
				"isPartial" : false,
				"indexVersion" : 2,
				"direction" : "forward",
				"indexBounds" : {
					"name" : [
						"[\"Daniel\", \"Daniel\"]"
					],
					"cpf" : [
						"[\"12345678900\", \"12345678900\"]"
					]
				}
			}
		},
		"rejectedPlans" : [ ]
	},
	"executionStats" : {
		"executionSuccess" : true,
		"nReturned" : 1,
		"executionTimeMillis" : 0,
		"totalKeysExamined" : 1,
		"totalDocsExamined" : 1,
		"executionStages" : {
			"stage" : "FETCH",
			"nReturned" : 1,
			"executionTimeMillisEstimate" : 0,
			"works" : 2,
			"advanced" : 1,
			"needTime" : 0,
			"needYield" : 0,
			"saveState" : 0,
			"restoreState" : 0,
			"isEOF" : 1,
			"docsExamined" : 1,
			"alreadyHasObj" : 0,
			"inputStage" : {
				"stage" : "IXSCAN",
				"nReturned" : 1,
				"executionTimeMillisEstimate" : 0,
				"works" : 2,
				"advanced" : 1,
				"needTime" : 0,
				"needYield" : 0,
				"saveState" : 0,
				"restoreState" : 0,
				"isEOF" : 1,
				"keyPattern" : {
					"name" : 1,
					"cpf" : 1
				},
				"indexName" : "name_1_cpf_1",
				"isMultiKey" : false,
				"multiKeyPaths" : {
					"name" : [ ],
					"cpf" : [ ]
				},
				"isUnique" : false,
				"isSparse" : false,
				"isPartial" : false,
				"indexVersion" : 2,
				"direction" : "forward",
				"indexBounds" : {
					"name" : [
						"[\"Daniel\", \"Daniel\"]"
					],
					"cpf" : [
						"[\"12345678900\", \"12345678900\"]"
					]
				},
				"keysExamined" : 1,
				"seeks" : 1,
				"dupsTested" : 0,
				"dupsDropped" : 0
			}
		}
	},
	"serverInfo" : {
		"host" : "PC-002387",
		"port" : 27017,
		"version" : "4.2.8",
		"gitVersion" : "43d25964249164d76d5e04dd6cf38f6111e21f5f"
	},
	"ok" : 1
}
```

O `explain()` exibe bastante detalhes sobre a query executada, mas vou dar destaque para apenas algumas informações. [As informações detalhadas sobre cada campo pode ser consultada clicando aqui](https://docs.mongodb.com/manual/reference/explain-results/).

#### winningPlan 

O *winningPlan* especifica o plano selecionado pelo otimizador de consultas do MongoDB, podendo ter de 1 até 3 estágios (*inputStage*).
Dentro do *winningPlan* são especificados os estágios da consulta. Vamos abordar 3 tipos de estágios aqui:
- **FETCH**: estágio responsável por retornar os documentos, normalmente busca os documentos das chaves retornadas no estágio anterior
- **COLLSCAN**: estágio responsável por escanear a collection
- **IXSCAN**: estágio responsável por escanear as chaves dos índices

Explicando de uma maneira mais objetiva, quando temos um *inputStage* de **COLLSCAN** significa que nesse estágio a collection foi escaneada documento por documento, ou seja, não fez uso do índice.
Já o estágio **IXSCAN** executa o *scanner* nas chaves do index, ou seja, se trata de uma busca mais performática por escanear dados estruturados.

Podemos notar que no *winningPlan* do exemplo existem 2 estágios, o de **FETCH** e o de **IXSCAN**. Então já sabemos que um índice está sendo usado, o nome do índice pode ser identificado no campo *indexName*.

Agora vamos dar uma olhada em alguns campos do *executionStats*:
```
"executionStats" : {
	"executionSuccess" : true,
	"nReturned" : 1,
	"executionTimeMillis" : 0,
	"totalKeysExamined" : 1,
	"totalDocsExamined" : 1,
```

- **executionStats.nReturned**: é o número de documentos que a consulta retornou
- **executionStats.executionTimeMillis**: é o tempo total da consulta
- **executionStats.totalKeysExamined**: é o número de chaves examinadas, ou seja, vai ser maior que 0 quando o **IXSCAN** for usado
- **executionStats.totalDocsExamined**: é o número de documentos examinados, quando o primeiro *stage* é de **FETCH**, então este será o mesmo número de documentos retornados

Levando em consideração os números do *executionStats*, podemos concluir que o resultado dessa consulta foi bom, pois para 1 resultado retornado foi examinado apenas 1 chave, ou seja, esse index cobre perfeitamente a consulta.

Após concluir que a primeira query estava 'performando' corretamente, realizei a mesma análise da segunda query:
```
$ db.person.find({occupation: "programmer", cpf: "12345678900"}).explain("executionStats")
{
	"queryPlanner" : {
		"plannerVersion" : 1,
		"namespace" : "test.person",
		"indexFilterSet" : false,
		"parsedQuery" : {
			"$and" : [
				{
					"cpf" : {
						"$eq" : "12345678900"
					}
				},
				{
					"occupation" : {
						"$eq" : "programmer"
					}
				}
			]
		},
		"winningPlan" : {
			"stage" : "FETCH",
			"filter" : {
				"cpf" : {
					"$eq" : "12345678900"
				}
			},
			"inputStage" : {
				"stage" : "IXSCAN",
				"keyPattern" : {
					"occupation" : 1
				},
				"indexName" : "occupation_1",
				"isMultiKey" : false,
				"multiKeyPaths" : {
					"occupation" : [ ]
				},
				"isUnique" : false,
				"isSparse" : false,
				"isPartial" : false,
				"indexVersion" : 2,
				"direction" : "forward",
				"indexBounds" : {
					"occupation" : [
						"[\"programmer\", \"programmer\"]"
					]
				}
			}
		},
		"rejectedPlans" : [ ]
	},
	"executionStats" : {
		"executionSuccess" : true,
		"nReturned" : 1,
		"executionTimeMillis" : 1101,
		"totalKeysExamined" : 799296,
		"totalDocsExamined" : 799296,
		"executionStages" : {
			"stage" : "FETCH",
			"executionTimeMillisEstimate" : 50,
			"docsExamined" : 799296,
			"filter" : {
				"cpf" : {
					"$eq" : "12345678900"
				}
			},
			"nReturned" : 1,
			"works" : 799297,
			"advanced" : 1,
			"needTime" : 799295,
			"needYield" : 0,
			"saveState" : 6244,
			"restoreState" : 6244,
			"isEOF" : 1,
			"alreadyHasObj" : 0,
			"inputStage" : {
				"stage" : "IXSCAN",
				"nReturned" : 799296,
				"executionTimeMillisEstimate" : 18,
				"works" : 799297,
				"advanced" : 799296,
				"needTime" : 0,
				"needYield" : 0,
				"saveState" : 6244,
				"restoreState" : 6244,
				"isEOF" : 1,
				"keyPattern" : {
					"occupation" : 1
				},
				"indexName" : "occupation_1",
				"isMultiKey" : false,
				"multiKeyPaths" : {
					"occupation" : [ ]
				},
				"isUnique" : false,
				"isSparse" : false,
				"isPartial" : false,
				"indexVersion" : 2,
				"direction" : "forward",
				"indexBounds" : {
					"occupation" : [
						"[\"programmer\", \"programmer\"]"
					]
				},
				"keysExamined" : 799296,
				"seeks" : 1,
				"dupsTested" : 0,
				"dupsDropped" : 0
			}
		}
	},
	"serverInfo" : {
		"host" : "PC-002387",
		"port" : 27017,
		"version" : "4.2.8",
		"gitVersion" : "43d25964249164d76d5e04dd6cf38f6111e21f5f"
	},
	"ok" : 1
}
```

Podemos ver que, assim como na query anterior, no *winningPlan* temos o estágio de **IXSCAN** e o de **FETCH**, então sabemos que um índice está sendo usado.
Vamos analisar o *executionStats*:
```
"executionStats" : {
		"nReturned" : 1,
		"executionTimeMillis" : 1101,
		"totalKeysExamined" : 799296,
		"totalDocsExamined" : 799296,
```

Agora sim percebe-se nitidamente uma quantidade absurda de documentos analisados e um tempo de execução (*executionTimeMillis*) bem maior que na query anterior.

Para entender o que aconteceu, é necessário observar o *executionStages*.
Como dito antes, o plano da query contém 2 estágios, vamos começar analisando o **IXSCAN** (a forma correta de leitura do plano é do estágio mais aninhado para fora):
```
"inputStage" : {
				"stage" : "IXSCAN",
				"nReturned" : 799296,
				"executionTimeMillisEstimate" : 18,
				"works" : 799297,
				"advanced" : 799296,
				"needTime" : 0,
				"needYield" : 0,
				"saveState" : 6244,
				"restoreState" : 6244,
				"isEOF" : 1,
				"keyPattern" : {
					"occupation" : 1
				},
				"indexName" : "occupation_1",
```

No **IXSCAN** nota-se que foi usado o índice do campo *occupation*, porém foram retornados *799296* documentos nesse estágio. O motivo desse resultado é que o índice contempla apenas 1 dos 2 campos pesquisados, então ele pode ajudar apenas filtrando o `occupation`, que por sua vez, retornou todos os *799296* documentos com *occupation* igual a *programmer*.

```
"executionStages" : {
			"stage" : "FETCH",
			"docsExamined" : 799296,
			"nReturned" : 1,
```

Agora no estágio **FETCH** o scanner precisou examinar todos os *799296* documentos para retornar apenas 1, ou seja, o scanner tá tendo muito documento para examinar, vamos tentar melhorar o índice para otimizar esse trabalho!



----`WIP`----