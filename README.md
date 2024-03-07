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
TODO
---------------------------------------------
### Using dynamic value as field name
TODO
---------------------------------------------
### Highly nested array
TODO
---------------------------------------------
### Storing as recusive structure
TODO
---------------------------------------------
### Scattering similar data/information in different collections/database
TODO