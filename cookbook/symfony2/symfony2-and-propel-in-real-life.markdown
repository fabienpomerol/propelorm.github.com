---
layout: documentation
title: Symfony2 And Propel In Real Life
---

# Symfony2 And Propel In Real Life #

Let's face it, one of the most common and challenging tasks for any application involves persisting and reading
information to and from a database. Unfortunately, Symfony2 does not come integrated with Propel but it's really
easy to integrate Propel into Symfony2. To get started, read [how to set up Propel into Symfony2](working-with-symfony2.html#installation).

## A Simple Example: A Product ##

In this section, you'll configure your database, create a `Product` object, persist it to the database and fetch it back out.

>**Code along with the example**<br />If you want to follow along with the example in this chapter, create an `AcmeStoreBundle` via: `php app/console generate:bundle --namespace=Acme/StoreBundle`.

### Configuring the Database ###

Before you really begin, you'll need to configure your database connection information.
By convention, this information is usually configured in an `app/config/parameters.ini` file:

{% highlight ini %}
;app/config/parameters.ini
[parameters]
    database_driver   = mysql
    database_host     = localhost
    database_name     = test_project
    database_user     = root
    database_password = password
    database_charset  = UTF8
{% endhighlight %}

>**Information**<br />Defining the configuration via `parameters.ini` is just a convention. The parameters defined in that file are referenced by the main configuration file when setting up Propel:

{% highlight yaml %}
propel:
    dbal:
        driver:     %database_driver%
        user:       %database_user%
        password:   %database_password%
        dsn:        %database_driver%:host=%database_host%;dbname=%database_name%;charset=%database_charset%
{% endhighlight %}

Now that Propel knows about your database, you can have it create the database for you:

    php app/console propel:database:create

>**Keep in mind**<br />In this example, you have one configured connection, named `default`. If you want to configure more than one connection, read the [PropelBundle configuration section](working-with-symfony2.html#project_configuration).

### Creating a Model Class ###

In Symfony2 world, **Model classes** also known as ActiveRecord classes in Propel are called **entities**.
With Propel, it's better to call them **Model classes** as generated Propel classes contain some business logic.

Suppose you're building an application where products need to be displayed. Let's writing a `schema.xml` inside
the `Resources/config/` directory of your `AcmeStoreBundle`:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<database name="default" namespace="Acme\StoreBundle\Model" defaultIdMethod="native">
    <table name="product">
        <column name="id" type="integer" required="true" primaryKey="true" autoIncrement="true" />
        <column name="name" type="varchar" primaryString="1" size="100" />
        <column name="price" type="decimal" />
        <column name="description" type="longvarchar" />
    </table>
</database>
{% endhighlight %}

### Building the Model ###

Once you wrote your `schema.xml`, you just have to generate it:

    php app/console propel:build-model

It will generate all classes to quickly develop your application in the `Model/` directory of your `AcmeStoreBundle` bundle.

### Creating the Database Tables/Schema ###

You now have a usable `Product` class and all you need to persist it. Of course, you don't yet have the corresponding
`product` table in your database.
Fortunately, Propel can automatically create all the database tables needed for every known entity in your application.
To do this, run:

    php app/console propel:build-sql

    php app/console propel:insert-sql --force


Your database now has a fully-functional `product` table with columns that match the schema you've specified.

>**Tip**<br />You can run the last three commands in once by using the following command: `php app/console propel:build --insert-sql`.

### Persisting Objects to the Database ###

Now that you have a `Product` object and corresponding `product` table, you're ready to persist data to the database.
From inside a controller, this is pretty easy. Add the following method to the `DefaultController` of the bundle:

{% highlight php %}
<?php
// src/Acme/StoreBundle/Controller/DefaultController.php
use Acme\StoreBundle\Model\Product;
use Symfony\Component\HttpFoundation\Response;
// ...

public function createAction()
{
    $product = new Product();
    $product->setName('A Foo Bar');
    $product->setPrice('19.99');
    $product->setDescription('Lorem ipsum dolor');

    $product->save();

    return new Response('Created product id '.$product->getId());
}
{% endhighlight %}

In this piece of code, you instantiate and work with the `$product` object. When you call the `save()` method on it, you persist
it in the database. No need to use other services, the object knows how to persist it itself.

>**Note**<br />If you're following along with this example, you'll need to create a route that points to this action to see it in work.

### Fetching Objects from the Database ###

Fetching an object back out of the database is even easier. For example, suppose you've configured a route to display
a specific `Product` based on its `id` value:

{% highlight php %}
<?php

use Acme\StoreBundle\Model\ProductQuery;

public function showAction($id)
{
    $product = ProductQuery::create()
        ->findPk($id);

    if (!$product) {
        throw $this->createNotFoundException('No product found for id '.$id);
    }

    // do something, like pass the $product object into a template
}
{% endhighlight %}

### Updating an Object ###

Once you've fetched an object from Propel, updating it is easy. Suppose you have a route that maps a product id
to an update action in a controller:

{% highlight php %}
<?php

use Acme\StoreBundle\Model\ProductQuery;

public function updateAction($id)
{
    $product = ProductQuery::create()
        ->findPk($id);

    if (!$product) {
        throw $this->createNotFoundException('No product found for id '.$id);
    }

    $product->setName('New product name!');
    $product->save();

    return $this->redirect($this->generateUrl('homepage'));
}
{% endhighlight %}

Updating an object involves just three steps:

1. fetching the object from Propel;
2. modifying the object;
3. saving it.

### Deleting an Object ###

Deleting an object is very similar, but requires a call to the `delete()` method on the object:

{% highlight php %}
<?php

$product->delete();
{% endhighlight %}

## Relationships/Associations ##

Suppose that the products in your application all belong to exactly one "category". In this case,
you'll need a `Category` object and a way to relate a `Product` object to a `Category` object.

Start by adding the `category` definition in your `schema.xml`:

{% highlight xml %}
<database name="default" namespace="Acme\StoreBundle\Model" defaultIdMethod="native">
    <table name="product">
        <column name="id" type="integer" required="true" primaryKey="true" autoIncrement="true" />
        <column name="name" type="varchar" primaryString="1" size="100" />
        <column name="price" type="decimal" />
        <column name="description" type="longvarchar" />

        <column name="category_id" type="integer" />
        <foreign-key foreignTable="category">
            <reference local="category_id" foreign="id" />
        </foreign-key>
    </table>

    <table name="category">
        <column name="id" type="integer" required="true" primaryKey="true" autoIncrement="true" />
        <column name="name" type="varchar" primaryString="1" size="100" />
   </table>
</database>
{% endhighlight %}

Create the classes:

    php app/console propel:build-model

Assuming you have products in your database, you won't to loose them. Thanks to migrations, Propel will
be able to update your database without loosing existing data.

    php app/console propel:migration:generate-diff

    php app/console propel:migration:migrate

Your database has been updated, you can continue to write your application.

### Saving Related Entities ###

Now, let's see the code in action. Imagine you're inside a controller:

{% highlight php %}
<?php
// ...
use Acme\StoreBundle\Model\Category;
use Acme\StoreBundle\Model\Product;
use Symfony\Component\HttpFoundation\Response;
// ...

class DefaultController extends Controller
{
    public function createProductAction()
    {
        $category = new Category();
        $category->setName('Main Products');

        $product = new Product();
        $product->setName('Foo');
        $product->setPrice(19.99);
        // relate this product to the category
        $product->setCategory($category);

        // save the whole
        $product->save();

        return new Response(
            'Created product id: '.$product->getId().' and category id: '.$category->getId()
        );
    }
}
{% endhighlight %}

Now, a single row is added to both the `category` and product tables. The `product.category_id` column for the
new product is set to whatever the id is of the new category. Propel manages the persistence of this relationship for you..

### Fetching Related Objects ###

When you need to fetch associated objects, your workflow looks just like it did before.
First, fetch a `$product` object and then access its related `Category`:

{% highlight php %}
<?php
// ...
use Acme\StoreBundle\Model\ProductQuery;

public function showAction($id)
{
    $product = ProductQuery::create()
        ->joinWithCategory()
        ->findPk($id)
        ;

    $categoryName = $product->getCategory()->getName();

    // ...
}
{% endhighlight %}

### More information on Associations ###

You will find more information on relations by reading the dedicated chapter on [relationships](../../documentation/04-relationships.html).

## Behaviors ##

All bundled behaviors in Propel are working with Symfony2. To get more information about how to use Propel behaviors,
look at the [behaviors reference section](http://www.propelorm.org/documentation/#behaviors_reference).

## Commands ##

You should read the dedicated section for [Propel commands in Symfony2](working-with-symfony2#commands).

## Summary ##

This documentation is quite similar with the [official Symfony2 Model chapter](http://symfony.com/doc/current/book/doctrine.html).
It shows you that Propel is well integrated into Symfony2 and it really works. All you can do with Propel is possible in Symfony2.
