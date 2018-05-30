# C# Driver

[Home](../../readme.md) > [MongoDb](../readme.md) > [C# Driver](./readme.md)

Nuget package: `MongoDB.Driver`. It can be used in .Net standard, .Net core and .net framework projects.


## Table of Contents

1. [Database context](#database-context)
    * [Conventions](#conventions)
2. [Filtering](#filtering)
    * [Filters](#filters)
    * [Autocompletion](#Autocompletion)


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

Where are many built-in filters defined in `FilterDefinitionBuilder`.

Examples of usage:

```csharp
// Equals: my property equals expected value
Builders<Item>.Filter.Eq(x => x.Owner.Name, searchParameters.Owner)

// In: my property in a list of expected values
Builders<Item>.Filter.In(x => x.Owner.Name, searchParameters.Owners)

// ElemMatch: my property in a list in a list of expected values
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

Example: `{"name": "Wonka Candy Company", "namePats": ["wonka", "candy", "company"] }`.

Then, in the filters:

```csharp
var nameParts = searchParameters.CompanyName.Split(' ').Where(x => !string.IsNullOrWhiteSpace(x)).Select(x => x.ToLower()).ToList();
foreach (var namePart in nameParts)
{
    filter = filter & Builders<Company>.Filter.StartsWith(x => x.NameParts, namePart);
}
```

