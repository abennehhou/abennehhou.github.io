# C# Driver

[Home](../../readme.md) > [MongoDb](../readme.md) > [C# Driver](./readme.md)

## Introduction

Nuget package: `MongoDB.Driver`. It can be used in .Net standard, .Net core and .net framework projects.

For all the topics below, check this github repository: [Playground](https://github.com/abennehhou/spielplatz) to see an example of implementation.

## Table of Contents

1. [Database context](#database-context)
    * [Conventions](#conventions)
2. [Filtering](#filtering)
    * [Filters](#filters)
    * [Autocompletion](#autocompletion)
3. [Operations history](#operations-history)
4. [Working with bson documents](#working-with-bson-documents)


## Database context

Mongo client is thread safe and not disposable. It can be reused, no need to open or close connections.

Generally, we create a database context and initialize it in the repository.

Example:

```csharp

public class PlaygroundContext
{
    private const string CollectionNameItems = "items";

    private readonly IMongoDatabase _playgroundDatabase;

    public PlaygroundContext(string connectionString, string databaseName)
    {
        var client = new MongoClient(connectionString);
        _playgroundDatabase = client.GetDatabase(databaseName);
    }

    public IMongoCollection<Item> GetItemsCollection()
    {
        return _playgroundDatabase.GetCollection<Item>(CollectionNameItems);
    }
}

public class ItemsRepository : IItemsRepository
{
    private readonly PlaygroundContext _playgroundContext;

    public ItemsRepository(string connectionString, string databaseName)
    {
        _playgroundContext = new PlaygroundContext(connectionString, databaseName);
    }

    public async Task<Item> GetById(ObjectId id)
    {
        var collection = _playgroundContext.GetItemsCollection();
        var item = await collection.Find(x => x.Id == id).FirstOrDefaultAsync();
        return item;
    }
}

```

### Conventions

By default, when we manipulate documents using a C# class, the fields in the documents have a first letter in upper case (convention of c# properties).
To have them lower case (mongodb convention), we need to specify it in the datamodel, or register conventions.
There are other cases where we need to use conventions: ignore null values, save enumerations as string, ignore elements that are in document but not in the model, etc.

We can register these conventions once when we initialize the context.

```csharp
public PlaygroundContext(string connectionString, string databaseName)
{
    RegisterConventions();
    var client = new MongoClient(connectionString);
    _playgroundDatabase = client.GetDatabase(databaseName);
}

private void RegisterConventions()
{
    ConventionRegistry.Register(
        "IgnoreNullValues",
        new ConventionPack
        {
            new IgnoreIfNullConvention(true)
        },
        t => true);

    ConventionRegistry.Register(
        "CamelCaseElementName",
        new ConventionPack
        {
            new CamelCaseElementNameConvention()
        },
        t => true);

    ConventionRegistry.Register(
        "EnumAsString",
        new ConventionPack
        {
            new EnumRepresentationConvention(BsonType.String)
        }, t => true);
    ConventionRegistry.Register(
        "IgnoreExtraElements",
        new ConventionPack
        {
            new IgnoreExtraElementsConvention(true)
        }, t => true);
}
```

## Filtering

When we query a collection, we can build filters using Linq, or using builders.

I prefer using builders, to be sure that the generated query corresponds to what I expect.
In all cases, we can log the query before sending it.

Example:

```csharp

public async Task<IPagedList<Item>> GetItems(ItemSearchParameter searchParameters)
{
    var collection = _playgroundContext.GetItemsCollection();
    var filter = Builders<Item>.Filter.Empty;

    if (!string.IsNullOrEmpty(searchParameters.Name))
        filter = filter & Builders<Item>.Filter.Eq(x => x.Name, searchParameters.Name);

    if (!string.IsNullOrEmpty(searchParameters.Owner))
        filter = filter & Builders<Item>.Filter.Eq(x => x.Owner, searchParameters.Owner);

    if (!string.IsNullOrEmpty(searchParameters.Tag))
        filter = filter & Builders<Item>.Filter.AnyEq(x => x.Tags, searchParameters.Tag);

    var query = collection.Find(filter)
        .SortBy(acc => acc.Id)
        .Skip(searchParameters.Skip)
        .Limit(searchParameters.Limit);

    _logger.LogDebug($"Get items query: {query}.");
    var items = await query.ToListAsync();
    var totalRows = (int)await collection.CountAsync(filter);
    return items.ToPagedList(searchParameters.Skip, searchParameters.Limit, totalRows);
}

```

### Filters

There are many built-in filters defined in `FilterDefinitionBuilder`.

Examples of usage:

```csharp
// Equals: my property equals the expected value
Builders<Item>.Filter.Eq(x => x.Owner.Name, searchParameters.Owner)

// In: my property is in a list of expected values
Builders<Item>.Filter.In(x => x.Owner.Name, searchParameters.Owners)

// ElemMatch: the property of at least one item of my collection equals the expected value
Builders<Post>.Filter.ElemMatch(x => x.Comments, y => y.Title == searchParameters.CommentTitle)
```

We can extend the filter definitions. Example:

```csharp
public static class FilterDefinitionBuilderExtensions
{
    public static FilterDefinition<TDocument> EqualsCaseInsensitive<TDocument>(this FilterDefinitionBuilder<TDocument> builder, Expression<Func<TDocument, object>> field, string value)
    {
        return Builders<TDocument>.Filter.Regex(field, new BsonRegularExpression($"/^{value}$/i"));
    }

    public static FilterDefinition<TDocument> StartsWith<TDocument>(this FilterDefinitionBuilder<TDocument> builder, Expression<Func<TDocument, object>> field, string value)
    {
        return Builders<TDocument>.Filter.Regex(field, new BsonRegularExpression($"/^{value}/"));
    }
}
```

### Autocompletion

For autocompletion, we search if the property _Contains_ a value. Example: search a Company with name like _Wonka Candy Company_.

```csharp
public static FilterDefinition<TDocument> ContainsCaseInsensitive<TDocument>(this FilterDefinitionBuilder<TDocument> builder, Expression<Func<TDocument, object>> field, string value)
{
    return Builders<TDocument>.Filter.Regex(field, new BsonRegularExpression($"/{value}/i"));
}

Builders<Company>.Filter.ContainsCaseInsensitive(x => x.Name, searchParameters.CompanyName);
```

Problem: it doesn't use the index on the property, if there is one. The search will be very slow if there are many documents: it will perform a full scan.

To fix this issue, save a new property with all the properties lower case.

Example: `{"name": "Wonka Candy Company", "nameParts": ["wonka", "candy", "company"] }`.

Then, in the filters:

```csharp
var nameParts = searchParameters.CompanyName.Split(' ').Where(x => !string.IsNullOrWhiteSpace(x)).Select(x => x.ToLower()).ToList();
foreach (var namePart in nameParts)
{
    filter = filter & Builders<Company>.Filter.StartsWith(x => x.NameParts, namePart);
}
```

To update a collection and add the missing properties for existing documents:

```javascript
var info = db.mycollection.aggregate([
  { $match: {"nameParts": {$exists: false}}},
  { $project : { nameParts : { $split: [{$toLower:"$name"}, " "] }, _id : 1 } }
]);


info.forEach(
    function (elem) {
        db.mycollection.update(
            {_id: elem._id},
            {
                $set: {
                    "nameParts": elem.nameParts
                }
            }
        );
    }
);
```

## Operations history

To save the history of operations (differences during a update, creation and delete operations), first, compute the differences between two BsonDocuments.

```csharp
public class Difference
{
    public string PropertyName { get; set; }

    public BsonValue Value1 { get; set; }

    public BsonValue Value2 { get; set; }
}

public static class BsonDocumentExtensions
{
    private const string PropertyNameDocument = "Document";

    public static List<Difference> GetDifferences(this BsonDocument document1, BsonDocument document2)
    {
        var differences = new List<Difference>();
        if (document1 == null && document2 == null)
            return differences;

        if (document1 == null || document2 == null)
        {
            differences.Add(new Difference { PropertyName = PropertyNameDocument, Value1 = document1, Value2 = document2 });
            return differences;
        }
        var objectsToCompare = new Stack<Difference>();
        objectsToCompare.Push(new Difference { PropertyName = PropertyNameDocument, Value1 = document1, Value2 = document2 });
        while (objectsToCompare.Count > 0)
        {
            var objectToCompare = objectsToCompare.Pop();
            var name = objectToCompare.PropertyName;
            var object1 = objectToCompare.Value1;
            var object2 = objectToCompare.Value2;
            var diff = new Difference { PropertyName = objectToCompare.PropertyName, Value1 = object1, Value2 = object2 };
            if (object1 == null && object2 == null)
                continue;

            if (object1 == null || object2 == null)
            {
                differences.Add(diff);
                continue;
            }

            // Checks for BsonDocument
            if (object1.IsBsonDocument)
            {
                if (!object2.IsBsonDocument)
                {
                    differences.Add(diff);
                    continue;
                }

                var elementsInObject1 = object1.AsBsonDocument.Elements.ToList();
                var elementsInObject2 = object2.AsBsonDocument.Elements.ToList();
                foreach (var element in elementsInObject1)
                {
                    var matchingElementValue = elementsInObject2.Where(x => x.Name == element.Name).Select(x => x.Value).FirstOrDefault();
                    objectsToCompare.Push(new Difference { PropertyName = $"{name}.{element.Name}", Value1 = element.Value, Value2 = matchingElementValue });
                }
                foreach (var element in elementsInObject2)
                {
                    var matchingElementValue = elementsInObject1.Where(x => x.Name == element.Name).Select(x => x.Value).FirstOrDefault();
                    if (matchingElementValue == null)
                        objectsToCompare.Push(new Difference { PropertyName = $"{name}.{element.Name}", Value1 = null, Value2 = element.Value });
                }
            }
            else if (object1.IsBsonArray)
            {
                // Checks for list
                if (!object2.IsBsonArray)
                {
                    differences.Add(diff);
                    continue;
                }
                var array1 = object1.AsBsonArray.OrderBy(x => x).ToArray();
                var array2 = object2.AsBsonArray.OrderBy(x => x).ToArray();
                for (int i = 0; i < array1.Length; i++)
                {
                    objectsToCompare.Push(new Difference { PropertyName = $"{name}[{i}]", Value1 = array1[i], Value2 = array2.ElementAtOrDefault(i) });
                }
                if (array2.Length > array1.Length)
                {
                    for (int i = array1.Length; i < array2.Length; i++)
                    {
                        objectsToCompare.Push(new Difference { PropertyName = $"{name}[{i}]", Value1 = null, Value2 = array2[i] });
                    }
                }
            }
            else
            {
                // Checks for simple element
                if (!object1.Equals(object2))
                {
                    differences.Add(new Difference { PropertyName = name, Value1 = object1, Value2 = object2 });
                }
            }
        }

        return differences;
    }
}
```

Create an operation class, to be instantiated during creation, update and delete.

```csharp
public class Operation
{
    [BsonId]
    public ObjectId Id { get; set; }
    public string EntityId { get; set; }
    public string EntityType { get; set; }
    public DateTime Date { get; set; }
    public string OperationType { get; set; }
    public List<Difference> Differences { get; set; }
}
```

Example of operations:

```json
{
    "id": "5b130e271469c43c08949dab",
    "entityId": "5b130e271469c43c08949daa",
    "entityType": "item",
    "date": "2018-06-02T21:37:43.381Z",
    "operationType": "Create"
},
{
    "id": "5b130f482f6fa952604bd3aa",
    "entityId": "5b130e271469c43c08949daa",
    "entityType": "item",
    "date": "2018-06-02T21:42:32.802Z",
    "operationType": "Update",
    "differences": [
    {
        "propertyName": "Document.tags[1]",
        "value2": "work"
    },
    {
        "propertyName": "Document.owner",
        "value1": "me",
        "value2": "you"
    },
    {
        "propertyName": "Document.description",
        "value1": "my awesome book, etc",
        "value2": "my book"
    }
    ]
},
{
    "id": "5b130ff005fead3d90756d17",
    "entityId": "5b130e271469c43c08949daa",
    "entityType": "item",
    "date": "2018-06-02T21:45:20.436Z",
    "operationType": "Delete"
}
```
More details in the github project provided in the introduction.

## Working with bson documents

Documents in a collection can be manipulated as bson documents. Example:

```csharp
var collection = playgroundDatabase.GetCollection<BsonDocument>(MyCollectionName);
var result = await collection.DeleteOneAsync(x => x["_id"] == id);
```

It is possible to use BsonDocuments in .Net Core api: map it to a _dynamic_ object.

Example for mapping in an AutoMapper profile:

```csharp
CreateMap<BsonDocument, dynamic>()
    .ConvertUsing((document, y) =>
    {
        if (document == null)
            return null;

        var json = document.ToJson(new JsonWriterSettings { OutputMode = JsonOutputMode.Strict });
        return Newtonsoft.Json.JsonConvert.DeserializeObject<dynamic>(json);
    });

CreateMap<dynamic, BsonDocument>()
    .ConvertUsing((x, y) =>
    {
        var json = (x == null) ? "{}" : Newtonsoft.Json.JsonConvert.SerializeObject(x);
        BsonDocument document = BsonDocument.Parse(json);
        return document;
    });
```

Example of usage in a controller:

```csharp

[HttpGet]
[ProducesResponseType(typeof(List<dynamic>), 200)]
public async Task<IActionResult> Get()
{
    var products = await _productsService.GetAllAsync();
    var result = _mapper.Map<List<dynamic>>(products);
    return Ok(result);
}

[HttpPut]
[ProducesResponseType(200)]
[Route("{id}")]
public async Task<IActionResult> Put(string id, [FromBody]dynamic product)
{
    if (product == null)
        throw new ValidationApiException(ApiErrorCode.MissingInformation, $"Parameter {nameof(product)} must be provided.");

    BsonDocument productDocument = _mapper.Map<BsonDocument>(product);

    var productId = productDocument.GetValue("_id", null)?.ToString();

    if (productId != id)
        throw new ValidationApiException(ApiErrorCode.InvalidInformation, $"ProductId must be the same in the uri and in the body. Provided values: '{id}' and '{productId}'.");

    await _productsService.ReplaceAsync(productDocument);

    var updatedProduct = _mapper.Map<dynamic>(productDocument);

    return Ok(updatedProduct);
}

```

