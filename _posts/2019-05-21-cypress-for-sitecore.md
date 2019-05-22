---
layout: post
title: Cypress For Sitecore
subtitle: A small implementation for running Cypress tests in Sitecore
gh-repo: sebastianbienko/cypressforsitecore
gh-badge: [star, fork, follow]
tags: [Cypress.io, Sitecore, testautomation]
comments: true
---

After having stuck with [Selenium](https://www.seleniumhq.org) as the only option for running automated UI-Tests for a couple of years now, there are finally arrising some alternatives at the horizon. Most knowingly [Cypress.io](https://cypress.io/) and [TestCafé Studio](https://www.devexpress.com/products/testcafestudio/). Both solutions share some common features, but also have some notable differences. For a good Side-by-Side comparison I recommend you to read this blog post: [https://xebia.com/blog/cypress-and-testcafe-a-comparison-part-one/](https://xebia.com/blog/cypress-and-testcafe-a-comparison-part-one/)

At Deloitte Digital we have given Cypress a shot. There has also already been an implementation, which has Cypress tests automatically run after a Sitecore publish by [Dave Leigh](https://github.com/daveaftershok). Check it out here: [https://github.com/daveaftershok/sitecore-test-run-on-publish](https://github.com/daveaftershok/sitecore-test-run-on-publish)\\
We figured, that it would be nice to know, if a feature will break, before publishing the changes. Also we wanted to get to know Cypress a little bit more in depth and see how flexible it actually is. So we took the "Sitecore Test Run On Publish" implementation of Dave as basis for a PoC, in which we wanted to acheive:

1. Being able to selectively run Cypress tests against the "Live" and "Preview" site.
2. Being able to run Cypress tests for individual pages and test only the page's actual renderings.
3. Have the visualization of the test result integrated in the Sitecore backend.

In the following I will outline, how we did this. If you haven't heard about Cypress before, I recommend you to check it out on [Cypress.io](https://cypress.io/) first (They also have a great documentation!) Let's go!

![CypressForSitecore](/assets/CypressForSitecore.gif)

## Starting Point

My starting point has been the implementation by Dave Leigh, which consists of two parts:

1. A Sitecore Module, which triggers the test runs and makes the tests results accessible from Sitecore. [GitHub Repo](https://github.com/daveaftershok/sitecore-test-run-on-publish)
2. A node.js web-server, which provides a simple Web-API to communicate with Cypress. [GitHub Repo](https://github.com/daveaftershok/cypress-node-api-windows)

A test run can be triggered by calling a specific endpoint ('/run'). Since we want Cypress to know which renderings and which page to test, we need to **pass additional parameters to the endpoint**, like the page url or the rendering IDs. Because Cypress does not know Sitecore or renderings we need to **implement some logic, which makes it possible to differentiate between different test cases by rendering definitions**. Last but not least, there was no **integration of results in the Sitecore back-end**, yet. So far the results were accessible on an external page, provided by the node.js web-server.

## Implementation

### Passing rendering and page information to Cypress

First, I added some buttons to the *Review* Strip of the Ribbon:

![ButtonsInRibbon](/assets/Buttons-In-Ribbon.png)

Those Button would trigger a command. On command execution, the context item is known and we can retrieve all the rendering informations. The only difference here between the "Test Live" and "Test Preview" button is, that the former is working against the *web* database and the latter against the *master* database.\\
Once we have all the renderings of the page item, we can extract the relevant informations. As a basis I am assuming a Sitecore solution, which follows the Helix principles. I found the following information to be relevant:

* Helix Layer: Project &#124; Feature &#124; Foundation
* Helix Module Name
* Rendering Name (could as well have used the ID)
* Datasource (if any)

The first two information about the Helix layer and module might help with performance, when filtering among large numbers on test files, later on. (Cypress gives you full flexibility in using any folder structure)\\
The rendering name or ID is needed to match the relevant test cases to the specific rendering.\\
The datasource can be used to specify the rendering even further. I am for example having the *Forms* rendering in mind, which might be used in very different cases and varies a lot, depending on the datasource used.

The result is being parsed to JSON and might look like this:

```json
"pageRenderings": [
  {
    "RenderingItemName": "Snippet",
    "HelixLayer": 2,
    "HelixModule": "Experience-Accelerator",
    "Datasource": "{874ED455-AF6F-438D-BEA8-1916B7FAB5BD}"
  },
  {
    "RenderingItemName": "Promo",
    "HelixLayer": 2,
    "HelixModule": "Experience-Accelerator",
    "Datasource": ""
  }
]
```

For simplicity I decided to pass the renderings as query parameter. Also, since I was already passing arguments to Cypress, I decided to have Sitecore "control" Cypress, by also passing Cypress settings as parameters.\\
This is fairly easy, because Cypress natively allows you to configure it by a configuration file, by passing command line arguments or by environment variables (Read more about it [here](https://docs.cypress.io/guides/references/configuration.html#Overriding-Options)). I have added a Sitecore configuration, where Cypress settings can be added. They will just be passed alongside the rendering informations as query parameter.

```xml
<sitecore>
  <cypress>
    <testRunners>
      <liveTestRunner type="Deloitte.Foundation.Testing.UI.Services.Cypress.CypressTestRunner, Deloitte.Foundation.Testing.UI">
        <param>http://localhost:3000/Run</param>
        <options hint="raw:AddOption">
          <option name="--config">baseUrl=http://habitathome.dev.local/</option>
          <option name="--config">video=false</option>
          <option name="--reporter">mochawesome</option>
          <option name="--record">false</option>
        </options>
      </liveTestRunner>
      <previewTestRunner type="Deloitte.Foundation.Testing.UI.Services.Cypress.CypressTestRunner, Deloitte.Foundation.Testing.UI">
        <param>http://localhost:3000/Run</param>
        <options hint="raw:AddOption">
          <option name="--config">baseUrl=http://habitathome.testing.dev.local/</option>
        </options>
      </previewTestRunner>
    </testRunners>
  </cypress>
</sitecore>
```

### Running relevant tests only

On the side of the node.js web-server the query parameters are being formated a little bit, before they are passed to Cypress (I won't get into this one). The following JS functions are used to check, if a test is relevant for the current run:

{% highlight javascript linenos %}
function containsRendering(pageRendering, allowedRendering){
  if (allowedRendering.datasource === undefined || allowedRendering.datasource === null || allowedRendering.datasource === ""){
    return pageRendering.RenderingItemName === replaceWhiteSpaces(allowedRendering.name);
  }else{
    return pageRendering.RenderingItemName === replaceWhiteSpaces(allowedRendering.name) && pageRendering.Datasource === replaceWhiteSpaces(allowedRendering.datasource);
  }
}

function pageHasRelevantRendering(pageRenderings, allowedRenderings){
  return allowedRenderings.some(function(allowedRendering, index) {
      if (pageRenderings.some((pageRendering, index) => containsRendering(pageRendering, allowedRendering))){
          return true;
      }
  })
}

function isRelevant(allowedRenderings){
  try{
    var pageRenderings = Cypress.env('pageRenderings');
    return pageHasRelevantRendering(pageRenderings, allowedRenderings);
  }
  catch(ex){
    return true;
  }
}

function replaceWhiteSpaces(value){
  var find = " ";
  var regEx = RegExp(find, 'g');
  return value.replace(regEx, "-")
}
{% endhighlight %}

Basically the *pageRendering* are read from the Cypress settings and we compare them to a given parameter *allowedRenderings*, which is an array of rendering names and datasources. It is fairly simple and even more easily explained by just giving an example.

We can define our relevant renderings at a central location:
```javascript
const renderings = {
    carousel: {
      name: 'Carousel',
      datasource: ''
    },
    snippet: {
      name: 'Snippet',
      datasource: ''
    }
  }
```

And later on simply select the renderings for which we want to run the test:
```javascript
describe("Carousel Rendering", function() {
  var relevantRenderings = [renderings.carousel];

  if (isRelevant(relevantRenderings)){
    it('Some Carousel Rendering Test', function() {
      cy.visit("/");
      //Test the rendering
    })
  }
})
```

There are probably more elegant ways to implement this, but it works just fine for this PoC. If there is a *datasource* defined for a rendering, then it will have to match the datasource passed by Sitecore as well as the name of the rendering.

### Making the test results available in the Sitecore back-end

I will keep this one very short: Before, the results had to be viewed outside of Sitecore on a different site. Now, this external site is being "integrated" by an Iframe.

![Cypress-TestResults-In-Sitecore](/assets/Cypress-TestResults-In-Sitecore.png)

The JS library [*mochawesome*](https://www.npmjs.com/package/mochawesome) is being used to generate the HTML file.

## Happy Testing!

I haven't said much about Cypress itself, there already is enough information about it out there to get you started. My take on it: *Cypress makes testing almost actually fun.* ;)

Take a peak at my implementation on GitHub:

* [CypressForSiteore](https://github.com/sebastianbienko/CypressForSitecore) *(Sitecore module)*
* [CypressForSitecore.Webserver](https://github.com/sebastianbienko/CypressForSitecore.Webserver) *(node.js webser)*

Feel free to contact me for questions or anything else!
