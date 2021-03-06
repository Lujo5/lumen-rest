# Lumen REST
> Lumen package for creating simple REST APIs.

## Installation
Install the package using composer by executing following command:
```sh
composer require lujo/lumen-rest
```
Or you can add in `composer.json` the followign line:
```sh
"require": {
    ...
    "lujo/lumen-rest": "*"
}
```
And then run: 
```sh
composer install
```

## Description
This Lumen package consists of two classes: `RestRoute` and `RestController`.
- `RestRoute` does all the routing operations in function `route(...)` for specific resource.
- `RestController` should be extended by some other controller class for specific resource.

## Usage

### 1. Create Eloquent model
In order to use this REST package, first you must create eloquent model for some resource.

Example Article model:
```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Article extends Model {

    protected $table = 'article';

    protected $fillable = [
        'title',
        'description',
        'content'
    ];

    // Optional realtions and other Eloquent stuff
    public function articleAuthor() {
        return $this->hasOne('App\Models\Author');
    }
}
```

### 2. Create Controller
After model is created, you can create simple Controller class and EXTEND the RestController class from this package.
Also, you must implement function `getModel()` which must return Eloquent model created in last step which is the
resource you want to expose through this REST API.

Example Controller class:
```php
<?php

namespace App\Http\Controllers;

use Illuminate\Database\Eloquent\Model;
use App\Models\Article;
use Lujo\Lumen\Rest\RestController;


class ArticleController extends RestController {

    /**
     * @return Model
     */
    protected function getModel() {
        return new Article();
    }
    
    // Optional override, transforms every model before returning it from API.
    protected function beforeGet($model) {
        return $model;
    }
    
    // Optional override, transform received request before creating new model from it.
    protected function beforeCreate($request) {
        return $request->all();
    }
    
    // Optional override, transform received request before updating existing model from it.
    protected function beforeUpdate($request) {
        return $request->all();
    }
    
    // Optional override, perform some action on/with model before it is deleted.
    protected function beforeDelete($model) {
        return null;
    }
    
    // Optional override, specify list of relations to return on specific action
    protected function getWith($request, $action) {
        if($action === 'INDEX') {
            return ['text'];
        }
        return ['author', 'comments', 'text'];
    }
    
    // Optional override, specify list of where statements to return on specific action
    protected function getWhere($request, $action) {
        if($action === 'DELETE') {
            return [['name', 'Test']];
        }
        return [['status', 'ACTIVE'], ['enabled', true]];
    }
    
    // Optional override, specify additional where query statement in form of a anonymous function
    protected function getWhereFunction($request, $action) {
        return function($q) {
            $q->where('name', 'Test')->orWhere('status', 'ACTIVE');
        };
    }
    
    // Optional override, when INDEX method is called and this method returns true, the returend JSON will contain 
    // count data used for pagination e.g. {result_count: 10, total_count: 45, data: [...results]}
    protected function withCountMetadata($request) {
        return false;
    }

}
```

All optional override functions are not required if you are not using them, they can be safely left out from
your controller class implementation.

### 3. Create routing
After previouse two steps are finished, open your Lumen routes in `routes/web.php`, and create routing structure
using `RestRoute` class static function `route($router, $prefix, $controller, $include = null)`.

Example `routes/web.php` rotuing:
```php
<?php 

use Laravel\Lumen\Routing\Router;
use Lujo\Lumen\Rest\RestRoute;

/**
 * @var $router Router
 */

// This will create all article routes ('INDEX', 'ONE', 'CREATE', 'UPDATE', 'DELETE') for routes /articles/*
RestRoute::route($router, 'articles', 'ArticleController', 'middleware1');

// This will create only 'INDEX' and 'ONE' for routes /users/*
RestRoute::route($router, 'users', 'UserController', ['middleware1', 'middleware2'], ['INDEX', 'ONE']);

// This will create only 'INDEX', 'CREATE' and 'UPDATE' routes for /example/* but apply middlewares only on CREATE and UPDATE
RestRoute::route($router, 'examples', 'ExampleController', ['CREATE' => ['middleware1', 'middleware2'], 'UPDATE' => 'middleware2'], 
        ['INDEX', 'CREATE', 'UPDATE']);

// This will create all routes and apply middleware1 and middleware2 to INDEX and ONE route, on others will middleware3 be applied.
RestRoute::route($router, 'users', 'UserController', ['INDEX,ONE' => ['middleware1', 'middleware2'], 'middleware3']);

// Example subgroup of routes (/article/authors/*) with middleware
$router->group(['prefix' => 'article', 'middleware' => 'middleware1'], function ($subRoute) {
        RestRoute::route($subRoute, 'authors', 'AuthorController');
    }
);
```

Generated REST resource routes are in the following format:

| Method | Path           | Function | Description          |
| ------ | -------------- | ---------|--------------------- |
| GET    | /resource      | Index    | Get all resources*   |
| GET    | /resource/{id} | One      | Get one resource     |
| POST   | /resource      | Create   | Create new resource  |
| PUT    | /resource/{id} | Update   | Update resource      |
| DELETE | /resource/{id} | Delete   | Delete resource      |

\* _Get all resources_ has following HTTP GET URL parameters:
* **skip** - How many resources to skip (e.g. 30)
* **limit** - How many resources to retreive (e.g. 15)
* **sort** - Filed on which to sort returned resources (e.g. 'first_name')
* **order** - Ordering of returend resources ('asc' or 'desc')

**Example HTTP GET request:** `http://site.com/api/resource?skip=30&limit=6&sort=name&order=desc`

**Response:**

_Headers:_
```
Content-type: application/json
X-Total-Count: 10
X-Result-Count: 6
```

_Body when method withCountMetadata() returns false:_
```
[
    {"id": 31, "resource_name": "Resource name", "description": "Some resource description"}, 
    {"id": 32, "resource_name": "Resource name", "description": "Some resource description"},
    {"id": 33, "resource_name": "Resource name", "description": "Some resource description"}, 
    {"id": 34, "resource_name": "Resource name", "description": "Some resource description"}, 
    {"id": 35, "resource_name": "Resource name", "description": "Some resource description"},
    {"id": 36, "resource_name": "Resource name", "description": "Some resource description"}
]
```

_Body when method withCountMetadata() returns true:_
```
{
    total_count: 10,
    result_count: 6,
    data: [
        {"id": 31, "resource_name": "Resource name", "description": "Some resource description"}, 
        {"id": 32, "resource_name": "Resource name", "description": "Some resource description"},
        {"id": 33, "resource_name": "Resource name", "description": "Some resource description"}, 
        {"id": 34, "resource_name": "Resource name", "description": "Some resource description"}, 
        {"id": 35, "resource_name": "Resource name", "description": "Some resource description"},
        {"id": 36, "resource_name": "Resource name", "description": "Some resource description"}
    ]
}
```

LICENSE
---
MIT