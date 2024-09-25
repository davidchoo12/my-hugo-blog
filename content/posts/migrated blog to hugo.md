---
title: Migrated blog from Jekyll to Hugo
published: 2024-09-25
---

I had the idea to migrate [my blog](https://davidchoo.netlify.app/) to Hugo since the first time I was introduced to it at a [Friday Hacks session](https://t.me/nushackers/114) by [Jethro Kuan](https://www.jethro.dev) back in 2020. My [previous blog](https://github.com/davidchoo12/my-jekyll-blog) was written using Jekyll, first deployed on 2019, so I wasn't just gonna migrate it after the effort I put in. Jekyll was decent but it felt clunky and old since it required ruby v2 the last time I set it up and I don't have much experience working with ruby. Hugo on the other hand is written in golang which I am now more experienced with.

I know there are other static site generators but Hugo seems to be the most popular (by github stars) and good enough for my use case.

## Theme

Choosing the theme was relatively easy. I simply picked the first one off the [themes list](https://themes.gohugo.io/) after reviewing the [demo site](https://adityatelange.github.io/hugo-PaperMod/). To use a theme, the theme repo needs to be imported as a git submodule. First time for me to use submodules although I knew it existed before, just never needed to use it.

The theme actually has other features like a [/tags](/tags) page and a [search feature](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#search-page). I decided to not use those to prevent cluttering the nav links and I don't have that many posts anyway. If I ever need to add more nav links, I may need to look into the adding a responsive nav menu like [this fork](https://github.com/solamarpreet/PaperMod-Responsive).

## Projects page

The migration is relatively painless except the part where I had to figure out how to render the projects page based on a json file and not markdown. Similar to Jekyll, posts in Hugo is written in markdown which is rendered as `{{ .Content }}` in the theme's layout file. In order to write logic in Hugo, I needed to use [go operators](https://gohugo.io/functions/go-template/) which just would not run in the markdown content and only rendered statically. After some googling and tinkering, I realized I can run shortcode using `{{ _.inline }}` and render the html containing the layout logic with `{{ partial }}`, like so:

`{{</* _.inline */>}}{{ partial "projects.html" . }}{{</* /_.inline */>}}`

So I could just use the go operators normally in the projects.html. This means the rendering path goes from the theme's layout -> my post's markdown -> my projects html layout. That was how I wrote the project.md file initially. Pretty sure that's not how I'm supposed to use Hugo. Hacky, I know but it worked.

Later, I figured the cleaner way would be to write a layout file specifically just for the projects page and point to it in the [markdown's frontmatter](https://gohugo.io/templates/lookup-order/#target-a-template). So that's what I have done. Layout simply copied and modified from the theme's layouts/_default/single.html.

## Syntax highlighting

Getting syntax highlighting in code blocks in Hugo is much more painless than in Jekyll. Hugo has [built in syntax highlighting](https://gohugo.io/content-management/syntax-highlighting/) with [many color schemes](https://xyproto.github.io/splash/docs/longer/all.html). I remember struggling to customize Jekyll's syntax highlighter theme and I had to copy the solarized dark color scheme css block from [a gist](https://gist.github.com/nicolashery/5765395).

To display line numbers in Hugo, I needed to config `noClasses: false` as the gaps between rounded borders would leak out a different background color. It's a minor but noticeable issue. Setting `noClasses: false` required me to [generate the color scheme css](https://gohugo.io/content-management/syntax-highlighting/#generate-syntax-highlighter-css).

## Colors

The original theme color scheme is just [dull black and white](https://adityatelange.github.io/hugo-PaperMod/). I picked the colors from [Tailwind's color palette](https://tailwindcss.com/docs/customizing-colors). I end up with pretty much the same color scheme with my [NUSWhispers Analysis](https://nuswhispers.fyi) project. What can I say, I love the combination of the slate and emerald hues. I believe it looks much fresher than the initial colors.

### Favicon

Favicon is simply a "D" character in Roboto Slab font with emerald-400 color and white outline drawn in [photopea](https://www.photopea.com/). Favicon assets were generated using [realfavicongenerator](https://realfavicongenerator.net/).

## Comment section

My previous blog used [utteranc.es](https://utteranc.es) which was probably the first comment system riding on Github's issues system. Since then, Github has launched a discussions feature which is more suitable for comments and then came [giscus](https://giscus.app/) built upon the discussions feature. I use giscus for this blog. I only needed to do minor changes to handle the light/dark mode color scheme.

## Deployment

Setting up deployment is quite smooth, simply using Github Pages following the [tutorial](https://gohugo.io/hosting-and-deployment/hosting-on-github/). After the domain is configured on the repo settings, I had to trigger another deployment for it to build using the new domain as the base url for the static assets.

## Domain registry

Never thought registering for a domain is such a pain. I kept facing payment issues on Cloudflare even as I tried different cards. Previously, I used Namecheap to buy the nuswhisers.fyi domain. Since then, I've discovered Cloudflare also sells domain names which I thought should be more trustworthy. To be fair, the payment issue likely originates from my bank or Stripe, their payment service provider. Anyway, I used Namecheap again to buy davidchoo.dev and the payment was seamless.

With the domain purchased, I just had to [configure its `A` records](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain).