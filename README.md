ElasticPress [![Build Status](https://travis-ci.org/10up/ElasticPress.svg?branch=master)](https://travis-ci.org/10up/ElasticPress)
=============

Integrate [Elasticsearch](http://www.elasticsearch.org/) with [WordPress](http://wordpress.org/).

* **Latest Stable**: [v0.9.3](https://github.com/10up/ElasticPress/releases/tag/v0.9.3)
* **Contributors**: [@aaronholbrook](https://github.com/AaronHolbrook), [@tlovett1](https://github.com/tlovett1), [@mattonomics](https://github.com/mattonomics), [@ivanlopez](https://github.com/ivanlopez), [@colegeissinger](https://github.com/colegeissinger), [@cmmarslender](https://github.com/cmmarslender)

## Background

Let's face it, WordPress search is rudimentary at best. Poor performance, inflexible and rigid matching algorithms (which means no comprehension of 'close' queries), the inability to search metadata and taxonomy information, no way to determine categories of your results and most importantly the overall relevancy of results is poor.

Elasticsearch is a search server based on [Lucene](http://lucene.apache.org/). It provides a distributed, multitenant-capable full-text search engine with a [REST](http://en.wikipedia.org/wiki/Representational_state_transfer)ful web interface and schema-free [JSON](http://json.org/) documents.

Coupling WordPress with Elasticsearch allows us to do amazing things with search including:

* Relevant results
* Autosuggest
* Fuzzy matching (catch misspellings as well as 'close' queries)
* Proximity and geographic queries
* Search metadata
* Search taxonomies
* Facets
* Search all sites on a multisite install
* [The list goes on...](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search.html)

## Purpose

The goal of ElasticPress is to integrate WordPress with Elasticsearch. This plugin integrates with the [WP_Query](http://codex.wordpress.org/Class_Reference/WP_Query) object returning results from Elasticsearch instead of MySQL.

There are other Elasticsearch integration plugins available for WordPress. ElasticPress, unlike others, offers multi-site search. Elasticsearch is a complex topic and integration results in complex problems. Rather than providing a limited, clunky UI, we elected to instead provide full control via [WP-CLI](http://wp-cli.org/).

## Installation

1. First, you will need to properly [install and configure](http://www.elasticsearch.org/guide/en/elasticsearch/guide/current/_installing_elasticsearch.html) Elasticsearch.
2. ElasticPress requires WP-CLI. Install it by following [these instructions](http://wp-cli.org).
3. Install the plugin in WordPress. You can download a [zip via Github](https://github.com/10up/ElasticPress/archive/master.zip) and upload it using the WP plugin uploader.

## Configuration

First, make sure you have Elasticsearch and WP-CLI configured properly.

1. Define the constant ```EP_HOST``` in your ```wp-config.php``` file with the connection (and port) of your Elasticsearch application. For example:

```php
define( 'EP_HOST', 'http://192.168.50.4:9200' );
```

The proceeding sets depend on whether you are configuring for single site or multi-site with cross-site search capabilities.

### Single Site

2. Activate the plugin.
3. Using wp-cli, do an initial sync (with mapping) with your ES server by running the following commands:

```bash
wp elasticpress index --setup
```

### Multisite Cross-site Search

2. Network activate the plugin
3. Using wp-cli, do an initial sync (with mapping) with your ES server by running the following commands:

```bash
wp elasticpress index --setup --network-wide
```

After your index finishes, ```WP_Query``` will be integrated with Elasticsearch and support a few special parameters.

### Creating Elasticsearch Indices

Creating indices is handled automatically by ElasticPress. Index names are automatically generated based on site URL.

## Usage

After running an index, ElasticPress integrates with WP_Query. The end goal is to support all the parameters available to WP_Query so the transition is completely transparent. Right now, our WP_Query integration supports *many* of the relevant WP_Query parameters and adds a couple special ones.

### Supported WP_Query Parameters

* ```s``` (*string*)

    Search keyword. By default used to search against ```post_title```, ```post_content```, and ```post_excerpt```.

* ```posts_per_page``` (*int*)

    Number of posts to show per page. Use -1 to show all posts (the ```offset``` parameter is ignored with a -1 value). Set the ```paged``` parameter to paginate based on ```posts_per_page```.

* ```tax_query``` (*array*)

    Filter posts by terms in taxonomies. Takes an array of form:

    ```php
    new WP_Query( array(
        's' => 'search phrase',
        'tax_query' => array(
            array(
                'taxonomy' => 'taxonomy-name',
                'terms' => array( ... ),
            )
        ),
    ));
    ```

    ```tax_query``` accepts an array of arrays where each inner array *only* supports ```taxonomy``` (string) and ```terms``` (string|array) parameters. ```terms``` is a slug, either in string or array form.

* ```post_type``` (*string*/*array*)

    Filter posts by post type. ```any``` wil search all public post types.

* ```offset``` (*int*)

    Number of posts to skip in ascending order.

* ```paged``` (*int*)

    Page number of posts to be used with ```posts_per_page```.

* ```author``` (*int*)

    Show posts associated with certain author ID.
    
* ```author_name``` (*string*)

    Show posts associated with certain author. Use ```user_nicename``` (NOT name).

The following are special parameters that are only supported by ElasticPress.

* ```search_tax``` (*array*)

    Applies the current search to terms within a taxonomy or taxonomies. The following will fuzzy search across the normal search fields AND terms within taxonomies ```category``` and ```post_tag```:

    ```php
    new WP_Query( array(
        's' => 'term search phrase',
        'search_tax' => array( 'category', 'post_tag' ),
    ));
    ```

* ```search_meta``` (*array*)

    Applies the current search to post meta. The following will fuzzy search across normal search fields AND post meta keys ```meta_key_1``` and ```meta_key_2```:

    ```php
    new WP_Query( array(
        's' => 'meta search phrase',
        'search_meta' => array( 'meta_key_1', 'meta_key_2' ),
    ));
    ```

* ```aggs``` (*array*)

    Add aggregation results to your search result. For example:
    
    ```php
    new WP_Query( array(
        's' => 'search phrase',
        'aggs' => array(
            'name' => 'name-of-aggregation', // (can be whatever you'd like)
            'use-filter' => true // (*bool*) used if you'd like to apply the other filters (i.e. post type, tax_query, author), to the aggregation
            'aggs' => array(
                'name' => 'name-of-sub-aggregation',
                'terms' => array(
                    'field' => 'terms.name-of-taxonomy.name-of-term',
                ),
            ),
        ),
    ));
    ```

* ```sites``` (*int*/*string*/*array*)

    This parameter only applies in a multi-site environment. It lets you search for posts on specific sites or across the network.

    By default, ```sites``` defaults to ```current``` which searches the current site on the network:

    ```php
    new WP_Query( array(
        's' => 'search phrase',
        'sites' => 'current',
    ));
    ```

    You can search on all sites across the network:

    ```php
    new WP_Query( array(
        's' => 'search phrase',
        'sites' => 'all',
    ));
    ```

    You can also specify specific sites by id on the network:

    ```php
    new WP_Query( array(
        's' => 'search phrase',
        'sites' => 3,
    ));
    ```

    ```php
    new WP_Query( array(
        's' => 'search phrase',
        'sites' => 3,
    ));
    ```

## Development

### Setup

Follow the configuration instructions above to setup the plugin.

### Testing

Within the terminal change directories to the plugin folder. Initialize your testing environment by running the
following command:

For VVV users:
```
bash bin/install-wp-tests.sh wordpress_test root root localhost latest
```

For VIP Quickstart users:
```
bash bin/install-wp-tests.sh wordpress_test root '' localhost latest
```

where:

* ```wordpress_test``` is the name of the test database (all data will be deleted!)
* ```root``` is the MySQL user name
* ```root``` is the MySQL user password (if you're running VVV). Blank if you're running VIP Quickstart.
* ```localhost``` is the MySQL server host
* ```latest``` is the WordPress version; could also be 3.7, 3.6.2 etc.


Our test suite depends on a running Elasticsearch server. You can supply a host to PHPUnit as an environmental variable like so:

```bash
EP_HOST="http://192.168.50.4:9200" phpunit
```

### Issues

If you identify any errors or have an idea for improving the plugin, please [open an issue](https://github.com/10up/ElasticPress/issues?state=open). We're excited to see what the community thinks of this project, and we would love your input!
