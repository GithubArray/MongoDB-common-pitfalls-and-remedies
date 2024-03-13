Below is a compiled list of common pitfalls I saw over the years on stackoverflow. I am going to propose common remedies to them also.


---------------------------------------------
### Storing dates as non-date objects
#### What is the Symptom?  
Date are not stored as [date object](https://www.mongodb.com/docs/manual/reference/method/Date/); but as strings or timestamp...
e.g.  
```
{
    "dateInString1": "2024-01-01",
    "dateInString2": "13/09/2024",
    "dateInString3": "13/Feb/2024",
    "dateInTimestamp": 1709777005000
}
```

#### Why this is bad?
MongoDB offers a rich set of date APIs like `$dateAdd`, `$dateSubstract`, `$dateTrunc`... You cannot leverage these APIs when you are not storing them as proper date objects.

#### What would be suggested way(s) to avoid this?
Storing date as proper date objects.
```
{
    "date": ISODate("2024-01-01")
}
```

#### What would be the remedy if this issue already happen?
1. clean up existing data by using [$toDate](https://www.mongodb.com/docs/manual/reference/operator/aggregation/toDate/); if they are in standard format or timestamp
```
db.collection.update({},
[
  {
    "$set": {
      "date": {
        "$toDate": "$dateInString1"
      }
    }
  }
])
```
[Mongo Playground](https://mongoplayground.net/p/-Cge5p7hEq1)

2. clean up exising data by using [$dateFromString](https://www.mongodb.com/docs/manual/reference/operator/aggregation/dateFromString/); if they are in some formatted date strings
```
db.collection.aggregate([
  {
    "$set": {
      "date": {
        "$dateFromString": {
          "dateString": "$dateInString2",
          "format": "%d/%m/%Y"
        }
      }
    }
  }
])
```
[Mongo Playground](https://mongoplayground.net/p/fCpiLzX6LHi)

3. break up the date into tokens using `$split` and use [$dateFromParts](https://www.mongodb.com/docs/manual/reference/operator/aggregation/dateFromParts/) to reform the date object if they are in non-standard format or you need extra parsing.

Below example assume:
1. the date format is dd MMM yyyy HH:mm:ss zzz
    - i.e. "01 Aug 2020 06:26:09 GMT"
1. the timezone is always in GMT(timezone info will be ignored in following conversion script)

Steps:
1. `$split` the date string into tokens for processing
1. try to locate the time string by checking `$indexOfCP` with `:`. If it is a time string, `$split` into tokens and put them back into the original array
1. use an array of month with `$indexOfArray` to convert them into int values(i.e. Jan to 1, Feb to 2 ...); Meanwhile, convert other string tokens into int
1. Use `$dateFromParts` with tokens to construct proper date object
1. `$merge` back to the collection for update

```js
db.collection.aggregate([
  // break into tokens for processing
  {
    "$addFields": {
      "tokens": {
        "$split": [
          "$date",
          " "
        ]
      }
    }
  },
  // try to parse time part and break into hh, mm, ss
  {
    "$addFields": {
      "tokens": {
        "$reduce": {
          "input": "$tokens",
          "initialValue": [],
          "in": {
            "$cond": {
              "if": {
                $ne: [
                  -1,
                  {
                    "$indexOfCP": [
                      "$$this",
                      ":"
                    ]
                  }
                ]
              },
              "then": {
                "$concatArrays": [
                  "$$value",
                  {
                    "$split": [
                      "$$this",
                      ":"
                    ]
                  }
                ]
              },
              "else": {
                "$concatArrays": [
                  "$$value",
                  [
                    "$$this"
                  ]
                ]
              }
            }
          }
        }
      }
    }
  },
  // try to 1. parse month part and 2. convert into int
  {
    "$addFields": {
      "tokens": {
        $let: {
          vars: {
            tokens: "$tokens",
            monthArray: [
              "dummy",
              "Jan",
              "Feb",
              "Mar",
              "Apr",
              "May",
              "Jun",
              "Jul",
              "Aug",
              "Sep",
              "Oct",
              "Nov",
              "Dec"
            ]
          },
          in: {
            "$map": {
              "input": "$$tokens",
              "as": "t",
              "in": {
                "$switch": {
                  "branches": [
                    {
                      "case": {
                        "$in": [
                          "$$t",
                          "$$monthArray"
                        ]
                      },
                      "then": {
                        "$indexOfArray": [
                          "$$monthArray",
                          "$$t"
                        ]
                      }
                    }
                  ],
                  default: {
                    "$convert": {
                      "input": "$$t",
                      "to": "int",
                      "onError": "$$t"
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  {
    "$addFields": {
      "parsedDate": {
        "$dateFromParts": {
          "year": {
            "$arrayElemAt": [
              "$tokens",
              2
            ]
          },
          "month": {
            "$arrayElemAt": [
              "$tokens",
              1
            ]
          },
          "day": {
            "$arrayElemAt": [
              "$tokens",
              0
            ]
          },
          "hour": {
            "$arrayElemAt": [
              "$tokens",
              3
            ]
          },
          "minute": {
            "$arrayElemAt": [
              "$tokens",
              4
            ]
          },
          "second": {
            "$arrayElemAt": [
              "$tokens",
              5
            ]
          }
        }
      }
    }
  },
  // cosmetics
  {
    "$project": {
      "date": "$parsedDate"
    }
  },
  // update back to collection
  {
    "$merge": {
      "into": "collection",
      "on": "_id",
      "whenMatched": "merge"
    }
  }
])
```
[Mongo Playground](https://mongoplayground.net/p/VPUAIMlRFeF)

---------------------------------------------
### Storing at wrong query level
#### What is the Symptom?  
Contrary to your frequent query pattern, you are storing your information in different level. For example in a customer data analytics system, your major query is concerned around customers transactions, like monthly statistics aggregation. However, your transaction data is stored as array entries in a customer document.

#### Why this is bad?  
1. You suffer from query complexity. Even if you can use complicated aggregation to wrangle the data, it introduces code smell and tech debt.
2. You are unlikely to take advantage from index.
3. When data grow, 16MB MongoDB single document size could be breached.

#### What would be suggested way(s) to avoid this?  
You should denormalize your data as an single document.  
Following our scenario of monthly transactions data aggregation, the `customer` collection may look like this:
```
db={
  "customer": [
    {
      "_id": 1,
      "name": "Alice",
      "transactions": [
        {
          _id: "tx1",
          date: ISODate("2024-01-01"),
          amount: 100
        },
        {
          _id: "tx2",
          date: ISODate("2024-01-02"),
          amount: 200
        },
        {
          _id: "tx3",
          date: ISODate("2024-02-01"),
          amount: 300
        },
        {
          _id: "tx4",
          date: ISODate("2024-03-01"),
          amount: 400
        }
      ]
    },
    {
      "_id": 2,
      "name": "Bob",
      "transactions": [
        {
          _id: "tx5",
          date: ISODate("2024-02-02"),
          amount: 1000
        },
        {
          _id: "tx6",
          date: ISODate("2024-03-03"),
          amount: 1000
        }
      ]
    }
  ]
}
```
You will need following (slightly) complex aggregation to get monthly sales aggregation data, which may be much more complex if you have sophisticated business logic.
```
db.customer.aggregate([
  {
    "$unwind": "$transactions"
  },
  {
    "$group": {
      "_id": {
        $dateTrunc: {
          date: "$transactions.date",
          unit: "month"
        }
      },
      "totalAmount": {
        "$sum": "$transactions.amount"
      }
    }
  }
])
```
[Mongo Playground](https://mongoplayground.net/p/XEL2c__eJSv)
You can refactor the schema to storing them as individual documents. There are 2 ways to do that:
1. denormalize common fields
If customer data is not that important or does not change frequently, you can simply duplicate/denormalize the field into new transactions documents.
```
db={
  "transactions": [
    {
      "_id": "tx1",
      "customerName": "Alice",
      "date": ISODate("2024-01-01"),
      "amount": 100
    },
    {
      "_id": "tx2",
      "customerName": "Alice",
      "date": ISODate("2024-01-02"),
      "amount": 200
    },
    {
      "_id": "tx3",
      "customerName": "Alice",
      "date": ISODate("2024-02-01"),
      "amount": 300
    },
    {
      "_id": "tx4",
      "customerName": "Alice",
      "date": ISODate("2024-03-01"),
      "amount": 400
    }
  ]
}
```
A single `$group` will serve our previous purpose.
```
db.transactions.aggregate([
  {
    "$group": {
      "_id": {
        $dateTrunc: {
          date: "$date",
          unit: "month"
        }
      },
      "totalAmount": {
        "$sum": "$amount"
      }
    }
  }
])
```
[Mongo Playground](https://mongoplayground.net/p/1ULw0HLtf4t)  

2. separating customer data into separate collection
If customer data is important/will be changed frequently, it might not be a good idea to scatter them around each transactions as it involves a lot of updates. You can segregate the data into an individual document.
```
db={
  "transactions": [
    {
      "_id": "tx1",
      "customerId": 1,
      "date": ISODate("2024-01-01"),
      "amount": 100
    },
    {
      "_id": "tx2",
      "customerId": 1,
      "date": ISODate("2024-01-02"),
      "amount": 200
    },
    {
      "_id": "tx3",
      "customerId": 1,
      "date": ISODate("2024-02-01"),
      "amount": 300
    },
    {
      "_id": "tx4",
      "customerId": 1,
      "date": ISODate("2024-03-01"),
      "amount": 400
    }
  ],
  "customer": [
    {
      "_id": 1,
      "name": "Alice"
    }
  ]
}
```
You can perform `$lookup` to the `customer` collection to retrieve customer data.

#### What would be the remedy if this issue already happen?  
You can refactor your schema and perform data migration according to above suggestions. The most important thing is that you need to identify your common query pattern. That would be a rather sophisticated topic so we don't go into details here. Once you identify the most common query, you can rafctor your schema towards a schema that is more suitable for your query needs.

---------------------------------------------
### Using dynamic value as field name
#### What is the Symptom?  
Instead of fixed field name, dynamic/non-constant values are used as field name. An forex exchange(FX) schema may looks like this:
```
[
  {
    "mainCurrency": "USD",
    "fxRate": {
      "CAD": {
        "rate": 1.35
      },
      "GBP": {
        "rate": 0.78
      },
      "EUR": {
        "rate": 0.91
      }
    }
  }
]
```

#### Why this is bad?  
1. This introduces unnecessary complexity to query. Usually `$objectToArray` and `$arrayToObject` is required to manipulate the object when you perform filtering / mapping, which is a non-trivial process.
2. Index cannot be built to improve performance
3. The schema cannot be well-defined. This may hinder documentation and future integration.

#### What would be suggested way(s) to avoid this?  
Make the field a key-value(kv) tuple, potentially as array entries.
```
[
  {
    "mainCurrency": "USD",
    "fxRate": [
      {
        "currency": "CAD",
        "rate": 1.35
      },
      {
        "currency": "GBP",
        "rate": 0.78
      },
      {
        "currency": "EUR",
        "rate": 0.91
      }
    ]
  }
]
```

In this way, you can avoid the complexity from object-array conversion and you can index the `fxRate.currency` field to improve performance. 

PS. if you found yourself frequently working at the `fxRate` level, consider refactoring them to individual element. See [Storing at wrong query level](#storing-at-wrong-query-level)

#### What would be the remedy if this issue already happen?  
1. You can refactor your schema and perform data migration according to above suggestions. 
2. You can use `$objectToArray` to convert the object to kv tuples.
```
db.collection.update({},
[
  {
    "$set": {
      "fxRate": {
        "$map": {
          "input": {
            "$objectToArray": "$fxRate"
          },
          "as": "kv",
          "in": {
            currency: "$$kv.k",
            rate: "$$kv.v.rate"
          }
        }
      }
    }
  }
])
```
[Mongo Playground](https://mongoplayground.net/p/9Akx8sXBHwR)
3. You may partially mitigate the issue through usage of [wildcard index](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-wildcard/) if your only concern is performance and relatively simple structure.

---------------------------------------------
### Highly nested array
#### What is the Symptom?  
You are nesting arrays, in a complicated and usually unnecessary manner. In this course management schema, you can see the students are allocated in nested arrays.
```
db={
  "courses": [
    {
      _id: 1,
      title: "Introduction to Computer Science",
      studentGroups: [
        {
          group: "A",
          students: [
            1,
            2
          ]
        },
        {
          group: "B",
          students: [
            3
          ]
        }
      ]
    },
    {
      _id: 2,
      title: "Chemistry",
      studentGroups: [
        {
          group: "C",
          students: [
            2,
            3
          ]
        },
        {
          group: "B",
          students: [
            4,
            5
          ]
        }
      ]
    }
  ],
  "students": [
    {
      _id: 1,
      name: "Henry",
      age: 15
    },
    {
      _id: 2,
      name: "Kim",
      age: 20
    },
    {
      _id: 3,
      name: "Michel",
      age: 14
    },
    {
      _id: 4,
      name: "Max",
      age: 16
    },
    {
      _id: 5,
      name: "Nathan",
      age: 19
    }
  ]
}
```

#### Why this is bad?  
1. This introduces unnecessary complexity to query. A often use case is to populate the innermost layer of array from another collection, like `students` collection's data in the above example. That will requires convoluted `$lookup`, `$mergeObjects`, `$map`/`$filter`...
2. Queries cannot be benefited from index.

#### What would be suggested way(s) to avoid this?  
Again, you need to figure our your most frequent query pattern first. There is no generic suggested way for this issue. However, you will likely find out the suggestions in [Storing at wrong query level](#storing-at-wrong-query-level) works here too.

#### What would be the remedy if this issue already happen?  
See [Storing at wrong query level](#storing-at-wrong-query-level)

---------------------------------------------
### Storing as recusive structure
#### What is the Symptom?  
Information are stored as recursive structure, often with dynamic recursive depth. Consider a family tree document like this:
```
db={
  "family": [
    {
      _id: 1,
      familyName: "Johnson",
      familyTree: {
        member: 1,
        children: [
          {
            member: 2,
            children: [
              {
                member: 3,
                children: [
                  {
                    member: 4,
                    children: []
                  },
                  {
                    member: 5,
                    children: []
                  }
                ]
              }
            ]
          }
        ]
      }
    }
  ]
}
```

#### Why this is bad?  
1. This introduces unnecessary complexity to query. A often use case is to populate the `member` data from another collection, which requires complicated `$lookup` and `$mergeObjects`...
2. For complex cases like a big family, 16 MB document limit size may be breached.
3. Queries cannot be benefited from index.

#### What would be suggested way(s) to avoid this?  
Storing the `member` node in individual document with link to parent/children. Use `$graphLookup` to construct the tree.
```
db={
  "member": [
    {
      _id: 1,
      name: "great grandpa",
      children: [
        2
      ]
    },
    {
      _id: 2,
      name: "grandpa",
      children: [
        3
      ]
    },
    {
      _id: 3,
      name: "dad",
      children: [
        4,
        5
      ]
    },
    {
      _id: 4,
      name: "son"
    },
    {
      _id: 5,
      name: "son2"
    }
  ]
}
```
Construct the tree with `$graphLookup`:
```
db.member.aggregate([
  {
    $match: {
      "_id": 1
    }
  },
  {
    "$graphLookup": {
      "from": "member",
      "startWith": "$_id",
      "connectFromField": "children",
      "connectToField": "_id",
      "as": "children"
    }
  }
])
```
[Mongo Playground](https://mongoplayground.net/p/Onz5w3k7vsC)

#### What would be the remedy if this issue already happen?  
You need to iterate your existing recursive structure and flatten the structure through some application level processing.

---------------------------------------------
### Scattering similar data/information in different collections/database
#### What is the Symptom?  
TODO
#### Why this is bad?  
TODO
#### What would be suggested way(s) to avoid this?  
TODO
#### What would be the remedy if this issue already happen?  
TODO