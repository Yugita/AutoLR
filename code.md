---
title: 项目最终使用代码汇总与实时更新
---

# 数据集创建

```python
import bibtexparser
import re
import time, json
import concurrent.futures

# preprocessing
with open(r'C:\Users\13607\Downloads\anthology+abstracts.bib\anthology+abstracts.bib', encoding='utf-8') as bibtex_file:
    bibtex = bibtex_file.read()
bibtex = re.sub(r'@proceedings{[^{}]+}', '', bibtex)
bibtex = re.sub(r'(month|pages|address) = .*\n', '', bibtex)
bibtex = re.sub(r'@inproceedings{[^{}]+language = "Chinese",[^{}]+}', '', bibtex)
bibtex_list = re.findall(r'@inproceedings{[^{}]+}', bibtex)

def parse(bibtex_str):
    bib_database = bibtexparser.loads(bibtex_str)
    return bib_database.entries[0]

if __name__ == '__main__':
    t1 = time.perf_counter()
    papers_json = []
    with concurrent.futures.ProcessPoolExecutor() as executor:
        for item in executor.map(parse, bibtex_list):
            if 'abstract' in item:
                papers_json.append(item)

    with open("../result/papers.json", "w") as write_file:
        json.dump(papers_json, write_file, indent=4)

    t2 = time.perf_counter()
    print(f'Finished in {t2-t1} seconds')
```

# arxiv数据集

```python
path = r"D:\pythonProject\result\arxiv-metadata-oai-snapshot.json"
import json
with open(path) as f:
    paper_list = json.loads("[" + f.read().replace("}\n{", "},\n{") + "]")

CL_paper_list = [paper for paper in paper_list if "cs" in paper["categories"]]
# cs.CL, cs.IR. cs.DB等，类别不太好区分

with open(r"D:\pythonProject\result\arxiv.json", "w") as f:
    json.dump(CL_paper_list, f, indent=4)
```

# 名词短语抽取与标注

```python
import spacy
nlp = spacy.load("en_core_web_sm")

import time, json, re
import concurrent.futures

with open(r"D:\pythonProject\result\papers.json", "r", encoding="utf-8") as read_file:
    papers_dict = json.load(read_file)

def chunking(paper):
    doc = nlp(paper['abstract'])
    index = []
    for token in doc:
        list_of_deps = ['agent', 'dobj', 'pobj', 'nsubj', 'nsubjpass', 'csubj', 'csubjpass', 'conj', 'attr', 'ccomp', 'pcomp', 'xcomp', 'appos', 'dative', 'dep']
        if token.pos_ in ['NOUN', 'PROPN', 'VBG'] and token.dep_ in list_of_deps:
            subtree_dict = {child.text: child.i for child in token.subtree if child.i < token.i}
            upper = token.i + 1
            if subtree_dict:
                lower = min(subtree_dict.values())
            else:
                lower = token.i
            index.append((lower, upper))

    if index:
        new_index = []
        for i in range(1, len(index)):
            if index[i][0] >= index[i-1][1]:
                new_index.append(index[i-1])
        new_index.append(index[-1])

        index = new_index

        text = ''
        tag_1 = '<span class="noun">'
        tag_2 = '</span>'

        for i in range(len(doc)):
            if index:
                if i == index[0][0]:
                    text += tag_1
                elif i == index[0][1]:
                    text += tag_2
                    index.pop(0)
            if doc[i].is_punct:
                text += doc[i].text
            else:
                text += ' ' + doc[i].text
        text = text.replace('<span class="noun"> ', ' <span class="noun">').replace(r'- ', '-').replace(r' -', '-')
        return text
    return None

# import pickle

if __name__ == '__main__':
    t1 = time.perf_counter()
    list_of_abstracts = []
    with concurrent.futures.ProcessPoolExecutor() as executor:
        for text in executor.map(chunking, papers_dict):
            list_of_abstracts.append(text)
    t2 = time.perf_counter()
    print(f'Finished in {t2-t1} seconds')

    list_papers_dict = []
    for i in range(len(list_of_abstracts)):
        new_papers_dict = {}
        if "author" in papers_dict[i]:
            new_papers_dict['author'] = papers_dict[i]['author']
        else:
            break
        new_papers_dict['title'] = papers_dict[i]['title']
        new_papers_dict['abstract'] = list_of_abstracts[i]
        new_papers_dict['year'] = papers_dict[i]['year']
        new_papers_dict['url'] = papers_dict[i]['url']
        list_papers_dict.append(new_papers_dict)

    print(len(list_papers_dict))
    with open(r"D:\毕业论文\visualization\data\papers.json", "w") as write_file:
        json.dump(list_papers_dict, write_file, indent=4)
```

# 展示标注

```python
# -*- coding:utf-8 -*-
import json
from flask import Flask, request
from flask import render_template
from flask_paginate import Pagination, get_page_parameter

app = Flask(__name__, static_folder="templates/static")

with open(r"D:\pythonProject\result\papers.json", "r", encoding="utf-8") as read_file:
    papers_dict = json.load(read_file)

@app.route("/")
def index():
    # Set the pagination configuration
    page = request.args.get(get_page_parameter(), type=int, default=1)
    start = (page-1) * 5
    end = start + 5
    papers = papers_dict[start:end]
    pagination = Pagination(page=page, per_page=5, total=len(papers_dict), bs_version=4, search=False)
    return render_template('index.html', papers_dict=papers, pagination=pagination)


if __name__ == "__main__":
    app.run(debug=True)
```
