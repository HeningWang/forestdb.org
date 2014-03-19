---
layout: default
title: Forest
all-model-statuses: [code, link, stub]
all-model-categories: [Concept Learning, Nested Inference, Inverse Dynamics, Machine Learning, Undirected Constraints, Nonparametric Models, Miscellaneous]
---

<div class="page-header">
  <h1>Models<br /></h1>
</div>

{% for category in page.all-model-categories %}

<div class="list-group">

  <div class="list-group-item" style="background-color: #F9F9F9">
    {{ category }}
  </div>

    {% for status in page.all-model-statuses %}
      {% for p in site.pages %}        
        {% if p.layout == 'model' %}
          {% if p.model-status == status %}
           {% if p.model-category == category %}
              <div class="list-group-item">
                  {% if p.model-status == 'stub' %}
                    <span class="label label-default pull-right">Stub</span> {{ p.title }}
                  {% else %}
                    {% if p.model-status == 'code' %}
                      <span class="label label-success pull-right">Code</span>            
                    {% elsif p.model-status == 'link' %}
                      <span class="label label-primary pull-right">Link</span>
                    {% else %}
                    {% endif %}
                    <a href="{{ p.url }}">{{ p.title }}</a>          
                  {% endif %}
                  <span class="subtitle">{{ p.model-description }}</span>
              </div>
            {% endif %}
          {% endif %}
        {% endif %}
      {% endfor %}
    {% endfor %}

</div>

{% endfor %}

<div class="btn-toolbar">
<a class="pull-right btn btn-success" id="github-add-link" href="https://github.com/stuhlmueller/forestdb.org/new/gh-pages/models">Add Model</a>
</div>
