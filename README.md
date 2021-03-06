# validy

Declarative validation with async validators support.
If you also want to work with filters and default values, take a look at [transformer-chain](https://github.com/Jokero/transformer-chain)

[![NPM version](https://img.shields.io/npm/v/validy.svg)](https://npmjs.org/package/validy)
[![Build status](https://img.shields.io/travis/Jokero/validy.svg)](https://travis-ci.org/Jokero/validy)

**Note:** This module works in browsers and Node.js >= 4.0. Use `Object.assign` and `Promise` polyfills for Internet Explorer

## Table of Contents

- [Demo](#demo)
- [Installation](#installation)
  - [Node.js](#nodejs)
  - [Browser](#browser)
- [Overview](#overview)
- [Usage](#usage)
  - [validy(object, schema, [options])](#validyobject-schema-options)
    - [Parameters](#parameters)
    - [Return value](#return-value)
  - [Validators](#validators)
    - [Built-in validators](#built-in-validators)
    - [Custom validator](#custom-validator)
      - [Synchronous validator](#synchronous-validator)
      - [Asynchronous validator](#asynchronous-validator)
  - [Error format](#error-format)
  - [Error messages](#error-messages)
  - [Return value](#return-value-1)
  - [Dynamic schema](#dynamic-schema)
- [Build](#build)
- [Tests](#tests)
- [License](#license)

## Demo

Try [demo](https://runkit.com/npm/validy) on RunKit.

## Installation

```sh
npm install validy
```

### Node.js
```js
const validy = require('validy');
```

### Browser
```
<script src="node_modules/validy/dist/validy.js">
```
or minified version
```
<script src="node_modules/validy/dist/validy.min.js">
```

You can use the module with AMD/CommonJS or just use `window.validy`.

## Overview

`validy` allows you to validate flat and nested objects using collection of default validators and your own validators. 

Validators can be asynchronous, you can do DB calls for example and so on.

To validate object you should define schema. It's simple object with your constraints:

```js
const book = { // object to validate
    name: 'The Adventures of Tom Sawyer',
    author: {
        name: 'Mark Twain'
    },
    reviews: [
        {
            author: 'Leo Tolstoy',
            text: 'Great novel'
        },
        {
            author: 'Fyodor Dostoyevsky',
            text: 'Very interesting'
        }
    ]
};

const schema = {
    name: {
        $validate: {
            required: true,
            string: true
        }
    },
    author: {
        $validate: { // you can omit check that "author" value is object, it will be done internally 
            required: true
        },
    
        name: {
            $validate: {
                required: true,
                string: true
            }
        }
    },
    reviews: [{ // define schema for array items
        author: {
            $validate: {
                required: true,
                string: true
            }
        },
        text: {
            $validate: {
                required: true,
                string: true
            }
        }
    }]
};

validy(book, schema)
    .then(errors => {
        if (errors) {
            // you have validation errors ("errors" is plain object)
        } else {
            // no errors ("errors" is undefined)
        }
    })
    .catch(err => {
        // application error (something went wrong)
    });

// async/await example
async function example() {
    try {
        const errors = await validy(book, schema);
        if (errors) {
            // you have validation errors ("errors" is plain object)
        } else {
            // no errors ("errors" is undefined)
        }
    } catch(err) {
        // application error (something went wrong)
    }
}
```

## Usage

### validy(object, schema, [options])

#### Parameters

- `object` (Object) - Object to validate
- `schema` (Object) - Schema which defines how to validate object
- `[options]` (Object) - Validation options
    - `[format=flat]` (string) - Format of object with validation errors (`flat`, `nested`)
    - `[reject=false]` (boolean) - Should return fulfilled promise with errors (`by default`) or rejected with `ValidationError`?

#### Return value

(Promise) - Result of validation. Promise is returned even for synchronous only validation

### Validators

#### Built-in validators

By default `validy` uses collection of simple and useful validators ([common-validators](https://github.com/tamtakoe/common-validators) module).

**Note**:
The basic principle of built-in validators is that most of them (except `required`, `notEmpty` and type validators `object`, `array`, ...) consider empty values as **valid values**.
Empty values are:
- `undefined`
- `null`
- `NaN`
- `''`
- `'   '` (whitespace only string)
- `[]`
- `{}`

Also they convert passed value to expected type. For example, `max` validator which checks that value is not greater than some limit will try to convert passed value to number (`Number(value)`).
All non-convertible values will be treated as `NaN` and validator will return validation error.

Some of built-in validators:

- **required (presence)** - validates that the value isn't `undefined`, `null`, `NaN`, empty or whitespace only string, empty array or object
- **notEmpty** - like `required` but `undefined` is valid value. It is useful for PATCH requests
- **object / array / string / number / integer / ...** - value is plain object, array, etc. `undefined` is valid value
- **max** - value is less than maximum
- **min** - value is greater than minimum
- **range** - value is in range
- **maxLength** - value length is greater than maximum
- **minLength** - value length is greater than minimum
- **pattern** - value matches the pattern
- **inclusion** - value is contained in white list
- **exclusion** - value is not contained in black list
- **email** - value is email address
- **url** - value is URL
- and many others (see [common-validators#validators](https://github.com/tamtakoe/common-validators#validators))

#### Custom validator

You can add your own validator:

```js
validy.validators.add('greaterThan', function(value, options) {
    // validator implementation
});

// or

validy.validators.add({ // this way you can add several validators at once
    greaterThan: function(value, options) {
        // validator implementation
    },
    anotherValidator: function(value, options) {
        // validator implementation
    }
});
```

Although in most cases you will have only two parameters in your own validators (`value` and `options`), some situations will require a bit knowledgeable validator.
So, full signature of validator is:

**validator(value, options, object, fullObject, path)**

- `value` (any) - Validated value
- `options` (Object) - Validator options
- `object` (Object) - Object whose property is validated at the moment
- `fullObject` (Object) - The whole validated object (object which was initially passed to `validy`)
- `path` (string[]) - Path to property

So imagine you wrote `validateSomething` validator:

```js
const book = {
    name: 'The Adventures of Tom Sawyer',
    author: {
        name: 'Mark Twain'
    }
};

const schema = {
    name: {
        $validate: {
            required: true,
            string: true
        }
    },
    author: { 
        $validate: {
            required: true
        },
        
        name: {
            $validate: {
                required: true,
                string: true,
                validateSomething: 'someArgument' // <--- and you want to use it here
            }
        }
    }
};
```

`validateSomething` validator will be called with the following arguments:

1) value

Value of `author.name` property.
```js
'Mark Twain' 
```

2) options

When you use non-object value as validator options it will be wrapped in object with `arg` property.
```js
{
    arg: 'someArgument'
}
```

3) object

Object with `name` property (`author` object).
```js
{
    name: 'Mark Twain'
}
```

4) fullObject

The whole validated object (`book` object).
```js
{
    name: 'The Adventures of Tom Sawyer',
    author: {
        name: 'Mark Twain'
    }
}
```

5) path

```js
['author', 'name']
```

##### Synchronous validator

Example:

```js
validy.validators.add('russianLettersOnly', function(value) {
    // it's ok to consider empty value as valid value
    // use "required" validator when this value must not be empty
    if (value === '') {
        // if value is valid just return nothing or falsy value 
        return;
    }
    
    if (typeof value !== 'string' || !/^[а-яё]+$/i.test(value)) {
        // when value is invalid, return string with error
        return 'must contain only Russian letters';
    }
});

// or

validy.validators.add({ // this way you can add several validators at once
    russianLettersOnly: function(value) { /**/ },
    
    anotherValidator: function(value) { /**/ }
});
```

And then just use it as any other validator:

```js
{
    name: {
        $validate: {
            // validator will be called only if its config is not equal to false/null/undefined 
            russianLettersOnly: true
        }
    }
}
```

##### Asynchronous validator

Almost the same as synchronous validator, just return fulfilled promise:

```js
validy.validators.add({
    /**
     * Check using mongoose model that value exists in mongodb 
     * 
     * @param {string} value
     * @param {Object} options
     * @param {Object}   options.model - Mongoose model
     * @param {string}   [options.field] - Which field to use for search
     *
     * @returns {Promise}
     */
    exists: function(value, options) {
        const errorMessage = 'does not exist';
        
        if (value === '') { 
            return Promise.resolve();
        }
        
        if (typeof value !== 'string') {
            return Promise.resolve(errorMessage);
        } 
    
        const model = options.model;
        const field = options.field || '_id';
    
        return model.count({ [field]: value })
            .then(count => {
                if (!count) {
                    // if value is invalid, return fulfilled promise with validation error
                    return Promise.resolve(errorMessage);
                }
            });
    }
});
```
### Error format

If there are no validation errors `validy` returns `undefined` as fulfillment value:

```js
validy(book, schema)
    .then(errors => {
        console.log(errors); // undefined
    })
```

If you have validation errors:

```js
const book = {
    name: '', // empty
    author: {
        name: 123456789 // not string
    }
};

const schema = {
    name: {
        $validate: {
            required: true,
            string: true
        }
    },
    author: {
        $validate: {
            required: true
        },
        
        name: {
            $validate: {
                required: true,
                string: true
            }
        }
    }
};

validy(book, schema)
    .then(errors => {
        console.log(errors);
    })
```

`errors` has flat structure by default:

```js
{
    name: [{ // errors are always placed in array
        error: 'required', // validator name
        message: 'Is required' // error message
    }],
    'author.name': [{
        error: 'string', 
        message: 'Must be a string'
    }]
}
```

But you can use nested structure:

```js
validy(book, schema, { format: 'nested' })
    .then(errors => {
        console.log(errors);
    })
```

`errors` with nested structure:

```js
{ 
    name: [{ 
        error: 'required', 
        message: 'Is required' 
    }],
    author: { 
        name: [{ 
            error: 'string', 
            message: 'Must be a string'
        }]
    }
}
```

### Error messages

You can customize error message for specific property:

```js
const schema = {
    url: {
        $validate: {
            url: { // instead of { url: true }
                message: 'invalid url' // message can be either string or function returning string
            }
        }
    }
};

const object = {
    url: 'not-url'
};

validy(object, schema)
    .then(errors => {
        console.log(errors); // { url: [ { error: 'url', message: 'invalid url' } ] }
    });
```

Or set default message for a validator:

```js
validy.validators.url.defaultOptions = { message: 'bad url' };

const schema = {
    url: {
        $validate: {
            url: true
        }
    }
};

const object = {
    url: 'not-url'
};

validy(object, schema)
    .then(errors => {
        console.log(errors); // { url: [ { error: 'url', message: 'bad url' } ] }
    });

```

### Return value 

By default `validy` returns fulfilled promise when validated object is not valid.
If for some reasons you want to use rejected promise with validation error instead of fulfilled promise, specify `reject` option:

```js
validy(object, schema, { reject: true })
    .then(() => {
        // no errors, everything is valid
    })
    .catch(err => {
        if (err instanceof validy.ValidationError) {
            // err.errors contains validation errors
        } else {
            // application error (something went wrong)            
        }
    });
```

### Dynamic schema

Sometimes you may need a way to validate some property differently depending on specific conditions.
Example with order of various products:

```js
const order = {
    products: [
        {
            type: 'book',
            name: 'The Adventures of Tom Sawyer',
            count: 1
        },
        {
            type: 'sugar',
            weight: 3000
        }
    ]
};

const productsSchemas = {
    book: {
        name: {
            $validate: {
                required: true,
                string: true
            }
        },
        count: {
            $validate: {
                required: true,
                integer: true,
                min: 1
            }
        }
    },

    sugar: {
        weight: {
            $validate: {
                required: true,
                integer: true,
                min: 1000
            }
        }
    }
};

const schema = {
    products: [(product/*, products, order, pathToItem*/) => {
        const productSchema = productsSchemas[product.type];
        return Object.assign({}, productSchema, {
            type: {
                $validate: {
                    required: true,
                    string: true,
                    inclusion: Object.keys(productsSchemas)
                }
            }
        });
    }]
};

// or you can do like this (products is marked as required)

const alternativeSchema = {
    products: {
        $validate: { // validate also "products" before items validation
            required: true,
            array: true
        },

        $items: function(product/*, products, order, pathToItem*/) {
            const productSchema = productsSchemas[product.type] || {};
            return Object.assign({}, productSchema, {
                type: {
                    $validate: {
                        required: true,
                        string: true,
                        inclusion: Object.keys(productsSchemas)
                    }
                }
            });
        }
    }
};
```

You can do similar things with `$validate` and specific validator:

```js
const bookSchema = {
    author: {
        name: {
            $validate: function(name, author, book, pathToName) {
                // implement your custom logic
                
                // validation will only run if you return object
                // so you can return null for example to skip validation 
                return {
                    required: function(name, author, book, pathToName) {
                        // implement your custom logic
                        // return undefined, null or false if you want skip validation
                    },
                    string: true
                };
            }
        }
    }
};
```

## Build

```sh
npm install
npm run build
```

## Tests

```sh
npm install
npm test
```

## License

[MIT](LICENSE)