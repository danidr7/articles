## Seu índice no MongoDB está sendo bem utilizado?

Recentemente no trabalho me deparei com uma collection no MongoDB que apresentou uma certa lentidão em suas queries. Essa lentidão não estava evidente, pois embora a collection seja grande (4 milhões de documento) a consulta a essa collection era realizada dentro de um processo maior, no qual tinham diversos procedimentos que somavam um tempo de retorno aceitável. 

O problema só foi identificado com a ajuda do [Sysdig](https://sysdig.com/), que apresentou as collection mais lentas do DB, e que para minha surpresa aquela era a collection com mais lentidão. [Você pode ler mais sobre como configurar um agente do sysdig no mongoDB clicando aqui](https://docs.sysdig.com/en/mongodb.html).

Para ilustrar melhor o problema vamos supor que temos uma collection de 4 milhões de pessoas:
```
{
	"_id": ObjectId("")
	"name": "Daniel",
	"cpf": "12345678900",
	"age": 28,
	"occupation": "programmer",
	"phone": "123456789",
	"favorite-dev-group": "whitestone"
}
```

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
		"ns" : "db.person"
	},
	{
		"v" : 2,
		"unique" : true,
		"key" : {
			"name" : 1,
			"cpf" : 1
		},
		"name" : "name_1_cpf_1_idx",
		"ns" : "db.person"
	},
	{
		"v" : 2,
		"key" : {
			"occupation" : 1
		},
		"name" : "occupation",
		"ns" : "db.person"
	}
]
```

Beleza, a gente tem um índice composto de 'name' e 'cpf' e outro índice individual de 'occupation', dá pra melhorar esses índices! Mas o que está acontecendo quando o MongoDB executa essas queries? O MongoDB está fazendo uso dos índices?
Aí que entra o [explain()](https://docs.mongodb.com/manual/reference/method/cursor.explain/)!
O explain vai ajudar a gente a entender qual é o plano da consulta:
```
$ db.person.find({name: "Daniel", cpf: "12345678900"}).explain()
{
	"queryPlanner" : {
		"plannerVersion" : 1,
		"namespace" : "db.person",
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
		"queryHash" : "3C4EBBFD",
		"planCacheKey" : "50CE7ABE",
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
	"serverInfo" : {
		"host" : "PC-002387",
		"port" : 27017,
		"version" : "4.2.8",
		"gitVersion" : "43d25964249164d76d5e04dd6cf38f6111e21f5f"
	},
	"ok" : 1
}

```

----`WIP`----