# Flattening JSON Data with Splunk

JSON data can be challenging to work with in Splunk, especially when you need to analyze nested structures. This article provides a step-by-step guide on how to flatten complex JSON data in Splunk.

## The Problem with Nested JSON

The fundamental challenge comes from the conceptual mismatch between Splunk's event model and JSON's hierarchical nature:

- **Most of Splunk commands expect**: One event = one row of data with simple field-value pairs
- **JSON provides**: One event potentially containing multiple logical events in a hierarchical structure

When dealing with JSON data that contains nested objects and arrays, a lot of Splunk commands (`|stats`, `|chart`, etc.) won't work as expected.
It's often necessary to flatten the structure to make analysis much easier.
Let's explore how to do this using a simple dataset of countries.

## Example Dataset

We'll be using a sample JSON dataset containing information about four countries:

```json
[
  {
    "name": "France",
    "capital": "Paris",
    "population": 67364357,
    "area": 551695,
    "currency": "Euro",
    "languages": [
      "French"
    ],
    "region": "Europe",
    "subregion": "Western Europe",
    "flag": "https://upload.wikimedia.org/wikipedia/commons/c/c3/Flag_of_France.svg"
  },
  {
    "name": "Germany",
    "capital": "Berlin",
    "population": 83240525,
    "area": 357022,
    "currency": "Euro",
    "languages": [
      "German"
    ],
    "region": "Europe",
    "subregion": "Western Europe",
    "flag": "https://upload.wikimedia.org/wikipedia/commons/b/ba/Flag_of_Germany.svg"
  },
  {
    "name": "United States",
    "capital": "Washington, D.C.",
    "population": 331893745,
    "area": 9833517,
    "currency": "USD",
    "languages": [
      "English"
    ],
    "region": "Americas",
    "subregion": "Northern America",
    "flag": "https://upload.wikimedia.org/wikipedia/commons/a/a4/Flag_of_the_United_States.svg"
  },
  {
    "name": "Belgium",
    "capital": "Brussels",
    "population": 11589623,
    "area": 30528,
    "currency": "Euro",
    "languages": [
      "Flemish",
      "French",
      "German"
    ],
    "region": "Europe",
    "subregion": "Western Europe",
    "flag": "https://upload.wikimedia.org/wikipedia/commons/6/65/Flag_of_Belgium.svg"
  }
]
```

## Key steps explained

### 1. Generate Sample Data

First, let's create our sample data (optionnal if you already collect some JSON data) :

```splunk
| makeresults 
| eval _raw = "[
  {\"name\": \"France\", \"capital\": \"Paris\", \"population\": 67364357, \"area\": 551695, \"currency\": \"Euro\", \"languages\": [\"French\"], \"region\": \"Europe\", \"subregion\": \"Western Europe\", \"flag\": \"https://upload.wikimedia.org/wikipedia/commons/c/c3/Flag_of_France.svg\"},
  {\"name\": \"Germany\", \"capital\": \"Berlin\", \"population\": 83240525, \"area\": 357022, \"currency\": \"Euro\", \"languages\": [\"German\"], \"region\": \"Europe\", \"subregion\": \"Western Europe\", \"flag\": \"https://upload.wikimedia.org/wikipedia/commons/b/ba/Flag_of_Germany.svg\"},
  {\"name\": \"United States\", \"capital\": \"Washington, D.C.\", \"population\": 331893745, \"area\": 9833517, \"currency\": \"USD\", \"languages\": [\"English\"], \"region\": \"Americas\", \"subregion\": \"Northern America\", \"flag\": \"https://upload.wikimedia.org/wikipedia/commons/a/a4/Flag_of_the_United_States.svg\"},
  {\"name\": \"Belgium\", \"capital\": \"Brussels\", \"population\": 11589623, \"area\": 30528, \"currency\": \"Euro\", \"languages\": [\"Flemish\", \"French\", \"German\"], \"region\": \"Europe\", \"subregion\": \"Western Europe\", \"flag\": \"https://upload.wikimedia.org/wikipedia/commons/6/65/Flag_of_Belgium.svg\"}
]"
```

### 1. Extract the First Level Array

The first step is to extract the top-level array items into a multi-value field :

```spl
| spath input=_raw output=countries path={}
```

