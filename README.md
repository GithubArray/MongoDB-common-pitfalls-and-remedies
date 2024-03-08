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
TODO
#### Why this is bad?  
TODO
#### What would be suggested way(s) to avoid this?  
TODO
#### What would be the remedy if this issue already happen?  
TODO
---------------------------------------------
### Highly nested array
#### What is the Symptom?  
TODO
#### Why this is bad?  
TODO
#### What would be suggested way(s) to avoid this?  
TODO
#### What would be the remedy if this issue already happen?  
TODO
---------------------------------------------
### Storing as recusive structure
#### What is the Symptom?  
TODO
#### Why this is bad?  
TODO
#### What would be suggested way(s) to avoid this?  
TODO
#### What would be the remedy if this issue already happen?  
TODO
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