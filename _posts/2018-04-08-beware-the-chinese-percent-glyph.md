---
layout: post
title: Beware the Chinese percent sign.
tags: ruby rails i18n localization internationalization chinese
---

# Localization problems.
## General Problems.

Localizing software is one of the most fundumental problems business faces when going to international market.
So many moving parts and headaches: How to organize the translation process? Where to find translators? How not to break application with invalid translation files?

On the one of my projects I am working on we are using [Lokalise](https://lokalise.co/). This is friendly service which helps us to organize our work related to translations.

## I18n. Rails. And mysterious percent sign.

Recently we happily uploaded a portion of our .yml files with translations to Chinese traditional and Chinese simplified. And we faced the following problem: `%{value}` was not substituted.
The whole team racked their brains to find out why I18n interpolation worked in all the languages except Chineese. The following worked well:

```ruby
I18n.locale ='en'
=> "en"
I18n.t('quest_conditions.stun_percentage', stun_percentage: 22)
=> "Deal at least 22 % of your team disable/stun"
```

And the next code did not, it was not interpolated:

```ruby
2.4.0 (main):0 > I18n.locale = 'zh_CN'
=> "zh_CN"
2.4.0 (main):0 > I18n.t('quest_conditions.stun_percentage', stun_percentage: 22)
=> "造成全队至少％{stun_percentage}的眩晕"
```

And the problem was that the Chinese people use nearly the same percent sign. If you look closer you will event notice the difference:

```ruby
'%'.ord
=> 37
'％'.ord
=> 65285
```

Always double check you translators work.

Happy Coding!