This command creates a new field called `countries` that contains an array with 4 values, one for each country object in the original JSON :

![image](https://github.com/user-attachments/assets/1409f18e-0ad6-4a57-9dec-03eadacdcea7)

### 2. Clean up `_raw` field
We no longer need the `_raw` field, so let's remove it:
```splunk
| fields - _raw
```

### 3. Expand the `countries` field

Now we need to expand the multi-value `countries` field, creating a separate event for each country:

```splunk
| mvexpand countries
```

This command transforms our single event with a multi-value field into 4 separate events, one for each country :

![image](https://github.com/user-attachments/assets/7340678d-98d9-4c73-84c2-a260ea54adf6)


### 4. Extract Fields from Each event of `countries` field

Next, we extract all fields from each country object:

```splunk
| spath input=countries
```

This extracts all first-level fields from each country object: `name`, `capital`, `population`, `area`, `currency`, `languages`, `region`, `subregion`, and `flag` :

![image](https://github.com/user-attachments/assets/8f4519fd-2d89-435b-9689-b8d8597ceb60)


### 5. Clean Up Intermediate Fields

We can now remove the intermediate `countries` field:

```splunk
| fields - countries _raw
```

### 6. Expand the Languages Array

We're left with one more multi-value field: `languages{}` :

![image](https://github.com/user-attachments/assets/af14772a-1c41-4182-84c3-9a5a18e17450)

Let's expand it to create separate events for each language:

```splunk
| mvexpand languages{}
```

This creates separate events for each language in each country. (Belgium, for example, will be expanded into three events, one for each of its languages).

After running this query, we'll have a flattened dataset where:

1. Each row represents a unique country-language combination
2. All fields from the original JSON are preserved
3. Nested structures are flattened into a tabular format

For example, Belgium will appear three times in the result set, once for each of its languages (Flemish, French, and German).

![image](https://github.com/user-attachments/assets/03786c45-e67b-46f3-8105-3b3c4f4ca40a)

## Complete and Commented SPL Query

Here's the complete SPL query that flattens our JSON data:

```splunk
| makeresults 
|eval _raw = 
"[
  {\"name\": \"France\", \"capital\": \"Paris\", \"population\": 67364357, \"area\": 551695, \"currency\": \"Euro\", \"languages\": [\"French\"], \"region\": \"Europe\", \"subregion\": \"Western Europe\", \"flag\": \"https://upload.wikimedia.org/wikipedia/commons/c/c3/Flag_of_France.svg\"},
  {\"name\": \"Germany\", \"capital\": \"Berlin\", \"population\": 83240525, \"area\": 357022, \"currency\": \"Euro\", \"languages\": [\"German\"], \"region\": \"Europe\", \"subregion\": \"Western Europe\", \"flag\": \"https://upload.wikimedia.org/wikipedia/commons/b/ba/Flag_of_Germany.svg\"},
  {\"name\": \"United States\", \"capital\": \"Washington, D.C.\", \"population\": 331893745, \"area\": 9833517, \"currency\": \"USD\", \"languages\": [\"English\"], \"region\": \"Americas\", \"subregion\": \"Northern America\", \"flag\": \"https://upload.wikimedia.org/wikipedia/commons/a/a4/Flag_of_the_United_States.svg\"},
  {\"name\": \"Belgium\", \"capital\": \"Brussels\", \"population\": 11589623, \"area\": 30528, \"currency\": \"Euro\", \"languages\": [\"Flemish\", \"French\", \"German\"], \"region\": \"Europe\", \"subregion\": \"Western Europe\", \"flag\": \"https://upload.wikimedia.org/wikipedia/commons/6/65/Flag_of_Belgium.svg\"}
]
"
### Extraction of the first level of json into a field that we'll call 'countries', which will be a multi-value field (mv field).
This field will contain as many values as there are entries in the first level of JSON, here : 
|spath input=_raw output=countries path={}

### We don't need the _raw field any more:
| fields - _raw

###  mvexpand will split the 'countries' field into as many events as it contains values, in this case 4 ### 
|mvexpand countries

### Extract JSON fields contained in the 'countries' field:
|spath input=countries

### We no longer need the 'countries' field:
| fields - countries

###  There's still one multi-value field left : 'languages{}'.
We proceed in the same way to split it into as many events as it contains values:
|mvexpand languages{}
```

