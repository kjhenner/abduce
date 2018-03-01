---
layout: post
title:  'citation mining episode 2: sourcing and parsing'
comments: true
---

## Sourcing data

In the last post on this project, I discussed some of my thoughts around
analyzing the semantic content of citations by applying a latent semantic model
to collections of article paragraphs that include a common reference. Here,
I'll dig in to the actual process of getting started on an implementation.

The code for this project can be found on
[GitHub](https://github.com/kjhenner/pubmed_data), though it may change from
the examples I've given here.

In my initial stab at implementing this model, I wanted to use papers in
cultural anthropology, or at least in the social sciences. I figured my
knowledge in this area would help me actually assess whether the model was able
to come up with useful results.

Unfortunately, the only databases of freely available academic papers I was
able to find in this area consisted of articles formatted as PDFs. For some, I
was able to extract actual text embedded in the articles, but formatting was
still a nightmare due to things like page-breaks, numbering, and diagrams that
disrupted the flow of text. For others, I tried optical character recognition,
which added its own set of character recognition issues on top of the same
formatting problems.

Even when I was able to get reasonably cohesive texts, I ended up with a
monstrous set of regular expressions to try to parse the citation data and
associate it back with inline citations. Though I was able to parse
bibliographies with something like 80% accuracy, A profusion of subtle
differences and inconsistencies in how citations were formatted led to quickly
diminishing returns in attempts to address that last 20%.

After setting the project aside for a while, I opted to abandon the idea of
using social sciences work and look for structured data in whatever field I
could find it. Eventually, I came happened upon the Open Access Subset at
[PubMed Central](https://www.ncbi.nlm.nih.gov/pmc/), a free archive of
biomedical and life sciences journal articles hosted by the National Institutes
of Health.

Not only did this archive include over a million articles, but the fact that
they were already structured as XML turned the seemingly intractable parsing
issue into a merely hairy one.

PMC's FTP server provides bulk access to the articles included in this list,
which allowed me to easily download the full list to disk.

## Setting up a development environment

After downloading and extracting the archive, my first step was to desing a
parser that would be up to the task of running through all million or-so
articles in a somewhat reasonable amount of time. I decided to go with the
Python [lxml](http://lxml.de/intro.html) module. It wraps the libxml2 C XML
processing library with conveneint Python bindings. Python is easy and C is
fast! What's not to like? Using Python here also had the advantage of
maintaining language consistency with the Python platforms like NLTK and NumPy
I'll want to use for later steps.

Installing Python modules with C library dependencies on MacOS always seems to
be a huge hassle. Rather than set up a development environment on MacOS, I
opted to use Docker to build my environment in a container. This also has the
advantage of making the environment easily portable, meaning that once I have
it set up locally, I can much more easily share it with collaborators or run
it in the cloud.

The installation process seemed to be easiest on Debian, so I created a
Dockerfile to define a Docker image with all my pre-requisites set up. From
past experience, I also knew that there were a few other potentially gnarly I
might want for analysis, so I went ahead and took care of these dependencies
up-front. Finally, I installed some Python tools for interacting with
PostgreSQL and Neo4j databases, which I'll discuss more when I get to the
database loading step.

Though it took some iteration and experimentation to get all of the right
dependencies installed, the resulting Dockerfile is very simple, consisting
only of `RUN` calls to `apt-get` and `pip`.

```
FROM debian:latest
MAINTAINER Kevin Henner <kjhenner@gmail.com>

RUN apt-get -qq update && apt-get -qq -y install python python-pip python-dev build-essential gfortran libatlas-base-dev libxml2-dev libxslt-dev lib32z1-dev curl
RUN pip install --upgrade pip
RUN pip install python-sql numpy scipy matplotlib scikit-learn nltk neo4j-driver lxml
```

From this Dockerfile, I built a `pubmed_env` image.

    docker build -t pubmed_env .

With this image built, I could use the `docker run` command to create a new
container instance from the image and run a specified command in that instance.
Typically, Docker is used to provide the minimal environment necessary to run a
single service. This allows services to be isolated without the overhead of a
full kernel and supporting services needed to run a full VM. Because I'm using
my image as a development environment, however, my approach is a little
different. Rather than using Docker to run an actual service, I wanted to run a
bash command-line session.

    docker run -it pubmed_env bash

This way, I could interact more freely with the environment, either running
Python scripts directly or opening an interactive Python session to experiment
and test code. For this to work, I needed to use the `-i` and `-t` flags to run
interactively and allocate a pseudo tty.

To make the development process simpler, I also pass in ``--rm -v `pwd`:/app``
to mount the directory where the command is run (my project directory) to the
`/app` directory on the container, and use `-w /app` to set the current working
directory on the container to that `/app` directory.

```
docker run -it \
  --rm -v `pwd`:/app \
  -w /app \
  pubmed_env bash
```

Mounting this directory lets me edit code in my project directory on my laptop
and seamlessly test and execute that code on the container. This also allows
code running on the container to read and write from my the project directory's
`data` and `output` subdirectories.

## The parser

With this environment set up, I began work on my parser by creating a simple
directory structure Python could use to organize my code into a module.

```
pmgraph/
├── __init__.py
├── parser.py
```

This structure allows me to import the module from a Python session running
in project directory (the container's `app` directory) with:

```python
import pmgraph.parser as parser
```

Functions defined in the `parser.py` file can then be run as:

```python
parser.function()
```

Next, I dove into the design for the parser itself. The input would be the name
of a base directory containing all the articles I wanted to parse, separated
out into distinct subdirectories for each journal. It looks like this:

```
data
├── Obes_Surg
│   ├── PMC2226018.nxml
│   └── ...
├── Oncogene
│   ├── PMC2628450.nxml
│   └── ...
└── ...
```

The output would be a set of CSVs suitable to be inserted into a relational
or graph database, where they could then be efficiently queried to provide
data in the shape needed for my analysis.

```
output/
├── articles.csv
├── cites.csv
├── contains.csv
├── contribs.csv
├── contributed.csv
├── ext_articles.csv
├── ext_contribs.csv
├── ext_journals.csv
├── journals.csv
├── paragraphs.csv
└── published.csv
```

Each CSV file either contains actual data extracted from the articles or
a join table to describe relationships between these data. For example, the
`articles.csv` file contains data about individual articles, while the
`cites.csv` file contains data about the relaionships between articles and
the paragraphs that cite them.

The files prepended by `ext` indicate articles, contributors, and journals
included in citations originating in articles being parsed, but not necessarily
appearing in the database themselves.

There are two primary challenges to be considered when parsing these files.
First, because the corpus is large, I must be conscious of runtime efficiency.

Second, because the articles are drawn from a large number of journals, I must
address the inevitable variations in how the articles are structured and which
data are actually available. The archive also includes several different kinds
of article, such as reviews, correspondence, and pre-publication manuscripts,
each of which has its own structural pecularities.

Addressing runtime efficiency is the less difficult problem. Because each
article can be parsed independently of the others, the time complexity is O(N).
Even with a large number of articles, this linear growth means that the parsing
step is unlikely to be the critical path in the pipeline as a whole. (The
singular value decomposition step of LSA has time complexity of O(min{mn^2,
m^2n}), making it far more concerning! I'll cross that bridge when I come to
it.) Though lxml is [very optimizable](http://lxml.de/2.1/performance.html), I
decided it would be better to avoid spending too much time here until I had
a specific need.

Still, I wanted to take some steps to avoid redundant traversals of the XML
document tree. To do this, I took a depth-first approach, extracting all data
I needed from each element subtree before moving to a sibling.

The overall structure of the `parse_file` function looks like this:

```python
parse_file(path):

    # Intiialize data dict with headers
    data = {
        'refs': {},
        'paragraphs': [],
        'contribs': [],
        'ext_contribs': [],
        'journals': [],
        'articles': [],
        'ext_journals': [],
        'ext_articles': []
    }

    #Parse the XML and get the tree and root
    tree = etree.parse(path)
    root = tree.getroot()

    # /article
    article = root.xpath('/article')[0]

    # /article/back
    back = if_xpath(article, 'back')
    if back:
        for ref in back.xpath('ref-list/ref'):
            [...]

    # /article/front
    front = article.xpath('front')[0]

    # /article/front/journal-meta
    journal_meta = front.xpath('journal-meta')[0]
    [...]

    # /article/front/article-meta
    article_meta = front.xpath('article-meta')[0]
    [...]

    data['articles'].append({
      [...]
    })

    # /article/front/article-meta/contrib-group
    contrib_group = article_meta.xpath('contrib-group')[0]
    contribs = contrib_group.xpath('contrib')
    for contrib in contribs:
        [...]

    # /articls/body
    body = article.xpath('body')[0]
    for paragraph in body.xpath('*//p')
        [...]

    return data
```

This seems to work well enough. I was able to parse several thousand articles
in a minute or two, so at worst, I figured leaving the process running for a
few hours would be sufficient to parse the whole archive.

This brought me to the somewhat gnarlier problem: format inconsistencies among
journal articles. While I still haven't fully resolved this issue, as some of
the fields I need to use as unique IDs to identify documents simply don't exist
in some of the articles! The pragmatic solution may be to simply exclude this
data for the time being and return to it if and when my results are compelling
enough to justify the effort.

For the non-essential fields, I simply implemented 'safe' functions that would
attempt one or more paths to a data field before giving in and returning an
empty string.
