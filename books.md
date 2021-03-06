---
title: Books
---

## Read a technical book each quarter (or even month)

| Year       | Book                                           | Done |
|------------|------------------------------------------------|------|
| 2020 (3/4) | Pragmatic Programmer                           | Done |
| 2020 (4/4) | Refactoring                                    | Done |
| 2020 (4/4) | The Deadline: A Novel About Project Management | Done |
| 2021 Jan   | The mythical Man-Month                         | Done |
| 2021 (1/4) | Clean architecture                             | WIP  |

Pool:

* Clean code (R. Martin)
* Peopleware
* Pearls of Functional Algorithm Design
* Reverse engineering for beginners

Advices from public speaking teacher (russian):

* Никита Непряхин «Убеждай и побеждай», «Гни свою линию». Его МК и вебинары
* Ника Косенкова «Метоника»
* Радислав Гандапас «Камасутра для оратора»
* Экзюпери «Маленький принц», «Планета людей»
* Луис Ривера «Матадор», «Матадор поневоле», «Змеелов»
* Ричард Бах «Чайка по имени Джонатан Ливингстон»
* Борис Пастернак, любые стихотворения читать вслух
* Так называемое зло


## Reviews

I'm not a critic. Just want to write some conclusion to put it in my head.
So there are some small reviews for books:

{% for post in site.categories['books'] %}
<article class="archive-item">
  <h4><a href="{{ site.baseurl }}{{ post.url }}">{{post.title}}</a></h4>
</article>
{% endfor %}
