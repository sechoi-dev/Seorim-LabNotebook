---
layout: page
title: Research Notebook
---

# Seorim Choi

Undergraduate researcher in the Roberts Lab at the
University of Washington.

## E5 Coral Project

I am working on adapting and evaluating an elastic-net
pipeline that predicts coral gene expression using miRNA,
lncRNA, and DNA methylation data.

My initial focus is extending the 26.7 pipeline to the
other two coral species and comparing results across species.

[Analysis repository](https://github.com/urol-e5/timeseries_molecular)
· [My working fork](https://github.com/sechoi-dev/timeseries_molecular)

## Research entries

{% for post in site.posts %}
- {{ post.date | date: "%B %-d, %Y" }}:
  [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
