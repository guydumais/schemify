# Schemify

[![Build Status](https://travis-ci.org/stevegrunwell/schemify.svg?branch=master)](https://travis-ci.org/stevegrunwell/schemify)
[![Code Climate](https://codeclimate.com/github/stevegrunwell/schemify/badges/gpa.svg)](https://codeclimate.com/github/stevegrunwell/schemify)
[![Test Coverage](https://codeclimate.com/github/stevegrunwell/schemify/badges/coverage.svg)](https://codeclimate.com/github/stevegrunwell/schemify/coverage)

> **Warning:** Schemify is very much in an experimental phase at this time, and is not recommended for use in a production environment!

Structured data allows developers and publishers to mark up content in a way that it's easy for machines to understand. Proper use of structured data [enables third-parties like Google to parse reviews, event data, and contact information in meaningful ways](https://developers.google.com/search/docs/guides/intro-structured-data), ensuring you're getting the most "bang" out of your publishing buck.

Fortunately, the major players in the Search game, including Google, Microsoft, Yahoo!, and Yandex) came together in the early 2010s to form [Schema.org](http://schema.org/docs/about.html), a collaborative, community-driven standard for structured data.

With the major search engines and communities behind it, we're all marking up everything with appropriate structured data now, right?

Unfortunately, the process of implementing Schema.org in a site – especially one driven by dynamic content – is less than straightforward. [WordPress](https://wordpress.org), for instance, [powers roughly a quarter of the web](https://ma.tt/2015/11/seventy-five-to-go/), but implementing structured data across hundreds of thousands of themes would be a nightmare.

Or, at least it would be, if it weren't for Schemify.


## What does Schemify do?

There are two approaches to adding structured data to a website: via the markup or [JSON for Linking Data (JSON-LD)](http://json-ld.org/).

Consider the following author information, which might appear at the bottom of a blog post:

```html
<div class="author">
	<img src="http://en.gravatar.com/avatar/88ea4e10ed968136228545d7112d82cb?s=200" alt="Steve Grunwell" />
	<h2><a href="https://stevegrunwell.com" rel="author">Steve Grunwell</a></h2>
	<p>Steve is a Senior Web Engineer + Project Lead at 10up. When he's not working, you can find him speaking at conferences, roasting coffee, or spending time with his wife and daughter</p>
</div>
```

If I wanted this information to use Schema.org markup, it would look something like this:

```html
<div class="author" itemscope itemtype="http://schema.org/Person">
	<img src="http://en.gravatar.com/avatar/88ea4e10ed968136228545d7112d82cb?s=200" alt="Steve Grunwell" itemprop="image" />
	<h2 itemprop="name"><a href="https://stevegrunwell.com" rel="author" itemprop="url">Steve Grunwell</a></h2>
	<p itemprop="description">Steve is a Senior Web Engineer + Project Lead at 10up. When he's not working, you can find him speaking at conferences, roasting coffee, or spending time with his wife and daughter</p>
</div>
```

While that may not _seem_ like a lot of work, that's still a lot of extra markup being added. Even worse, a lot of that markup might normally be generated by WordPress helper functions like [`get_avatar()`](https://developer.wordpress.org/reference/functions/get_avatar/), which means extra work to get the necessary `itemprop` attribute in place.

Fortunately, the other approach for adding structured data is much more straight-forward in a theme, as it's simply a JSON-LD object:

```html
<script type="application/ld+json">
{
	"@context": "http://schema.org",
	"@type": "Person",
	"name": "Steve Grunwell",
	"url": "https://stevegrunwell.com",
	"image": "http://en.gravatar.com/avatar/88ea4e10ed968136228545d7112d82cb?s=200",
	"description": "Steve is a Senior Web Engineer + Project Lead at 10up. When he's not working, you can find him speaking at conferences, roasting coffee, or spending time with his wife and daughter"
}
</script>
```

We simply generate this JSON-LD object and append it to our document's `<body>`. When Google or anyone else who supports JSON+LD structured data parses our page, they can immediately understand that Steve Grunwell is a person and determine his website, avatar, and biography.

The best part? There's no need to change the markup within our theme!

**Schemify aims to automate the generation of JSON-LD objects for content within WordPress.** Its flexible structure and reasonable defaults enables drop-in support for structured data, regardless of the WordPress theme.


## Installation

After uploading Schemify to your WordPress instance, activate it via the Plugins screen in wp-admin.


### Registering post type support

Out of the box, Schemify has support for the "post", "page", and "attachment" post types, but you can easily enable it on your own custom post types via [`add_post_type_support()`](https://codex.wordpress.org/Function_Reference/add_post_type_support):

```php
/**
 * Add Schemify support to the "book" custom post type.
 */
function mytheme_add_schemify_support() {
	add_post_type_support( 'book', 'schemify' );
}
add_action( 'init', 'mytheme_add_schemify_support' );
```

> **Note:** You may need to adjust the action ordering to ensure that your `add_post_type_support()` callback runs *after* your custom post type is registered. You may alternately choose to add "schemify" to the `supports` parameter within [`register_post_type()`](https://codex.wordpress.org/Function_Reference/register_post_type#supports) as the custom post type is registered.


## Working with Schemify

Out of the box, Schemify will append a JSON-LD object to the footer of your site's homepage and any single posts that have registered post type support for Schemify.

While that might be plenty for most sites, more complex sites will likely want to tweak the way Schemify works to optimize the way content is represented.


### Overriding Schemify properties

When constructing Schemify objects, it's important to remember that a single, top-level Schema can have any number of nested objects. For example, a single post may be represented with a `BlogPosting` object, which has properties like "author", "publisher", and "image", which are represented by `Person`, `Organization`, and `ImageObject` objects, respectively.

As each object is built, its values are passed through the `schemify_get_properties_$schema` filter, where `$schema` is the current object's declared Schema.

The filter is passed up to four arguments:

<dl>
	<dt>(array) $data</dt>
	<dd>The collection of properties assembled for this object.</dd>
	<dt>(string) $schema</dt>
	<dd>The current schema being filtered.</dd>
	<dt>(int) $object_id</dt>
	<dd>The object ID being constructed.</dd>
	<dt>(bool) $is_main</dt>
	<dd>Is this the top-level JSON-LD schema being constructed?</dd>
</dl>

#### Example

Let's say you've added a custom `twitter_url` user meta property to user profiles on your site, and wish to inject this information into the "sameAs" property on a `Person` object:

```php
/**
 * Add a user's "twitter_url" user meta.
 *
 * @param array  $data      The collection of properties assembled for this object.
 * @param string $schema    The current schema being filtered.
 * @param int    $object_id The object ID being constructed.
 */
function add_twitter_url( $data, $schema, $object_id ) {
	if ( $twitter = get_user_meta( $object_id, 'twitter_url', true ) ) {
		$data['sameAs'] = $twitter;
	}

	return $data;
}
add_filter( 'schemify_get_properties_Person', __NAMESPACE__ . '\add_twitter_url', 10, 3 );
```

Using the nested nature of Schema objects, you can easily use Schemify to manipulate your data objects.
