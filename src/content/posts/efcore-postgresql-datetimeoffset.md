---
title: EFCore PostgreSql DateTimeOffset报错解决方案
published: 2024-11-18
description: ''
image: ''
tags: ["CSharp", "EFCore", "PostgreSql"]
category: '笔记'
draft: false 
lang: ''
---

在EFCore的PostgreSql中使用DateTimeOffset可能会报错:

```log
  ---> System.ArgumentException: Cannot write DateTimeOffset with Offset=08:00:00 to PostgreSQL type 'timestamp with time zone', only offset 0 (UTC) is supported.  (Parameter 'value')
```

Npgsql 支持将 DateTimeOffset 读取和写入具有时区的时间戳，但仅限于 Offset=0  

官方文档说明: <https://www.npgsql.org/doc/types/datetime.html#timestamps-and-timezones>

## 解决方案1

在 Main 方法中加上 `AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);`

## 解决方案2

使用转换器  

```csharp
public class DateTimeOffsetUtcConverter : ValueConverter<DateTimeOffset, DateTimeOffset>
{
    public static ValueConverterInfo Instance { get; } = new ValueConverterInfo(typeof(DateTimeOffset), typeof(DateTimeOffset), (i) => new DateTimeOffsetUtcConverter(i.MappingHints));

    public DateTimeOffsetUtcConverter() : this(null) { }

    public DateTimeOffsetUtcConverter(ConverterMappingHints? mappingHints) : base((v) => ToUtc(v), (v) => ToLocal(v), mappingHints) { }

    public static DateTimeOffset ToLocal(DateTimeOffset v)
        => v.ToLocalTime();

    public static DateTimeOffset ToUtc(DateTimeOffset v)
        => v.ToUniversalTime();
}
```

然后在 DbContext 中:  

```csharp
public class MyDbContext 
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        if (provider.Contains("postgresql", StringComparison.OrdinalIgnoreCase))
        {
            foreach (var entityType in modelBuilder.Model.GetEntityTypes())
            {
                var properties = entityType.ClrType.GetProperties()
                    .Where(p => p.PropertyType == typeof(DateTimeOffset) || p.PropertyType == typeof(DateTimeOffset?));
                foreach (var property in properties)
                {
                    modelBuilder
                        .Entity(entityType.Name)
                        .Property(property.Name)
                        .HasConversion(new DateTimeOffsetUtcConverter());
                }
            }
        }
    }
}

```
