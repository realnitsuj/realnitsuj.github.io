---
date: '2025-09-18T23:34:55-04:00'
title: 'Gestion de mon CV'
tags:
- pandoc
- cv
- divers
---

Pour la gestion de mon CV, je me suis fixé les critères suivants :

- il doit être simple à lire et à écrire
- je dois pouvoir le convertir en PDF et en HTML simplement

Le tout en centralisant les informations, pour éviter des incohérences entre versions.


J'ai donc séparé mon CV en plusieurs fichiers :

- deux en-têtes (`header-pdf.md` et `header-blog.md`)
- deux fichiers d'informations (`infos.md` et `infos-min.md`)
- le corps du CV (`cv.md`)

Avec trois sorties :

1. `cv.pdf` : mon CV complet
2. `cv-min.pdf` : mon CV public (sans *photo*, *adresse*, *numéro de téléphone*)
3. `$(BLOG)/cv.md` : une version compatible [Goldmark][][^1], sans informations, qui se retrouve [ici](/cv/)

[^1]: J'entends par *compatible* sans syntaxe [LaTeX][], tant dans l'en-tête que le corps.

## Les en-têtes

Ils contiennent toutes les métadonnées nécessaires à chaque sortie. J'y ai également rajouté la première ligne du corps de chaque version, puisqu'elle n'est pas commune.

### `header-blog.md`

```md
---
title: 'Curriculum Vitæ'
menu: main
weight: 10
---

[[Version PDF](cv.pdf)] : *aucune information personnelle n'y est présente, vous pouvez [me contacter](../contact/) pour un CV complet.*

***
```

Ce fichier contient le nécessaire pour une page avec [Hugo][], ainsi qu'une mention quant à la version PDF.

### `header-pdf.md`

```md
---
author-meta: Justin Bossard
title-meta: Curriculum Vitæ de Justin Bossard
date-meta: 20250911
keywords:
- CV
- ingénieur
- informatique
- étudiant
papersize: a4
fontfamily: FiraMono
fontsize: 10pt
geometry: margin=6.5mm
pagestyle: empty
urlcolor: blue
lang: fr-FR
header-includes:
- \renewcommand*\familydefault{\ttdefault}
- \usepackage[fixed]{fontawesome5}
- \usepackage{titlesec}
- \titleformat{\subsection}[block]{\filcenter\normalfont\large\bfseries}{}{0pt}{}
---

# \centering Justin Bossard

```

- Les options `*-meta` et `keywords` contiennent les métadonnées du document
- Les autres options sont pour la mise en forme
- `\renewcommand*\familydefault{\ttdefault}` sert à utiliser la variante `typewriter` de `FiraMono` (sinon on a la version `serif`)


## Les informations

Je ne veux pas rendre publiques certaines données personnelles. Le reste des informations est commun aux deux versions.

Ainsi, éditer à la main `infos.md` et `infos-min.md` si, par exemple, je change de courriel, n'est pas très élégant. J'ai donc opté pour la solution suivante :

- un fichier `infos.yaml`, regroupant toutes mes informations

  ```yaml
  photo:
    path: ./photo.png
    height: 115px
  birthday: 2003-07-08
  license: Permis B, véhiculé
  address:
    street: 15 rue Margemarkdo
    city: 93006 Bagnolet
    country: France
  phone:
    country: US
    number: 3353656565
  mail: jbossard02@gmail.com
  website: realnitsuj.github.io
  github: realnitsuj
  linkedin: in/justin-bossard
  ```

- un script `gen-infos.py` qui lit et traite le premier fichier, pour générer deux fichiers de sorties via un template [Jinja2][]

  ```python
  #!/usr/bin/env python3

  import yaml
  import phonenumbers
  from phonenumbers import PhoneNumberFormat
  import locale
  from datetime import datetime, date
  from jinja2 import Environment, FileSystemLoader

  locale.setlocale(locale.LC_TIME, "fr_FR.UTF-8")

  env = Environment(loader=FileSystemLoader("."))
  template = env.get_template("infos-template.md")

  ## Charger le YAML #############################################################
  with open("infos.yaml") as f:
      data = yaml.safe_load(f)

  ## Traiter les données #########################################################

  bday = data["birthday"]
  today = date.today()
  data["birthday"] = bday.strftime("%-d %B %Y")
  data["age"] = today.year - bday.year - ((today.month, today.day) < (bday.month, bday.day))

  parsed = phonenumbers.parse(str(data["phone"]["number"]), data["phone"]["country"])
  formatted = phonenumbers.format_number(parsed, PhoneNumberFormat.INTERNATIONAL)
  e164 = phonenumbers.format_number(parsed, PhoneNumberFormat.E164)
  data["phone"] = f"[{formatted}](tel:{e164})"

  ## Écrire résultat #############################################################
  with open("infos.md", "w") as f:
      f.write(template.render(**data, mode="full"))

  with open("infos-min.md", "w") as f:
      f.write(template.render(**data, mode="min"))
  ```

  C'est plutôt explicite, on charge les données du YAML, puis on traite l'anniversaire et le numéro de téléphone, avant d'écrire les deux fichiers.

  > `data["birthday"]` est bien traité comme une date par PyYAML, pas besoin de le convertir explicitement.

