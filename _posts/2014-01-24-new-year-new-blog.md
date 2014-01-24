---
layout: post
title: New Year, New Blog
categories:
- basic
tags: []
status: publish
type: post
published: true
---
Based on Stefan Baumgartners presentation on [Jekyll](http://jekyllrb.com) at the last [Technologieplauscherl](http://www.technologieplauscherl.at) meetup, I decided to migrate my blog to [GitHub Pages](https://help.github.com/articles/what-are-github-pages). GitHub pages are public webpages freely hosted on GitHub. One way to generate these GitHub pages is by using Jekyll. Jekyll is a so-called _static website generator_. In the case of Jekyll, the web generators code is in Ruby. As the static web site generator category seems to be some kind of hipster thing right now, there are generators in/for various programming languages: Java, Groovy, Clojure etc. The main advantage of Jekyll is its GitHub support.

The overall workflow when using the generator is:

- generate a new project via `jekyll new something`
- write a post by placing a *.md or *.html file in the `_posts` directory
- check the post by locally running `jekyll serve` and looking up the page at `localhost:4000`
- `git push origin master` to push the changes to GitHub

After pushing the changes to GitHub, the GitHub pages generator will recognize the Jekyll project and generate the HTML files from the given sources.

## Modifying Jekyll Generated HTML

As my blog is an original Wordpress blog, I needed to migrate all the existing posts and pages. There is a tool called `jekyll-import` that takes the Wordpress XML and generates separate HTML pages to be used by Jekyll. The resulting HTML files need to be slightly adapted to the needs of Jekyll. For example, all the source code examples where surrounded by a `[source language="..."]` block. In Jekyll, the `highlight ...` tag is used to surround and therefore format later on source code examples. Thus I needed a little script to convert the Jekyll generated HTML pages. 

I decided to write the script in [Clojure](http://www.clojure.org). As I am currently using Clojure in a project, I thought it would be a good fit for doing a little bit more "hands-on" training.

Here is the script that implemented various HTML transformations for my Wordpress posts:

<div class="highlightjs">
	<pre class="clojure">
(ns clj-wp-import.core
  (:require [clojure.java.io :as jio]
            [clojure.string  :as str]))
 
(defn post-files
  [f]
  (file-seq (jio/file f)))
 
(defn remove-pre-code 
  [txt]
  (str/replace txt #&quot;(?s)(?i)&lt;pre&gt;(.+?)&lt;/pre&gt;&quot; &quot;{&#37; highlight groovy &#37;} $1 {&#37; endhighlight &#37;}&quot;))
 
(defn remove-code-language
  ([tag attr txt] (remove-code-language (vector &quot;groovy&quot;) tag attr txt))
  ([lang-coll tag attr txt]
     (reduce #(str/replace &#37;1
                           (re-pattern (str  &quot;(?s)(?i)\[&quot; tag &quot; &quot; attr &quot;=&quot;&quot; &#37;2 &quot;&quot;\](.+?)\[/&quot; tag &quot;\]&quot;))
                           (str &quot;{&#37; highlight &quot; &#37;2 &quot; &#37;} $1 {&#37; endhighlight &#37;}&quot;)) txt lang-coll)))
 
(defn format-link-section [txt]
  (str/replace txt
               (re-pattern &quot;(\[\d{1,2}\] &lt;a.*)&quot;)
               (str &quot;&lt;div&gt;$1&lt;/div&gt;&quot;)))
 
(defn format-first-link-of-link-section [txt]
  (str/replace txt
               (re-pattern &quot;(&lt;div&gt;\[0\] &lt;a.*)&quot;)
               (str &quot;&lt;br&gt;&lt;br&gt;$1&quot;)))
 
(defn remove-double-curly-braces [txt]
  (str/replace (str/replace txt &quot;{{&quot; &quot;&quot;) &quot;}}&quot; &quot;&quot;))
 
(defn replace-html-entites [txt]
  (let [entities [[&quot;&amp;quot;&quot; &quot;&quot;&quot;] [&quot;&amp;lt;&quot; &quot;&lt;&quot;] [&quot;&amp;gt;&quot; &quot;&gt;&quot;]]]
    (reduce #(str/replace &#37;1 (nth &#37;2 0) (nth &#37;2 1)) txt entities)))
 
(defn replace-html-entites-in-highlight [txt]
  (reduce #(str/replace &#37;1 (first &#37;2) (replace-html-entites (first &#37;2))) txt (re-seq (re-pattern &quot;(?s)\{&#37; highlight (.*?) &#37;\} (.*?) \{&#37; endhighlight &#37;\}&quot;) txt)))
 
(defn process-post-file [file]
  (let [s (slurp file)]
    (-&gt;&gt; (remove-pre-code s)
 
         (remove-code-language [&quot;groovy&quot;, &quot;xml&quot;, &quot;java&quot;, &quot;scala&quot;, &quot;kotlin&quot;] &quot;code&quot; &quot;language&quot;)
         (remove-code-language [&quot;groovy&quot;, &quot;xml&quot;, &quot;java&quot;, &quot;scala&quot;, &quot;kotlin&quot;] &quot;code&quot; &quot;lang&quot;)
  
         (remove-code-language [&quot;groovy&quot;, &quot;xml&quot;, &quot;java&quot;, &quot;scala&quot;, &quot;kotlin&quot;] &quot;sourcecode&quot; &quot;language&quot;)
         (remove-code-language [&quot;groovy&quot;, &quot;xml&quot;, &quot;java&quot;, &quot;scala&quot;, &quot;kotlin&quot;] &quot;sourcecode&quot; &quot;lang&quot;)
 
         (format-link-section)
         (format-first-link-of-link-section)
 
         (remove-double-curly-braces)
 
         (replace-html-entites-in-highlight)
         )))
		</pre>
</div>

As it turned out after the migration and deployment, the script is not quite complete but the majority of my blog posts have been created successfully. During the migration, I moved all comments to [Disqus](http://www.disqus.com), which allows for easy embedding of comment sections for various types of blogs.

Overall, I am quite happy with the status quo, although I am going to replace [pygments](http://pygments.org) (the default Jekyll highlighter) with [highlight.js](http://highlightjs.org). In the next couple of days, I'll do some manual cleanup for all the blog posts, so please be patient if there are little formatting errors every now and then.