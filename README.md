# TERO

Tero is written in php 5.3 (which supports the use of anonymous functions with bind), designed to run both that version and server versions 5.6 and 7+

It is optional to use friendly urls but if you want it to work you should use it in apache with mod_rewrite enabled.

For database is intended to integrate almost any database through PDO, Tero was widely used in MySQL 5.5 / Mariadb / SQLite / Mssql Server 2005+

It is supported on both Windows (wamp) and Linux (lamp) base servers

To install tero just go with your console to the web directory and run composer

### Install

```
composer create-project dromero86/tero project_name
```

Tero, by default, has this folder structure:

```
   MyWebsite/
	|--app/
	|  |--config/
	|  |--library/
	|  |--model/
	|  |--schema/
	|  |--third_party/
	|  |--vendor/
	|
	|--ui/
	|  |--images/
	|  |--themes/
	|     |--mytheme
	|
	|--index.php
```

index.php acts as bootstrapper to execute the core, this also has two tasks, the first is to load all the libraries needed for the project and the second is to load all the model-controllers written by the user and finally execute the one required by the url.

The folder app has the structure of the framework that is not public, therefore it is not accessible via web instead the folder ui has all the resources that will be of web use as images, scripts and css files.

### Hello World with tero is:

```php
//1#
<?php if ( !defined('BASEPATH')) exit('No direct script access allowed');

//2# 
$App = core::getInstance();  

//3# 
$App->get("index", function()
{    
    //4# 
    echo "Hello world!";
});
```
The first line specifies the environment of tero (similar to codeigniter) and is valid for all files defined by the user. It is a security measure to not access the file directly and exploit a vulnerability.

The second line obtains the core instance, important for defining our controllers and accessing defined libraries / helpers

The third line defines our controller, with the first argument it will interpret the web call and with the function it will resolve the content to be returned

The fourth line is the content printed by the browser. for example, in this case if our project would be at http://localhost/MyWebsite/  el
content to be returned by this driver would be:

```
Hello World!
```

Note that the use of "index" refers to the default controller, therefore there is no need to add parameters to the url.

## Setup Apps

To include libraries or helpers, core.json is used, this file contains the load configuration and other options to improve or limit the performance of the app, this file has this syntax:

```json
{
    "loader":
    [
        {"file":"app/vendor/Parser"    , "library":{ "class":"Parser"   , "rename":"parser" } },
        {"file":"app/vendor/Skeleton"  , "library":{ "class":"Skeleton" , "rename":"view"   } },
        {"file":"app/vendor/Dataset"   , "library":{ "class":"Dataset"  , "rename":"data"   } },
        {"file":"app/vendor/input"     , "library":{ "class":"input"    , "rename":"input"  } },
        {"file":"app/model/docs"       , "helper" : true },
        {"file":"app/model/sketch"     , "helper" : true }  
    ],
    "debug"   : false,
    "error"   : "On" ,
    "leak"    : "50M",
    "timezone":"America/Vancouver",
    "encoding": "UTF-8" 
}
```
Within the loader, the elements to be loaded are stacked, the order of loading is from ascending to descending, and as a criterion "from which the smallest dependencies have the most".

The "File" attribute indicates where the file is located and starts from the folder where the file is located to the file without ".php"
 
