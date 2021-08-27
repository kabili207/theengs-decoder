# 1decoder

# Using

Call `decodeBLEJson(JsonObject)` with the input being of the Arduino JSON JsonObject type. If the device is known the JsonObject will have the decoded device data added to it.

### Example
Input JsonObject:
```
{
  "servicedata": "712098000163b6658d7cc40d0410024001"
}
```

JsonObject after decoding:
```
{
  "servicedata": "712098000163b6658d7cc40d0410024001"
  "brand":"Xiaomi",
  "model":"miflora",
  "model_id":"HHCCJCY01HHCC",
  "tempc":32,
  "tempf":89.6
}
```

# Adding device decoding

Device decode specifications are located in the [device_json.h](src/device_json.h) file. The format is:
```
R""""(
{
   "brand":"Xiaomi",
   "model":"miflora",
   "model_id":"HHCCJCY01HHCC",
   "condition":["servicedata", "contain", "209800"],
   "properties":{
      "tempc":{
         "condition":["servicedata", 25, "4"],
         "decoder":["value_from_hex_data", "servicedata", 30, 4, true],
         "post_proc":['/', 10]
      },
      "moi":{
         "condition":["servicedata", 25, "8"],
         "decoder":["value_from_hex_data", "servicedata", 30, 2, false]
      },
      "lux":{
         "condition":["servicedata", 25, "7"],
         "decoder":["value_from_hex_data", "servicedata", 30, 6, true]
      },
      "fer":{
         "condition":["servicedata", 25, "9"],
         "decoder":["value_from_hex_data", "servicedata", 30, 4, true]
      }
   }
})"""",
```

Each device must provide a `brand`, `model`, `model_id`, `condition`, and `properties`.
- `brand` = brand name of the device.
- `model` = model name of the device.
- `model_id` = model id number of the device.

### Condition
`condition` is a JSON array, which must contain as the first parameter, the data source to test for the condtion. Valid inputs are:
- "servicedata"
- "manufacturerdata"
- "name"
- "uuid"

The second parameter is how the data should be tested. Valid inputs are:
- "contain" tests if the specified value (see below) exists the data source 
- "index" tests if the specified value exists at the index location (see below) in the data source

The third parameter can be either the index value or the data value to find. If the second parameter is `contain`, the third parameter should be the value to look for in the data source. If the second parameter is `index`, the third parameter should be the location in the data source to look for the value provided as the fourth parameter.

`condition` can have multiple conditions chanined together using '|' and '&' between them.  
For example: `"condition":["servicedata", "index", 0, "0804", '|', "servicedata", "index", 0, "8804"]`  
This will match if the service data at index 0 is "0804" `OR` "8804".

### Properties
Properties is a nested JSON object containing one or more JSON objects. In the example above it looks like:
```
 "properties":{
      "tempc":{
         "condition":["servicedata", 25, "4"],
         "decoder":["value_from_hex_data", "servicedata", 30, 4, true],
         "post_proc":['/', 10]
      },
```

Here we have a single property that defines a value that we want to decode. The key "tempc" will be used as the key in the JsonObject provided when `decodeBLEJson(JsonObject)` is called. "tempc" in this example is another JSON object that has an (optional, explained below) `condition`, `decoder`, and `post_proc`.

`condition` is a JSON array. The first parameter defines the data source of the condition to test and must be one of:
- "servicedata"
- "manufacturerdata"

The second parameter is the index of the data source to look for the value. The third parameter is the value to test for.
If the condition is met the data will be decoded and added to the JsonObject.

`decoder` is a JSON array that specifies the decoder function and parameters to decode the value. The first parameter is the name of the function to call, currently only "value_from_hex_data" is valid. The other parameters are:
- "servicedata", Extract the value from the service data. Could also be "manufacturerdata"
- 30, The index of the data source where the value exists.
- 4, The length of the data in bytes (characters in the string).
- true/false, If the value in the data source should have it's endianness reversed before converting.
- (optional)true/false, Sets if the resulting value can be a negative number.

`post_proc` This specifies any post processing of the resulting decoded value. This is a JSON array that should be written in the order that the operation order is desired. In the simple example the first parameter is the '/' divide operation and the second parameter (10) is the value to divide the result by. Multiple operations can be chained together in this array to perform more complex calculations. Valid operations are:
- '/' divide
- '*' multiply
- '+' add
- '-' subtract
- '<' shift left
- '>' shift right
- '!' Not (invert), useful for bool types

`val_bits` (Not shown in the example) is an additional parameter that can be added to define the value. It will convert the post processed result into a value with the number of bits specified by `val_bits`. Valid values are: 1 (for bool), 8, 16, 32. Double is the default type if this is omitted.
