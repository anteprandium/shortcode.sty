# Hugo shortcodes in LaTeX's Lua Markdown

This package implements an extension to LaTeX's Lua Markdown parser (caveat: by someone who has never programmed in Lua) to process Hugo shortcodes. We define the `shortcode.lua` markdown  extension, but you still need to provide the corresponding LaTex definition for each shortcode. 

Let me state it again: You need to `\usepackage{shortcode}`, and then **you need to define your own macros for each shortcode**.  

See test.tex for examples on how to define shortcodes or read on for an explanation.

## How it works

The idea behind this package is to code a markdown extension so that shortcodes

```
{{< shortcodename key1=value1 key2=value2 ... >}}
```

gets translated into LaTeX as

```latex
\shortcode{shortcodename}[key1=value1,key2=value2,...]
```

This command actually invokes the following command, which you have to define:

```
\shortcodeshortcodename[key1=value1,key2=value2,...]
```



Likewise,

```
{{< /shortcodename >}}
```

gets translated to 

```latex
\closeshortcode{shortcodename}
```

which in turn calls

```latex
\closeshortcodeshortcodename
```

which, again, you have to define.

Let's see a few examples.

## How to define your own LaTeX shortcodes

### Simplest case

The simplest case is a shortcode with no options, like

```
{{< hr >}}
```

This is translated by the markdown extension to 

````
\shortcode{hr}
````

which in turn, calls the following macro that you have to define:

```
\shortcodehr
```

which you could, for instance, set like 

``` 
% simplest shortcode
\newcommand\shortcodehr{\par\medskip\hrule\par\medskip}
```

### Not so simple: one argument.

Suppose now that you want to use a shortcode like this: `{{ icon github }}` to insert an icon for GitHub. This shortcode gets translated by the markdown extension to

```
\shortocode{icon}[github]
```

which in turn calls

````
\shortcodeicon[github]
````

This would be easy to define:

```
% shortcode with one argument, no processing. 
\RequirePackage{fontawesome5}
\newcommand\shortcodeicon[1][]{\faIcon{#1}}
```



### Next step: named arguments, defaults, 

If you want a more nuanced approach, you could use a key-value approach, with a variable name of arguments. A shortcode like this, for instance

```
{{< faicon name=building >}} or {{<faicon syle=regular name=building >}}
```

(In any order). This gets translated by the markdown extension to 

```latex
\shortcode{faicon}[name=building] or \shortcode{faicon}[name=building,style=regular]
```

and then, when these are expanded in LaTeX, to

````
\shortcodefaicon[name=building] or \shortcodefaicon[name=building,style=regular]
````

but we still need to provide the LaTeX definitions. Here you can leverage a key-value package; I'm partial to `pgfkeys`. So you can define something like

```latex
\RequirePackage{fontawesome5}
\RequirePackage{pgfkeys}
\pgfkeys{/faicon/.is family, /faicon/.cd, 
    name/.default={},
    name/.estore in=\shtcIconName,
    style/.estore in=\shtcIconStyle,
    style/.default=solid,
    name, style % execute defaults!
}
\newcommand\shortcodefaicon[1][]{\pgfkeys{/faicon/.cd,#1}%
	\faIcon[\shtcIconStyle]{\shtcIconName}}
```



### Simple environments

Some shortcodes have a closing shortcode:

```
{{< abstract >}}
lorem ipsum...
{{< /abstract >}}
```

This gets translated to 

```latex
\shortcode{abstract}
lorem ipsum...
\closeshortcode{abstract}
```

which gets expanded to

```latex
\shortcodeabstract
lorem ipsum...
\closeshortcodeabstract
```

So a sensible definition would be 

```latex
\newcommand{\shortcodeabstract}{\begin{abstract}}
\newcommand{\closeshortcodeabstract}{\end{abstract}}
```

### From environments to commands

Occasionally, a pair of opening/closing shortodes have to be translated to a single LaTeX command:

````
{{< alert warning >}}
Lorem ipsum ...
{{< /alert >}}
````

You need some TeX treachery to transform the above to `\alertwarning{Lorem ipsum...}`. For starters, the above gets translated to 

```latex
\shortcode{alert}[warning]
Lorem ipsum...
\closeshortode{alert}
```

which, when expanded, translates to 

```latex
\shortcodealert[warning]
Lorem ipsum...
\closeshortodealert
```

So a good LaTeX definition would be:

```latex
\RequirePackage{alertmessage}
\def\shortcodealert[#1]#2\closeshortcode#3{\expandafter\csname alert#1\endcsname{#2}}
```

For simpler shortcodes, without optional arguments, a similar trick works: say you want to use a `sidenote` shortcode:

````markdown
In principle {{< sidenote >}}This is of course, the content of a sidenote{{< /sidenote >}} you should be able to manipulate the contents of...
````

Here, the content of the sidenote shortcode should be given as argument to the `\sidenote`command, To do that, you can use the following definition:

```latex
\def\shortcodesidenote#1\closeshortcode#2{\sidenote{#1}}
```



### Managing quoted strings

Say you want to use the `theorem` shortcode above, 

``` 
{{< theorem name="Chinese Remainder Theorem" >}} The groups $\mathbb{Z}/(ab)$ and $\mathbb{Z}/(a)\times \mathbb{Z}/(b)$ are isomorphic if and only if $\gcd(a,b)=1$.
{{< /theorem >}}
```

To make thing worse, suppose you also would like to use the following form, both in the same document.

``` 
{{< theorem "Chinese Remainder Theorem" >}} The groups $\mathbb{Z}/(ab)$ and $\mathbb{Z}/(a)\times \mathbb{Z}/(b)$ are isomorphic if and only if $\gcd(a,b)=1$.
{{< /theorem >}}
```

In this case, managing the quoted strings and the optional `name=` part becomes cumbersome. But here is a solution that uses `pgfkeys`.

```latex
\RequirePackage{pgfkeys}
\makeatletter
\def\auxshortcodemaybeunquote#1\pgfeov{\@ifnextchar"%
    {\auxshortcodemaybeunquoteQuote}%
    {\auxshortcodemaybeunquoteNotQuote}#1\pgfeov}
\def\auxshortcodemaybeunquoteQuote"#1"#2\pgfeov{\edef\optionalStatementName{#1}}
\def\auxshortcodemaybeunquoteNotQuote#1\pgfeov{\edef\optionalStatementName{#1}}
\pgfkeyslet{/theorem/name/.@cmd}{\auxshortcodemaybeunquote}
\pgfkeys{% this bit actually deals with an unnamed quoted string
    /theorem/.is family, /theorem/.cd,
    /handlers/first char syntax=true,
    /handlers/first char syntax/the character "/.initial=\auxshortcodeinitialquote,
}
\def\auxshortcodeinitialquote#1{\auxshortcodemaybeunquoteQuote#1\pgfeov}
\makeatother
%% Environment shortcode
\newcommand\shortcodetheorem[1][]{%
    \pgfkeys{/theorem/.cd,#1}% see above
    \begin{theorem}[\optionalStatementName]}
\newcommand\closeshortcodetheorem{\end{theorem}}

```



The tricky part is

```latex
\pgfkeyslet{/theorem/name/.@cmd}{\auxshortcodemaybeunquote}
```

We then have to define a macro `\auxshortcodemaybeunquote` that checks if the following character is a quote, to cover both use cases: `{{<theorem name="Lagrange" >}}` and `{{<theorem name=Lagrange >}}`.

