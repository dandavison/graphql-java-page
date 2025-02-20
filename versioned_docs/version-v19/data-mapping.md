---
title: "Data mapping"
date: 2018-09-09T12:52:46+10:00
description: How graphql-java maps object data to graphql types
---
# Mapping data

## How graphql maps object data to types

At its heart graphql is all about declaring a type schema and mapping that over backing runtime data.

As the designer of the type schema, it is your challenge to get these elements to meet in the middle.

For example imagine we want to have a graphql type schema as follows:

```graphql
type Query {
  products(match : String) : [Product]   # a list of products
}

type Product {
  id : ID
  name : String
  description : String
  cost : Float
  tax : Float
}
```

We could then run queries over this simple schema via a something like the following:

```graphql
query ProductQuery {
  products(match : "Paper*")
  {
    id, name, cost, tax
  }
}
```

We will have a ``DataFetcher`` on the ``Query.products`` field that is responsible for finding a list of products that match
the argument passed in.

Now imagine we have 3 downstream services.  One that gets product information, one that gets product cost information and one that calculates
product tax information.

graphql-java works by running data fetchers over objects for all that information and mapping that back to the types specified in the schema.

Our challenge is to take these 3 sources of information and present them as one unified type.

We could specify data fetchers on the ``cost`` and ``tax`` fields that does those calculations but this is more to maintain and likely to lead to
`N+1 performance problems`.

We would be better to do all this work in the ``Query.products`` data fetcher and create a unified view of the data at that point.

```java
DataFetcher productsDataFetcher = new DataFetcher() {
    @Override
    public Object get(DataFetchingEnvironment env) {
        String matchArg = env.getArgument("match");

        List<ProductInfo> productInfo = getMatchingProducts(matchArg);

        List<ProductCostInfo> productCostInfo = getProductCosts(productInfo);

        List<ProductTaxInfo> productTaxInfo = getProductTax(productInfo);

        return mapDataTogether(productInfo, productCostInfo, productTaxInfo);
    }
};
```

So looking at the code above we have 3 types of information that need to be combined in a way such that a graphql query above can get access to
the fields ``id, name, cost, tax``

We have two ways to create this mapping.  One is via using a not type safe ``List<Map>`` structure and one by creating a type safe ``List<ProductDTO>`` class that
encapsulates this unified data.

The ``Map`` technique could look like this.

```java
private List<Map> mapDataTogetherViaMap(List<ProductInfo> productInfo, List<ProductCostInfo> productCostInfo, List<ProductTaxInfo> productTaxInfo) {
    List<Map> unifiedView = new ArrayList<>();
    for (int i = 0; i < productInfo.size(); i++) {
        ProductInfo info = productInfo.get(i);
        ProductCostInfo cost = productCostInfo.get(i);
        ProductTaxInfo tax = productTaxInfo.get(i);

        Map<String, Object> objectMap = new HashMap<>();
        objectMap.put("id", info.getId());
        objectMap.put("name", info.getName());
        objectMap.put("description", info.getDescription());
        objectMap.put("cost", cost.getCost());
        objectMap.put("tax", tax.getTax());

        unifiedView.add(objectMap);
    }
    return unifiedView;
}
```

The more type safe ``DTO`` technique could look like this.

```java
class ProductDTO {
    private final String id;
    private final String name;
    private final String description;
    private final Float cost;
    private final Float tax;

    public ProductDTO(String id, String name, String description, Float cost, Float tax) {
        this.id = id;
        this.name = name;
        this.description = description;
        this.cost = cost;
        this.tax = tax;
    }

    public String getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getDescription() {
        return description;
    }

    public Float getCost() {
        return cost;
    }

    public Float getTax() {
        return tax;
    }
}

private List<ProductDTO> mapDataTogetherViaDTO(List<ProductInfo> productInfo, List<ProductCostInfo> productCostInfo, List<ProductTaxInfo> productTaxInfo) {
    List<ProductDTO> unifiedView = new ArrayList<>();
    for (int i = 0; i < productInfo.size(); i++) {
        ProductInfo info = productInfo.get(i);
        ProductCostInfo cost = productCostInfo.get(i);
        ProductTaxInfo tax = productTaxInfo.get(i);

        ProductDTO productDTO = new ProductDTO(
                info.getId(),
                info.getName(),
                info.getDescription(),
                cost.getCost(),
                tax.getTax()
        );
        unifiedView.add(productDTO);
    }
    return unifiedView;
}
```

The graphql engine will now use that list of objects and run the query sub fields ``id, name, cost, tax`` over it.

The default data fetcher in graphql-java is ``graphql.schema.PropertyDataFetcher`` which has both map support and POJO support.

For every object in the list it will look for an ``id`` field, find it by name in a map or via a `getId()` getter method and that will be sent back in the graphql
response.  It does that for every field in the query on that type.

By creating a "unified view" at the higher level data fetcher, you have mapped between your runtime view of the data and the graphql schema view of the data.
