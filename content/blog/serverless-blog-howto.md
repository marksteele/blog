---
layout: blog
title: Serverless blog HOWTO
date: '2018-04-22T12:36:25-04:00'
thumbnail: /images/serverlessblog.png
categories:
  - Serverless
  - Lambda
  - AWS
  - Netlify CMS
  - TravisCI
  - Github
---
# Serverless content management

In this post, I will go over how to setup a completely serverless blog that runs with no servers and is for all intents and purposes free to run! 

It leverages static content generation, will be served from a content delivery network, and has a browser based UI for managing content with a review/approval workflow.

It's also highly secure, as there is virtually no attack surface for the bad guys to get a hold of.

![null](/images/serverlessblog.png)

<!--more-->

# Moving parts

* [Hugo](https://gohugo.io): Static site generator
* [Github](https://github.com): Code repository. We'll use 2 repos for this project. One contains the source, and one contains the HTML that we want published (using Github pages).
* [TravisCI](https://travis-ci.org): The continuous integration tool that we can leverage to build the site when the source gets updated.
* [Netlify CMS](https://www.netlifycms.org/): The browser-based content management system that we'll integrate with Github to be able to create posts straight from a browser.
* [AWS Lambda](https://aws.amazon.com/lambda/): In order to authenticate the CMS with Github, we need an application in the middle to handle the OAuth2 handshake (and keep our secrets secret). We'll use a Lambda function [that I wrote](https://github.com/marksteele/netlify-serverless-oauth2-backend) with the [Serverless framework](https://serverless.com) to do just that.

## Static Site Generator - Hugo

I chose to leverage a static content site generator for this, as I wanted to keep things as simple as possible as well as ensure that the site will load fast and be cacheable.

The framework that I landed on was [Hugo](https://gohugo.io/). Getting Hugo up and running is pretty straight-forward.

Follow the instructions in the [quick-start guide](https://gohugo.io/getting-started/quick-start/), and you'll be ready for the next steps.

By the end of the quick-start, you should be able to test a working static site on your local machine.

Here's what my configuration file looks like (config.toml):

```
baseURL = "https://www.control-alt-del.org/"
languageCode = "en-us"
title = "Cogito Interruptus Vulgaris"
theme = "hugo-theme-bleak"
copyright = "Mark Steele"
pygmentsCodeFences = "true"
PygmentsStyle = "perldoc"

[blackfriday]
  plainIDAnchors = true
  hrefTargetBlank = false
  latexDashes = true
  smartypants = false
  factions = true
  extensions = ["hardLineBreak"]

[params]
    cover = "/images/IMG_1191.jpg"
    Subtitle = "Mark's blog"
    description = "Everything is awesome !"

[params.posts]
    foldername = "post"
    pagesize = "6"
    featuredpost = "true"
```

## Setup some Github repositories

Next we'll want to create a couple Github repositories to store the source and published sites. For the source site, create a repo. Mine's called 'blog'.

For the published site, I'm using Github pages, so my repo for that is 'marksteele.github.io'.

Read on [here](https://pages.github.com/) for more details on Github pages.

## Set origins

Head on over to your blog source (~/blog in this example), and set the origin. Something like this:

```
cd ~/blog
git init
git remote add origin git@github.com:marksteele/blog.git 
```

Next we want to make sure we point our `public` folder to the repository we want to publish to. We'll use a git submodule for this.

```
cd
mkdir marksteele.github.io
cd marksteele.github.io
touch README.md
git init
git remote add origin git@github.com:marksteele/marksteele.github.io.git 
git add .
git commit -m 'initial import'
git push origin master
```

Next we want to create a configuration file for TravisCI. Head on over to the blog source and create a file names .travis.yml with the following contents (edit to match your settings):

```yaml
language: python

install:
  - sudo apt-get install python-pygments
  - sudo wget -qO hugo_0.39_Linux-64bit.deb https://github.com/gohugoio/hugo/releases/download/v0.39/hugo_0.39_Linux-64bit.deb
  - sudo dpkg -i hugo_0.39_Linux-64bit.deb

script:
  - git submodule foreach -q --recursive 'branch="$(git config -f $toplevel/.gitmodules submodule.$name.branch)"; [ -z "$branch" ] && git checkout master || git checkout $branch'
  - hugo # This commands builds your website on travis

after_success: |
    cd public
    git add .
    git status
    git commit -m 'Site push'
    git push https://$GITHUB_USER:$GITHUB_API_KEY@github.com/marksteele/marksteele.github.io master

branches:
  only:
  - master
```

This job will install a few packages, then run a couple scripts. The first script ensures that submodules are pointing to the branch HEAD, and the second script generates the site. Once the site is built, the new content is published to the publishing repo.

Head back over into the blog, remove the public folder and setup the submodule pointing to our publishing repo. Then push.

```
cd ~/blog
rm -rf public
git submodule add -b master https://github.com/marksteele/marksteele.github.io.git public
git submodule syncgit add .git commit -m 'initial import'git push origin master
```

## Setup API keys in Github

Head on over to Github, in the Settings page -> Developer settings, you'll find a menu for personal access tokens. Create a token, call it something meaningful (eg: TravisCI), and give the token access to everything in the 'repo' ACL section. Copy paste that somewhere for later.

## Setup TravisCI

Head on over to [Travis](https://travis-ci.org) and login. Give Travis access to your repositories. Next, navigate to the blog repo in the Travis UI, and click on the settings tab. You want to add two variables: GITHUB_USER and GITHUB_API_KEY. The GITHUB_API_KEY is the key you just created, and the GITHUB_USER is your github username.

![null](/images/travis-settings.png)

## Checkpoint - What we have so far

If all went well, at this point we should have Travis automatically triggering every time something is pushed to the blog repo, and pushing changes after running Hugo to the publishing repo.

But why stop here? Let's create a UI to make this easier for non technical folks to update content.

## Setup the OAuth2 Lambda

Before setting up the CMS, we'll want to setup an OAuth2 middle-man to hold our secrets. 

There is a simpler version of this setup that trusts the nice folks at Netlify to do this for us, but being the paranoid person that I am, I'd rather run that myself.

Since I don't expect a lot of volume on this endpoint, unless I start receiving more than a million requests a month I can run this on AWS for free.

### Create an OAuth2 client id/secret in Github

Head on over to Github settings -> developer settings and create a 

If you don't yet have an AWS account, here is where you'd go sign up. Once you've got an AWS account created, we're going to 

## Setup NetlifyCMS - Part 1

Netlify CMS is a content management system build as a React web app, which integrates into Github to allow you to edit content straight from your browser. Setting it up is as easy as dropping an HTML page into your site.

```
cd ~/blog
mkdir -p static/admin/
```
Create the CMS configuration file (~/blog/static/admin/config.yml):

```
publish_mode: editorial_workflow
backend:
  name: github
  repo: marksteele/blog
  base_url: https://www.control-alt-del.org
  auth_endpoint: /oauth/auth
site_id: www.control-alt-del.org
media_folder: "static/images" 
public_folder: "/images" 
collections:
  - name: "blog" # Used in routes, e.g., /admin/collections/blog
    label: "Blog" # Used in the UI
    folder: "content/blog"
    create: true # Allow users to create new documents in this collection
    fields: # The fields for each document, usually in front matter
      - {label: "Layout", name: "layout", widget: "hidden", default: "blog"}
      - {label: "Title", name: "title", widget: "string"}
      - {label: "Publish Date", name: "date", widget: "datetime"}
      - {label: "Featured image", name: "thumbnail", widget: "image"}
      - {label: "Categories", name: "categories", widget: "list", default: ["news"]}
      - {label: "Body", name: "body", widget: "markdown"}
```

Create the CMS webpage (~/blog/static/admin/index.html):

```
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Content Manager</title>

  <!-- Include the styles for the Netlify CMS UI, after your own styles -->
  <link rel="stylesheet" href="https://unpkg.com/netlify-cms@^1.0.0/dist/cms.css" />

</head>
<body>
  <!-- Include the script that builds the page and powers Netlify CMS -->
  <script src="https://unpkg.com/netlify-cms@^1.0.0/dist/cms.js"></script>
<script>
CMS.registerEditorComponent({
  // Internal id of the component
  id: "youtube",
  // Visible label
  label: "Youtube",
  // Fields the user need to fill out when adding an instance of the component
  fields: [{name: 'id', label: 'Youtube Video ID', widget: 'string'}],
  // Pattern to identify a block as being an instance of this component
  pattern: /^{{< youtube (\S+) >}}$/,
  // Function to extract data elements from the regexp match
  fromBlock: function(match) {
    return {
      id: match[1]
    };
  },
  // Function to create a text block from an instance of this component
  toBlock: function(obj) {
    return `{{< youtube ${obj.id} >}}`;
  },
  // Preview output for this component. Can either be a string or a React component
  // (component gives better render performance)
  toPreview: function(obj) {
    return (
      `<img src="https://img.youtube.com/vi/${obj.id}/maxresdefault.jpg" alt="Youtube Video"/>`
    );
  }
});

CMS.registerEditorComponent({
  // Internal id of the component
  id: "figure",
  // Visible label
  label: "Figure",
  // Fields the user need to fill out when adding an instance of the component
  fields: [
    {name: 'src', label: 'Image', widget: 'image'},
    {name: 'link', label: 'Link', widget: 'string'},
    {name: 'target', label: 'Target window', widget: 'string'},
    {name: 'rel', label: 'Rel (optional if link set)', widget: 'string'},
    {name: 'alt', label: 'Image alternate text', widget: 'string'},
    {name: 'title', label: 'Figure title', widget: 'string'},
    {name: 'caption', label: 'Figure caption', widget: 'string'},
    {name: 'className', label: 'HTML class', widget: 'string'},
    {name: 'height', label: 'Image height', widget: 'number', min: 1},
    {name: 'width', label: 'Image width', widget: 'string', min: 1},
    {name: 'attr', label: 'Figure attribution text', widget: 'string'},
    {name: 'attrlink', label: 'Figure attribution link', widget: 'string'}
  ],
  // Pattern to identify a block as being an instance of this component
  pattern: /^{{< figure (.*?) >}}$/,
  // Function to extract data elements from the regexp match
  fromBlock: function(matchIn) {
    var match = matchIn[1].match(/(?:(\b\w+\b)\s*=\s*("[^"]*"|'[^']*'|[^"'<>\s]+)\s*)/g);
    let results = {};
    for (i=0; i < match.length; i++) {
      const pair = match[i].match(/(\b\w+\b)\s*=\s*("[^"]*"|'[^']*'|[^"'<>\s]+)\s*/);
      results[pair[1]] = pair[2].replace(/['"]+/g,'');
    }
    return results;
  },
  // Function to create a text block from an instance of this component
  toBlock: function(obj) {
    let results = '{{< figure ';
    Object.keys(obj).forEach((e) => {
      results += `${e}="${obj[e]}" `;
    });
    results += '>}}';
    return results;
  },
  // Preview output for this component. Can either be a string or a React component
  // (component gives better render performance)
  toPreview: function(obj) {
    return (
      `
      <figure>
        <h3>${obj.title || ''}</h3>
        <a href="${obj.link || ''}" rel="${obj.rel || ''}" target="${obj.target || ''}">
        <img src="${obj.src}" alt="${obj.alt || ''}" width="${obj.width}" height="${obj.height}" />
        </a>
        <figcaption>
          <h4 class="${obj.className}">${obj.caption || ''}</h4>
        </figcaption>
        <small><a href="${obj.attrlink || ''}">${obj.attr || ''}</a></small>
      </figure>
      `
    );
  }
});

CMS.registerEditorComponent({
  // Internal id of the component
  id: "instagram",
  // Visible label
  label: "Instagram",
  // Fields the user need to fill out when adding an instance of the component
  fields: [{name: 'id', label: 'ID', widget: 'string'}],
  // Pattern to identify a block as being an instance of this component
  pattern: /^{{< instagram (\S+) >}}$/,
  // Function to extract data elements from the regexp match
  fromBlock: function(match) {
    return {
      id: match[1]
    };
  },
  // Function to create a text block from an instance of this component
  toBlock: function(obj) {
    return `{{< instagram ${obj.id} >}}`;
  },
  // Preview output for this component. Can either be a string or a React component
  // (component gives better render performance)
  toPreview: function(obj) {
    return (
      `<h4>No preview</h4>
      <a href="https://www.instagram.com/p/{$obj.id}/">Click here</a>`
    );
  }
});

CMS.registerEditorComponent({
  // Internal id of the component
  id: "vimeo",
  // Visible label
  label: "Vimeo",
  // Fields the user need to fill out when adding an instance of the component
  fields: [{name: 'id', label: 'Video ID', widget: 'string'}],
  // Pattern to identify a block as being an instance of this component
  pattern: /^{{< vimeo (\S+) >}}$/,
  // Function to extract data elements from the regexp match
  fromBlock: function(match) {
    return {
      id: match[1]
    };
  },
  // Function to create a text block from an instance of this component
  toBlock: function(obj) {
    return `{{< vimeo ${obj.id} >}}`;
  },
  // Preview output for this component. Can either be a string or a React component
  // (component gives better render performance)
  toPreview: function(obj) {
    return (
      `<h4>No preview</h4>`
    );
  }
});

CMS.registerEditorComponent({
  // Internal id of the component
  id: "gist",
  // Visible label
  label: "Gist",
  // Fields the user need to fill out when adding an instance of the component
  fields: [{name: 'id', label: 'Gist ID', widget: 'string'}],
  // Pattern to identify a block as being an instance of this component
  pattern: /^{{< gist (\S+) >}}$/,
  // Function to extract data elements from the regexp match
  fromBlock: function(match) {
    return {
      id: match[1]
    };
  },
  // Function to create a text block from an instance of this component
  toBlock: function(obj) {
    return `{{< gist ${obj.id} >}}`;
  },
  // Preview output for this component. Can either be a string or a React component
  // (component gives better render performance)
  toPreview: function(obj) {
    return (
      `<h4>No preview</h4>`
    );
  }
});

</script>
</body>
</html>
```
I've created a few custom widgets to add some more Hugo markdown shortcodes. Enjoy!

Add the newly create config/html to the blog source

```
cd ~/blog
git add .
git commit -m 'adding cms'
git push origin master
```


