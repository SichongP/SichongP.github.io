---
title: Write scientific paper in markdown
layout: post
author: Sichong Peng
date: '2019-08-07'
excerpt_separator: <!--more-->
math: true
---

If you have ever written a scientific paper in MS Word (or any other word processing software) then you're probably experienced frustrations like me: Word constantly crashing when loaded with too many high-resolution figures, citations getting messed up when working with others that use different citation managers, or hard to keep track of hundreds of versions of the draft, etc.
<!--more-->
## Introduction

Of course, Latex is always an excellent solution to all those problems. It separates content and formatting, allowing you to focus on content and leave the rest to those with expertise at it. A Tex file is also a pure text format so it's easy to manage with a version control tool like git. However, Latex does have a rather steep learning curve. And a Tex file is not the most intuitive format to look at. For example, look at the below Tex format: ![](/img/Tex_example.png)

Another problem with writing in Latex is, while it's not easy to learn Latex, it's even harder to get all your collaborators on board, which is usually the case with scientific papers.

That led me to think: markdown is a fantastic format that is both intuitive and powerful. Why not write in markdown? After a short search, I'm happy to report that this is indeed possible!

## What does writing a scientific paper require?

Content aside, to write a scientific paper efficiently, one needs at least the following elements:

1. Easy way to insert and manage citations.
2. Easy way to insert and manage high resolution figures
3. Easy way to format tables
4. Can be converted to other formats (pdf, html, etc)
5. Allows collaborative writing

Atom is an excellent text editor that with packages can do almost all of the above!

## Required software and packages

- [Atom](https://atom.io/)
  - [markdown-preview-enhanced](https://atom.io/packages/markdown-preview-enhanced)
  - [zotero-citations](https://atom.io/packages/zotero-citations)

- [Pandoc](https://pandoc.org/)

- [A Tex distribution for your OS](https://www.latex-project.org/get/) (I used Tex Live for my linux system)

- [Zotero citation manager](https://www.zotero.org/)
  - [Bbibtex Add-on](https://retorque.re/zotero-better-bibtex/installation/)

Once you have all required packages installed, it's time to start writing!

## Write a paper

Writing a paper in markdown is as easy as it sounds: use `#` for titles and subtitles, use `|` to format a table, among other things.

### Write math equations

You can use Latex syntax to write a math equation, like so:

```
$f_A^2+2f_Af_B+f_B^2=1$
```

$f_A^2+2f_Af_B+f_B^2=1$

### Import figures:

You can insert a figure in typical markdown way:

```
![alt text](figure.png)
```

This would import the figure at current location. You can even further configure the figure like so:

```
![alt tex](figure.png){#id .class width=300px height=200px}
```

Note: Because `markdown preview enhanced` by default uses a different render engine to render the preview, you will not see change to figure size in the preview but it will be rendered when converting to another output. You can also change its default markdown render engine to pandoc to see size change in preview.

### Insert citations:

In order to insert a citation, you need to have zotero open. With zotero open, you can call up zotero citation picker by either pressing `ctrl` + `alt` + `P` or pressing `ctrl` + `shift` + `P` to call up the command palate and search for "zotero citations: pick". In the citation picker window like below: ![](/img/zotero_picker.png)

type in the reference you wish to insert and hit `enter`. A string of citation key should appear in your document:

```
@bryoisEvaluationChromatinAccessibility2018
```

Surround this reference key with `[]` and you're good to go! Once you export this document to a pdf, word, or html format, the reference will be automatically appended to the end of your document and the ref key will be replaced with corresponding references, like so: ![](/img/citation_eg.png)

To add multiple citations, use `;` to separate ref keys, like so:

```
[@bintuSuperresolutionChromatinTracing2018; @bryoisEvaluationChromatinAccessibility2018]
```

Lastly, in order for pandoc to render citations correctly, you must provide a `bib` file that contains all your citations and their corresponding ref keys.

To do this, head on to zotero and click `File` -> `Export library`, in the format section, choose `Better Bibtex`. If you want the exported library up to date with your zotero library, check `Keep updated`. Once you have the exported library, add yaml frontmatter to the beginning of your document, like so:

```
---
bibliography: My Library.bib
---
```

### Export to other formats

With pandoc, you can easily convert the markdown format to other text formats like pdf, word, and tex. To do this, first add `output: word_document` to your document's frontmatter, like so:

```
---
bibliography: My Library.bib
output: word_document
---
```

and then you can right clock on the preview panel on the right to export the document through pandoc: ![](/img/pandoc_right_click.png) This will output your document to a Word document. To output to PDF, simple change the value of `output` in frontmatter to `pdf_document`.

### Conclusion

Markdown is an incredibly intuitive text format and yet still versatile and powerful enough for productive writing. It has recently surged to new favorite of many among developers, scientists, and journalists. It will certainly keep raising in popularity due to its friendliness and expandability.

Writing a scientific document in markdown has freed me of many frustrations caused by Word and with incredible `pandoc`, I never have to wrestle with any of my collaborators over which text editor we are to use. It has been a life saver!