- le template `infos-template.md`

  ````jinja
  {% if mode == "min" %}
  > *Ceci est une version sans informations trop personnelles, n'hésitez pas à me contacter pour une version complète.*
  {% endif %}
  :::: three-columns
  {% if mode == "full" %}
  ```{=latex}
  \begin{center}
  ```

  ![]({{photo.path}}){height={{photo.height}}}

  ```{=latex}
  \end{center}
  ```
  {% endif %}
  \faIcon{birthday-cake} {{birthday}} ({{age}} ans)

  \faIcon{car} {{license}}
  {% if mode == "full" %}
  \faIcon{map-marker-alt} {{address.street}}\newline
  \hspace*{1.5em} {{address.city}}\newline
  \hspace*{1.5em} {{address.country}}

  \faIcon{phone-alt} {{phone}}
  {% endif %}
  \faIcon{envelope} <{{mail}}>

  \faIcon{globe} [{{website}}](https://{{website}})

  \faIcon{github} [{{github}}](https://github.com/{{github}})

  \faIcon{linkedin} [{{linkedin}}](https://www.linkedin.com/{{linkedin}})

  ::::
  ````


## Le corps du CV

Le reste est dans `cv.md`.

J'utilise [FontAwesome][] avec [LaTeX][] (e.g. `\faIcon{globe}`) pour avoir de jolies icônes en PDF. Je réfléchis à une solution pour les intégrer aussi en HTML avec un filtre Lua et [cet article](https://somethingstrange.com/posts/hugo-with-fontawesome/).

J'utilise [columns.lua](https://github.com/dialoa/columns) pour les doubles colonnes[^2], le reste est écrit en [Markdown][] (avec beaucoup de listes de définitions).

[^2]: Au lieu d'un environnement `multicol`, pour garder une syntaxe la plus simple possible


## `Makefile`

Une fois qu'on a tout ça, on fait un Makefile, avec toutes les recettes qu'il faut, en utilisant [Pandoc][] :

```Makefile
.PHONY: all

PANDOC=pandoc
PYTHON=python

BLOG=~/website/content/cv
OUTPUT=./output

all: $(OUTPUT)/cv.pdf $(OUTPUT)/cv-min.pdf $(BLOG)/index.md

$(OUTPUT)/cv.pdf: cv.md header-pdf.md infos.md
	$(PANDOC) --lua-filter columns.lua header-pdf.md infos.md cv.md -o $@

$(OUTPUT)/cv-min.pdf: cv.md header-pdf.md infos-min.md
	$(PANDOC) --lua-filter columns.lua header-pdf.md infos-min.md cv.md -o $@
	cp $(OUTPUT)/cv-min.pdf $(BLOG)/cv.pdf

$(BLOG)/index.md: cv.md header-blog.md
	$(PANDOC) --standalone --lua-filter columns.lua header-blog.md cv.md -o $@ -t markdown_strict+definition_lists+yaml_metadata_block

infos.md infos-min.md: gen-infos.py infos.yaml
	$(PYTHON) gen-infos.py
```

Pour la conversion en `$(BLOG)/cv.md`, j'utilise le format `markdown_strict`, avec en extension les listes de définitions et les en-têtes YAML, tous deux compatibles avec la convention [Goldmark][].

***

## Conclusion

Et voilà ! Ce système me permet d’avoir un CV centralisé et exportable dans différents formats, sans duplication d’informations.


[Goldmark]: https://github.com/yuin/goldmark/
[Hugo]: https://gohugo.io/
[Jinja2]: https://pypi.org/project/Jinja2/
[FontAwesome]: https://fontawesome.com/
[LaTeX]: https://www.latex-project.org/
[Pandoc]: https://pandoc.org/
[Markdown]: https://daringfireball.net/projects/markdown/