It is important to differentiate a class or library from a helper because core saves the instance of the class as public property (accessible within the user's driver), however, for a helper only loads it.

For libraries I can rename the instance for example:

```php
$App->get("...", function(...))
{
     //data is rename of dataset
     $this->data->set(...);
});
```

## Router URLs

To take the arguments of the url as parameters and to understand how the processes are processed, we will explain how each of them works.

Tero accepts (where "index" is an example method):

- /?action=index
- /index
- /index-:id => :id is param
- /index?another=param
- /index-:id?another=param


Method url: INDEX
Param order: left to right

extract url method in this order:
- Match Simple: find method in rewrited url without regex url, if not
- Match Params: find method in rewrited url with regex expression, if not
- Default     : parse GET params

if method exist and is callable, call it 
if not call default method “index”

To work with friendly urls in apache remember to enable "mod_rewrite" and include .htaccess with:

```
RewriteEngine On

RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_FILENAME} !-f

RewriteRule ^ index.php [QSA,L]
```

This will enable type urls
/my-cool-page or parameterized as
/product/:name-:id where ": name" and ": id" are the parameters that will be used in the method:

http://localhost/myterocart/product/cool-glass-red-2345

```php 
$App->get("product/:name-:id", function($name, $id)) //name unnused in the example
{
     //take params
     $id = (int)$id; 

     $this->db->query(“SELECT price FROM products WHERE product_id = {$id}”);
});
```

Later we will see more uses of the parameters.

### Redirections

We often need to jump from one page to another from the server, this can be solved with the native "redirect" function that is implemented like this

```php
$App->get("product/:id", function($id)) 
{
	$category = 36;
	redirect(“products/category/{$category}”);
  ...
```
In the example, it means that you will jump from the products page to the categories page from the server, this function also allows you to indicate headers for when you perform permanent redirects.

# Requesting

Capturing the get variables of the url:

A) the simple native path: the get parameters are serialized and passed to the model function as follows:

```php
//http://localhost/museum_library/?action=download_pdf&file=myfile.pdf

$App->get("download_pdf", function($file)) 
{
...
```
Here the processed variable is file, note that "action" is a reserved word and is used internally by the core of tero.

We must also understand that Tero returns all the parameters in string, it is our responsibility to perform the appropriate cast to avoid security cracks.

B) using friendly urls: Parameters are obtained as a result of using regular expressions on the pattern of the url (the get is not used) the example we saw earlier

```php
//http://localhost/terocart/product/36
$App->get("product/:id", function($id)) 
{
...
```
Here it will take "product/36" and in this case, it will take "36" and pass it as an argument of the method.

C) a bit more complex (and fun): Tero can combine the 2 things, on the one hand, extract variables through a regular expression as well as process the get parameters, leaving a funny monster

```php
//http://localhost/clothestero/tshirt-category/36?color=blue&size=xxl
$App->get("tshirt-category/:category, function($category, $color=””, $size=””)) 
{
...
```

For our Frankenstein to work, we must consider some things:

1. The url pattern goes first
2. The get arguments are not written as if they were regular expressions
3. The get arguments can be placed as optional using php syntax for functions {$ variable = ""}
4. When we use Get we must respect the order of the parameters

## Working with POST

The use of post is not native to the core of tero, but if it is included as a library, therefore we must include it in core.json

```json
{
    "loader":
    [
        ...
        {"file":"app/vendor/input" , "library":{ "class":"input" , "rename":"input"  } 

```
Its use is really simple

```php
$App->get("...”, function(...)) 
{

   $post = $this->input->post();
   ...
```
The post() input method converts the _POST array into an object (with the possibility of treating the elements), as an object it will be useful later when we use database, but even if we do not need it for that, its use is really comfortable

```
	$post = $this->input->post();
	
	$post->name 
	$post->phone
	etc
```

Check if I have post elements

```
$App->get("...”, function(...)) 
{
   if($this->input->has_post())
   {
     $post = $this->input->post();
   }
```

## SETUP DATABASE

To integrate our database we must first enable it from core.json, this is done as follows:

```json
{
    "loader":
    [
      ...
      {"file":"app/vendor/database", "library":{ "class":"database" , "rename":"db"  } 
```

This indicates that we will have access to the database through the core attribute "$this->db" but we still have to configure the access to the database, for this we will have to edit db.json

```json
{
    "database" :
    {
        "driver"    : "mysqli",
        "user"      : "mydbuser"      ,
        "pass"      : "mydbpass"          ,
        "host"      : "myhostname" ,
        "db"        : "mydbname"        ,
        "charset"   : "utf8"      ,
        "collate"   : "utf8_general_ci",
        "debug"     : false
    }
}
```

If everything is ok, we can check if the connection works like this

```php
$App->get("...”, function(...)) 
{
   var_dump($this->db->is_ready());

```

If I return TRUE it means that we connect successfully.
