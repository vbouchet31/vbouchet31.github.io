---
layout: post
title:  "Integrate Google Keep with Kalliope"
date:   2019-01-16 10:13:54 +0100
categories: [kalliope]
---
To integrate Kalliope with Google Keep, I used [gkeep-neuron](https://github.com/corus87/gkeep-neuron) from [corus87](https://github.com/corus87).

To install the community module:
```
kalliope install --git-url https://github.com/corus87/gkeep-neuron
```

Follow the instructions from the [github page](https://github.com/corus87/gkeep-neuron).

The given synapse example must be created in `brains/`. The given template example must be created in `templates/`. If you copy/paste the example neuron, the template name must be `gkeep.j2`.

### Example
```
- name: "add-shopping-list"
    signals:
      - order: "Ajoute du \{\{items\}\} à la liste des courses"
      - order: "Ajoute des {{items}} à la liste des courses"
      - order: "Ajoute de l'{{items}} à la liste des courses"
    neurons:
      - gkeep:
          login: "<email>"
          password: "<password>"
          list: "shopping"
          items: "{{ items }}"
          option: "add"
          split_word: "et"
          pin_list: True
          file_template: "templates/gkeep.j2"
```
Source: [Kalliope Gkeep neuron](https://github.com/corus87/gkeep-neuron)