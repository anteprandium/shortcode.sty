# Hugo shortcodes in LaTeX's Lua Markdown

Lualatex has a Markdown parser package (called, you might have never guessed, `markdown`). So you can write Markdown, and have it beautifully rendered by LaTeX. Hugo is a static site builder. It works on Markdown files (among others) but it extends the Markdown syntax using shortcodes.

Shortcodes have this aspect: `{{< shortcodename param1 param2 ... >}}`, which is transformed by Hugo to some HTML. 

For a more concrete example, suppose you define a `theorem` shortcode in this way: 

```html
<p><strong>Theorem {{ with .Get "name" }} ({{ . }}){{ end }}</strong>
    <em>{{ .Inner | markdownify }}</em></p>

```

Then you can use this shortcode like so:

```markdown
{{< theorem name="Chinese Remainder Theorem" >}} The groups $\mathbb{Z}/(ab)$ and $\mathbb{Z}/(a)\times \mathbb{Z}/(b)$ are isomorphic if and only if $\gcd(a,b)=1$.
{{< /theorem >}}
```

But if you use shortcodes, you cannot use the Markdown file as a LaTeX's `markdown`  input, since shortcodes aren't processed by `markdown`. Well, not anymore.

---

This package implements an extension to LaTeX's Lua Markdown parser (caveat: by someone who has never programmed in Lua) to process shortcodes. Mind you: this is mostly proof of concept for my personal use. You need to `\usepackage{shortcode}`, and then you need to define macros for each shortcode. Say you want to use the `theorem` shortcode above. Then you would define something like:

```tex
\newcommand\shortcodetheorem[1][]{%
	\begin{theorem}[#1]%
}
\newcommand\closeshortcodetheorem{\end{theorem}}
```

You do not need to define `\closeshortcode...` for simple (non terminated) shortcodes.

For shortcodes whose content should be the argument to another command (as opposed to an environment), there is some extra trickery involved. Say you want to use a `sidenote` shortcode:

````markdown
In principle {{< sidenote >}}This is of course, the content of a sidenote{{< /sidenote >}} you should be able to manipulate the contents of...
````

Here, the content of the sidenote shortcode should be given as argument to the `\sidenote`command, To do that, you can use the following definition:

```latex
\def\shortcodesidenote#1\closeshortcode#2{\sidenote{#1}}
```

