---
layout: post
title: '.NET Performance Analysis: Newtonsoft.Json vs System.Text.Json in .NET 9'
tags: json parsing performance
categories: blog
date: 2024-12-02
---

Today, I want to talk about something that's been on my mind lately - JSON serialization in .NET. As a passionate developer, I'm always curious about the best ways to solve problems, and JSON serialization is no exception.

Recently, I came across an article by [Trevor McCubbin](https://trevormccubbin.medium.com/net-performance-analysis-newtonsoft-json-vs-system-text-json-in-net-8-34520c21d054) that compared the performance of `System.Text.Json` and `Newtonsoft.Json` in .NET 8. I was intrigued by the results, and it got me thinking - how do these two popular JSON serialization libraries stack up against each other in .NET 9?

In this article, I'll share my findings and provide some insights into the performance and features of both `System.Text.Json` and `Newtonsoft.Json`.

## Benchmark Setup
To accurately compare the performance of `System.Text.Json` and `Newtonsoft.Json` in JSON serialization, I'll be focusing on two main use cases:

Serialization and Deserialization of a Single Large Dataset: This will provide insights into how each library handles bulk data.
Serialization and Deserialization of Many Small Datasets: This will showcase how each library performs when dealing with a large number of smaller payloads.
To generate diverse test data, I'll be utilizing the Bogus NuGet package to generate random users with unique identities. This will help us evaluate how each library handles different types of data.

```csharp
[GlobalSetup]
    public void GlobalSetup()
    {
        var faker = new Faker<Person>()
        .RuleFor(c => c.Name, f => f.Name.FullName())
        .RuleFor(c => c.Age, f => f.Random.Int(1, 100))
        .RuleFor(c => c.DateOfBirth, f => f.Date.Recent())
        .RuleFor(c => c.IsMarried, f => f.Random.Bool())
        .RuleFor(c => c.Children, f => f.Lorem.Words(f.Random.Int(1, 5)));

        persons = faker.Generate(Count);
        personsJson = System.Text.Json.JsonSerializer.Serialize(persons);

        personListJson = persons.Select(c => System.Text.Json.JsonSerializer.Serialize(c)).ToList();
    }
```

## Performance

### Deserialize Big Data
In this benchmark, we will compare deserialization performance of both approaches using a single large object.
```csharp
[BenchmarkCategory("DeserializeBigData"), Benchmark]
public void JsonSerializer_Deserialize_BigData()
{
    _ = System.Text.Json.JsonSerializer.Deserialize<List<Person>>(personsJson);
}

[BenchmarkCategory("DeserializeBigData"), Benchmark(Baseline = true)]
public void JsonConvert_Deserialize_BigData()
{
    _ = Newtonsoft.Json.JsonConvert.DeserializeObject<List<Person>>(personsJson);
}
```

### Results

| Method                             | Count | Mean     | Error    | StdDev   | Ratio | RatioSD | Allocated | Alloc Ratio |
|----------------------------------- |------ |---------:|---------:|---------:|------:|--------:|----------:|------------:|     
| JsonSerializer_Deserialize_BigData | 10000 | 20.88 ms | 0.394 ms | 0.368 ms |  0.78 |    0.02 |   3.72 MB |        0.85 |     
| JsonConvert_Deserialize_BigData    | 10000 | 26.78 ms | 0.535 ms | 0.657 ms |  1.00 |    0.03 |   4.39 MB |        1.00 |

 Just like in the .NET 8 analysis, `System.Text.Json` is the clear winner when it comes to deserialization performance

### Deserialize Small Data Objects
In this scenario, we test the deserialization of multiple small JSON objects.

```csharp
[BenchmarkCategory("DeserializeSmallData"), Benchmark]
    public void JsonSerializer_Deserialize_SmallData()
    {
        foreach (var person in personListJson)
        {
            _ = System.Text.Json.JsonSerializer.Deserialize<Person>(person);
        }
    }

    [BenchmarkCategory("DeserializeSmallData"), Benchmark(Baseline = true)]
    public void JsonConvert_Deserialize_SmallData()
    {
        foreach (var person in personListJson)
        {
            _ = Newtonsoft.Json.JsonConvert.DeserializeObject<Person>(person);
        }
    }
```
### Results

| Method                               | Count | Mean     | Error    | StdDev   | Ratio | Allocated | Alloc Ratio |
|------------------------------------- |------ |---------:|---------:|---------:|------:|----------:|------------:|
| JsonSerializer_Deserialize_SmallData | 10000 | 11.36 ms | 0.122 ms | 0.115 ms |  0.63 |   7.96 MB |        0.26 |
| JsonConvert_Deserialize_SmallData    | 10000 | 18.14 ms | 0.222 ms | 0.197 ms |  1.00 |  30.55 MB |        1.00 |

`System.Text.Json` once more demonstrates its superiority, outperforming `Newtonsoft.Json` in both speed and memory efficiency. Notably, it achieves this while using less than half the memory.

### Serialize Big Data Object
Now lets look into serialization of big data object

```csharp
[BenchmarkCategory("SerializeBigData"), Benchmark]
public void JsonSerializer_Serialize_BigData()
{
    _ = System.Text.Json.JsonSerializer.Serialize(persons);
}

[BenchmarkCategory("SerializeBigData"), Benchmark(Baseline = true)]
public void JsonConvert_Serialize_BigData()
{
    _ = Newtonsoft.Json.JsonConvert.SerializeObject(persons);
}
```

### Results

| Method                           | Count | Mean      | Error     | StdDev    | Ratio | RatioSD | Allocated | Alloc Ratio |    
|--------------------------------- |------ |----------:|----------:|----------:|------:|--------:|----------:|------------:|    
| JsonSerializer_Serialize_BigData | 10000 |  7.784 ms | 0.0830 ms | 0.0736 ms |  0.66 |    0.02 |   2.69 MB |        0.42 |    
| JsonConvert_Serialize_BigData    | 10000 | 11.788 ms | 0.2297 ms | 0.4022 ms |  1.00 |    0.05 |    6.4 MB |        1.00 |

Yet again, `System.Text.Json` performs better than `Newtonsoft.Json` in both execution time and memory.

### Serialise Small Data Object
This is a common scenario where we serialize several small data objects to JSON.
```csharp
[BenchmarkCategory("SerializeSmallData"), Benchmark]
    public void JsonSerializer_Serialize_SmallData()
    {
        foreach (var person in persons)
        {
            _ = System.Text.Json.JsonSerializer.Serialize(person);
        }
    }

    [BenchmarkCategory("SerializeSmallData"), Benchmark(Baseline = true)]
    public void JsonConvert_Serialize_SmallData()
    {
        foreach (var person in persons)
        {
            _ = Newtonsoft.Json.JsonConvert.SerializeObject(person);
        }
    }
```
### Results
| Method                             | Count | Mean      | Error     | StdDev    | Ratio | Allocated | Alloc Ratio |
|----------------------------------- |------ |----------:|----------:|----------:|------:|----------:|------------:|
| JsonSerializer_Serialize_SmallData | 10000 |  7.917 ms | 0.0683 ms | 0.0605 ms |  0.72 |   5.89 MB |        0.33 |
| JsonConvert_Serialize_SmallData    | 10000 | 10.990 ms | 0.0766 ms | 0.0717 ms |  1.00 |  17.63 MB |        1.00 |

And there are no surprises. `System.Text.Json` still outperforms `Newtonsoft.Json` in serializing multiple small data objects.

## Features
While `System.Text.Json` is a more lightweight and performant option, `Newtonsoft.Json` has some features that may be important to your application. Here are a few examples:

- **Serialization of complex objects**: `Newtonsoft.Json` has better support for serializing complex objects, such as those with circular references.
- **Custom converters**: `Newtonsoft.Json` has a more extensive set of custom converters, which can be useful for handling specific serialization scenarios.
- **LINQ to JSON**: `Newtonsoft.Json` has a LINQ to JSON API, which allows you to query and manipulate JSON data using LINQ.

However, `System.Text.Json` has some features that are not available in `Newtonsoft.Json`, such as:

- **Async serialization**: `System.Text.Json` has built-in support for async serialization, which can improve performance in certain scenarios.
- **Streaming**: `System.Text.Json` has built-in support for streaming, which can be useful for handling large JSON payloads.

## Conclusion
In conclusion, `System.Text.Json` is a more performant and lightweight option for working with JSON in .NET 9. While `Newtonsoft.Json` has some features that may be important to your application.

You can download this sample code from github https://github.com/updateaman/JsonSerializerVsJsonConvert/blob/main/Program.cs 
