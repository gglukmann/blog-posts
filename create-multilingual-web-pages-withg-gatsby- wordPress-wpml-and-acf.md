# Create Multilingual Web Pages With Gatsby, WordPress, WPML, and ACF

Bring your site to the world

![Gatsby page with English and Estonian content from WordPress](https://cdn-images-1.medium.com/max/2000/1*vVTDgiAoliRU_lI93lpLWQ.jpeg)_Gatsby page with English and Estonian content from WordPress_

Gatsby is a great static-site generator to use today. Its ecosystem is really large, and you get a lot out of the box. Getting Lighthouse maximum scores is almost default with Gatsby. Anyone who is working with WordPress and wants to separate the CMS and the website itself should at least try to create something with Gatsby. It is really easy to use, and the documentation is very straightforward.

Gatsby is using GraphQL for getting data from local files or from external endpoints. If you want use it with WordPress for getting pages, posts, media, ACF fields, etc., you don’t have to manually figure it out. There’s a library that creates schema from the WordPress REST API to GraphQL and it’s backed by Gatsby. It’s [WPGraphQL](https://github.com/wp-graphql/wp-graphql), and there’s a Gatsby plugin, [gatsby-source-wordpress](https://www.gatsbyjs.org/packages/gatsby-source-wordpress/), to connect your WordPress site to. It’s using that connector library underneath.

This piece expects you have WordPress set up with WPML and ACF plugins activated. It also expects you have the gatsby-source-wordpress plugin set up in `gatsby-config.js`. In the [example repository](https://github.com/gglukmann/multilingual-wp-gatsby), you can see how I connected to WordPress from Gatsby.

## The Problem: Changing the Language in GraphQL

There’s only one problem. Lets say you’re creating a page with only one view, and this will be on the root URL `//your-site.domain/`. Now, you’ll need to create the same page in a different language that’ll sit in the `//your-site.domain/et` URL — just like when using standard WordPress. How do you do it using WPML in Wordpress and creating pages with Gatsby?

The WordPress REST API endpoint gets you content for the default language. Example: `//your-site.domain/wp-json/wp/v2/pages` is in your WPML default language. You can switch languages by adding the `?lang=et` parameter, but with GraphQL you can’t add parameters like that. You’ll have to add it as a filter to the query. The GraphQL schema in Gatsby doesn’t have that filter for WMPL. We have to add it ourselves.

## Creating Pages in Gatsby

I have created a page in WordPress with the slug `homepage` and with the ACF fields `title` and `description`.

![ACF fields in WordPress](https://cdn-images-1.medium.com/max/3032/1*kRHiHmk_sOzzTwmNAfDMYg.png)_ACF fields in WordPress_

Make sure every page with a different language has the same slug because WordPress creates new slugs for different languages. When I created a new page in the Estonian language, WordPress created the slug `homepage-2`. You could, of course, query it with its ID, too, but it’ll be easier to query data for that page with a known slug. You’ll see later where we’re going to use it.

Creating pages in Gatsby is usually done by adding new JavaScript files to the `src/pages` folder with the name that’ll be the route. Like the `about.js` file would have `/about` as its URL. When you’re creating pages from WordPress, you’ll have to create them during the build time. You’ll need to open `gatsby-node.js` and use the [`createPages`](https://www.gatsbyjs.org/docs/node-apis/#createPages) function that Gatsby provides.

For our case, we’ll need to create a separate page for all languages. Our index page will have the URLS `/` for English and `/et` for the Estonian language.

```javascript
const path = require(`path`)

const languages = [
  {
    path: "/",
    code: "en_US", <- taken from WPML language codes
  },
  {
    path: "/et",
    code: "et",
  },
]

exports.createPages = async ({ actions: { createPage } }) => {
  const HomepageTemplate = path.resolve("./src/templates/Homepage.js")

  languages.forEach(lang => {
    createPage({
      path: lang.path,
      component: HomepageTemplate,
      context: {
        lang: lang.code,
      },
    })
  })
}
```

We’ve created an array with languages that match our WordPress WPML setup. This will be looped over, and one page will be created for each language with a given path.

You can see there’s a template file from `./src/templates/Homepage.js`. This will be the template that has our index-page components inside — just like when you’d add a new page inside the `src/pages` folder.

Next, as you’d think, we’ll have to create that template file. Create the folder `templates` inside `./src`, and inside it, create a file named `Homepage.js`.

```javascript
import React from "react"
import { Link } from "gatsby"

import Layout from "../components/layout"

const HomepageTemplate = () => {
  return (
    <Layout title="Title">
      <p>Description</p>

      <Link to="/">English</Link>
      <Link to="/et">Estonian</Link>
    </Layout>
  )
}

export default HomepageTemplate
```

The hardcoded texts `Title` and `Description` will be replaced with the texts from WordPress a bit later.

If you run `gatsby develop`, then you can switch between those two views. But right now, the content is exactly the same.

## Getting Data from WordPress

In your `Homepage.js` file, add the following GraphQL query before `export default HomepageTemplate`. Make sure to add `graphql` to import from `gatsby` as the named import.

```javascript
import { graphql, Link } from "gatsby"

...

export const query = graphql`
  query {
    wordpressPage(
      slug: { eq: "homepage" }
    ) {
      acf {
        title
        description
      }
    }
  }
`

export default HomepageTemplate
```

Here you can see we’re querying a WordPress page with slug that equals `"homepage"` and two ACF fields — `title` and `description` — that we had set up in WordPress. The query result is added to your `HomepageTemplate` component as the prop `data`.

```javascript
const HomepageTemplate = ({
  data: {
    wordpressPage: {
      acf: { title, description },
    },
  },
}) => {

...
```

With object destructuring, we have `title` and `description` ready to use inside our React component. We can change our HTML.

```javascript
<Layout title={title}>
<p>{description}</p>
```

Now if you run it in your browser, it shows text in the default language and switching between those pages will still not change anything yet again. We’ll get to that now.

## Adding Content in Other Languages to the WordPress REST API So GraphQL Can Create Schema

Switching pages doesn’t change the language because the WordPress REST API is only giving out data in one language, and we’ll have to change that.

First, look at the WordPress REST API `//your-site.domain/wp-json/wp/v2/pages`, and you can see only one object with content in the default language there. But we’ll need to have both languages there in different objects.

For that, you’ll need to open your currently active theme code, located at `./wp-content/themes/example-theme/`. Open the file `functions.php`, and add the following lines there.

```javascript
add_action('rest_api_init', function () {
  if (defined('REST_REQUEST') && REST_REQUEST) {
    // Add all WPML language translations to rest api when type is page
    add_action('parse_query', function ($q) {
      if ($q->query_vars['post_type'] == 'page') {
        $q->query_vars['suppress_filters'] = true;
      }
    });
  }
});
```

This trick is taken from [wmpl.org forum](https://wpml.org/forums/topic/how-can-i-get-all-pages-original-and-translated-in-the-api-response/#post-3072937). Now if you look at the WordPress REST API, `//your-site.domain/wp-json/wp/v2/pages`, you can see there are two objects with different languages.

That means GraphQL can now create schema for both languages.

Before we can start using it inside our React component, we need to be able to get the current language code, too. If you look closely into the REST API response, you’ll see that `title` and `description` are in different languages inside different objects, but there’s no way to get the language code.

For that, you’ll need the [WPML REST API](https://github.com/shawnhooper/wpml-rest-api) plugin activated inside WordPress. For us, it adds `wpml_current_locale` to the REST API response. This way we can know which language to query from GraphQL.

## Getting Texts in the Right Language From GraphQL

If you look at the `gatsby-node.js` file, you can see in our languages array, we have `code` defined for each language. This `code` is exactly the same as `wpml_current_locale`. If you look at where we’re using the `createPage` function, you’ll see we’re giving the `context` as the property with that `code`.

```javascript
createPage({
  path: lang.path,
  component: HomepageTemplate,
  context: {
    lang: lang.code, <- sending language code to GraphQL query
  },
})
```

We’ll get this as a GraphQL variable inside `Homepage.js`, where we’re going to make the query.

Update the `Homepage.js` GraphQL query with the following code.

```javascript
export const query = graphql`
  query($lang: String) {
    wordpressPage(
      slug: { eq: "homepage" }
      wpml_current_locale: { eq: $lang }
    ) {
      acf {
        title
        description
      }
    }
  }
`
```

`$lang` is the language code we sent with the context from the `createPage` function. We pass it to query the filter as equal to `wpml_current_local`.

And we’ve done it!

Now if you run it in a browser, it shows the text in English, and when switching it to a different language, `title` and `description` are going to change.

## Conclusion

This solution is pretty standard for [creating pages with Gatsby and getting data from Wordpress](https://www.gatsbyjs.org/docs/recipes/sourcing-data#sourcing-from-wordpress), but that one little trick inside the WordPress theme `functions.php` is what matters to get data for all available WPML languages.

Thanks.

Here’s a link to the [example repository](https://github.com/gglukmann/multilingual-wp-gatsby).
