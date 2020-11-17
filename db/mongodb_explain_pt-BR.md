## Seu índice no MongoDB está sendo bem utilizado?

Recentemente no trabalho me deparei com uma collection no MongoDB que apresentou uma certa lentidão em suas queries. Essa lentidão não estava evidente, pois embora a collection seja razoavelmente grande (4 milhões de documentos) as consultas a ela eram realizadas dentro de um processo maior, no qual tinham diversos procedimentos que somavam um tempo de retorno aceitável.

O problema só foi identificado com a ajuda do [Sysdig](https://sysdig.com/), que apresentou as collection mais lentas do DB, e que para minha surpresa aquela era a collection com mais lentidão. [Você pode ler mais sobre como configurar um agente do sysdig no mongoDB clicando aqui](https://docs.sysdig.com/en/mongodb.html).

Para ilustrar melhor a situação, vamos supor que temos uma collection de 4 milhões de pessoas com o seguinte modelo:
```
{
	"_id": ObjectId("5fab44b1153579ed0dacdb88")
	"name": "danidr7",
	"cpf": "12345678900",
	"age": 28,
	"occupation": "programmer",
	"phone": "123456789"
}
```
[Usei esse script para gerar dados aleatórios no MongoDB e simular a situação.](https://gist.github.com/danidr7/f4e87ff2548db48001584e4b2466a913)

A primeira coisa que fiz foi identificar quais campos estavam sendo consultados naquela collection:
```
1. db.person.find({name: "danidr7", cpf: "12345678900"})
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
$ db.person.find({name: "danidr7", cpf: "12345678900"}).explain("executionStats")
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
						"$eq" : "danidr7"
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
						"[\"danidr7\", \"danidr7\"]"
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
						"[\"danidr7\", \"danidr7\"]"
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
		"host" : "my-host",
		"port" : 27017,
		"version" : "4.2.8",
		"gitVersion" : "43d25964249164d76d5e04dd6cf38f6111e21f5f"
	},
	"ok" : 1
}
```

O `explain()` exibe bastante detalhes sobre a consulta executada, mas vou dar destaque para 2 informações, o *winningPlan* e o *executionStats*. [As informações detalhadas sobre cada campo pode ser consultada clicando aqui](https://docs.mongodb.com/manual/reference/explain-results/).

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

- **nReturned**: é o número de documentos que a consulta retornou
- **executionTimeMillis**: é o tempo total da consulta
- **totalKeysExamined**: é o número de chaves examinadas, ou seja, vai ser maior que 0 quando o **IXSCAN** for usado
- **totalDocsExamined**: é o número de documentos examinados durante a execução da consulta, normalmente documentos são examinados nos estágios de **FETCH** e **COLLSCAN**

Levando em consideração os números do *executionStats*, podemos concluir que o resultado dessa consulta foi bom, pois o estágio de **IXSCAN** filtrou as chaves de index de forma objetiva, retornando apenas 1 resultado para o estágio de **FETCH** retornar o documento.

Após concluir que a primeira *query* estava 'performando' corretamente, realizei a mesma análise da segunda *query* (para melhorar a visualização vamos abstrair o resultado):
```
$ db.person.find({occupation: "programmer", cpf: "12345678900"}).explain("executionStats")
{
	...

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
	}
}
```

Podemos ver que, assim como na *query* anterior, temos o estágio de **IXSCAN** e o de **FETCH**, então sabemos que um índice está sendo usado.
Na raiz do *executionStats* podemos ver os seguintes números:
```
"executionStats" : {
	"nReturned" : 1,
	"executionTimeMillis" : 1101,
	"totalKeysExamined" : 799296,
	"totalDocsExamined" : 799296,
```

Agora sim percebe-se nitidamente uma quantidade absurda de documentos analisados e um tempo de execução (*executionTimeMillis*) bem maior que na *query* anterior.

Para entender o que aconteceu, é necessário observar o *executionStages*.
Como dito antes, o plano desta *query* contém 2 estágios, vamos começar analisando o **IXSCAN** (a forma correta de leitura do plano é do estágio mais aninhado para fora):
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

No **IXSCAN** nota-se que foi usado o índice do campo *occupation*, porém foram retornados *799296* documentos nesse estágio. O motivo desse resultado é que o índice contempla apenas 1 dos 2 campos pesquisados, então ele pode ajudar apenas filtrando o campo `occupation`, que por sua vez, retornou todos os *799296* documentos com *occupation* igual a *programmer*.

```
"executionStages" : {
			"stage" : "FETCH",
			"docsExamined" : 799296,
			"nReturned" : 1,
```

Agora no estágio **FETCH** foi examinado todos os *799296* documentos para retornar apenas 1, ou seja, o estágio está examinando documentos demasiadamente, vamos tentar melhorar o índice para otimizar esse trabalho!

```
$ db.person.createIndex({cpf: 1, occupation: 1})
{
	"createdCollectionAutomatically" : false,
	"numIndexesBefore" : 3,
	"numIndexesAfter" : 4,
	"ok" : 1
}
```

Foi criado um índice composto de *cpf* e *occupation*, mas se já temos um índice de *occupation*, porquê não criar só mais um índice de *cpf*?
O MongoDB ao criar o *winningPlan* vai selecionar apenas 1 índice para ser utilizado na consulta, então deve-se analisar bem quais campos são interessantes incluir no índice, atualmente o MongoDB suporta índices compostos de até 32 campos.

Agora vamos ver como ficou a performance após a criação do novo índice:
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
			"inputStage" : {
				"stage" : "IXSCAN",
				"keyPattern" : {
					"cpf" : 1,
					"occupation" : 1
				},
				"indexName" : "cpf_1_occupation_1",
				"isMultiKey" : false,
				"multiKeyPaths" : {
					"cpf" : [ ],
					"occupation" : [ ]
				},
				"isUnique" : false,
				"isSparse" : false,
				"isPartial" : false,
				"indexVersion" : 2,
				"direction" : "forward",
				"indexBounds" : {
					"cpf" : [
						"[\"12345678900\", \"12345678900\"]"
					],
					"occupation" : [
						"[\"programmer\", \"programmer\"]"
					]
				}
			}
		},
		"rejectedPlans" : [
			{
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
			}
		]
	},
	"executionStats" : {
		"executionSuccess" : true,
		"nReturned" : 1,
		"executionTimeMillis" : 11,
		"totalKeysExamined" : 1,
		"totalDocsExamined" : 1,
		"executionStages" : {
			"stage" : "FETCH",
			"nReturned" : 1,
			"executionTimeMillisEstimate" : 10,
			"works" : 3,
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
				"executionTimeMillisEstimate" : 10,
				"works" : 2,
				"advanced" : 1,
				"needTime" : 0,
				"needYield" : 0,
				"saveState" : 0,
				"restoreState" : 0,
				"isEOF" : 1,
				"keyPattern" : {
					"cpf" : 1,
					"occupation" : 1
				},
				"indexName" : "cpf_1_occupation_1",
				"isMultiKey" : false,
				"multiKeyPaths" : {
					"cpf" : [ ],
					"occupation" : [ ]
				},
				"isUnique" : false,
				"isSparse" : false,
				"isPartial" : false,
				"indexVersion" : 2,
				"direction" : "forward",
				"indexBounds" : {
					"cpf" : [
						"[\"12345678900\", \"12345678900\"]"
					],
					"occupation" : [
						"[\"programmer\", \"programmer\"]"
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
		"host" : "my-host",
		"port" : 27017,
		"version" : "4.2.8",
		"gitVersion" : "43d25964249164d76d5e04dd6cf38f6111e21f5f"
	},
	"ok" : 1
}
```

Podemos ver uma grande melhoria no tempo de resposta (*executionTimeMillis*) que passou de 1101ms para 11ms. No estágio de **IXSCAN** dentro do *executionStages* é apresentado que foi utilizado o índice composto que criamos anteriormente (*cpf_1_occupation_1*) e a quantidade de chaves retornadas para o estágio de **FETCH**, que foi apenas *1*. Já na consulta realizada antes de criar o índice composto, tivemos um retorno de *799296* no estágio de **IXSCAN**, ou seja, o índice criado funcionou adequadamente para essa consulta, passando a retornar apenas 1 chave neste mesmo estágio.

Outro campo interessante a se notar é o *rejectedPlans*, ele indica planos de consulta que eram candidatos a serem utilizados, mas o otimizador de consultas do MongoDB acabou rejeitando-o e optando por outro plano no qual considerou mais performático.
No *rejectedPlans* da nossa última consulta possui um plano utilizando o índice antigo *occupation_1*, nesse caso o índice não é mais útil nas consultas, então o melhor a se fazer é remover ele da collection.

#### Considerações finais

Devemos ser sempre cautelosos ao escolher os índices e lembrar que o MongoDB utiliza apenas 1 índice por consulta. A quantidade de índices também vai impactar na performance de escrita e para cada documento inserido na collection vai ser criado também uma nova chave para cada índice. [Você pode ler mais sobre a performance de escrita clicando aqui](https://docs.mongodb.com/manual/core/write-performance/).

Para fins de análise da *query* o *explain* é um grande aliado, podendo nos dar um visão clara do que acontece na execução da *query*. Porém muitas vezes a má performance das *queries* não é evidente, para isso temos algumas ferramentas no mercado que podem nos ajudar a identificar isso em um contexto mais amplo, como o [prometheus](https://prometheus.io/docs/instrumenting/exporters/) e o [sysdig](https://docs.sysdig.com/en/mongodb.html).