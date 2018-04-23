One
===========

[One](https://github.com/resugary/hugo-theme-one) is a minimal blog theme for Hugo, which is forked from [onetwothree](https://github.com/schollz/onetwothree). A demo is available [here](https://resugary.github.io/hugo-theme-one).

![Screenshot](https://github.com/resugary/hugo-theme-one/blob/master/images/screenshot.png)

It provides some new features and simplifications from original onetwothree. I tried to keep it minimal with less configuration to write a blog rather than play with a theme instead.

Features:  
- Add archives support for all posts in a single page  
- Homepage displayed with 7 latest posts default  
- Sytax highlighting support with `highlight.js`  
- Google Analytics support  
- Full-text RSS support

## Installation

Clone this repository to your hugo theme directory.

```
git clone https://github.com/resugary/hugo-theme-one.git themes/one
hugo server -t=one
```

## Create New Posts

Posts should generally go under a `content/posts` directory, you may start like this:

```
hugo new posts/hello.md
```

## Create a fixed Page

Fixed pages such as an About page should be present at the root of the `content` directory:

```
hugo new about.md
```

To enable Archives, you should create a new file called `archives.md`:

```
hugo new archives.md

# then add the following line in the front matter
type: "archives"
```

## Configuration

Copy the `config.toml` in the root director of your hugo site. 

```toml
baseURL = "https://example.com"
languageCode = "en-us"
title = "My Hugo Site"

theme = "one"
googleAnalytics = "UA-123-45"

[params]
    navigation = ["archives.md", "about.md"]

```

Feel free to change the strings in this theme.

## License

Licensed under the MIT License.
