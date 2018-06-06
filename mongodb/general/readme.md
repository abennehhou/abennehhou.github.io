# General

[Home](../../readme.md) > [MongoDb](../readme.md) > [General](./readme.md)

## Table of Contents

1. [Tips](#tips)
    * [Connect to a replicaset](#connect-to-a-replicaset)
    * [Backup and restore](#backup-and-restore)
    * [Get largest document](#get-largest-document)
    * [Find duplicates](#find-duplicates)
    * [Load file](#load-file)
    * [Save to file](#Save-to-file)

## Tips

### Connect to a replicaset

* From mongo shell: `mongo --host rs0/host1:port1,host2:port2,host3:port3 --username myuser --password mypassword mydb`
* From driver: `mongodb://myuser:mypassword@host1:port1,host2:port2,host3:port3?replicaSet=rs0&w=majority&readPreference=primary&w=majority`

### Backup and restore
* Create backup: `mongodump --host myhost --port 27017 --db mycollection --out "D:\backup\mongodump_2017_10_13"`
* Restore backup: `mongorestore --host myhost --port 27017 "D:\backup\mongodump_2017_10_13"`

### Get largest document

```javascript
var max = 0;
var id = 0;
db.mycollection.find().forEach(function(obj) {
    var curr = Object.bsonsize(obj); 
    if(max < curr) {
        max = curr;
		id = obj._id;
    } 
})
print(max);
print(id);
```

### Find duplicates

```javascript
db.mycollection.aggregate(
    {"$group" : { "_id": "$myproperty", "count": { "$sum": 1 } } },
    {"$match": {"_id" :{ "$ne" : null } , "count" : {"$gt": 1} } }, 
    {"$project": {"myproperty" : "$_id", "_id" : 0} }
)
```

### Load file

Example of loading a javascript file containing index creation:

```javascript
print('-- Reset indexes for mycollection --');

print('*** Drop indexes ***');
var dropResult = db.mycollection.dropIndexes();
printjson(dropResult);

print('*** Create indexes ***');
var createResult = db.mycollection.createIndexes(
    [
        { "myfield1" : 1, "date" : -1 },
        { "nested.myfield2" : 1}
    ]);

printjson(createResult);

var createIndexesSparse = db.mycollection.createIndexes(
    [
        { "sparse.field1": 1 },
        { "sparse.field2" : -1 }
    ],
    {
        sparse: true
    });

printjson(createIndexesSparse);

print('*** Get all indexes ***');
var indexes = db.mycollection.getIndexes();
indexes.forEach(function (index) {
    print(index.ns + '.' + index.name);
});
```
In the shell, type: `load("D:\\path\\to\\file\\initialize_indexes.js")`.

### Save to file

Example to save the result to a file: 
`mongo --quiet mydatabase --eval "printjson(db.mycollection.findOne({'_id' : ObjectId('59511bd38186f446441fc552')}))" > result.json`.
